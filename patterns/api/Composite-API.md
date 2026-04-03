# Composite API

## Problem

Modern enterprise clients — mobile applications, single-page web applications, partner portals — routinely need data that spans multiple backend services. Without a composition layer, clients must issue multiple sequential or parallel API calls, manage partial failures independently, correlate results from different schemas, and handle different authentication mechanisms per service. This places an unreasonable burden on client developers, creates tight coupling between client code and backend service topology, and produces observable patterns of over-fetching and under-fetching that degrade performance.

At scale, this pattern breaks down further: changes to any one backend service can require coordinated updates across all consumers, network chattiness increases infrastructure cost and latency, and operational visibility becomes fragmented across calls that logically belong to a single user-facing operation.

## Solution

The Composite API pattern introduces a server-side aggregation layer — typically implemented in Azure API Management combined with a backend orchestration function — that fans out to multiple upstream services, collects their responses, and assembles a single normalized payload for the caller. The client makes one request; all fan-out, error handling, correlation, and transformation occurs behind the API surface.

The pattern operates in two primary modes:

- **Sequential composition:** Call B depends on a value returned by call A. Useful when there is a dependency chain (e.g., fetch customer ID, then fetch orders for that customer).
- **Parallel (fan-out) composition:** All upstream calls are independent and can be issued simultaneously. Results are merged when all (or a configurable subset) respond.

An **envelope pattern** is typically used for the response: a standard outer structure carries metadata (request ID, timestamp, component statuses), and each aggregated resource occupies a named field within the body. This allows partial success — returning data from services that responded successfully while indicating which components failed.

## When to Use

- Use when clients consistently need data from two or more backend services to render a single view or complete a single operation.
- Use when you need to shield clients from the internal decomposition of your backend (strangler fig, microservice migration).
- Use when downstream services have different latency profiles and parallel fan-out can hide that latency from the client.
- Use when you want centralized control over backend call retry, timeout, and circuit-breaking behavior.
- Avoid when aggregation logic is highly dynamic or requires complex conditional branching that cannot be expressed cleanly in an orchestration layer — consider a dedicated BFF service instead.
- Avoid when all data is available from a single backend and aggregation adds no value.
- Avoid when response payload assembly is extremely large (>10 MB) and APIM policy memory limits become a constraint — offload composition to a Function or Logic App.

## Azure Implementation

### Primary Composition via Azure API Management + Azure Functions

**APIM** handles the inbound API contract, authentication, rate limiting, and correlation ID injection. **Azure Functions** (isolated worker, .NET 8) performs the actual fan-out using `HttpClientFactory` with named clients configured per upstream service, and assembles the composite response.

```xml
<!-- APIM inbound policy: route composition requests to a dedicated Function -->
<inbound>
    <base />
    <set-header name="X-Correlation-ID" exists-action="skip">
        <value>@(Guid.NewGuid().ToString())</value>
    </set-header>
    <set-backend-service base-url="https://composite-api-func.azurewebsites.net/api" />
    <set-header name="x-functions-key" exists-action="override">
        <value>{{composite-func-key}}</value>
    </set-header>
</inbound>
```

**Azure Function — parallel fan-out with partial success:**

