# Dead-Letter Recovery

## Problem

In asynchronous messaging systems, some messages cannot be processed successfully. The cause may be a malformed payload, a schema mismatch, a permanently unavailable downstream system, a business rule violation, or a bug in the consumer. When a message cannot be processed and cannot be retried to success, it must go somewhere — otherwise it either blocks queue processing or is silently discarded.

Without an explicit dead-letter strategy, failed messages are invisible. Operations teams discover problems through degraded downstream systems or missing data, not through proactive alerting on the messaging layer. Diagnosing what went wrong requires reconstructing the failed message from system logs, if they exist at all. Recovery — re-processing valid messages that failed due to a transient infrastructure issue — is manual, ad hoc, and error-prone.

## Solution

The Dead-Letter Recovery pattern treats dead-lettered messages as first-class operational artifacts rather than discarded failures. It establishes:

1. **Structured dead-lettering:** Consumers dead-letter messages with meaningful reason codes and error detail — not just "failed."
2. **DLQ monitoring:** Dead-letter queues are monitored with alerting thresholds that trigger operational response.
3. **Recovery tooling:** A recovery workflow that can inspect, classify, replay, or discard dead-lettered messages — with audit logging of every recovery action.
4. **Root cause feedback:** Metrics on DLQ reason codes feed back to development teams to drive bug fixes and schema improvements.

On Azure Service Bus, every queue and topic subscription automatically has a dead-letter sub-queue (suffix `/deadletterqueue`). Messages that exceed `MaxDeliveryCount` or are explicitly dead-lettered by the consumer are moved there. Azure Event Grid similarly routes failed delivery events to a dead-letter storage blob container.

## When to Use

- Apply this pattern as a standard operational requirement for every Service Bus queue and topic subscription — it is not optional.
- Apply to Event Grid subscriptions where delivery failures would represent data loss.
- The recovery tooling investment is proportional to the business criticality of the messages in the queue. High-criticality domains (financial transactions, order processing) require automated recovery workflows; lower-criticality domains may tolerate manual replay tooling.

## Azure Implementation

### Structured Dead-Lettering in Consumer Functions

```csharp
[Function("ProcessOrderMessage")]
public async Task Run(
    [ServiceBusTrigger("orders-queue", Connection = "ServiceBus")]
    ServiceBusReceivedMessage message,
    ServiceBusMessageActions messageActions)
{
    try
    {
        var payload = JsonSerializer.Deserialize<OrderMessage>(message.Body.ToArray());

        if (payload == null)
        {
            await messageActions.DeadLetterMessageAsync(message,
                deadLetterReason:           "DeserializationFailed",
                deadLetterErrorDescription: "Message body could not be deserialized to OrderMessage");
            return;
        }

        if (!IsValidOrder(payload, out var validationErrors))
        {
            await messageActions.DeadLetterMessageAsync(message,
                deadLetterReason:           "ValidationFailed",
                deadLetterErrorDescription: string.Join("; ", validationErrors));
            return;
        }

        await _orderService.ProcessAsync(payload);
        await messageActions.CompleteMessageAsync(message);
    }
    catch (SchemaVersionException ex)
    {
        // Schema mismatch — unlikely to succeed on retry
        await messageActions.DeadLetterMessageAsync(message,
            deadLetterReason:           "SchemaVersionMismatch",
            deadLetterErrorDescription: ex.Message);
    }
    catch (Exception ex)
    {
        // Unknown failure — abandon for retry up to MaxDeliveryCount
        _logger.LogError(ex, "Transient failure processing order message {MessageId}", message.MessageId);
        await messageActions.AbandonMessageAsync(message,
            propertiesToModify: new Dictionary<string, object>
            {
                ["LastError"]     = ex.Message,
                ["LastFailedAt"]  = DateTimeOffset.UtcNow.ToString("o")
            });
    }
}
```

### DLQ Monitor — Azure Function Reading Dead-Letter Queue

A dedicated Function monitors the DLQ and emits metrics and alerts:

```csharp
[Function("DeadLetterQueueMonitor")]
public async Task Run([TimerTrigger("0 */5 * * * *")] TimerInfo timer)
{
    var client = new ServiceBusAdministrationClient(_connectionString);
    var queues = new[] { "orders-queue", "payments-queue", "inventory-queue" };

    foreach (var queueName in queues)
    {
        var props = await client.GetQueueRuntimePropertiesAsync(queueName);
        var dlqCount = props.Value.DeadLetterMessageCount;

        // Emit metric to Application Insights
        _telemetryClient.TrackMetric(
            metricName: "ServiceBus.DeadLetterCount",
            value:      dlqCount,
            properties: new Dictionary<string, string>
            {
                ["QueueName"] = queueName,
                ["Namespace"] = _namespaceName
            });

        if (dlqCount > _alertThreshold)
        {
            _logger.LogWarning("DLQ threshold exceeded: {Queue} has {Count} dead-lettered messages",
                queueName, dlqCount);
        }
    }
}
```

### Recovery Workflow — Inspect, Classify, and Replay

The recovery workflow is a Logic App or Durable Function that allows operations teams to inspect DLQ messages, classify them, and choose a recovery action:

