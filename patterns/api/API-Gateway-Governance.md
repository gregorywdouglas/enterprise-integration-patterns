# API Gateway Governance

## Problem

In organizations with multiple development teams exposing APIs, each team tends to solve common cross-cutting concerns independently — implementing their own authentication mechanisms, applying inconsistent rate limits, logging in idiosyncratic formats, and omitting distributed tracing entirely. The result is an API estate where security posture is uneven, operational visibility is fragmented, and consumer experience varies unpredictably across APIs owned by different teams.

This becomes a critical risk when: a high-traffic consumer exhausts backend capacity without any throttling safeguard; a security audit reveals that some APIs accept unauthenticated requests; or a production incident cannot be traced across service boundaries because correlation identifiers were not propagated. Retrofitting governance controls after the fact is expensive and operationally disruptive.

## Solution

API Gateway Governance centralizes cross-cutting API concerns in Azure API Management through a layered policy architecture. All APIs entering or leaving the platform traverse a shared gateway where authentication, authorization, rate limiting, correlation ID propagation, and observability are enforced by policy — not by individual API implementation teams. Teams own the business logic of their APIs; the platform owns the operational envelope.

The governance model operates at three levels in APIM:

1. **Global (All APIs) policy** — Applies to every request through the gateway. Enforces baseline authentication, injects/validates correlation IDs, emits standardized diagnostic logs.
2. **Product policy** — Applies to all APIs within a product. Enforces consumer-tier-specific rate limits and quota.
3. **API / Operation policy** — Applies specific transformations, backend routing, or additional authorization for individual APIs or operations.

Policies at lower levels inherit and extend higher-level policies via the `<base />` directive. This inheritance model is the key to consistent enforcement: if a team forgets to add rate limiting to a new API operation, the product-level policy still applies it.

## When to Use

- Use as the baseline for any organization operating more than two or three APIs on a shared infrastructure — the governance overhead is justified almost immediately.
- Use when multiple development teams share an API platform and you need to enforce consistent security and operational standards without requiring each team to implement them independently.
- Use when compliance or audit requirements mandate demonstrable access control, logging, and rate limiting across all APIs.
- Avoid using a single monolithic APIM instance for governance across completely unrelated business domains with radically different security requirements — consider whether a second APIM instance with separate policy inheritance is more appropriate.

## Azure Implementation

### Global Policy — Baseline Governance

The global policy applies to every request. Configure it conservatively — it should enforce only what truly must apply everywhere.

```xml
<!-- Global scope: All APIs -->
<policies>
    <inbound>
        <!-- Inject correlation ID if not present -->
        <set-header name="X-Correlation-ID" exists-action="skip">
            <value>@(Guid.NewGuid().ToString())</value>
        </set-header>
        <!-- Propagate W3C traceparent if present -->
        <set-header name="traceparent" exists-action="skip">
            <value>@($"00-{Guid.NewGuid():N}{Guid.NewGuid().ToString("N").Substring(0,16)}-01")</value>
        </set-header>
        <!-- Remove internal headers from inbound consumer requests -->
        <set-header name="X-Internal-Secret" exists-action="delete" />
    </inbound>
    <backend>
        <forward-request timeout="30" />
    </backend>
    <outbound>
        <!-- Expose correlation ID to consumer in response -->
        <set-header name="X-Correlation-ID" exists-action="override">
            <value>@(context.Request.Headers.GetValueOrDefault("X-Correlation-ID", ""))</value>
        </set-header>
        <!-- Remove internal response headers -->
        <set-header name="X-Powered-By" exists-action="delete" />
        <set-header name="Server" exists-action="delete" />
    </outbound>
    <on-error>
        <base />
        <set-header name="X-Correlation-ID" exists-action="override">
            <value>@(context.Request.Headers.GetValueOrDefault("X-Correlation-ID", ""))</value>
        </set-header>
        <set-body>@{
            return new JObject(
                new JProperty("error", context.LastError.Message),
                new JProperty("correlationId", context.Request.Headers.GetValueOrDefault("X-Correlation-ID", "")),
                new JProperty("timestamp", DateTimeOffset.UtcNow)
            ).ToString();
        }</set-body>
    </on-error>
</policies>
```

### Product Policy — Rate Limiting and Quota by Consumer Tier

Define APIM Products corresponding to consumer tiers (e.g., Free, Standard, Enterprise). Apply rate limits and quotas at the product level:

