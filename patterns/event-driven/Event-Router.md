# Event Router

## Problem

In a system with multiple event producers and multiple event consumers, the naive approach is point-to-point connectivity: each producer knows about each consumer and delivers events directly. This creates a fully connected dependency graph where every producer change requires consumer awareness and vice versa. As the number of participants grows, the coupling becomes unmanageable. Adding a new consumer requires modifying producers; removing a consumer requires cleaning up point-to-point connections; adding a new event type requires updating every consumer that doesn't care about it to explicitly ignore it.

Additionally, point-to-point event delivery provides no centralized visibility into event flow, no standard retry or dead-letter handling, and no mechanism for routing events differently based on content or context (e.g., routing high-priority events to dedicated processing lanes).

## Solution

The Event Router pattern interposes a routing intermediary between event producers and event consumers. Producers publish events to the router without knowledge of or dependency on consumers. The router delivers events to consumers based on configurable routing rules: event type, event content, consumer subscription filters, or contextual metadata.

On Azure, the Event Router is implemented through a combination of:

- **Azure Event Grid** — For cloud event routing based on event type and subject filtering, with native integration to Azure services as event sources
- **Azure Service Bus Topics + Subscriptions** — For content-based routing with rich SQL-filter expressions, ordered delivery requirements, and reliable at-least-once delivery semantics
- **Azure Functions** — For dynamic routing logic that cannot be expressed in declarative filter expressions

The choice between Event Grid and Service Bus depends on delivery guarantees, message size, ordering requirements, and filter complexity.

## When to Use

- Use when multiple consumers need to independently receive and process the same events without producer awareness of consumer topology.
- Use when different consumers need different subsets of events from the same producer (content-based routing).
- Use when consumers need to be added or removed without modifying producers.
- Use Azure Event Grid when event sources are Azure services (Blob Storage, Resource Manager, IoT Hub) or when push-based delivery with HTTP webhooks is acceptable.
- Use Azure Service Bus Topics when delivery guarantees (at-least-once), message ordering (sessions), large message size (up to 256 KB standard / 100 MB premium), or rich filter expressions are required.
- Avoid when all events must go to a single consumer — a router adds unnecessary indirection in point-to-one scenarios.

## Azure Implementation

### Event Grid — Cloud-Native Event Routing

Event Grid routes events from system topics (Azure services) and custom topics to multiple subscriptions (consumers). Each subscription has independent filter rules:

```bicep
// Custom Event Grid Topic for application events
resource eventGridTopic 'Microsoft.EventGrid/topics@2023-12-15-preview' = {
  name: 'platform-events'
  location: resourceGroup().location
  properties: {
    inputSchema: 'CloudEventSchemaV1_0'
    publicNetworkAccess: 'Disabled'  // Use private endpoint
  }
}

// Subscription 1: Order service receives only order events
resource orderEventSubscription 'Microsoft.EventGrid/topics/eventSubscriptions@2023-12-15-preview' = {
  parent: eventGridTopic
  name: 'order-service-subscription'
  properties: {
    destination: {
      endpointType: 'ServiceBusQueue'
      properties: {
        resourceId: orderServiceQueue.id
      }
    }
    filter: {
      includedEventTypes: ['com.yourplatform.order.created', 'com.yourplatform.order.updated']
      enableAdvancedFilteringOnArrays: true
    }
    deadLetterDestination: {
      endpointType: 'StorageBlob'
      properties: {
        resourceId: deadLetterStorageAccount.id
        blobContainerName: 'event-grid-dlq'
      }
    }
    retryPolicy: {
      maxDeliveryAttempts: 30
      eventTimeToLiveInMinutes: 1440  // 24 hours
    }
  }
}

// Subscription 2: Audit service receives ALL events
resource auditEventSubscription 'Microsoft.EventGrid/topics/eventSubscriptions@2023-12-15-preview' = {
  parent: eventGridTopic
  name: 'audit-service-subscription'
  properties: {
    destination: {
      endpointType: 'ServiceBusQueue'
      properties: {
        resourceId: auditQueue.id
      }
    }
    filter: {
      includedEventTypes: ['*']  // All event types
    }
  }
}
```

### Publishing Events to Event Grid

```csharp
// Publishing CloudEvents from an Azure Function
[Function("PublishOrderEvent")]
public async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "post", Route = "orders/{orderId}/events")]
    HttpRequest req, string orderId)
{
    var orderEvent = new CloudEvent(
        source: "/orders-service",
        type:   "com.yourplatform.order.created",
        data:   new OrderCreatedData
        {
            OrderId    = orderId,
            CustomerId = "cust-12345",
            TotalAmount = 299.99m,
            Currency    = "USD"
        })
    {
        Id      = Guid.NewGuid().ToString(),
        Time    = DateTimeOffset.UtcNow,
        Subject = $"orders/{orderId}"
    };

    await _eventGridClient.SendEventAsync(orderEvent);

    return new AcceptedResult();
}
```

