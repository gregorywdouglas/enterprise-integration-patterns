# Azure Event Grid

## Overview

Azure Event Grid is a managed event routing service that delivers events from sources (publishers) to handlers (subscribers) using a push model. It is designed for high-throughput, low-latency event notification rather than reliable message queuing. Event Grid is the integration hub for Azure platform events (resource state changes, Blob Storage operations, IoT Hub messages) and custom application events.

Event Grid's primary value is its native integration with Azure services as event sources and handlers, enabling event-driven workflows with minimal configuration. It is not a replacement for Service Bus — the two services are complementary and are often used together (Event Grid routes to Service Bus as the handler).

---

## Key Capabilities

### Event Schema

Event Grid supports two schemas:

**Event Grid Schema** (legacy, Azure-specific):
```json
{
  "id":          "event-12345",
  "topic":       "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{account}",
  "subject":     "/blobServices/default/containers/orders/blobs/order-001.json",
  "eventType":   "Microsoft.Storage.BlobCreated",
  "eventTime":   "2025-01-15T10:00:00Z",
  "dataVersion": "1.0",
  "data": {
    "api":             "PutBlob",
    "blobType":        "BlockBlob",
    "contentType":     "application/json",
    "contentLength":   1024,
    "url":             "https://..."
  }
}
```

**CloudEvents 1.0 Schema** (recommended for new implementations):
```json
{
  "specversion": "1.0",
  "id":          "event-12345",
  "source":      "/orders-service",
  "type":        "com.yourplatform.order.created",
  "subject":     "orders/order-001",
  "time":        "2025-01-15T10:00:00Z",
  "datacontenttype": "application/json",
  "data": {
    "orderId":   "order-001",
    "customerId": "cust-456"
  }
}
```

Use CloudEvents schema for new custom topic implementations. CloudEvents is a CNCF standard and supported by all major cloud providers and messaging systems, enabling cross-platform event exchange.

### Topics

**System Topics** are created by Azure for events emitted by Azure services. They do not require separate creation — you create subscriptions against existing system topics for a given resource.

Supported system topic sources include:
- Azure Blob Storage (BlobCreated, BlobDeleted, BlobRenamed)
- Azure Container Registry (ImagePushed, ImageDeleted)
- Azure Event Hubs (CaptureFileCreated)
- Azure IoT Hub (DeviceCreated, DeviceConnected, DeviceTelemetry)
- Azure Resource Manager (ResourceWriteSuccess, ResourceDeleteSuccess)
- Azure Service Bus (ActiveMessagesAvailable, DeadletterMessagesAvailable)
- Azure SignalR, Azure Maps, Azure Machine Learning, and others

**Custom Topics** receive events from custom application code. Applications publish events via an HTTPS POST to the topic endpoint with an authentication key or Managed Identity token.

**Domain Topics** are a grouping mechanism for large-scale custom topic deployments (thousands of topics) under a single Event Grid Domain resource. Each tenant or entity gets its own topic within the domain.

### Event Subscriptions

Subscriptions define the handler (destination) and optional filters for events from a topic.

**Supported handlers:**
| Handler | Notes |
|---|---|
| Azure Function | Direct invocation; best for compute-intensive handlers |
| Logic App | HTTP trigger; use for workflow-based handling |
| Service Bus Queue | Reliable delivery; use for durable processing |
| Service Bus Topic | Fan-out to multiple subscribers |
| Storage Queue | Simple durable handler |
| Event Hub | High-throughput streaming; use for analytics pipelines |
| Webhook (HTTPS) | Any HTTP endpoint; includes validation handshake |
| Hybrid Connection | On-premises relay via Service Bus Relay |

For event-driven processing requiring reliable delivery, dead-letter handling, or complex retry logic, route Event Grid events to **Service Bus** as the handler — not directly to Functions or webhooks.

### Event Filtering

Each subscription supports filtering to receive only matching events:

```bicep
resource subscription 'Microsoft.EventGrid/topics/eventSubscriptions@2023-12-15-preview' = {
  parent: customTopic
  name: 'high-priority-orders'
  properties: {
    filter: {
      includedEventTypes: ['com.yourplatform.order.created']
      subjectBeginsWith:  '/orders/priority/'
      advancedFilters: [
        {
          operatorType: 'NumberGreaterThan'
          key:          'data.totalAmount'
          value:        10000
        }
        {
          operatorType: 'StringIn'
          key:          'data.region'
          values:       ['US-EAST', 'US-WEST']
        }
      ]
      enableAdvancedFilteringOnArrays: true
    }
  }
}
```

