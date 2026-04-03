# Azure Service Bus

## Overview

Azure Service Bus is a fully managed enterprise message broker providing reliable, asynchronous messaging between decoupled components. It supports queues (point-to-point) and topics with subscriptions (publish-subscribe), with configurable delivery guarantees, message ordering, dead-letter handling, and session-based processing. Service Bus is the primary messaging infrastructure for event-driven integration patterns on Azure.

Service Bus operates on the AMQP 1.0 protocol, which provides strong delivery semantics and supports cross-platform clients (.NET, Java, Python, JavaScript, Go). It also provides an HTTP REST interface for environments where AMQP cannot be used.

---

## Key Capabilities

### Queues

Queues implement point-to-point messaging with at-least-once delivery. A message sent to a queue is received by exactly one consumer. Key properties:

| Property | Description |
|---|---|
| `MaxSizeInMegabytes` | Maximum queue storage (1 GB – 80 GB; Premium: up to 80 GB) |
| `DefaultMessageTimeToLive` | TTL for unprocessed messages (default: 14 days) |
| `LockDuration` | How long a received message is locked before becoming available again (max 5 minutes) |
| `MaxDeliveryCount` | Attempts before auto-dead-lettering (default: 10) |
| `RequiresDuplicateDetection` | Reject duplicate messages within the detection window |
| `RequiresSession` | Enable session-based ordered processing |
| `DeadLetteringOnMessageExpiration` | Move expired messages to DLQ |

### Topics and Subscriptions

Topics implement publish-subscribe. A message published to a topic is delivered to every subscription that matches the message. Each subscription acts as an independent queue.

Subscription filters:
- **SQL Filter:** SQL92 expression on message application properties and system properties
- **Correlation Filter:** Equality match on standard message properties (more efficient for simple cases)
- **True Filter:** Accept all messages (default if no filter specified)
- **False Filter:** Accept no messages

See [Publish-Subscribe with Filtering](../patterns/event-driven/Publish-Subscribe-with-Filtering.md) for implementation details.

### Dead-Letter Queue

Every queue and topic subscription has an automatically created dead-letter sub-queue (DLQ), accessible at `{entity}/$deadletterqueue`. Messages are moved to the DLQ when:
- `MaxDeliveryCount` is exceeded
- A consumer explicitly dead-letters a message (`DeadLetterMessageAsync`)
- The message TTL expires (if `DeadLetteringOnMessageExpiration` is enabled)
- A subscription filter evaluation error occurs

DLQ messages include `DeadLetterReason` and `DeadLetterErrorDescription` properties set by Service Bus or the consumer. See [Dead-Letter Recovery](../patterns/event-driven/Dead-Letter-Recovery.md).

### Sessions

Sessions enable FIFO processing for groups of related messages. Messages sharing the same `SessionId` are delivered to the same consumer instance in order. Sessions are required for:
- Processing messages related to the same entity in arrival order (e.g., all events for an order)
- State management per session (session state stored in Service Bus)
- Competing consumers with ordering guarantees per entity

Sessions must be enabled at queue/subscription creation — they cannot be added retroactively.

### Message Properties

| Property | Type | Description |
|---|---|---|
| `MessageId` | string | Unique identifier; used for duplicate detection |
| `CorrelationId` | string | Correlation with other messages or requests |
| `SessionId` | string | Session group key (required if sessions enabled) |
| `ReplyTo` | string | Queue to send replies to (request-reply pattern) |
| `ReplyToSessionId` | string | Session ID for reply routing |
| `Subject` (Label) | string | Application-defined message type/subject |
| `ContentType` | string | MIME type of message body |
| `TimeToLive` | TimeSpan | Message-level TTL override |
| `ScheduledEnqueueTimeUtc` | DateTime | Defer delivery to a future time |
| `ApplicationProperties` | Dictionary | Custom properties (filterable by SQL subscriptions) |

---

## SKU Comparison

