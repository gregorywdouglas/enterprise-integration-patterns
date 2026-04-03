# Competing Consumers

## Problem

High-volume message processing workloads cannot be served by a single consumer instance. A single processor working through a queue serializes work that could be performed in parallel, creating a throughput ceiling. When the queue depth grows — due to traffic spikes, slow consumer processing, or downstream system latency — latency increases and messages age in the queue waiting to be processed.

Scaling the consumer is the obvious solution, but naive scaling introduces new problems: multiple consumers competing for the same messages may process a message more than once, or consumers may interfere with each other when processing messages related to the same entity. Simply adding more consumers without coordination produces correctness issues even as it increases raw throughput.

## Solution

The Competing Consumers pattern distributes message processing across multiple consumer instances that compete for messages from a shared queue. Each message is processed by exactly one consumer. The queue provides the coordination mechanism — its message lock protocol ensures that once a consumer acquires a message, other consumers cannot see it until the lock expires. If the acquiring consumer fails to process the message, the lock expires and another consumer can pick it up.

On Azure Service Bus, the competing consumer pattern is the default behavior for queue consumers. Azure Functions with a Service Bus trigger implements this pattern automatically: the Functions runtime scales out worker instances (within configured limits) and the Service Bus SDK handles message lock management, completion, and abandonment.

The pattern supports two scaling strategies:
- **Horizontal scale-out:** Add consumer instances to increase throughput proportionally to the number of concurrent messages the queue can serve.
- **Session-based ordering:** When messages related to the same entity must be processed in order, group them by session ID. Each session is exclusive to one consumer at a time, preserving order without sacrificing parallelism across sessions.

## When to Use

- Use when a single consumer cannot keep up with message production rates.
- Use when messages are independent of each other and can be processed in any order across consumer instances.
- Use when you need elastic scaling — consumer count grows during peak load and shrinks during quiet periods.
- Use session-based competing consumers when messages for the same entity must be ordered but parallelism across different entities is acceptable.
- Avoid when all messages must be processed in strict global arrival order — this requires a single consumer with sessions or Event Hubs with partition-aware consumers.
- Avoid when message processing has a side effect that is not idempotent — duplicate processing due to lock expiry must be handled by the processing logic.

## Azure Implementation

### Azure Functions + Service Bus Queue (Auto-Scaling)

```csharp
// Functions automatically scales out to process messages concurrently
[Function("ProcessOrderMessage")]
public async Task Run(
    [ServiceBusTrigger("orders-queue", Connection = "ServiceBus")]
    ServiceBusReceivedMessage message,
    ServiceBusMessageActions messageActions)
{
    var correlationId = message.CorrelationId ?? Guid.NewGuid().ToString();

    try
    {
        var order = JsonSerializer.Deserialize<OrderMessage>(message.Body)
            ?? throw new InvalidOperationException("Cannot deserialize message body");

        // Idempotency check: verify this order hasn't already been processed
        if (await _processedOrderStore.ExistsAsync(order.OrderId))
        {
            _logger.LogWarning("Duplicate message detected for order {OrderId}, completing without processing",
                order.OrderId);
            await messageActions.CompleteMessageAsync(message);
            return;
        }

        await _orderProcessor.ProcessAsync(order, correlationId);
        await _processedOrderStore.MarkProcessedAsync(order.OrderId);

        await messageActions.CompleteMessageAsync(message);

        _logger.LogInformation("Order {OrderId} processed successfully on instance {InstanceId}",
            order.OrderId, Environment.MachineName);
    }
    catch (TransientException ex)
    {
        // Abandon: message returns to queue for retry
        _logger.LogWarning(ex, "Transient failure processing order, abandoning for retry");
        await messageActions.AbandonMessageAsync(message,
            new Dictionary<string, object> { ["LastError"] = ex.Message });
    }
    catch (PermanentException ex)
    {
        // Dead-letter: message won't succeed with retry
        _logger.LogError(ex, "Permanent failure processing order, dead-lettering message");
        await messageActions.DeadLetterMessageAsync(message,
            deadLetterReason: "ProcessingFailed",
            deadLetterErrorDescription: ex.Message);
    }
}
```

### host.json — Concurrency and Prefetch Configuration

```json
{
  "version": "2.0",
  "extensions": {
    "serviceBus": {
      "prefetchCount": 10,
      "messageHandlerOptions": {
        "maxConcurrentCalls": 16,
        "autoComplete": false,
        "maxAutoRenewDuration": "00:05:00"
      }
    }
  },
  "functionTimeout": "00:10:00"
}
```

