# Saga Pattern

## Problem

Distributed business transactions that span multiple services cannot use database-level ACID transactions. A traditional two-phase commit (2PC) requires all participating services to hold locks during the coordination phase, which is incompatible with independent service deployability, horizontal scaling, and polyglot persistence. Even where 2PC is technically achievable, its operational complexity and failure modes make it inadvisable for modern distributed architectures.

Yet many business operations require consistency across service boundaries: placing an order must simultaneously confirm inventory, charge payment, and create a shipment record. If any of these steps fails after others have succeeded, the system is in an inconsistent state. Without an explicit coordination mechanism, this inconsistency accumulates silently — charged customers with no order, decremented inventory with no payment, or shipment records with no payment confirmation.

## Solution

The Saga pattern manages long-running, multi-step business transactions across services using a sequence of local transactions, where each step publishes an event or message that triggers the next step, and compensation transactions reverse completed steps when a failure occurs.

Two implementation variants:

**Choreography:** Each service publishes events when its local transaction completes. Other services subscribe to those events and execute their own local transactions. No central coordinator exists — the workflow emerges from event reactions. Simple to implement, but coordination logic is distributed across services, making end-to-end workflow visibility difficult.

**Orchestration:** A central orchestrator (typically an Azure Durable Function or Logic App) explicitly calls each service step and manages the overall saga state. Services are called by the orchestrator and do not need to know about each other. More complex to implement but provides centralized visibility, easier debugging, and cleaner failure handling.

For complex multi-step sagas in enterprise environments, orchestration is generally preferred.

## When to Use

- Use when a business operation must coordinate state changes across two or more independently deployed services with separate data stores.
- Use orchestration when you need centralized visibility into saga progress and explicit failure handling at each step.
- Use choreography when services are truly independent, the coordination flow is simple (2–3 steps), and you want to minimize coupling to a central orchestrator.
- Avoid when all the data you need to modify lives in a single service with a single database — use a local transaction instead.
- Avoid using saga for read-heavy operations — sagas are for write coordination only.
- Consider [Request-Reply with Compensation](../api/Request-Reply-with-Compensation.md) for synchronous, short-duration operations; use Saga for asynchronous, long-running operations.

## Azure Implementation

### Orchestrated Saga via Azure Durable Functions

The order fulfillment saga: inventory reservation → payment charge → shipment creation → customer notification.

```csharp
[FunctionName("OrderFulfillmentSaga")]
public static async Task<SagaResult> RunOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext context,
    ILogger log)
{
    var request = context.GetInput<OrderFulfillmentRequest>();
    var sagaLog = new SagaLog(context.InstanceId, request.OrderId);

    try
    {
        context.SetCustomStatus("ReservingInventory");

        // Step 1: Reserve inventory (synchronous activity)
        var reservation = await context.CallActivityWithRetryAsync<InventoryReservation>(
            "ReserveInventory",
            new RetryOptions(firstRetryInterval: TimeSpan.FromSeconds(5), maxNumberOfAttempts: 3),
            new ReserveInventoryCommand
            {
                OrderId  = request.OrderId,
                Items    = request.Items,
                SagaId   = context.InstanceId
            });

        sagaLog.RecordStep("InventoryReserved", reservation.ReservationId);
        context.SetCustomStatus("ChargingPayment");

        // Step 2: Charge payment
        var payment = await context.CallActivityWithRetryAsync<PaymentResult>(
            "ChargePayment",
            new RetryOptions(firstRetryInterval: TimeSpan.FromSeconds(10), maxNumberOfAttempts: 3),
            new ChargePaymentCommand
            {
                OrderId    = request.OrderId,
                CustomerId = request.CustomerId,
                Amount     = request.TotalAmount,
                Currency   = request.Currency,
                SagaId     = context.InstanceId
            });

        sagaLog.RecordStep("PaymentCharged", payment.TransactionId);
        context.SetCustomStatus("CreatingShipment");

        // Step 3: Create shipment record
        var shipment = await context.CallActivityAsync<ShipmentRecord>(
            "CreateShipment",
            new CreateShipmentCommand
            {
                OrderId       = request.OrderId,
                ReservationId = reservation.ReservationId,
                PaymentId     = payment.TransactionId,
                Address       = request.ShippingAddress,
                SagaId        = context.InstanceId
            });

        sagaLog.RecordStep("ShipmentCreated", shipment.ShipmentId);

        // Step 4: Notify customer (fire-and-forget; failure doesn't trigger compensation)
        await context.CallActivityAsync("NotifyCustomer",
            new NotifyCustomerCommand
            {
                CustomerId  = request.CustomerId,
                OrderId     = request.OrderId,
                ShipmentId  = shipment.ShipmentId,
                SagaId      = context.InstanceId
            });

        context.SetCustomStatus("Completed");
        return new SagaResult { Status = "completed", OrderId = request.OrderId };
    }
    catch (Exception ex)
    {
        log.LogError(ex, "Saga {SagaId} failed at step {CurrentStep}", context.InstanceId, context.CustomStatus);
        context.SetCustomStatus("Compensating");

        await ExecuteCompensationsAsync(context, sagaLog);

        context.SetCustomStatus("Failed");
        return new SagaResult
        {
            Status    = "failed",
            OrderId   = request.OrderId,
            FailReason = ex.Message
        };
    }
}

private static async Task ExecuteCompensationsAsync(
    IDurableOrchestrationContext context,
    SagaLog sagaLog)
{
    // Compensate in reverse order
    if (sagaLog.HasStep("ShipmentCreated"))
        await context.CallActivityAsync("CancelShipment",
            new CancelShipmentCommand { ShipmentId = sagaLog.GetStepId("ShipmentCreated") });

    if (sagaLog.HasStep("PaymentCharged"))
        await context.CallActivityAsync("RefundPayment",
            new RefundPaymentCommand { TransactionId = sagaLog.GetStepId("PaymentCharged") });

    if (sagaLog.HasStep("InventoryReserved"))
        await context.CallActivityAsync("ReleaseInventoryReservation",
            new ReleaseReservationCommand { ReservationId = sagaLog.GetStepId("InventoryReserved") });
}
```

