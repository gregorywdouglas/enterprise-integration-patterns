# Azure Functions

## Overview

Azure Functions is a serverless compute service that executes event-driven code on demand without provisioning or managing infrastructure. Functions are the primary compute layer in event-driven integration architectures — they process messages from Service Bus, respond to Event Grid events, implement orchestration workflows via Durable Functions, and serve as lightweight API backends behind APIM.

Functions supports .NET (isolated worker model), Java, Python, Node.js, PowerShell, and custom handlers. For enterprise integration on Azure, .NET isolated worker model (targeting .NET 8 LTS) is the recommended implementation language for performance, tooling maturity, and SDK support.

---

## Key Capabilities

### Triggers

Functions execute in response to triggers. Common integration triggers:

| Trigger | Use Case |
|---|---|
| **Service Bus Trigger** | Process messages from queues or topic subscriptions |
| **Event Grid Trigger** | Handle Event Grid event deliveries |
| **Event Hub Trigger** | Process high-throughput event streams |
| **HTTP Trigger** | Implement API backends callable from APIM |
| **Timer Trigger** | Scheduled tasks (monitoring, cleanup, reports) |
| **Cosmos DB Trigger** | React to Cosmos DB Change Feed |
| **Blob Storage Trigger** | Process files on blob creation/modification |
| **Durable Orchestration Trigger** | Start or signal Durable Functions orchestrations |

### Isolated Worker Model (.NET 8)

The isolated worker model runs Function code in a separate process from the Functions host. Benefits over the in-process model:
- Full control of .NET version (not tied to host runtime)
- Standard ASP.NET Core dependency injection and middleware
- Better performance isolation
- Support for native AOT compilation (preview)

```csharp
// Program.cs — isolated worker model host configuration
var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults(worker =>
    {
        worker.UseMiddleware<CorrelationIdMiddleware>();
        worker.UseMiddleware<ExceptionHandlingMiddleware>();
    })
    .ConfigureServices((context, services) =>
    {
        services.AddSingleton<IEventPublisher, ServiceBusEventPublisher>();
        services.AddHttpClient("orders-api", client =>
        {
            client.BaseAddress = new Uri(context.Configuration["OrdersApi:BaseUrl"]!);
            client.DefaultRequestHeaders.Add("X-API-Version", "2");
        });
        services.AddAzureClients(builder =>
        {
            builder.AddServiceBusClient(context.Configuration["ServiceBus:ConnectionString"]);
            builder.UseCredential(new DefaultAzureCredential());
        });
    })
    .Build();

await host.RunAsync();
```

### Durable Functions

Durable Functions extends Azure Functions with stateful workflow orchestration using the virtual actor pattern. The orchestrator function defines the workflow; activity functions perform individual steps. State is persisted automatically in Azure Storage (default) or Azure SQL / Cosmos DB (configurable).

Three entity types:
- **Orchestrator functions:** Define the workflow logic. Must be deterministic. Cannot perform I/O directly.
- **Activity functions:** Perform actual work (call APIs, read/write data). Callable from orchestrators with retry.
- **Entity functions:** Represent stateful entities (virtual actors) that can be called from orchestrators or external clients.

Durable Functions is used in:
- [Request-Reply with Compensation](../patterns/api/Request-Reply-with-Compensation.md) — orchestrate compensation steps
- [Saga Pattern](../patterns/event-driven/Saga-Pattern.md) — orchestrate long-running business transactions
- [Dead-Letter Recovery](../patterns/event-driven/Dead-Letter-Recovery.md) — orchestrate DLQ inspection and replay workflows

### Hosting Plans

| Plan | Cold Start | Max Duration | VNet Integration | Scale |
|---|---|---|---|---|
| **Consumption** | Yes | 10 minutes | No | Auto (0–200 instances) |
| **Flex Consumption** | Reduced | Unlimited | Yes | Auto with pre-provisioned instances |
| **Premium** | No | Unlimited | Yes (VNet integration) | Auto (min–max instances) |
| **Dedicated (App Service)** | No | Unlimited | Yes | Manual or auto |
| **Container Apps** | Configurable | Unlimited | Yes | KEDA-based auto |