`prefetchCount` controls how many messages a single instance fetches from the broker at once (reducing round-trip latency). `maxConcurrentCalls` controls how many messages one Function instance processes simultaneously. Total parallelism = instances × maxConcurrentCalls.

### KEDA-Based Scaling (Container Apps or AKS)

For container-based consumers requiring more predictable scaling than Functions provides, KEDA scales pods based on Service Bus queue depth:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
spec:
  scaleTargetRef:
    name: order-processor-deployment
  minReplicaCount: 1
  maxReplicaCount: 20
  pollingInterval: 15
  cooldownPeriod: 300
  triggers:
    - type: azure-servicebus
      metadata:
        namespace: yourplatform-servicebus
        queueName: orders-queue
        messageCount: "5"          # Scale up when >5 messages per pod
        connectionFromEnv: SERVICE_BUS_CONNECTION_STRING
```

### Session-Based Ordering with Competing Consumers

When processing order events that must be applied in sequence per order ID:

```csharp
// Session receiver — each session (orderId) is processed by one consumer at a time
[Function("ProcessOrderEventSession")]
public async Task RunSession(
    [ServiceBusTrigger("order-events-queue", IsSessionsEnabled = true, Connection = "ServiceBus")]
    ServiceBusReceivedMessage message,
    ServiceBusSessionMessageActions sessionActions)
{
    var orderId = message.SessionId;  // OrderId is the session key

    _logger.LogInformation("Processing event {EventType} for order {OrderId} (sequence {Seq})",
        message.Subject, orderId, message.ApplicationProperties["SequenceNumber"]);

    try
    {
        await _orderEventProcessor.ApplyEventAsync(
            orderId:      orderId,
            eventType:    message.Subject,
            eventPayload: message.Body.ToArray(),
            sequence:     (long)message.ApplicationProperties["SequenceNumber"]);

        await sessionActions.CompleteMessageAsync(message);
    }
    catch (Exception ex)
    {
        await sessionActions.DeadLetterMessageAsync(message,
            deadLetterReason: "SessionProcessingFailed",
            deadLetterErrorDescription: ex.Message);
    }
}
```

### Idempotency Store

The idempotency store prevents duplicate processing when a message lock expires and the message is redelivered:

```csharp
public class RedisIdempotencyStore
{
    private readonly IConnectionMultiplexer _redis;

    public async Task<bool> TryAcquireAsync(string messageId, TimeSpan expiry)
    {
        var db = _redis.GetDatabase();
        // SetNX (set if not exists) returns true only if this is the first time
        return await db.StringSetAsync(
            key:   $"processed:{messageId}",
            value: DateTimeOffset.UtcNow.ToString("o"),
            expiry: expiry,
            when:  When.NotExists);
    }
}
```

## Key Considerations

**Idempotency is Required:** Service Bus delivers at-least-once by default. Lock expiry, consumer crashes, and network failures can all cause a message to be redelivered to a different consumer instance. Processing logic must be idempotent — processing the same message twice must produce the same observable outcome as processing it once. Implement an idempotency check using the message's `MessageId` before executing side effects.

**Delivery Count and Max Delivery:** Configure `MaxDeliveryCount` on the queue or subscription (default: 10). After this many delivery attempts, Service Bus automatically dead-letters the message. Set this value based on your retry budget — too low and transient failures exhaust retries; too high and permanently broken messages cycle through the queue consuming resources. A value of 3–5 with exponential delay between attempts is common.

**Lock Duration:** Configure message lock duration (`LockDuration`) to exceed the expected maximum processing time. If processing takes longer than the lock duration, the lock expires and the message is redelivered to another consumer, resulting in duplicate processing. For variable processing times, use lock renewal (enabled via `maxAutoRenewDuration` in host.json). Set `lockDuration` conservatively and monitor lock renewal failures.

**Throughput vs. Ordering:** Competing consumers and strict global ordering are mutually exclusive. If you need both throughput and ordering, use sessions (which provide per-entity ordering while maintaining cross-entity parallelism). Document the ordering guarantee level your implementation provides.

**Queue Depth Monitoring:** Monitor the active message count and scheduled message count on the queue as health signals. A steadily growing queue depth indicates consumer throughput is insufficient for the production rate. Alert when queue depth exceeds a threshold representing acceptable processing lag for your SLA.

**Poison Message Detection:** A message that consistently causes consumer failure but does not reach `MaxDeliveryCount` (because the consumer crashes before completing or abandoning it) can stall processing. Implement explicit poison message detection: if a message's `DeliveryCount` is greater than 1 when received, log it and apply additional scrutiny before processing.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