```csharp
[Function("DLQRecoveryOrchestrator")]
public static async Task<RecoveryResult> RunOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var request = context.GetInput<RecoveryRequest>();

    // Step 1: Read batch of messages from DLQ
    var messages = await context.CallActivityAsync<List<DLQMessage>>(
        "ReadDLQBatch",
        new ReadDLQRequest
        {
            QueueName  = request.QueueName,
            BatchSize  = request.BatchSize ?? 20
        });

    // Step 2: Classify messages by dead-letter reason
    var classified = await context.CallActivityAsync<ClassifiedMessages>(
        "ClassifyMessages", messages);

    // Step 3: Auto-replay messages that failed due to transient infrastructure issues
    var replayResults = await context.CallActivityAsync<List<ReplayResult>>(
        "ReplayMessages",
        classified.TransientFailures.Select(m => new ReplayRequest
        {
            Message         = m,
            TargetQueue     = request.QueueName.Replace("/$deadletterqueue", ""),
            ReplayedBy      = request.InitiatedBy,
            ReplayReason    = "Auto-recovery: transient failure resolved"
        }).ToList());

    // Step 4: Route permanent failures to operations review
    if (classified.PermanentFailures.Any())
    {
        await context.CallActivityAsync("CreateOperationsTicket",
            new TicketRequest
            {
                Messages  = classified.PermanentFailures,
                QueueName = request.QueueName,
                Summary   = $"{classified.PermanentFailures.Count} messages require manual review"
            });
    }

    return new RecoveryResult
    {
        Replayed  = replayResults.Count(r => r.Success),
        Failed    = replayResults.Count(r => !r.Success),
        Escalated = classified.PermanentFailures.Count
    };
}
```

### Replay Activity — Re-enqueue with Audit Trail

```csharp
[Function("ReplayMessages")]
public async Task<List<ReplayResult>> ReplayMessages(
    [ActivityTrigger] List<ReplayRequest> requests)
{
    var results = new List<ReplayResult>();
    var sender  = _serviceBusClient.CreateSender(requests.First().TargetQueue);

    foreach (var request in requests)
    {
        try
        {
            var replayMessage = new ServiceBusMessage(request.Message.Body)
            {
                MessageId     = request.Message.MessageId,
                CorrelationId = request.Message.CorrelationId,
                ContentType   = request.Message.ContentType,
                Subject       = request.Message.Subject
            };

            // Copy original properties
            foreach (var prop in request.Message.ApplicationProperties)
                replayMessage.ApplicationProperties[prop.Key] = prop.Value;

            // Add replay audit metadata
            replayMessage.ApplicationProperties["ReplayedAt"]     = DateTimeOffset.UtcNow.ToString("o");
            replayMessage.ApplicationProperties["ReplayedBy"]     = request.ReplayedBy;
            replayMessage.ApplicationProperties["ReplayReason"]   = request.ReplayReason;
            replayMessage.ApplicationProperties["OriginalDLQReason"] = request.Message.DeadLetterReason;

            await sender.SendMessageAsync(replayMessage);

            // Complete the DLQ message to remove it
            await _dlqReceiver.CompleteMessageAsync(request.Message.ReceivedMessage);

            results.Add(new ReplayResult { MessageId = request.Message.MessageId, Success = true });
        }
        catch (Exception ex)
        {
            results.Add(new ReplayResult
            {
                MessageId = request.Message.MessageId,
                Success   = false,
                Error     = ex.Message
            });
        }
    }

    return results;
}
```

### Event Grid Dead-Letter Configuration

```bicep
resource eventSubscription 'Microsoft.EventGrid/eventSubscriptions@2023-12-15-preview' = {
  name: 'orders-subscription'
  properties: {
    deadLetterDestination: {
      endpointType: 'StorageBlob'
      properties: {
        resourceId: storageAccount.id
        blobContainerName: 'event-grid-deadletter'
      }
    }
    retryPolicy: {
      maxDeliveryAttempts: 30
      eventTimeToLiveInMinutes: 1440
    }
  }
}
```

## Key Considerations

**DLQ Reason Code Taxonomy:** Establish a standard set of dead-letter reason codes shared across all consumer teams: `DeserializationFailed`, `ValidationFailed`, `SchemaVersionMismatch`, `ProcessingFailed`, `MaxRetriesExceeded`, `BusinessRuleViolation`. Without a taxonomy, DLQ reason codes are freeform text that is difficult to aggregate, alert on, and triage efficiently.

**Alert Thresholds:** Set DLQ depth alerts at two levels: a warning threshold (e.g., >5 messages) that triggers investigation, and a critical threshold (e.g., >50 messages) that triggers immediate escalation. Thresholds must be calibrated per queue — a queue that processes 10,000 messages per minute has a different risk profile than one that processes 10 per hour.

**DLQ Message Retention:** Service Bus retains DLQ messages up to the entity's `MessageTimeToLive` (TTL) setting. If recovery takes longer than the TTL, messages are permanently lost. Set TTL conservatively on high-criticality queues (7–14 days). For compliance-sensitive domains, archive DLQ messages to Azure Blob Storage or Cosmos DB immediately on dead-lettering.

**Replay Idempotency:** Replayed messages may land at a consumer that has already partially processed the original message (before the failure that caused dead-lettering). Consumer processing logic must be idempotent with respect to the message ID, not just transient infrastructure retries.

**Root Cause Tracking:** Treat DLQ reason code metrics as engineering quality signals, not just operational noise. A sustained increase in `SchemaVersionMismatch` errors means producers and consumers are not coordinating schema evolution. Feed this data back to the team responsible for the schema contract.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
