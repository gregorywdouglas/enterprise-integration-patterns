# Backend for Frontend (BFF)

## Problem

A single general-purpose API designed to serve all consumer types — mobile applications, browser-based single-page applications, partner integrations, internal dashboards — inevitably satisfies none of them well. Mobile clients operating on constrained bandwidth and intermittent connectivity need compact, denormalized payloads optimized for specific screens. Web applications need richer datasets to populate complex UIs. Partner systems need stable, versioned contracts with predictable schemas. Internal tools need operational data fields that should never be exposed externally.

When a single API tries to serve all these audiences, one of two failure modes occurs: the API grows to the union of all consumer needs (bloated payloads, excessive fields), or consumers receive only the intersection of their needs (under-serving every consumer). Changing the API to satisfy a new requirement from one consumer type risks breaking others. The result is a release coordination bottleneck that slows delivery for all teams.

## Solution

The Backend for Frontend (BFF) pattern creates dedicated API surfaces tailored to each distinct consumer type. Each BFF is a thin adaptation layer that transforms, filters, and aggregates data from shared backend services into the exact shape each consumer needs. BFFs are owned and deployed by the teams that build the corresponding consumer applications — the mobile team owns the mobile BFF, the web team owns the web BFF.

On Azure, BFFs are most commonly implemented as:
- Azure Functions (HTTP-triggered, isolated worker) for lightweight, stateless adaptation
- Azure Container Apps for heavier BFF implementations requiring persistent connections or complex middleware
- Azure API Management Products for policy-based consumer-type differentiation without separate compute

Each BFF is exposed through a dedicated APIM product or API with its own versioning, rate limits, and subscription tier. Shared backend services remain unchanged and serve all BFFs through stable internal contracts.

## When to Use

- Use when two or more consumer types have meaningfully different data shape, field, or interaction requirements for the same underlying domain.
- Use when mobile and web teams are releasing independently and need to evolve their API contracts without coordinating with each other or with backend teams.
- Use when you need to enforce strict field-level data exposure differences between internal and external consumers.
- Use when consumer-specific aggregation logic is complex enough that encoding it in APIM policy XML would be unmaintainable.
- Avoid when all consumers genuinely need identical data and the differentiation is only cosmetic — a single well-designed API with optional fields is preferable.
- Avoid when the organization cannot sustain the operational overhead of maintaining multiple BFF deployments with separate CI/CD pipelines and monitoring.
- Avoid using BFF as a backdoor to bypass centralized API governance — each BFF must still be fronted by APIM.

## Azure Implementation

### Architecture Overview

```
Mobile App      → APIM (mobile product) → Mobile BFF (Function)  → Backend APIs
Web SPA         → APIM (web product)    → Web BFF (Function)     → Backend APIs
Partner System  → APIM (partner product)→ Partner BFF (Function) → Backend APIs
```

All BFFs share access to the same backend services but produce different response shapes. APIM enforces authentication, rate limiting, and correlation ID injection for all three products. Each BFF Function handles data transformation and aggregation.

### Mobile BFF — Compact Payload Optimized for Screen

```csharp
// Mobile BFF: returns minimal order list for order history screen
[Function("GetMobileOrderHistory")]
public async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "orders")]
    HttpRequest req)
{
    var userId = req.HttpContext.User.FindFirst("sub")?.Value;
    var orders = await _ordersService.GetOrdersAsync(userId);

    // Mobile only needs: order ID, date, total, status — not full line items
    var mobilePayload = orders.Select(o => new
    {
        id      = o.OrderId,
        date    = o.CreatedAt.ToString("yyyy-MM-dd"),
        total   = o.TotalAmount,
        status  = o.StatusCode,
        canCancel = o.StatusCode is "PENDING" or "PROCESSING"
    });

    return new OkObjectResult(new { orders = mobilePayload });
}
```

### Web BFF — Rich Payload with Line Items and Metadata

```csharp
// Web BFF: returns full order detail including line items for order management UI
[Function("GetWebOrderHistory")]
public async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "orders")]
    HttpRequest req)
{
    var userId = req.HttpContext.User.FindFirst("sub")?.Value;

    // Web BFF performs aggregation across orders + inventory + customer services
    var ordersTask   = _ordersService.GetOrdersWithLineItemsAsync(userId);
    var customerTask = _customerService.GetPreferencesAsync(userId);

    await Task.WhenAll(ordersTask, customerTask);

    return new OkObjectResult(new
    {
        customer = customerTask.Result,
        orders   = ordersTask.Result,
        metadata = new
        {
            lastUpdated = DateTimeOffset.UtcNow,
            totalOrders = ordersTask.Result.Count
        }
    });
}
```

