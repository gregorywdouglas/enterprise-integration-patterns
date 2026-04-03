# OpenTelemetry Integration

## Overview

OpenTelemetry (OTel) is the CNCF-standard framework for distributed observability — providing APIs, SDKs, and tooling for generating, collecting, and exporting traces, metrics, and logs from distributed systems. It replaces proprietary vendor SDKs (including the Azure Application Insights SDK's direct instrumentation) with a vendor-neutral layer that can export to any backend (Application Insights, Jaeger, Prometheus, Datadog, Grafana Tempo).

For Azure integration architectures, OpenTelemetry integration enables:
- End-to-end distributed traces spanning APIM, Azure Functions, Logic Apps, and Service Bus
- Consistent span naming across all services
- Automatic context propagation via W3C Trace Context headers
- Exportation to Application Insights without taking a hard dependency on the App Insights SDK

This document defines instrumentation guidance, span naming conventions, and exporter configuration for all services in the integration platform.

---

## Core Concepts

**Trace:** A collection of spans that represent the complete path of a single request or operation through a distributed system.

**Span:** A unit of work within a trace. Each span has a name, start time, duration, status, and optional attributes. Spans are nested — a root span (e.g., APIM request) has child spans (Function execution, Service Bus send).

**Context propagation:** The mechanism by which trace context (trace ID, span ID) is carried between services. W3C Trace Context (`traceparent` header) is the standard propagation format.

**Exporter:** Sends OTel telemetry to a backend. For Azure, the Azure Monitor exporter sends to Application Insights.

**Collector:** An intermediate agent (OpenTelemetry Collector) that receives telemetry from services, processes it (batch, filter, transform), and exports to one or more backends.

---

## SDK Setup: Azure Functions (.NET 8 Isolated)

```csharp
// Program.cs: configure OpenTelemetry for Functions isolated worker
using Azure.Monitor.OpenTelemetry.AspNetCore;
using OpenTelemetry;
using OpenTelemetry.Trace;
using OpenTelemetry.Metrics;

var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults()
    .ConfigureServices((context, services) =>
    {
        services.AddOpenTelemetry()
            .WithTracing(tracing =>
            {
                tracing
                    .SetResourceBuilder(ResourceBuilder
                        .CreateDefault()
                        .AddService(
                            serviceName:    "order-processing-function",
                            serviceVersion: "1.2.0",
                            serviceInstanceId: Environment.MachineName))
                    .AddSource("OrderProcessing")     // Custom activity source
                    .AddAspNetCoreInstrumentation()
                    .AddHttpClientInstrumentation(opts =>
                    {
                        opts.RecordException = true;
                        // Enrich span with correlation ID from outbound request
                        opts.EnrichWithHttpRequestMessage = (activity, request) =>
                        {
                            if (request.Headers.TryGetValues("X-Correlation-ID", out var values))
                                activity.SetTag("correlationId", values.First());
                        };
                    })
                    .AddAzureMonitorTraceExporter(opts =>
                    {
                        opts.ConnectionString = context.Configuration["APPLICATIONINSIGHTS_CONNECTION_STRING"];
                    });
            })
            .WithMetrics(metrics =>
            {
                metrics
                    .AddMeter("OrderProcessing")
                    .AddHttpClientInstrumentation()
                    .AddAzureMonitorMetricExporter(opts =>
                    {
                        opts.ConnectionString = context.Configuration["APPLICATIONINSIGHTS_CONNECTION_STRING"];
                    });
            });
    })
    .Build();

await host.RunAsync();
```

### Custom Activity Source

```csharp
// Define a static ActivitySource for custom instrumentation
public static class Telemetry
{
    public static readonly ActivitySource Source = new("OrderProcessing", "1.0.0");
}

// Use in function code
[Function("ProcessOrder")]
public async Task Run(
    [ServiceBusTrigger("orders-queue", Connection = "ServiceBus")]
    ServiceBusReceivedMessage message,
    ServiceBusMessageActions messageActions)
{
    // Restore W3C Trace Context from message (propagation from publisher)
    var propagator = Propagators.DefaultTextMapPropagator;
    var parentContext = propagator.Extract(
        default,
        message.ApplicationProperties,
        (props, key) => props.ContainsKey(key) ? new[] { props[key].ToString()! } : Array.Empty<string>());

    Baggage.Current = parentContext.Baggage;

    using var activity = Telemetry.Source.StartActivity(
        "ProcessOrder",
        ActivityKind.Consumer,
        parentContext.ActivityContext);

    var correlationId = message.CorrelationId ?? Guid.NewGuid().ToString();

    activity?.SetTag("messaging.system",          "azure_service_bus");
    activity?.SetTag("messaging.destination",     "orders-queue");
    activity?.SetTag("messaging.message_id",      message.MessageId);
    activity?.SetTag("messaging.operation",       "receive");
    activity?.SetTag("correlationId",             correlationId);
    activity?.SetTag("order.id",                  /* extracted from payload */);

    try
    {
        await ProcessOrderAsync(message, correlationId);
        await messageActions.CompleteMessageAsync(message);

        activity?.SetStatus(ActivityStatusCode.Ok);
    }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        activity?.RecordException(ex);
        throw;
    }
}
```

---

## SDK Setup: ASP.NET Core API (BFF or Integration Service)

```csharp
// Program.cs: configure OTel for ASP.NET Core service
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .SetResourceBuilder(ResourceBuilder.CreateDefault()
                .AddService("mobile-bff", serviceVersion: "2.1.0")
                .AddAttributes(new Dictionary<string, object>
                {
                    ["deployment.environment"] = builder.Environment.EnvironmentName,
                    ["cloud.provider"]         = "azure",
                    ["cloud.region"]           = Environment.GetEnvironmentVariable("REGION") ?? "unknown"
                }))
            .AddAspNetCoreInstrumentation(opts =>
            {
                opts.RecordException = true;
                opts.EnrichWithHttpRequest = (activity, request) =>
                {
                    activity.SetTag("correlationId",
                        request.Headers["X-Correlation-ID"].FirstOrDefault() ?? "");
                    activity.SetTag("http.consumer",
                        request.Headers["X-Consumer-Id"].FirstOrDefault() ?? "");
                };
            })
            .AddHttpClientInstrumentation(opts => opts.RecordException = true)
            .AddAzureMonitorTraceExporter(opts =>
            {
                opts.ConnectionString = builder.Configuration["APPLICATIONINSIGHTS_CONNECTION_STRING"];
            });
    });
```

---

## W3C Trace Context Propagation for Service Bus

When publishing messages to Service Bus, inject the current trace context into message properties:

```csharp
public static class ServiceBusTracingExtensions
{
    private static readonly TextMapPropagator Propagator = Propagators.DefaultTextMapPropagator;

    public static ServiceBusMessage InjectTraceContext(this ServiceBusMessage message)
    {
        // Inject W3C traceparent and tracestate into ApplicationProperties
        Propagator.Inject(
            new PropagationContext(Activity.Current?.Context ?? default, Baggage.Current),
            message.ApplicationProperties,
            (props, key, value) => props[key] = value);

        return message;
    }

    public static PropagationContext ExtractTraceContext(this ServiceBusReceivedMessage message)
    {
        return Propagator.Extract(
            default,
            message.ApplicationProperties,
            (props, key) => props.ContainsKey(key)
                ? new[] { props[key].ToString()! }
                : Array.Empty<string>());
    }
}

// Usage: publish with trace context
var message = new ServiceBusMessage(payload)
{
    MessageId     = eventId,
    CorrelationId = correlationId
}.InjectTraceContext();

await sender.SendMessageAsync(message);
```

---

## APIM: OpenTelemetry Tracing

APIM (Premium and Standard v2) natively exports traces to an OpenTelemetry Collector via OTLP:

```bicep
// APIM diagnostic configuration for OTel export
resource apimOtelDiagnostic 'Microsoft.ApiManagement/service/diagnostics@2023-03-01-preview' = {
  parent: apimService
  name: 'opentelemetry'
  properties: {
    loggerId:    '/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ApiManagement/service/{apim}/loggers/otel-logger'
    alwaysLog:   'allErrors'
    verbosity:   'information'
    sampling: {
      samplingType: 'fixed'
      percentage:   100
    }
    frontend: {
      request:  { headers: ['X-Correlation-ID', 'traceparent', 'User-Agent'] }
      response: { headers: ['X-Correlation-ID', 'Content-Type'] }
    }
    backend: {
      request:  { headers: ['X-Correlation-ID', 'traceparent'] }
      response: { headers: ['X-Correlation-ID'] }
    }
  }
}
```

---

## OpenTelemetry Collector Configuration

Deploy the OTel Collector as a Kubernetes DaemonSet or a standalone container to receive telemetry from all services and export to Application Insights:

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: "0.0.0.0:4317"
      http:
        endpoint: "0.0.0.0:4318"

processors:
  batch:
    timeout:        10s
    send_batch_size: 512
  resource:
    attributes:
      - action: insert
        key:    cloud.provider
        value:  azure

exporters:
  azuremonitor:
    connection_string: "${APPLICATIONINSIGHTS_CONNECTION_STRING}"
    spaneventsenabled: true
  logging:
    verbosity: normal

service:
  pipelines:
    traces:
      receivers:  [otlp]
      processors: [batch, resource]
      exporters:  [azuremonitor]
    metrics:
      receivers:  [otlp]
      processors: [batch]
      exporters:  [azuremonitor]
    logs:
      receivers:  [otlp]
      processors: [batch]
      exporters:  [azuremonitor]
```

---

## Span Naming Conventions

Consistent span naming enables meaningful Application Map visualization in Application Insights and clean trace queries.

### HTTP Server Spans (inbound requests)
Format: `{HTTP Method} {route template}`
Examples:
- `GET /customers/{customerId}/summary`
- `POST /orders`
- `GET /v2/orders/{orderId}`

### HTTP Client Spans (outbound calls)
Format: `{HTTP Method} {host}`
Examples:
- `GET profile-api.internal`
- `POST payments-api.internal`

### Messaging Consumer Spans (Service Bus receive)
Format: `{queue/topic name} receive`
Examples:
- `orders-queue receive`
- `platform-events/inventory-service receive`

### Messaging Producer Spans (Service Bus send)
Format: `{queue/topic name} send`
Examples:
- `orders-queue send`
- `platform-events send`

### Internal/Custom Spans
Format: `{ServiceName}.{OperationName}`
Examples:
- `OrderService.ProcessOrder`
- `InventoryService.ReserveStock`
- `PaymentService.ChargeCard`

---

## Application Insights Queries

```kql
// View complete distributed trace for a correlation ID
union traces, requests, exceptions, dependencies
| where customDimensions["correlationId"] == "3fa85f64-5717-4562-b3fc-2c963f66afa6"
| project timestamp, itemType, name, message, operation_Id, operation_ParentId,
          cloud_RoleName, duration, success
| order by timestamp asc

// Identify slow spans across all services
dependencies
| where timestamp > ago(1h)
| where duration > 1000  // > 1 second
| summarize avg(duration), max(duration), count()
    by name, cloud_RoleName, target
| order by avg_duration desc

// Error rate by service and operation
requests
| where timestamp > ago(1h)
| summarize TotalRequests = count(),
            FailedRequests = countif(success == false),
            ErrorRate = todouble(countif(success == false)) / count() * 100
    by cloud_RoleName, name
| order by ErrorRate desc
```

---

*Part of the [Enterprise Integration Patterns](../README.md) library by Cheops Consulting Services.*
