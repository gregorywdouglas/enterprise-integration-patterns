# Request-Reply with Compensation

## Problem

Synchronous request-reply is the dominant integration pattern for user-facing operations: a client calls an API, waits for a response, and acts on the result. This model works reliably when all participating systems are available, consistent, and respond within acceptable latency bounds. In real enterprise environments, those conditions are frequently violated: downstream services time out, partial writes succeed before a failure occurs, and the caller receives an ambiguous error state.

Without a compensation strategy, failed synchronous operations leave the system in an inconsistent state. A payment that was charged but not applied to the order. A reservation that was created but the inventory was not decremented. An account that was created in system A but the corresponding record in system B was never written. These half-completed operations are operationally expensive to detect and correct manually, and in regulated industries, they represent compliance risk.

## Solution

The Request-Reply with Compensation pattern extends synchronous request-reply with an explicit rollback capability. When a multi-step synchronous operation fails after one or more steps have succeeded, the pattern invokes compensating actions against the steps that completed successfully, restoring the system to a pre-operation consistent state.

The pattern has three elements:

1. **Forward path:** Execute steps sequentially (or in parallel where dependencies permit). Record each completed step and its identifier (e.g., order ID, payment transaction ID) before proceeding to the next.
2. **Compensation registry:** As each step completes, register its compensating action (what to call, with what identifier, to undo it) in a local structure or durable store.
3. **Compensation path:** On any step failure, execute compensating actions in reverse order for all previously completed steps. Log compensation success and failure separately from forward path logging.

This pattern does not guarantee true ACID atomicity — compensating transactions are business-level undos, not database rollbacks. Systems that processed the forward request must be idempotent when receiving compensation requests.

## When to Use

- Use when a synchronous operation must write to two or more systems and partial failure leaves observable inconsistency.
- Use when business rules require "all or nothing" semantics for a user-facing operation, but a distributed transaction (2PC) is not available or appropriate.
- Use when all downstream systems expose compensation APIs (cancel, reverse, delete, undo) — or can be extended to expose them.
- Avoid when downstream systems do not support idempotent operations — a compensation call that is retried must not double-reverse a transaction.
- Avoid when the operation involves irreversible real-world effects (email sent, payment settled, physical shipment dispatched) where compensation must go through a human-mediated business process rather than an API call.
- Consider the [Saga Pattern](../event-driven/Saga-Pattern.md) instead when the overall transaction involves long-running steps or asynchronous processing.

## Azure Implementation

### Implementation via Azure Durable Functions

Azure Durable Functions provides the coordination runtime for implementing compensation via the orchestration pattern. The orchestrator function manages step sequencing, step tracking, and compensation invocation.

```csharp
[FunctionName("OrderOrchestrator")]
public static async Task<OrderResult> RunOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var request = context.GetInput<OrderRequest>();
    var compensations = new Stack<(string Activity, object Input)>();

    try
    {
        // Step 1: Reserve inventory
        var reservationId = await context.CallActivityAsync<string>(
            "ReserveInventory", new InventoryRequest
            {
                ProductId = request.ProductId,
                Quantity  = request.Quantity
            });
        compensations.Push(("CancelInventoryReservation", new { ReservationId = reservationId }));

        // Step 2: Charge payment
        var paymentId = await context.CallActivityAsync<string>(
            "ChargePayment", new PaymentRequest
            {
                CustomerId = request.CustomerId,
                Amount     = request.TotalAmount,
                Currency   = request.Currency
            });
        compensations.Push(("RefundPayment", new { PaymentId = paymentId }));

        // Step 3: Create order record
        var orderId = await context.CallActivityAsync<string>(
            "CreateOrder", new CreateOrderRequest
            {
                CustomerId    = request.CustomerId,
                ReservationId = reservationId,
                PaymentId     = paymentId,
                Items         = request.Items
            });
        // No compensation needed for order creation if payment and inventory succeed —
        // the order is the successful outcome

        return new OrderResult { OrderId = orderId, Status = "confirmed" };
    }
    catch (Exception ex)
    {
        context.SetCustomStatus("compensating");

        // Execute compensations in reverse order
        var compensationErrors = new List<string>();
        while (compensations.TryPop(out var compensation))
        {
            try
            {
                await context.CallActivityAsync(compensation.Activity, compensation.Input);
            }
            catch (Exception compEx)
            {
                // Log compensation failure — requires manual intervention
                compensationErrors.Add($"{compensation.Activity}: {compEx.Message}");
            }
        }

        if (compensationErrors.Any())
        {
            // Compensation partially failed — alert operations team
            await context.CallActivityAsync("AlertOperationsTeam", new
            {
                OrchestrationId = context.InstanceId,
                OriginalError   = ex.Message,
                CompensationErrors = compensationErrors
            });
        }

        return new OrderResult { Status = "failed", Reason = ex.Message };
    }
}
```