Advanced filter operators: `NumberIn`, `NumberNotIn`, `NumberLessThan`, `NumberGreaterThan`, `NumberLessThanOrEquals`, `NumberGreaterThanOrEquals`, `BoolEquals`, `StringIn`, `StringNotIn`, `StringBeginsWith`, `StringNotBeginsWith`, `StringEndsWith`, `StringNotEndsWith`, `StringContains`, `StringNotContains`, `IsNullOrUndefined`, `IsNotNull`.

### Delivery and Retry

Event Grid retries delivery with an exponential backoff schedule:
- Immediate
- 10 seconds
- 30 seconds
- 1 minute
- 5 minutes
- 10 minutes
- 30 minutes
- 1 hour (repeating up to TTL)

Default event TTL: 24 hours. Maximum retry period: 24 hours. Configure per-subscription:

```bicep
retryPolicy: {
  maxDeliveryAttempts:      30
  eventTimeToLiveInMinutes: 1440  // 24 hours
}
```

### Dead-Letter Configuration

Events that exceed the retry policy are routed to a dead-letter storage blob container:

```bicep
deadLetterDestination: {
  endpointType: 'StorageBlob'
  properties: {
    resourceId:       storageAccount.id
    blobContainerName: 'event-grid-deadletter'
  }
}
```

Monitor dead-lettered events by reading the blob container or by setting up Event Grid monitoring alerts on the `DeadLetteredCount` metric.

---

## Common Use in Integration Patterns

| Pattern | Event Grid Role |
|---|---|
| [Event Router](../patterns/event-driven/Event-Router.md) | Primary event routing for Azure service events and custom topics |
| [Hybrid Data Synchronization](../patterns/hybrid/Hybrid-Data-Synchronization.md) | Trigger sync workflows from storage or service events |
| [Event-Driven AI Trigger](../patterns/event-driven/Event-Driven-AI-Trigger.md) | Route document upload events to AI trigger functions |
| [Dead-Letter Recovery](../patterns/event-driven/Dead-Letter-Recovery.md) | DLQ blob storage for undeliverable events |

---

## Configuration Considerations

### Custom Topic Publishing Authentication

For custom topics, use Managed Identity authentication (not SAS keys) when publishing from Azure-hosted producers:

```csharp
// Publish with managed identity (no API key required)
var credential = new DefaultAzureCredential();
var client     = new EventGridPublisherClient(
    new Uri("https://your-topic.eastus-1.eventgrid.azure.net/api/events"),
    credential);

await client.SendEventAsync(new CloudEvent(
    source: "/orders-service",
    type:   "com.yourplatform.order.created",
    data:   orderData));
```

### Private Networking

Event Grid supports private endpoints (Public Network Access disabled) for custom topics. Webhook handlers must be publicly reachable HTTPS endpoints or use the Hybrid Connection handler for on-premises endpoints.

### Event Grid vs. Service Bus Decision Guide

| Scenario | Use Event Grid | Use Service Bus |
|---|---|---|
| Azure service event notifications | ✓ | |
| Push-based delivery to HTTPS endpoints | ✓ | |
| Message size > 1 MB | | ✓ |
| FIFO / ordered processing required | | ✓ |
| At-least-once with settlement protocol | | ✓ |
| Rich SQL filter expressions | | ✓ |
| Message sessions | | ✓ |
| Long TTL (> 24 hours) | | ✓ |
| Request-reply pattern | | ✓ |
| High-volume fan-out (millions/second) | ✓ | |

### Service Limits

| Limit | Value |
|---|---|
| Max event size | 1 MB |
| Max events per publish request | 1 MB total |
| Max subscriptions per topic | 500 |
| Max advanced filters per subscription | 25 |
| Event TTL | 24 hours |
| Throughput (custom topic) | 10 MB/s (scalable) |

---

*Part of the [Enterprise Integration Patterns](../README.md) library by Cheops Consulting Services.*
