# Versioning and Deprecation

## Problem

APIs are consumed by clients that evolve at different rates and are controlled by different organizations. A breaking change to an API — a removed field, a changed data type, a renamed endpoint, an altered authentication requirement — may be inconsequential to some consumers and catastrophic to others. Without a disciplined versioning and deprecation strategy, API producers face an impossible choice: either freeze the API in place indefinitely to protect existing consumers, or break consumers when making necessary changes.

In enterprise environments, the problem is compounded by partner and B2B integrations where the consumer is an external organization with its own release cycles, change management processes, and SLA commitments. Breaking a partner integration is not a bug fix — it is a contractual incident.

## Solution

The Versioning and Deprecation pattern establishes explicit lifecycle management for APIs: how new versions are introduced, how old versions are maintained, and how consumers are migrated. It separates the API surface (the contract consumers depend on) from the backend implementation (which can evolve independently).

Three primary versioning strategies are used in practice:

1. **URI path versioning** (`/v1/orders`, `/v2/orders`) — The most explicit and most widely understood approach. Makes the version visible in every request and response. Recommended for partner and external APIs.
2. **Header versioning** (`Api-Version: 2024-01-01`) — Keeps URIs clean; version is expressed as a header. Preferred by some API design standards (Microsoft REST API Guidelines use date-based header versioning). More opaque to consumers unfamiliar with the convention.
3. **Query parameter versioning** (`?api-version=2024-01-01`) — Simple to test in a browser. Can cause caching complications. Common in Azure REST APIs.

In Azure API Management, all three strategies are implementable through API version sets. URI versioning is the default and most operationally straightforward choice.

## When to Use

- Use URI versioning for any API with external or partner consumers who have no control over the request path.
- Use header or query parameter versioning for internal APIs where consumers can be coordinated, and where clean URI namespaces are a priority.
- Introduce a new version whenever a change would break existing consumers: removing or renaming fields, changing data types, altering authentication requirements, removing endpoints.
- Do not version for additive changes: adding optional response fields, adding new endpoints, adding optional request parameters — these are backward-compatible and should not require a version increment.
- Deprecate a version when the successor has been available long enough for consumer migration (minimum 6 months for internal consumers; 12–18 months for external/partner consumers is typical).

## Azure Implementation

### APIM API Version Set

Create an APIM Version Set to manage multiple versions of the same logical API:

```bicep
resource orderApiVersionSet 'Microsoft.ApiManagement/service/apiVersionSets@2023-03-01-preview' = {
  parent: apimService
  name: 'orders-api-version-set'
  properties: {
    displayName: 'Orders API'
    versioningScheme: 'Segment'  // URI path versioning: /v1, /v2
    description: 'Order management API versioned by URI segment'
  }
}

resource ordersApiV1 'Microsoft.ApiManagement/service/apis@2023-03-01-preview' = {
  parent: apimService
  name: 'orders-api-v1'
  properties: {
    displayName: 'Orders API v1'
    apiVersion: 'v1'
    apiVersionSetId: orderApiVersionSet.id
    path: 'orders'
    protocols: ['https']
    subscriptionRequired: true
  }
}

resource ordersApiV2 'Microsoft.ApiManagement/service/apis@2023-03-01-preview' = {
  parent: apimService
  name: 'orders-api-v2'
  properties: {
    displayName: 'Orders API v2'
    apiVersion: 'v2'
    apiVersionSetId: orderApiVersionSet.id
    path: 'orders'
    protocols: ['https']
    subscriptionRequired: true
  }
}
```

### Deprecation Headers — Communicating Sunset

Inject deprecation notice headers into responses from deprecated API versions. This allows consumers to detect and respond to deprecation notices programmatically:

```xml
<!-- Applied to deprecated v1 API policy -->
<outbound>
    <base />
    <!-- RFC 8594 Sunset header: date after which the API will be decommissioned -->
    <set-header name="Sunset" exists-action="override">
        <value>Sat, 31 Dec 2025 23:59:59 GMT</value>
    </set-header>
    <!-- Deprecation header: indicates this version is deprecated -->
    <set-header name="Deprecation" exists-action="override">
        <value>Tue, 01 Jul 2025 00:00:00 GMT</value>
    </set-header>
    <!-- Link to successor version documentation -->
    <set-header name="Link" exists-action="override">
        <value>&lt;https://developer.yourplatform.com/api/orders/v2&gt;; rel="successor-version"</value>
    </set-header>
</outbound>
```

### Traffic Monitoring for Deprecated Versions