```csharp
[Function("GetCustomerSummary")]
public async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "get", Route = "customers/{customerId}/summary")]
    HttpRequest req, string customerId)
{
    var correlationId = req.Headers["X-Correlation-ID"].FirstOrDefault()
                        ?? Guid.NewGuid().ToString();

    var profileTask = _profileClient.GetAsync($"/customers/{customerId}");
    var ordersTask  = _ordersClient.GetAsync($"/customers/{customerId}/orders?top=5");
    var balanceTask = _billingClient.GetAsync($"/accounts/{customerId}/balance");

    await Task.WhenAll(profileTask, ordersTask, balanceTask);

    var envelope = new CompositeResponse
    {
        RequestId  = correlationId,
        Timestamp  = DateTimeOffset.UtcNow,
        Components = new Dictionary<string, ComponentResult>
        {
            ["profile"]  = await ExtractResult(profileTask),
            ["orders"]   = await ExtractResult(ordersTask),
            ["balance"]  = await ExtractResult(balanceTask)
        }
    };

    return new OkObjectResult(envelope);
}

private async Task<ComponentResult> ExtractResult(Task<HttpResponseMessage> task)
{
    try
    {
        var response = await task;
        if (response.IsSuccessStatusCode)
            return new ComponentResult
            {
                Status = "ok",
                Data   = await response.Content.ReadFromJsonAsync<JsonElement>()
            };
        return new ComponentResult
        {
            Status    = "error",
            ErrorCode = response.StatusCode.ToString()
        };
    }
    catch (Exception ex)
    {
        return new ComponentResult
        {
            Status    = "error",
            ErrorCode = "upstream_timeout",
            Detail    = ex.Message
        };
    }
}
```

### Composition Directly in APIM Policy (Lightweight Aggregation)

For simpler cases with two or three small upstream responses, APIM `send-request` policies can perform fan-out without a separate compute layer. This reduces hops but limits testability and pushes logic into policy XML.

```xml
<inbound>
    <base />
    <send-request mode="new" response-variable-name="profileResponse" timeout="5" ignore-error="true">
        <set-url>@($"https://profile-api.internal/customers/{context.Variables["customerId"]}")</set-url>
        <set-method>GET</set-method>
    </send-request>
    <send-request mode="new" response-variable-name="ordersResponse" timeout="5" ignore-error="true">
        <set-url>@($"https://orders-api.internal/customers/{context.Variables["customerId"]}/orders")</set-url>
        <set-method>GET</set-method>
    </send-request>
</inbound>
<outbound>
    <base />
    <set-body>@{
        var profile = ((IResponse)context.Variables["profileResponse"]).Body.As<JObject>();
        var orders  = ((IResponse)context.Variables["ordersResponse"]).Body.As<JArray>();
        return new JObject(
            new JProperty("profile", profile),
            new JProperty("recentOrders", orders)
        ).ToString();
    }</set-body>
</outbound>
```

### Response Envelope Schema

Define a canonical envelope schema and publish it to the APIM developer portal:

```json
{
  "requestId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "timestamp": "2025-01-15T10:30:00Z",
  "components": {
    "profile": {
      "status": "ok",
      "data": {}
    },
    "orders": {
      "status": "error",
      "errorCode": "upstream_timeout",
      "detail": "orders-api did not respond within 5s"
    }
  }
}
```

## Key Considerations

**Partial Failure Semantics:** Define and document whether the composite API returns HTTP 200 with component-level error indicators, or whether failure of any critical component produces a non-200 response. Clients must understand the contract. A common approach: mandatory components failing → 502/503; optional components failing → 200 with component-level error flags.

**Timeout Budgets:** The composite timeout budget must be less than the client-facing SLA. For parallel fan-out, total wall-clock time is `max(t1, t2, ..., tN)`. Set APIM policy timeouts 500ms–1s below client SLA to allow error assembly time.

**Correlation ID Propagation:** Inject a correlation ID on entry and propagate it as a header to every upstream call. Log it at every hop. This is non-negotiable for debugging composite failures in production. See [Correlation ID Strategy](../../observability/Correlation-ID-Strategy.md).

**Caching:** Component responses may be cacheable even if the composite response is not. Consider component-level caching in the Function using `IMemoryCache` or Redis for high-read, low-change data to reduce upstream load.

**Upstream Service Authentication:** Each upstream service may require different authentication. Use managed identity for Azure-native services, OAuth 2.0 client credentials for OIDC-compatible services, and named values in APIM for legacy shared-secret cases.

**Testing:** Composite APIs require contract testing, not just integration testing. Use WireMock or Azure Functions test hosts to simulate upstream responses — including slow responses, partial failures, and malformed payloads.

**Cost:** Monitor Function execution time and memory under load. A composite function holding multiple open HTTP connections for several seconds generates more cost than simple passthrough functions. Factor this into capacity planning.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