| Feature | Standard | Premium |
|---|---|---|
| Max message size | 256 KB | 100 MB |
| Private endpoints | No | Yes |
| VNet service endpoints | Yes | Yes |
| Geo-redundancy | No | Yes (Geo-DR, Active Geo-Replication) |
| Availability zones | No | Yes |
| Customer-managed keys | No | Yes |
| Messaging units | Shared | Dedicated (1, 2, 4, 8, 16) |

For production environments with security requirements (private endpoints, CMK) or large message support, use **Premium**.

---

## Common Use in Integration Patterns

| Pattern | Service Bus Role |
|---|---|
| [Competing Consumers](../patterns/event-driven/Competing-Consumers.md) | Queue with parallel consumer scale-out |
| [Dead-Letter Recovery](../patterns/event-driven/Dead-Letter-Recovery.md) | DLQ monitoring and recovery workflows |
| [Publish-Subscribe with Filtering](../patterns/event-driven/Publish-Subscribe-with-Filtering.md) | Topics with SQL/correlation subscription filters |
| [Event Router](../patterns/event-driven/Event-Router.md) | Topic as fan-out destination from Event Grid |
| [Saga Pattern](../patterns/event-driven/Saga-Pattern.md) | Choreography-based saga message exchange |
| [Secure Hybrid Messaging](../patterns/hybrid/Secure-Hybrid-Messaging.md) | Private endpoint + managed identity + CMK |

---

## Configuration Considerations

### Authentication

Service Bus supports two authentication mechanisms:

1. **Shared Access Signatures (SAS):** Connection string–based authentication. Provides namespace-level or entity-level access. Should be disabled (`disableLocalAuth: true`) in production.
2. **Azure AD (RBAC):** Recommended for all production workloads. Roles:
   - `Azure Service Bus Data Owner` — full access (admin operations)
   - `Azure Service Bus Data Sender` — send messages only
   - `Azure Service Bus Data Receiver` — receive/settle messages only

```bicep
// Assign sender role to Function managed identity
resource senderRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(functionApp.id, queue.id, 'sender')
  scope: queue
  properties: {
    roleDefinitionId: subscriptionResourceId(
      'Microsoft.Authorization/roleDefinitions',
      '69a216fc-b8fb-44d8-bc22-1f3c2cd27a39')  // Service Bus Data Sender
    principalId: functionApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

### Prefetch and Lock Duration Tuning

- **PrefetchCount:** Number of messages fetched locally by the client before the application requests them. Reduces round trips. Set to 2–5× `maxConcurrentCalls` for high-throughput consumers.
- **LockDuration:** Must exceed maximum processing time. Default 60 seconds. Maximum 5 minutes. Use lock renewal for processing that may exceed the lock duration.

### Namespace Configuration (Bicep)

```bicep
resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'sb-integration-prod'
  location: resourceGroup().location
  sku: { name: 'Premium', tier: 'Premium', capacity: 1 }
  properties: {
    publicNetworkAccess: 'Disabled'
    disableLocalAuth:    true
    minimumTlsVersion:   '1.2'
    zoneRedundant:       true
  }
}

resource ordersQueue 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = {
  parent: serviceBusNamespace
  name: 'orders-queue'
  properties: {
    maxSizeInMegabytes:                    5120
    defaultMessageTimeToLive:              'P7D'     // 7 days
    lockDuration:                          'PT1M'    // 1 minute
    maxDeliveryCount:                      5
    requiresDuplicateDetection:            true
    duplicateDetectionHistoryTimeWindow:   'PT10M'
    deadLetteringOnMessageExpiration:      true
    enableBatchedOperations:               true
  }
}
```

### Service Limits

| Limit | Standard | Premium |
|---|---|---|
| Max namespace size | 40 GB | 1 TB per messaging unit |
| Max message size | 256 KB | 100 MB |
| Max concurrent connections | 1,000 | 4,000 per messaging unit |
| Max subscriptions per topic | 2,000 | 2,000 |
| Max SQL filter length | 1,024 characters | 1,024 characters |
| Max message TTL | 14 days | No limit |
| Throughput | Shared | Up to 1 GB/s per messaging unit |

---

*Part of the [Enterprise Integration Patterns](../README.md) library by Cheops Consulting Services.*