### Choreography-Based Saga via Service Bus

Each service reacts to events and publishes outcomes. The saga progresses through event reactions:

```csharp
// Inventory Service — reacts to OrderPlaced, publishes InventoryReserved or InventoryReservationFailed
[Function("InventoryService_OnOrderPlaced")]
public async Task Run(
    [ServiceBusTrigger("platform-events", "inventory-saga-subscription", Connection = "ServiceBus")]
    ServiceBusReceivedMessage message,
    ServiceBusMessageActions messageActions)
{
    if (message.Subject != "com.yourplatform.order.placed") { await messageActions.CompleteMessageAsync(message); return; }

    var order = JsonSerializer.Deserialize<OrderPlacedEvent>(message.Body)!;

    try
    {
        var reservationId = await _inventoryService.ReserveAsync(order.Items);

        await _publisher.PublishAsync(new InventoryReservedEvent
        {
            OrderId       = order.OrderId,
            ReservationId = reservationId,
            SagaId        = order.SagaId,
            CorrelationId = message.CorrelationId
        });

        await messageActions.CompleteMessageAsync(message);
    }
    catch (InsufficientStockException ex)
    {
        await _publisher.PublishAsync(new InventoryReservationFailedEvent
        {
            OrderId     = order.OrderId,
            SagaId      = order.SagaId,
            Reason      = ex.Message,
            CorrelationId = message.CorrelationId
        });

        await messageActions.CompleteMessageAsync(message);
    }
}

// Order Service — reacts to InventoryReservationFailed, executes compensation
[Function("OrderService_OnInventoryFailed")]
public async Task Run(
    [ServiceBusTrigger("platform-events", "order-compensation-subscription", Connection = "ServiceBus")]
    ServiceBusReceivedMessage message,
    ServiceBusMessageActions messageActions)
{
    if (message.Subject != "com.yourplatform.inventory.reservation.failed")
    {
        await messageActions.CompleteMessageAsync(message);
        return;
    }

    var failed = JsonSerializer.Deserialize<InventoryReservationFailedEvent>(message.Body)!;
    await _orderService.CancelOrderAsync(failed.OrderId, "InsufficientInventory");
    await messageActions.CompleteMessageAsync(message);
}
```

### Saga State Persistence (Orchestrator Pattern)

Durable Functions persists saga state automatically in Azure Storage. For scenarios requiring external visibility into saga progress (e.g., operational dashboards), project saga state to Cosmos DB from the orchestrator:

```csharp
// Activity function: write saga state for external visibility
[FunctionName("PersistSagaState")]
public async Task Run([ActivityTrigger] SagaStateRecord state)
{
    await _container.UpsertItemAsync(state, new PartitionKey(state.OrderId));
}
```

### Saga Monitoring via Durable Functions Management API

```http
GET https://{function-app}.azurewebsites.net/runtime/webhooks/durabletask/instances?code={key}&runtimeStatus=Running,Failed

# Response: list of running and failed saga instances
```

## Key Considerations

**Saga vs. Process Manager:** A Saga manages state transitions and compensation for a business transaction. A Process Manager additionally routes messages between services and manages complex conditional routing logic. The distinction matters for design: sagas should be relatively stateless (state is the current step and compensation history); process managers require richer state modeling.

**Idempotency in Every Service:** Every service participating in a saga must implement idempotent local transactions. The orchestrator may retry a failed activity — if the activity is not idempotent, the retry produces a duplicate side effect (double reservation, double charge). Use the SagaId + step name as the idempotency key for each service call.

**Compensation Reliability:** Compensation steps can also fail. Implement retry logic on every compensation step. If a compensation step fails after exhausting retries, do not silently continue with remaining compensations — emit an alert and create an operations ticket. A partially compensated saga is an inconsistency that requires human resolution.

**Observability:** Implement centralized saga tracking — a Cosmos DB or Table Storage record per saga instance that records each step's status, timestamp, and output. The Durable Functions management API provides this for orchestrated sagas, but export it to Application Insights or a custom dashboard for operational visibility. Alert on sagas that remain in a non-terminal state beyond the expected maximum duration.

**Saga Duration Limits:** Durable Functions orchestrators have a practical limit on history size (tens of thousands of events). For very long-running sagas (days or weeks), implement periodic checkpointing using the continue-as-new pattern to reset the history while preserving logical saga state.

**Testing:** Integration-test sagas with failure injection at every step. Specifically test: failure at step 1 (no compensations needed), failure at step N-1 (all preceding compensations executed), compensation failure at step K (remaining compensations still execute). These are not edge cases — they are the primary failure modes in production.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
