# Azure API Management

## Overview

Azure API Management (APIM) is a managed API gateway service that provides a unified entry point for APIs across an organization. It separates the API contract (what consumers see and call) from the backend implementation (what actually processes the request), enabling centralized governance of authentication, authorization, rate limiting, transformation, caching, and observability for all APIs without requiring changes to backend services.

APIM is a foundational component in enterprise integration architectures on Azure. Nearly every integration pattern in this library that involves synchronous API communication routes through or is governed by APIM.

---

## Key Capabilities

### Gateway

The gateway is the runtime component that processes API requests. It:
- Evaluates inbound policies (authentication, transformation, rate limiting)
- Routes requests to configured backends
- Evaluates outbound policies (response transformation, header injection)
- Handles errors via on-error policies
- Emits telemetry to configured loggers (Application Insights, Azure Monitor)

Multiple gateway types are available:
- **Managed gateway:** Hosted in Azure, maintained by Microsoft. Scales automatically within the region.
- **Self-hosted gateway:** Deployed as a container on Kubernetes, on-premises or in other clouds. Syncs configuration from the Azure APIM control plane. See [Self-Hosted Gateway pattern](../patterns/hybrid/Self-Hosted-Gateway.md).
- **Workspace gateway (preview):** Dedicated gateway resources within a single APIM instance for multi-team isolation.

### Policy Engine

APIM's policy engine is the core differentiator. Policies are XML documents that execute at four points in the request lifecycle:

- `<inbound>` — Before the request reaches the backend
- `<backend>` — During backend request forwarding
- `<outbound>` — After the backend response is received
- `<on-error>` — When a policy or backend error occurs

Policies at different scopes (global, product, API, operation) form an inheritance hierarchy via the `<base />` directive.

Commonly used policies:
| Policy | Purpose |
|---|---|
| `validate-jwt` | Validate OAuth 2.0 / OpenID Connect tokens |
| `rate-limit` / `rate-limit-by-key` | Throttle requests per consumer |
| `quota` | Enforce call volume limits per subscription period |
| `set-header` | Add, modify, or remove request/response headers |
| `set-body` | Transform request or response body (supports C# expressions) |
| `rewrite-uri` | Modify the request URI before backend forwarding |
| `send-request` | Issue additional HTTP requests (fan-out, webhook notification) |
| `cache-lookup` / `cache-store` | Cache backend responses |
| `retry` | Retry failed backend calls with backoff |
| `forward-request` | Forward the request to the configured backend |
| `return-response` | Short-circuit the pipeline and return a custom response |
| `choose` | Conditional logic (if/when/otherwise) |
| `set-variable` | Store computed values for use later in the pipeline |

### Developer Portal

The APIM Developer Portal is an auto-generated, customizable website where:
- API consumers discover available APIs and products
- Consumers self-service subscribe to products
- Consumers test APIs interactively (try-it console)
- Consumers access API documentation (OpenAPI specification)
- Administrators manage subscriptions and approvals

The portal is Gatsby-based and fully customizable. For enterprise deployments, configure the portal with organizational branding, custom domain, and custom pages.

### Subscriptions and Products

**Products** are containers of APIs with associated policies (rate limits, quota, approval requirements). Consumers subscribe to products, not directly to APIs.

**Subscriptions** represent a consumer's access grant to a product. Each subscription generates one or two subscription keys. Subscription key validation is enforced by the `subscription-required` property on the API or product.

### Named Values

Named Values are key-value pairs (optionally secret) that can be referenced in policies using the `{{named-value-key}}` syntax. They externalize configuration from policy XML and support:
- **Plain values:** Visible in the portal and API
- **Secret values:** Encrypted at rest, masked in UI
- **Key Vault references:** Secret stored in Azure Key Vault; APIM fetches and caches the value using its managed identity

Use Named Values for all externalized configuration — backend URLs, API keys, feature flags — and back secrets with Key Vault references.

---

## Common Use in Integration Patterns

| Pattern | APIM Role |
|---|---|
| [Composite API](../patterns/api/Composite-API.md) | Front-end contract; routes to composition function |
| [API Abstraction Layer](../patterns/api/API-Abstraction-Layer.md) | Protocol/schema transformation via policies |
| [API Gateway Governance](../patterns/api/API-Gateway-Governance.md) | Central governance enforcement via policy hierarchy |
| [Backend for Frontend](../patterns/api/Backend-for-Frontend.md) | Product per consumer type; independent rate limits |
| [Versioning and Deprecation](../patterns/api/Versioning-and-Deprecation.md) | API Version Sets; deprecation headers via policy |
| [Self-Hosted Gateway](../patterns/hybrid/Self-Hosted-Gateway.md) | Control plane for SHG configuration sync |
| [Legacy Modernization Bridge](../patterns/hybrid/Legacy-Modernization-Bridge.md) | REST facade over legacy SOAP/WCF endpoints |

---

## Configuration Considerations

### SKU Selection

| SKU | VNet Integration | Private Endpoints | Self-Hosted Gateway | Scale Units | Use Case |
|---|---|---|---|---|---|
| Developer | None | No | 1 | 1 | Development/testing only |
| Basic | None | No | No | 2 | Low-traffic non-production |
| Standard | External VNet | No | 10 | 4 | Production without VNet isolation |
| Premium | Internal/External VNet | Yes | Unlimited | Unlimited | Enterprise production |
| Standard v2 | VNet injection | Yes | No | Auto | Production with VNet, simplified management |

For enterprise integration with private endpoints and VNet integration, use **Premium** or **Standard v2**.

### Managed Identity

Enable system-assigned managed identity on APIM to:
- Fetch secrets from Azure Key Vault for Named Values
- Authenticate to backend services using managed identity (for Azure-native backends)
- Authenticate to Azure Monitor and Application Insights

```bicep
resource apimService 'Microsoft.ApiManagement/service@2023-03-01-preview' = {
  name: 'apim-integration-prod'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    publisherEmail: 'platform-team@yourorg.com'
    publisherName:  'Platform Engineering'
    sku: {
      name:     'Premium'
      capacity: 1
    }
    virtualNetworkType: 'Internal'
    virtualNetworkConfiguration: {
      subnetResourceId: '${vnet.id}/subnets/snet-apim'
    }
  }
}
```

### Application Insights Integration

```bicep
resource apimLogger 'Microsoft.ApiManagement/service/loggers@2023-03-01-preview' = {
  parent: apimService
  name: 'appinsights-logger'
  properties: {
    loggerType: 'applicationInsights'
    credentials: {
      instrumentationKey: appInsights.properties.InstrumentationKey
    }
    resourceId: appInsights.id
  }
}

resource apimDiagnostic 'Microsoft.ApiManagement/service/diagnostics@2023-03-01-preview' = {
  parent: apimService
  name: 'applicationinsights'
  properties: {
    loggerId: apimLogger.id
    alwaysLog: 'allErrors'
    verbosity: 'information'
    sampling: {
      samplingType: 'fixed'
      percentage:   100
    }
  }
}
```

### Service Limits

| Limit | Value |
|---|---|
| Max request body size | 1 MB (configurable up to 2 GB with streaming) |
| Max policy execution time | 60 seconds |
| Max `send-request` body | 1 MB |
| Max concurrent connections per unit | 2,500 (Premium) |
| Max APIs per service | 50 (Developer), 250 (Standard), 10,000 (Premium) |
| Max Named Values | 1,000 |

---

*Part of the [Enterprise Integration Patterns](../README.md) library by Cheops Consulting Services.*