### Service Bus Topics — Content-Based Routing with SQL Filters

Service Bus subscription filters support SQL92 expressions on message properties, enabling rich content-based routing:

```csharp
// Create subscriptions with SQL filters via Service Bus Administration Client
var adminClient = new ServiceBusAdministrationClient(connectionString);

// Subscription: High-priority orders (value > $10,000) → priority processing queue
await adminClient.CreateSubscriptionAsync(
    new CreateSubscriptionOptions("orders-topic", "high-value-orders"),
    new CreateRuleOptions("high-value-filter",
        new SqlRuleFilter("OrderTotal > 10000 AND Currency = 'USD'")));

// Subscription: International orders → compliance review subscription
await adminClient.CreateSubscriptionAsync(
    new CreateSubscriptionOptions("orders-topic", "international-orders"),
    new CreateRuleOptions("international-filter",
        new SqlRuleFilter("ShippingCountry <> 'US'")));

// Subscription: All orders → audit log
await adminClient.CreateSubscriptionAsync(
    new CreateSubscriptionOptions("orders-topic", "all-orders-audit"),
    new CreateRuleOptions("all-filter", new TrueRuleFilter()));
```

### Publishing with Message Properties for SQL Filtering

```csharp
// Producer: set message properties that subscription filters evaluate
var message = new ServiceBusMessage(JsonSerializer.SerializeToUtf8Bytes(orderPayload))
{
    MessageId      = orderId,
    CorrelationId  = correlationId,
    ContentType    = "application/json",
    Subject        = "order.created"
};

// Application properties used by SQL filters
message.ApplicationProperties["OrderTotal"]      = order.TotalAmount;
message.ApplicationProperties["Currency"]        = order.Currency;
message.ApplicationProperties["ShippingCountry"] = order.ShippingAddress.Country;
message.ApplicationProperties["Priority"]        = order.IsPriority ? "high" : "normal";

await sender.SendMessageAsync(message);
```

### Dynamic Routing via Azure Function Router

For routing logic that exceeds declarative filter capabilities — for example, routing based on a database lookup, an external API response, or complex multi-step logic:

```csharp
[Function("DynamicEventRouter")]
[ServiceBusOutput("routed-events-{routeTarget}", Connection = "ServiceBus")]
public async Task<ServiceBusMessage> Route(
    [ServiceBusTrigger("inbound-events", Connection = "ServiceBus")] ServiceBusReceivedMessage message)
{
    var eventType   = message.ApplicationProperties["EventType"]?.ToString();
    var tenantId    = message.ApplicationProperties["TenantId"]?.ToString();

    // Dynamic routing decision via configuration service or database
    var routeTarget = await _routingService.ResolveRouteAsync(eventType, tenantId);

    var outbound = new ServiceBusMessage(message.Body)
    {
        CorrelationId = message.CorrelationId,
        Subject       = message.Subject
    };

    foreach (var prop in message.ApplicationProperties)
        outbound.ApplicationProperties[prop.Key] = prop.Value;

    outbound.ApplicationProperties["RoutedTo"] = routeTarget;
    outbound.ApplicationProperties["RoutedAt"] = DateTimeOffset.UtcNow.ToString("o");

    return outbound;
}
```

## Key Considerations

**Event Grid vs. Service Bus:** Event Grid is optimized for event notification (at-most-once to at-least-once delivery, no ordering guarantees, events up to 1 MB). Service Bus is optimized for reliable message delivery (at-least-once, FIFO with sessions, up to 100 MB on Premium, message lock and settlement protocol). Choose based on what your consumers need — notification vs. reliable processing. Many architectures use both: Event Grid routes to Service Bus queues as the endpoint, combining Event Grid's fan-out with Service Bus's delivery reliability.

**Filter Complexity and Maintenance:** SQL filter expressions in Service Bus subscriptions are powerful but operationally invisible — there is no central view of all active filters across all subscriptions. Document filter expressions in source control alongside the infrastructure code that creates them. Test filters explicitly during integration testing, including edge cases (null property values, type mismatches).

**Dead-Letter Handling:** Configure dead-letter queues (DLQ) for all subscriptions. Monitor DLQ depth as an operational health signal. An accumulating DLQ means events are being routed but not successfully processed — this is a routing or consumer bug, not a transient failure. See [Dead-Letter Recovery](Dead-Letter-Recovery.md).

**Ordering:** If consumers require ordered event processing, use Service Bus sessions. Group related events under the same session ID (e.g., `orderId` as the session ID for all events related to a specific order). Event Grid does not guarantee ordering.

**Correlation ID Propagation:** Ensure the correlation ID set at event creation is preserved through all routing hops. In Event Grid, include it in the CloudEvent `data` payload or as an extension attribute. In Service Bus, use the `CorrelationId` property. This ensures distributed traces are complete from producer to consumer.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