### Partner BFF — Versioned, Schema-Stable Contract

Partner integrations require long-lived, stable schemas. The Partner BFF exposes a versioned contract and performs explicit field mapping to decouple the partner schema from internal domain model evolution:

```csharp
// Partner BFF: explicit field mapping to v2 partner contract schema
[Function("GetPartnerOrders")]
public async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "v2/orders")]
    HttpRequest req)
{
    var partnerId = req.HttpContext.User.FindFirst("partner_id")?.Value;
    var orders = await _ordersService.GetOrdersByPartnerAsync(partnerId);

    // Explicit mapping to partner v2 contract — insulates partner from internal model changes
    var partnerOrders = orders.Select(o => new PartnerOrderV2
    {
        OrderReference  = o.OrderId,
        PlacedOn        = o.CreatedAt.ToUniversalTime(),
        GrandTotal      = new MonetaryAmount { Amount = o.TotalAmount, Currency = o.Currency },
        OrderStatus     = MapToPartnerStatus(o.StatusCode),
        LineItems       = o.Items.Select(i => new PartnerLineItemV2
        {
            Sku       = i.ProductSku,
            Quantity  = i.Quantity,
            UnitPrice = new MonetaryAmount { Amount = i.UnitPrice, Currency = o.Currency }
        }).ToList()
    });

    return new OkObjectResult(new { orders = partnerOrders });
}
```

### APIM Product Configuration per BFF

Each consumer type gets a dedicated APIM Product with its own rate limits, JWT audience validation, and subscription approval workflow:

```xml
<!-- Mobile product policy: tighter rate limits, mobile-specific auth -->
<inbound>
    <base />
    <rate-limit calls="300" renewal-period="60" />
    <validate-jwt header-name="Authorization">
        <openid-config url="https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration" />
        <audiences><audience>api://mobile-bff</audience></audiences>
    </validate-jwt>
    <set-backend-service base-url="https://mobile-bff-func.azurewebsites.net/api" />
</inbound>
```

### Shared Backend Service Authentication

BFF Functions authenticate to shared backend services using their system-assigned managed identity. No shared secrets or API keys are passed between BFF and backend:

```bicep
// Grant mobile BFF Function role on Orders API backend
resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(mobileBffFunc.id, ordersApiScope)
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '{custom-orders-reader-role-id}')
    principalId: mobileBffFunc.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

## Key Considerations

**Ownership Model:** The value of BFF is tightly coupled to the team ownership model. If the mobile team does not own the mobile BFF (and its deployment pipeline), the BFF becomes another shared bottleneck rather than an enabler. Make the ownership contract explicit before committing to the pattern.

**Code Duplication vs. Shared Library:** Multiple BFFs will share logic for calling the same backend services. Resist the urge to create a heavyweight shared library that all BFFs depend on — this recreates the tight coupling the BFF pattern is meant to eliminate. Instead, share thin HTTP client configurations and authentication helpers. Accept some duplication in the adaptation layer; it is the price of independent deployability.

**Authentication Consistency:** Each BFF validates its own JWT audience, but the issuer and tenant should be the same for all BFFs unless there is a specific reason for separate identity providers. Centralizing identity configuration through APIM's `validate-jwt` policy applied at the product level reduces duplication.

**Versioning:** Partner BFFs require explicit API versioning (URI versioning: `/v1/`, `/v2/`) with documented deprecation timelines. Mobile and web BFFs can use more aggressive versioning strategies since the team controls both the BFF and the consumer application. See [Versioning and Deprecation](Versioning-and-Deprecation.md).

**Operational Overhead:** Three BFF Functions means three deployment pipelines, three monitoring configurations, and three sets of Application Insights dashboards. Use an Infrastructure-as-Code template (Bicep module) that standardizes BFF deployment and monitoring configuration — parameterize only what genuinely differs between consumer types.

**Avoid BFF Sprawl:** The BFF pattern works for two to five distinct consumer types. If you find yourself creating a BFF per screen or per feature, the architecture has drifted into microservice sprawl for APIs. Consolidate BFFs by consumer persona, not by UI screen.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