For enterprise integration requiring VNet connectivity (private endpoints) and no cold starts, use **Premium** or **Flex Consumption** plan. For high-throughput, container-based deployments, **Container Apps** with KEDA scaling is increasingly common.

---

## Common Use in Integration Patterns

| Pattern | Functions Role |
|---|---|
| [Composite API](../patterns/api/Composite-API.md) | Fan-out and aggregation behind APIM |
| [Competing Consumers](../patterns/event-driven/Competing-Consumers.md) | Service Bus–triggered consumer at scale |
| [Dead-Letter Recovery](../patterns/event-driven/Dead-Letter-Recovery.md) | DLQ monitor, replay orchestrator, recovery activities |
| [Saga Pattern](../patterns/event-driven/Saga-Pattern.md) | Durable Functions orchestrator and activities |
| [Request-Reply with Compensation](../patterns/api/Request-Reply-with-Compensation.md) | Durable Functions orchestrator with compensation |
| [Event-Driven AI Trigger](../patterns/event-driven/Event-Driven-AI-Trigger.md) | Service Bus–triggered AI inference and result publishing |
| [Hybrid Data Synchronization](../patterns/hybrid/Hybrid-Data-Synchronization.md) | CDC event consumer, cloud target writer |

---

## Configuration Considerations

### Managed Identity for Downstream Service Access

Always use managed identity to access Azure services from Functions. Never use connection strings or API keys in application settings:

```bicep
resource functionApp 'Microsoft.Web/sites@2023-01-01' = {
  name: 'func-integration-prod'
  kind: 'functionapp'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      appSettings: [
        // Service Bus connection using managed identity (no connection string)
        { name: 'ServiceBus__fullyQualifiedNamespace'
          value: '${serviceBusNamespace.name}.servicebus.windows.net' }
        { name: 'APPLICATIONINSIGHTS_CONNECTION_STRING'
          value: appInsights.properties.ConnectionString }
      ]
    }
  }
}
```

### host.json — Concurrency and Retry

```json
{
  "version": "2.0",
  "extensions": {
    "serviceBus": {
      "prefetchCount": 10,
      "messageHandlerOptions": {
        "maxConcurrentCalls": 16,
        "autoComplete":       false,
        "maxAutoRenewDuration": "00:05:00"
      }
    }
  },
  "retry": {
    "strategy":          "exponentialBackoff",
    "maxRetryCount":     5,
    "minimumInterval":   "00:00:02",
    "maximumInterval":   "00:00:30"
  },
  "logging": {
    "logLevel": {
      "default":                    "Warning",
      "Function":                   "Information",
      "Microsoft.Azure.Functions":  "Warning"
    }
  }
}
```

### VNet Integration (Premium Plan)

```bicep
resource networkConfig 'Microsoft.Web/sites/networkConfig@2023-01-01' = {
  parent: functionApp
  name: 'virtualNetwork'
  properties: {
    subnetResourceId: '${vnet.id}/subnets/snet-functions'
    swiftSupported:   true
  }
}
```

With VNet integration enabled, the Function can reach private endpoints for Service Bus, Storage, Cosmos DB, and Key Vault without traversing the public internet.

### Service Limits

| Limit | Consumption | Premium |
|---|---|---|
| Max execution timeout | 10 minutes | Unlimited |
| Max instances | 200 | 100 per plan |
| Max HTTP request size | 100 MB | 100 MB |
| Max concurrent connections per instance | 600 (dynamic) | 600 (dynamic) |
| Durable Functions max history events | ~50,000 | ~50,000 |
| Durable Functions storage operations | Per Azure Storage billing | Per Azure Storage billing |

---

*Part of the [Enterprise Integration Patterns](../README.md) library by Cheops Consulting Services.*