```xml
<!-- Product scope: Standard Tier -->
<policies>
    <inbound>
        <base />
        <!-- Validate subscription key -->
        <rate-limit calls="100" renewal-period="60" />
        <quota calls="50000" renewal-period="86400" />
        <!-- Require valid JWT from Azure AD -->
        <validate-jwt header-name="Authorization" failed-validation-httpcode="401" failed-validation-error-message="Unauthorized">
            <openid-config url="https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration" />
            <audiences>
                <audience>api://your-platform-api</audience>
            </audiences>
            <required-claims>
                <claim name="scp" match="any">
                    <value>api.read</value>
                    <value>api.write</value>
                </claim>
            </required-claims>
        </validate-jwt>
    </inbound>
    <outbound>
        <base />
    </outbound>
</policies>
```

### API-Level Policy — Additional Authorization and Observability

```xml
<!-- API scope: Orders API -->
<policies>
    <inbound>
        <base />
        <!-- Extract caller identity for downstream propagation -->
        <set-header name="X-Consumer-Id" exists-action="override">
            <value>@(context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt()?.Claims.GetValueOrDefault("sub", "unknown"))</value>
        </set-header>
        <!-- Scope-based authorization: write operations require additional scope -->
        <choose>
            <when condition="@(context.Request.Method == "POST" || context.Request.Method == "PUT" || context.Request.Method == "DELETE")">
                <set-variable name="requiredScope" value="orders.write" />
                <choose>
                    <when condition="@(!context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt()?.Claims["scp"].Contains("orders.write") ?? true)">
                        <return-response>
                            <set-status code="403" reason="Forbidden" />
                            <set-body>{"error": "Insufficient scope for write operations"}</set-body>
                        </return-response>
                    </when>
                </choose>
            </when>
        </choose>
    </inbound>
    <outbound>
        <base />
    </outbound>
</policies>
```

### Observability: Application Insights Integration

Configure APIM diagnostics to emit structured logs to Application Insights:

```json
// APIM diagnostic settings (ARM/Bicep)
{
  "loggerId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ApiManagement/service/{apim}/loggers/appinsights-logger",
  "alwaysLog": "allErrors",
  "verbosity": "information",
  "sampling": {
    "samplingType": "fixed",
    "percentage": 100
  },
  "frontend": {
    "request": {
      "headers": ["X-Correlation-ID", "X-Consumer-Id", "traceparent"],
      "body": { "bytes": 0 }
    },
    "response": {
      "headers": ["X-Correlation-ID", "Content-Type"],
      "body": { "bytes": 0 }
    }
  },
  "backend": {
    "request": {
      "headers": ["X-Correlation-ID", "traceparent"],
      "body": { "bytes": 0 }
    },
    "response": {
      "headers": ["X-Correlation-ID"],
      "body": { "bytes": 0 }
    }
  }
}
```

### Bicep Snippet: APIM Named Value backed by Key Vault

```bicep
resource namedValue 'Microsoft.ApiManagement/service/namedValues@2023-03-01-preview' = {
  name: '${apimService.name}/backend-api-key'
  properties: {
    displayName: 'backend-api-key'
    secret: true
    keyVault: {
      secretIdentifier: 'https://${keyVaultName}.vault.azure.net/secrets/backend-api-key'
      identityClientId: apimIdentity.properties.clientId
    }
    tags: ['backend', 'credentials']
  }
}
```

## Key Considerations

**Policy Inheritance Discipline:** Enforce the `<base />` convention strictly. A policy that omits `<base />` silently bypasses all higher-level policies, breaking governance invariants. Use APIM policy linting (available via the Azure portal or ARM API) and include policy validation in CI/CD pipelines.

**Rate Limit Accuracy:** APIM rate limiting operates per gateway node when multiple scale units are deployed. With multiple units, a consumer may exceed limits by a factor equal to the number of units before the limit is enforced. For strict enforcement, use rate-limit-by-key with a counter key that hashes to a consistent gateway node, or implement rate limiting in a centralized cache (Redis) via an Azure Function called from APIM policy.

**JWT Validation Caching:** APIM caches OIDC discovery documents and signing keys. If you rotate signing keys, allow up to 1 hour for APIM to pick up the new keys via its cache refresh interval. Plan key rotations accordingly.

**Consumer Identity Propagation:** Extract the consumer identity (subscription key, JWT subject or object ID) at the gateway and propagate it as a standardized header (`X-Consumer-Id`) to all backend services. This enables per-consumer logging and auditing downstream without each service implementing JWT parsing independently.

**Governance Drift Detection:** APIM policies are configuration, not code, and they can be edited manually in the portal by anyone with Contributor access. Implement Defender for APIs or use an APIM backup/export job in CI to detect unauthorized policy changes. Treat all policy files as code: store in source control, deploy via pipeline, alert on direct portal edits.

**Developer Portal:** Enable the APIM Developer Portal and populate it with API documentation, try-it consoles, and product descriptions. Governance friction is reduced when consumers can self-serve onboarding rather than requiring manual subscription approvals and documentation requests.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