### Activity Function — Idempotent Compensation

Compensation activity functions must be idempotent. If a compensation is retried due to transient failure, it must not double-refund or double-cancel:

```csharp
[FunctionName("RefundPayment")]
public static async Task RefundPayment(
    [ActivityTrigger] RefundRequest request,
    ILogger log)
{
    // Check if refund already exists before issuing (idempotency guard)
    var existing = await _paymentClient.GetRefundAsync(request.PaymentId);
    if (existing?.Status == "completed")
    {
        log.LogInformation("Refund for payment {PaymentId} already completed, skipping", request.PaymentId);
        return;
    }

    await _paymentClient.RefundAsync(request.PaymentId, idempotencyKey: request.PaymentId + "-refund");
}
```

### APIM Integration — Synchronous Response with Async Orchestration

For operations where the orchestration may take more than 10–15 seconds, use the APIM + Durable Functions async polling pattern. The client receives a 202 Accepted with a status URL immediately; the orchestration runs to completion in the background:

```xml
<!-- APIM inbound: pass X-Correlation-ID to orchestrator -->
<inbound>
    <base />
    <set-header name="X-Correlation-ID" exists-action="skip">
        <value>@(Guid.NewGuid().ToString())</value>
    </set-header>
</inbound>
<!-- APIM outbound: rewrite Location header from Function to APIM-relative URL -->
<outbound>
    <base />
    <redirect-content-urls />
</outbound>
```

### Compensation State Persistence

For long-running compensations or compensation audit requirements, persist compensation state to Azure Table Storage or Cosmos DB before and after compensation execution:

```csharp
// Persist compensation record before executing
var compensationRecord = new CompensationRecord
{
    PartitionKey  = request.CorrelationId,
    RowKey        = Guid.NewGuid().ToString(),
    Activity      = compensation.Activity,
    Input         = JsonSerializer.Serialize(compensation.Input),
    Status        = "pending",
    Timestamp     = DateTimeOffset.UtcNow
};
await _tableClient.UpsertEntityAsync(compensationRecord);

// Update after completion
compensationRecord.Status    = "completed";
compensationRecord.CompletedAt = DateTimeOffset.UtcNow;
await _tableClient.UpdateEntityAsync(compensationRecord, ETag.All);
```

## Key Considerations

**Compensation Is Not Rollback:** Compensating transactions execute at the business layer, not the database layer. Between the forward step succeeding and the compensation executing, other processes may have read or acted on the written data. Downstream consumers of that data may need to be notified of the reversal. Design your domain model to accommodate the compensated state (e.g., "order cancelled", "payment refunded") as explicit, observable business states — not silent deletions.

**Compensation Failure:** Compensation can fail. A payment processor may reject a refund request. An inventory system may be down when the cancellation is attempted. Implement a compensation failure alert that routes to an operations team queue and includes all context needed to execute the compensation manually. Log the original forward step IDs (reservation ID, payment ID) alongside the failure.

**Idempotency is Mandatory:** Every forward step and every compensation step must be idempotent. Durable Functions retries activity functions on transient failure — if the activity is not idempotent, a partial-success retry will double-execute the side effect. Use a client-provided or derived idempotency key for every external API call in both the forward and compensation paths.

**Timeout Configuration:** Set activity timeouts in Durable Functions explicitly. An orchestration waiting indefinitely for an unresponsive downstream service will consume resources and delay compensation. Configure `RetryOptions` with a maximum retry count and exponential backoff for transient failures; fail fast after the retry budget is exhausted so compensation can proceed.

**Observability:** Emit structured log entries at each step transition (step started, step completed, step failed, compensation started, compensation completed, compensation failed). Include the orchestration instance ID, correlation ID, and step identifier in every log entry. This is the minimum required to reconstruct the full operation timeline from Application Insights during an incident.

**Testing Compensation Paths:** Unit testing the happy path is easy; testing compensation invocation requires deliberately failing steps at each position in the forward sequence and verifying that the correct subset of compensations was invoked. Build explicit test scenarios for: failure at step 1, failure at step 2, failure at step N, and compensation failure during compensation. These are the production scenarios that will actually occur.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
