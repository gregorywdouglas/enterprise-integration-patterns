# Exception Handling Standards

## Overview

Distributed integration systems fail in ways that monolithic systems do not: network partitions, downstream service timeouts, message schema mismatches, transient infrastructure throttling, partial writes, and message lock expirations all produce failure conditions that require distinct handling strategies. An ad hoc approach to exception handling — try/catch with a generic log statement — is insufficient at enterprise scale. It produces inconsistent error messages for consumers, unreliable retry behavior, silent message loss, and operations teams who cannot diagnose incidents from log data.

This document defines the canonical exception handling framework for all integration patterns in this library: how faults are classified, how retry policies are configured, how dead-letter routing works, what alerts are required, and how circuit breaking is implemented.

---

## Fault Classification

Every caught exception must be classified before determining the handling strategy. Two primary categories:

### Transient Faults
Temporary conditions that are expected to resolve without intervention:
- Network timeouts (HTTP 429, 408, 503, 502)
- Azure service throttling (Service Bus 429, Cosmos DB 429, APIM 429)
- Transient database connection failures
- Lock expiry on Service Bus messages (redelivery, not an application error)

**Handling:** Retry with exponential backoff. Do not alert immediately. Log at Warning level.

### Permanent Faults
Conditions that will not resolve through retry and require intervention:
- Message deserialization failure (malformed payload)
- Schema version mismatch (consumer cannot process message format)
- Business rule violation (invalid order state, insufficient permissions)
- Authentication failure (expired certificate, invalid token)
- Downstream service returning 400 (client error from the integration's perspective)

**Handling:** Dead-letter the message (for async) or return error response (for sync). Alert if volumes exceed threshold. Log at Error level with full context.

---

## Retry Policy Configurations

### Service Bus Consumer — Retry via Message Lock Renewal and Redelivery

For Service Bus consumers (Azure Functions with Service Bus trigger), retry behavior is controlled by:

1. **Lock renewal:** Extend the message lock if processing takes longer than `lockDuration`
2. **Abandon:** Return the message to the queue for redelivery (increments `DeliveryCount`)
3. **MaxDeliveryCount:** After N abandonments, Service Bus auto-dead-letters the message

```json
// host.json: Service Bus retry behavior
{
  "version": "2.0",
  "extensions": {
    "serviceBus": {
      "prefetchCount": 10,
      "messageHandlerOptions": {
        "maxConcurrentCalls":     16,
        "autoComplete":          false,
        "maxAutoRenewDuration":  "00:05:00"
      }
    }
  }
}
```

```csharp
// Consumer: explicit retry classification
[Function("ProcessOrder")]
public async Task Run(
    [ServiceBusTrigger("orders-queue", Connection = "ServiceBus")]
    ServiceBusReceivedMessage message,
    ServiceBusMessageActions messageActions)
{
    try
    {
        await ProcessMessageAsync(message);
        await messageActions.CompleteMessageAsync(message);
    }
    catch (SchemaException ex)
    {
        // Permanent: dead-letter immediately, do not retry
        _logger.LogError(ex, "Schema mismatch for message {MessageId}", message.MessageId);
        await messageActions.DeadLetterMessageAsync(message,
            deadLetterReason:           "SchemaVersionMismatch",
            deadLetterErrorDescription: ex.Message);
    }
    catch (HttpRequestException ex) when (IsTransient(ex))
    {
        // Transient: abandon for retry (Service Bus will redeliver)
        _logger.LogWarning(ex, "Transient failure on message {MessageId} (delivery {Count})",
            message.MessageId, message.DeliveryCount);
        await messageActions.AbandonMessageAsync(message,
            new Dictionary<string, object>
            {
                ["LastError"]      = ex.Message,
                ["LastAttemptAt"]  = DateTimeOffset.UtcNow.ToString("o"),
                ["DeliveryCount"]  = message.DeliveryCount
            });
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Unexpected failure on message {MessageId}", message.MessageId);
        // Abandon if delivery count is below threshold; dead-letter if at limit
        if (message.DeliveryCount < 3)
            await messageActions.AbandonMessageAsync(message);
        else
            await messageActions.DeadLetterMessageAsync(message,
                deadLetterReason:           "MaxRetriesExceeded",
                deadLetterErrorDescription: ex.Message);
    }
}

private static bool IsTransient(HttpRequestException ex)
    => ex.StatusCode is HttpStatusCode.TooManyRequests
                     or HttpStatusCode.ServiceUnavailable
                     or HttpStatusCode.GatewayTimeout
                     or HttpStatusCode.BadGateway;
```

### HTTP Client — Retry with Polly

For outbound HTTP calls in Functions or API services, use Polly via `AddPolicyHandler` on `HttpClientFactory`:

```csharp
// Program.cs: configure HTTP retry and circuit breaker for downstream API clients
services.AddHttpClient("orders-api", client =>
{
    client.BaseAddress = new Uri(configuration["OrdersApi:BaseUrl"]!);
    client.Timeout     = TimeSpan.FromSeconds(30);
})
.AddPolicyHandler(GetRetryPolicy())
.AddPolicyHandler(GetCircuitBreakerPolicy());

static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()           // 429, 5xx, network errors
        .OrResult(r => r.StatusCode == HttpStatusCode.RequestTimeout)
        .WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: attempt =>
                TimeSpan.FromSeconds(Math.Pow(2, attempt))    // 2s, 4s, 8s
                + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 500)),  // Jitter
            onRetry: (outcome, timespan, attempt, context) =>
            {
                Log.Warning("Retry {Attempt} for {Operation} after {Delay}ms. Reason: {StatusCode}",
                    attempt, context.OperationKey, timespan.TotalMilliseconds,
                    outcome.Result?.StatusCode);
            });
}

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30),
            onBreak:    (outcome, duration) =>
                Log.Error("Circuit breaker opened for {Duration}s. Last error: {Error}",
                    duration.TotalSeconds, outcome.Exception?.Message ?? outcome.Result?.StatusCode.ToString()),
            onReset:    () => Log.Information("Circuit breaker reset"),
            onHalfOpen: () => Log.Information("Circuit breaker in half-open state"));
}
```

### APIM Retry Policy

```xml
<!-- APIM: retry backend calls on transient failure -->
<backend>
    <retry condition="@(context.Response.StatusCode == 429 || context.Response.StatusCode == 503 || context.Response.StatusCode == 502)"
           count="3"
           interval="5"
           max-interval="30"
           delta="5"
           first-fast-retry="false">
        <forward-request timeout="30" />
    </retry>
</backend>
```

### Durable Functions — Activity Retry Options

```csharp
// Orchestrator: retry activity functions on transient failure
var retryOptions = new RetryOptions(
    firstRetryInterval:  TimeSpan.FromSeconds(5),
    maxNumberOfAttempts: 3)
{
    BackoffCoefficient = 2.0,
    MaxRetryInterval   = TimeSpan.FromSeconds(30),
    Handle             = ex => ex is HttpRequestException or TimeoutException
};

var result = await context.CallActivityWithRetryAsync<string>(
    "CallDownstreamService", retryOptions, input);
```

---

## Dead-Letter Routing Standards

### Reason Code Taxonomy

All dead-lettered messages must use one of these standard reason codes (set as `DeadLetterReason` on the Service Bus message):

| Reason Code | When to Use |
|---|---|
| `DeserializationFailed` | Message body cannot be deserialized to expected type |
| `SchemaVersionMismatch` | Message schema version is incompatible with consumer |
| `ValidationFailed` | Message content fails business validation rules |
| `BusinessRuleViolation` | Content is valid but violates business logic (e.g., duplicate order) |
| `ProcessingFailed` | General processing failure after exhausting retries |
| `MaxRetriesExceeded` | Delivery count exceeded without resolution |
| `AuthenticationFailed` | Cannot authenticate to downstream service |
| `DownstreamUnavailable` | Downstream service consistently unavailable after retries |
| `PoisonMessage` | Message structure causes consistent consumer crash |

### DLQ Monitoring

Monitor DLQ depth as a first-class operational metric:

```csharp
// Timer-triggered Function: monitor DLQ depth across all queues
[Function("DLQDepthMonitor")]
public async Task Run([TimerTrigger("0 */5 * * * *")] TimerInfo timer)
{
    var adminClient = new ServiceBusAdministrationClient(
        $"{_namespace}.servicebus.windows.net",
        new DefaultAzureCredential());

    var queues = new[] { "orders-queue", "payments-queue", "inventory-queue" };

    foreach (var queueName in queues)
    {
        var props = await adminClient.GetQueueRuntimePropertiesAsync(queueName);
        var dlqCount = props.Value.DeadLetterMessageCount;

        _telemetryClient.TrackMetric($"DLQ.{queueName}.Count", dlqCount,
            new Dictionary<string, string> { ["queue"] = queueName });

        if (dlqCount > _warningThreshold)
            _logger.LogWarning("DLQ warning: {Queue} has {Count} dead-lettered messages (threshold: {Threshold})",
                queueName, dlqCount, _warningThreshold);
    }
}
```

---

## Alert Thresholds

Configure the following alerts in Azure Monitor:

| Signal | Warning Threshold | Critical Threshold | Action |
|---|---|---|---|
| DLQ message count (per queue) | > 5 | > 50 | Page on-call integration team |
| Function execution failure rate | > 1% | > 5% | Page on-call team |
| APIM backend error rate (5xx) | > 0.5% | > 2% | Page API operations team |
| Service Bus active message count | > 1,000 | > 10,000 | Scale consumer; alert team |
| Service Bus namespace throttled requests | Any | > 100/min | Review throughput; scale Premium MU |
| Azure OpenAI 429 rate (AI trigger) | > 10/min | > 50/min | Reduce consumer concurrency |
| Orchestration (Durable Functions) failure count | > 5/hour | > 20/hour | Alert and investigate |

### Alert Rule (Bicep)

```bicep
resource dlqAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'dlq-depth-warning-orders-queue'
  properties: {
    severity:            2  // Warning
    enabled:             true
    scopes:              [serviceBusNamespace.id]
    evaluationFrequency: 'PT5M'
    windowSize:          'PT5M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [{
        name:            'DLQDepth'
        metricName:      'DeadletteredMessages'
        metricNamespace: 'Microsoft.ServiceBus/namespaces'
        operator:        'GreaterThan'
        threshold:       5
        timeAggregation: 'Maximum'
        dimensions: [{
          name:     'EntityName'
          operator: 'Include'
          values:   ['orders-queue']
        }]
      }]
    }
    actions: [{ actionGroupId: integrationOpsActionGroup.id }]
  }
}
```

---

## Canonical Error Response Format

All integration APIs exposed through APIM must return errors in a consistent envelope format. Define this in the APIM global `on-error` policy:

```json
{
  "error": {
    "code":          "UPSTREAM_TIMEOUT",
    "message":       "The request could not be completed in the allotted time",
    "correlationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "timestamp":     "2025-01-15T10:30:00Z",
    "details":       []
  }
}
```

```xml
<!-- APIM global on-error policy: canonical error response -->
<on-error>
    <base />
    <set-status code="@(context.Response.StatusCode == 0 ? 500 : context.Response.StatusCode)"
                reason="@(context.Response.StatusReason)" />
    <set-header name="Content-Type" exists-action="override">
        <value>application/json</value>
    </set-header>
    <set-header name="X-Correlation-ID" exists-action="override">
        <value>@(context.Request.Headers.GetValueOrDefault("X-Correlation-ID", Guid.NewGuid().ToString()))</value>
    </set-header>
    <set-body>@{
        var errorCode = context.LastError.Source == "backend"
            ? "UPSTREAM_ERROR"
            : context.LastError.Source == "policy"
            ? "POLICY_ERROR"
            : "GATEWAY_ERROR";

        return new JObject(
            new JProperty("error", new JObject(
                new JProperty("code",          errorCode),
                new JProperty("message",       context.LastError.Message),
                new JProperty("correlationId", context.Request.Headers.GetValueOrDefault("X-Correlation-ID", "")),
                new JProperty("timestamp",     DateTimeOffset.UtcNow)
            ))
        ).ToString();
    }</set-body>
</on-error>
```

---

## Circuit Breaking for APIM Backends

APIM does not natively implement circuit breaking, but you can approximate it using a combination of the `retry` policy, `forward-request` timeouts, and a backend health check Function:

```xml
<!-- APIM: circuit breaker approximation via cached health flag -->
<inbound>
    <base />
    <!-- Check cached circuit state (set by health check Function) -->
    <cache-lookup-value key="@($"circuit-{context.Api.Id}")" variable-name="circuitOpen" />
    <choose>
        <when condition="@(context.Variables.GetValueOrDefault<bool>("circuitOpen", false))">
            <return-response>
                <set-status code="503" reason="Service Unavailable" />
                <set-body>{"error":{"code":"CIRCUIT_OPEN","message":"Backend temporarily unavailable"}}</set-body>
            </return-response>
        </when>
    </choose>
</inbound>
```

For robust circuit breaking, use Azure Front Door or Azure Application Gateway as the load balancer in front of APIM backends, which provides native health probe–based circuit breaking.

---

## Exception Logging Standards

Every caught exception must be logged with the following properties as structured fields (not embedded in the message string):

| Property | Description |
|---|---|
| `correlationId` | Current operation's correlation ID |
| `messageId` | Service Bus message ID (for async consumers) |
| `exceptionType` | Full exception type name |
| `exceptionMessage` | Exception message |
| `stackTrace` | Stack trace (Error level only; Warning may omit) |
| `faultClassification` | `transient` or `permanent` |
| `deliveryCount` | Service Bus delivery count (async consumers) |
| `operation` | Current operation name |

```csharp
_logger.LogError(ex,
    "Permanent failure processing message {MessageId} for correlation {CorrelationId}. " +
    "Classification: {FaultClassification}. DeliveryCount: {DeliveryCount}",
    message.MessageId,
    correlationId,
    "permanent",
    message.DeliveryCount);
```

---

*Part of the [Enterprise Integration Patterns](../README.md) library by Cheops Consulting Services.*
