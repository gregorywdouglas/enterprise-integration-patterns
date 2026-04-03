# Publish-Subscribe with Filtering

## Problem

As the number of event types and event consumers in a platform grows, unfiltered publish-subscribe becomes unmanageable. Without filtering, consumers receive every event published to a topic, even events they have no interest in. This wastes consumer processing resources, increases message processing latency for events the consumer actually cares about, and requires consumer-side filtering logic that is difficult to test, maintain, and evolve.

Conversely, if filtering is implemented only on the consumer side (after message receipt), the broker still delivers all messages to all consumers — wasting network bandwidth, Service Bus message operations (which are billed per operation), and compute cycles on deserialization of irrelevant messages.

The problem is compounded in multi-tenant or multi-domain platforms where different consumers have both different event type interests and different data access permissions. Without server-side filtering, a consumer could receive events intended for a different tenant or domain.

## Solution

Publish-Subscribe with Filtering adds server-side filter evaluation between the publisher (topic) and each subscriber (subscription). Each subscription carries its own filter expression. The broker evaluates the filter against each incoming message and delivers only matching messages to that subscription. Non-matching messages are silently skipped — the subscription never sees them.

Azure Service Bus Topic Subscriptions support two filter types:
- **SQL Filter:** A SQL92 subset expression evaluated against message application properties (e.g., `EventType = 'OrderCreated' AND Region = 'US-EAST'`)
- **Correlation Filter:** A hash-based filter on well-known message properties (ContentType, Label/Subject, MessageId, CorrelationId, ReplyTo, SessionId, and custom properties). More efficient than SQL filters for simple equality matching.

Azure Event Grid subscriptions support advanced filtering on event properties (data fields, subject, event type) with operators including string contains, begins-with, ends-with, and numeric comparisons.

## When to Use

- Use when multiple consumers subscribe to the same topic but each consumer only needs a subset of the published events.
- Use when consumers have different access levels and should not receive events from domains or tenants they are not authorized for.
- Use correlation filters (over SQL filters) when filtering on a single well-known property — they are evaluated more efficiently by the broker.
- Use SQL filters when filtering requires compound logic (AND/OR), range comparisons, or evaluation of custom application properties.
- Avoid when all consumers genuinely need all messages — a subscription with a TrueFilter is a valid and correct configuration.
- Avoid when filter logic is extremely complex (multiple nested conditions, string pattern matching beyond simple contains) — this is a signal to split into separate topics or implement application-level routing.

## Azure Implementation

### Service Bus Topic with Multiple Filtered Subscriptions

```csharp
// Infrastructure setup: create topic with filtered subscriptions
var adminClient = new ServiceBusAdministrationClient(connectionString);

// Create the topic
await adminClient.CreateTopicAsync(new CreateTopicOptions("platform-events")
{
    DefaultMessageTimeToLive   = TimeSpan.FromDays(7),
    RequiresDuplicateDetection = true,
    DuplicateDetectionHistoryTimeWindow = TimeSpan.FromMinutes(10)
});

// Subscription 1: Inventory service — receives only inventory-affecting events
await adminClient.CreateSubscriptionAsync(
    new CreateSubscriptionOptions("platform-events", "inventory-service")
    {
        MaxDeliveryCount           = 5,
        DeadLetteringOnMessageExpiration = true,
        LockDuration               = TimeSpan.FromSeconds(60)
    },
    new CreateRuleOptions("inventory-event-filter",
        new SqlRuleFilter(
            "EventType IN ('OrderConfirmed', 'OrderCancelled', 'ReturnRequested') " +
            "AND EventVersion >= 2")));

// Subscription 2: Notification service — receives events requiring customer communication
await adminClient.CreateSubscriptionAsync(
    new CreateSubscriptionOptions("platform-events", "notification-service")
    {
        MaxDeliveryCount = 5,
        LockDuration     = TimeSpan.FromSeconds(30)
    },
    new CreateRuleOptions("notification-filter",
        new SqlRuleFilter(
            "RequiresNotification = true " +
            "AND CustomerEmail IS NOT NULL")));

// Subscription 3: Audit service — uses correlation filter for efficiency (all events)
await adminClient.CreateSubscriptionAsync(
    new CreateSubscriptionOptions("platform-events", "audit-service")
    {
        MaxDeliveryCount = 10,
        LockDuration     = TimeSpan.FromSeconds(60)
    },
    new CreateRuleOptions("all-events",
        new CorrelationRuleFilter { Subject = "*" }));

// Subscription 4: Regional filter — EU events only (GDPR boundary)
await adminClient.CreateSubscriptionAsync(
    new CreateSubscriptionOptions("platform-events", "eu-compliance-service"),
    new CreateRuleOptions("eu-region-filter",
        new SqlRuleFilter("DataResidencyRegion = 'EU'")));
```

### Publisher — Setting Application Properties for Filtering

Publishers must set the message properties that subscriptions filter on. This is a contract between publisher and subscriber mediated through the topic:

```csharp
public class PlatformEventPublisher
{
    private readonly ServiceBusSender _sender;

    public async Task PublishAsync<T>(PlatformEvent<T> platformEvent, CancellationToken ct = default)
        where T : class
    {
        var message = new ServiceBusMessage(
            JsonSerializer.SerializeToUtf8Bytes(platformEvent.Payload))
        {
            MessageId     = platformEvent.EventId,
            CorrelationId = platformEvent.CorrelationId,
            ContentType   = "application/json",
            Subject       = platformEvent.EventType
        };

        // Application properties used by subscription filters
        message.ApplicationProperties["EventType"]           = platformEvent.EventType;
        message.ApplicationProperties["EventVersion"]        = platformEvent.SchemaVersion;
        message.ApplicationProperties["Domain"]              = platformEvent.Domain;
        message.ApplicationProperties["TenantId"]            = platformEvent.TenantId;
        message.ApplicationProperties["DataResidencyRegion"] = platformEvent.DataResidencyRegion;
        message.ApplicationProperties["RequiresNotification"]= platformEvent.RequiresNotification;
        message.ApplicationProperties["Priority"]            = platformEvent.Priority;

        // Only set if present — SQL IS NOT NULL filter works correctly on absent properties
        if (platformEvent.CustomerEmail != null)
            message.ApplicationProperties["CustomerEmail"] = platformEvent.CustomerEmail;

        await _sender.SendMessageAsync(message, ct);
    }
}
```

### Consumer — Receiving Filtered Messages

Consumers receive only the messages matching their subscription filter. The consumer implementation does not need to re-apply the filter:

```csharp
[Function("InventoryEventConsumer")]
public async Task Run(
    [ServiceBusTrigger(
        topicName:        "platform-events",
        subscriptionName: "inventory-service",
        Connection:       "ServiceBus")]
    ServiceBusReceivedMessage message,
    ServiceBusMessageActions messageActions)
{
    var eventType = message.ApplicationProperties["EventType"]?.ToString()
                    ?? message.Subject;

    _logger.LogInformation("Inventory service received {EventType} event {MessageId}",
        eventType, message.MessageId);

    switch (eventType)
    {
        case "OrderConfirmed":
            var order = JsonSerializer.Deserialize<OrderConfirmedPayload>(message.Body)!;
            await _inventoryService.ReserveStockAsync(order.Items, message.CorrelationId);
            break;

        case "OrderCancelled":
            var cancelled = JsonSerializer.Deserialize<OrderCancelledPayload>(message.Body)!;
            await _inventoryService.ReleaseReservationAsync(cancelled.OrderId, message.CorrelationId);
            break;

        case "ReturnRequested":
            var returnReq = JsonSerializer.Deserialize<ReturnRequestedPayload>(message.Body)!;
            await _inventoryService.InitiateReturnAsync(returnReq, message.CorrelationId);
            break;
    }

    await messageActions.CompleteMessageAsync(message);
}
```

### Event Grid Subscription with Advanced Filtering

```bicep
resource orderEventsSubscription 'Microsoft.EventGrid/systemTopics/eventSubscriptions@2023-12-15-preview' = {
  parent: orderSystemTopic
  name: 'high-value-order-alerts'
  properties: {
    destination: {
      endpointType: 'WebHook'
      properties: {
        endpointUrl: 'https://alert-service.internal/api/events'
      }
    }
    filter: {
      includedEventTypes: ['com.yourplatform.order.created']
      advancedFilters: [
        {
          operatorType: 'NumberGreaterThan'
          key:          'data.totalAmount'
          value:        10000
        }
        {
          operatorType: 'StringIn'
          key:          'data.currency'
          values:       ['USD', 'EUR', 'GBP']
        }
      ]
      enableAdvancedFilteringOnArrays: true
    }
  }
}
```

### Managing Filter Versions via Infrastructure as Code

Subscription filters are infrastructure, not configuration. Manage them through Bicep or Terraform and deploy via CI/CD pipelines. Avoid modifying filters directly in the Azure portal — changes are not tracked and can break consumer expectations silently.

```bicep
resource inventorySubscriptionRule 'Microsoft.ServiceBus/namespaces/topics/subscriptions/rules@2022-10-01-preview' = {
  parent: inventorySubscription
  name: 'inventory-event-filter'
  properties: {
    filterType: 'SqlFilter'
    sqlFilter: {
      sqlExpression: "EventType IN ('OrderConfirmed', 'OrderCancelled', 'ReturnRequested') AND EventVersion >= 2"
    }
  }
}
```

## Key Considerations

**Filter/Publisher Contract:** Subscription filters are a contract between publisher and subscriber mediated by message properties. Document the full set of application properties that publishers set on messages for each topic, and which subscriptions depend on which properties. Schema drift (publisher stops setting a property that a filter depends on) breaks filtering silently — messages that should match no longer match, and consumers process an incorrect subset of events.

**SQL Filter Cost:** SQL filters are evaluated on every message delivery per subscription. Compound SQL filters with multiple clauses are more expensive than correlation filters. For high-throughput topics (thousands of messages per second), SQL filter evaluation CPU cost on the broker is measurable. Use correlation filters where possible; reserve SQL filters for genuinely complex conditions.

**NULL Property Handling in SQL Filters:** A SQL filter expression referencing a property that is absent on the message evaluates the property as NULL. `EventType = 'OrderCreated'` will not match a message where `EventType` is not set in application properties — the comparison evaluates to unknown. If your publisher sometimes omits a property, account for this explicitly in your filter expression: `EventType = 'OrderCreated' OR EventType IS NULL` if absent should be treated as a match.

**Filter Testing:** Test subscription filters explicitly, including edge cases — null properties, unexpected property values, integer vs. string type mismatches. Service Bus SQL filter evaluation is strict about type compatibility. Write infrastructure tests that publish messages with various property combinations and verify each subscription receives exactly the expected messages.

**Multi-Tenant Filtering:** For multi-tenant platforms, a `TenantId` filter on every subscription is a critical security control — not just a performance optimization. Verify in testing that a message published for Tenant A is not delivered to a subscription filtering for Tenant B, including edge cases like shared tenant IDs or missing TenantId properties.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