Before decommissioning a deprecated version, verify that consumer traffic has actually migrated. Use APIM's built-in analytics or query Application Insights:

```kql
// Kusto query: requests to deprecated v1 API by consumer over last 30 days
requests
| where timestamp > ago(30d)
| where url contains "/v1/orders"
| summarize RequestCount = count(), LastSeen = max(timestamp)
    by SubscriptionId = tostring(customDimensions["Ocp-Apim-Subscription-Name"])
| order by RequestCount desc
```

This query identifies which subscriptions (consumers) are still calling the deprecated version, enabling targeted outreach before decommission.

### Routing Old Versions to New Backend (Facade Migration)

When the backend changes significantly between versions but the old APIM version needs to remain live during migration, use APIM policy to route v1 calls through a transformation adapter:

```xml
<!-- v1 API policy: transform v1 request format to v2 backend format -->
<inbound>
    <base />
    <!-- Map legacy v1 field names to v2 backend schema -->
    <set-body>@{
        var body     = context.Request.Body.As<JObject>();
        var v2Body   = new JObject();
        // Rename fields from v1 to v2 naming conventions
        v2Body["customerId"]  = body["customer_id"];
        v2Body["productSku"]  = body["sku"];
        v2Body["quantity"]    = body["qty"];
        v2Body["deliveryDate"]= body["requested_delivery"];
        return v2Body.ToString();
    }</set-body>
    <!-- Route to v2 backend -->
    <set-backend-service base-url="https://orders-api-v2.internal" />
</inbound>
<outbound>
    <base />
    <!-- Transform v2 response back to v1 response schema -->
    <set-body>@{
        var v2 = context.Response.Body.As<JObject>();
        return new JObject(
            new JProperty("order_id",   v2["orderId"]),
            new JProperty("status",     v2["orderStatus"]),
            new JProperty("created_at", v2["createdAt"])
        ).ToString();
    }</set-body>
</outbound>
```

### Version Deprecation Communication Workflow

Establish a standard communication workflow executed for every version deprecation:

1. **Announcement (T-12 months for external; T-6 months for internal):** Post to developer portal, send email to all active subscriptions, update API documentation with sunset date.
2. **Header injection (T-12 months):** Add `Sunset` and `Deprecation` headers to all responses from the deprecated version. Monitor consumer alerting for subscription uptake.
3. **Traffic reporting (ongoing):** Weekly or monthly report of subscription traffic on deprecated version, distributed to API owner and consumer relationship owners.
4. **Targeted outreach (T-3 months):** Direct engagement with subscriptions still active on deprecated version. Offer migration assistance.
5. **Soft decommission (T-1 month):** Return HTTP 429 with a `Retry-After` header on a small percentage of calls (APIM policy) to force consumer awareness.
6. **Hard decommission (T=0):** Delete the APIM API version. Confirm all subscriptions have migrated. Archive the version's documentation.

## Key Considerations

**Semantic Versioning vs. Date Versioning:** URI path versions (`v1`, `v2`) are stable identifiers that never change meaning. Date-based versions (`2024-01-01`) can accumulate rapidly and make it difficult to reason about compatibility relationships between versions. For external APIs with infrequent major changes, URI segment versioning is clearer. For APIs following Microsoft REST API Guidelines, date versioning aligns with platform conventions.

**Breaking vs. Non-Breaking Changes:** The most common versioning mistake is over-versioning. Adding an optional response field is not a breaking change and does not require a version increment — it is a backward-compatible extension. Define in writing what constitutes a breaking change for your API program and enforce it in your API review process. Breaking change categories: removing any field, making an optional field required, changing a field data type, changing HTTP method, changing authentication mechanism, changing error response schema.

**APIM Revision vs. Version:** APIM has both Revisions and Versions. Revisions are for non-breaking changes that do not require a new version — useful for testing a policy change before making it current. Versions are for breaking changes. Use them appropriately and do not conflate the two.

**Consumer Inventory:** You cannot deprecate what you cannot inventory. Maintain a registry of all API subscriptions including consumer organization, technical contact, consuming system, and last-seen timestamp. APIM subscription metadata and Application Insights telemetry combined provide this data. Without it, deprecation outreach is a broadcast rather than a targeted engagement.

**Policy Maintenance Cost:** Maintaining transformation policies that adapt old version requests to new backend schemas (the facade migration approach above) has a cost. Each version requires its own policy file, CI/CD deployment, and regression testing. Calculate the maintenance cost against the cost of requiring consumer migration. For small consumer populations, requiring migration is often cheaper than maintaining perpetual translation layers.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
