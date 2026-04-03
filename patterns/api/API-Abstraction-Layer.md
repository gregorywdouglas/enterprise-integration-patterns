# API Abstraction Layer

## Problem

Enterprise backend systems rarely expose interfaces that are suitable for direct consumption by external clients or even internal teams outside the owning domain. Backends carry legacy naming conventions, technology-specific response formats (SOAP, EDI, flat files), authentication mechanisms tied to on-premises infrastructure, and schemas that reflect internal data models rather than consumer-useful contracts. When consumers integrate directly against these backends, every backend change — a renamed field, a refactored endpoint, a migrated data store — risks breaking consumer integrations. The cost of coordinated change across dozens of consumer teams is high, and the risk of regression is substantial.

Additionally, backends often provide more data than any single consumer needs, creating security risk (over-exposure of sensitive fields) and performance cost (large payloads traversing constrained networks).

## Solution

The API Abstraction Layer (AAL) pattern interposes a managed API gateway — Azure API Management — between all consumers and all backends. APIM acts as a stable, versioned contract surface that consumers depend on. The underlying backend can change topology, protocol, schema, and location without affecting consumers, as long as the APIM layer handles the translation.

Key capabilities implemented in the APIM layer:

- **Protocol transformation:** Convert inbound REST to backend SOAP, convert responses back to JSON
- **Schema transformation:** Map backend field names to consumer-facing names using APIM policy
- **Field filtering:** Strip sensitive or irrelevant fields from responses before returning to consumers
- **Backend routing:** Direct requests to different backends based on consumer identity, environment, or request content
- **Credential mediation:** Accept OAuth tokens from consumers, exchange for backend-specific credentials

## When to Use

- Use when backends expose protocols, schemas, or authentication mechanisms inappropriate for direct consumer access.
- Use when you need to insulate consumers from planned or ongoing backend modernization or migration.
- Use when multiple backends provide semantically equivalent data and you want to present a unified interface to consumers.
- Use when you must enforce field-level access control (different consumers see different subsets of the same resource).
- Avoid when the abstraction layer would need to contain substantial business logic — that belongs in a dedicated API or service layer, not in APIM policy.
- Avoid when backends are already well-designed consumer APIs and the abstraction layer would add latency and operational complexity without meaningful benefit.

## Azure Implementation

### Policy-Driven Protocol Transformation (REST to SOAP)

A common enterprise scenario: consumer calls a RESTful JSON endpoint exposed in APIM; the backend is a legacy SOAP/WCF service.

```xml
<!-- APIM inbound: transform REST request to SOAP envelope -->
<inbound>
    <base />
    <rewrite-uri template="/CustomerService.svc" />
    <set-header name="Content-Type" exists-action="override">
        <value>text/xml;charset=UTF-8</value>
    </set-header>
    <set-header name="SOAPAction" exists-action="override">
        <value>"http://tempuri.org/ICustomerService/GetCustomer"</value>
    </set-header>
    <set-body>@{
        var customerId = context.Request.MatchedParameters["customerId"];
        return $@"<?xml version=""1.0"" encoding=""utf-8""?>
<soap:Envelope xmlns:soap=""http://schemas.xmlsoap.org/soap/envelope/"">
  <soap:Body>
    <GetCustomer xmlns=""http://tempuri.org/"">
      <customerId>{customerId}</customerId>
    </GetCustomer>
  </soap:Body>
</soap:Envelope>";
    }</set-body>
</inbound>

<!-- APIM outbound: transform SOAP response to JSON -->
<outbound>
    <base />
    <set-header name="Content-Type" exists-action="override">
        <value>application/json</value>
    </set-header>
    <set-body>@{
        var xml = context.Response.Body.As<XDocument>();
        var ns  = XNamespace.Get("http://tempuri.org/");
        var customer = xml.Descendants(ns + "GetCustomerResult").First();
        return new JObject(
            new JProperty("id",       customer.Element(ns + "CustomerId")?.Value),
            new JProperty("name",     customer.Element(ns + "FullName")?.Value),
            new JProperty("email",    customer.Element(ns + "EmailAddress")?.Value),
            new JProperty("status",   customer.Element(ns + "AccountStatus")?.Value)
        ).ToString();
    }</set-body>
</outbound>
```

### Field-Level Filtering by Consumer Identity

Different consumers (e.g., internal operations team vs. external partner) receive different subsets of the same response. APIM subscription keys or JWT claims identify the consumer:

```xml
<outbound>
    <base />
    <choose>
        <when condition="@(!context.User.Groups.Contains("internal-ops"))">
            <!-- Strip PII fields for non-internal consumers -->
            <set-body>@{
                var body = context.Response.Body.As<JObject>();
                body.Remove("ssn");
                body.Remove("dateOfBirth");
                body.Remove("internalAccountNotes");
                return body.ToString();
            }</set-body>
        </when>
    </choose>
</outbound>
```

### Backend Routing by Consumer or Environment

Direct traffic to different backends based on subscription product or environment header:

```xml
<inbound>
    <base />
    <choose>
        <when condition="@(context.Subscription.Name.StartsWith("partner-"))">
            <set-backend-service base-url="https://partner-api.backend.internal" />
        </when>
        <when condition="@(context.Request.Headers.GetValueOrDefault("X-Target-Env", "prod") == "staging")">
            <set-backend-service base-url="https://staging-api.backend.internal" />
        </when>
        <otherwise>
            <set-backend-service base-url="https://prod-api.backend.internal" />
        </otherwise>
    </choose>
</inbound>
```

### Credential Mediation (OAuth to Basic/API Key)

Consumer authenticates with Azure AD OAuth tokens; APIM validates the token and exchanges for a backend credential stored in a Named Value (backed by Key Vault):

```xml
<inbound>
    <base />
    <!-- Validate consumer JWT from Azure AD -->
    <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
        <openid-config url="https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration" />
        <audiences><audience>api://your-api-audience</audience></audiences>
    </validate-jwt>
    <!-- Replace with backend API key from Key Vault-backed Named Value -->
    <set-header name="Authorization" exists-action="override">
        <value>@($"Basic {{{{"backend-api-credentials"}}}}")</value>
    </set-header>
</inbound>
```

### Networking Considerations

Backends called from APIM must be reachable from the APIM subnet. Use private endpoints for Azure-native backends. For on-premises backends, route through ExpressRoute or VPN with APIM deployed in a VNet (Developer tier does not support VNet integration — use Standard v2 or Premium). Use APIM Named Values backed by Azure Key Vault references for all backend credentials — never hardcode secrets in policy XML.

### Backend Health and Circuit Breaking

APIM does not natively provide circuit breaking, but you can approximate it using the `retry` and `forward-request` policies combined with a backend health check. For robust circuit breaking across the abstraction layer, consider fronting the backend with an Azure Application Gateway or integrating with Azure Front Door health probes.

## Key Considerations

**Policy Complexity Creep:** APIM policy XML can become difficult to maintain as transformation logic accumulates. Define a complexity threshold: if a transformation requires more than ~30 lines of C# in `set-body`, extract it to an Azure Function and call it via `send-request`. Keep policy XML focused on routing, header manipulation, and simple field mapping.

**Backend Error Passthrough:** Decide explicitly how backend errors (4xx, 5xx) surface to consumers. Transform backend error schemas to a canonical error envelope at the APIM layer. Do not expose internal error messages, stack traces, or backend-specific error codes directly to consumers.

**Performance:** Every policy execution adds latency. For high-throughput, low-latency APIs, profile the impact of heavy `set-body` transformations. XML/JSON conversions in policy are CPU-bound operations running in the APIM gateway process — they do not scale horizontally without scaling the APIM unit count.

**Schema Drift:** When the backend schema changes, APIM transformation policies must be updated in coordination. Without automated contract testing between APIM and the backend, schema drift will cause silent data loss (mapped fields returning null). Implement APIM policy unit tests using the APIM test console and monitor transformation errors in Application Insights.

**Observability:** Log the backend service name, backend response time, and transformation outcome in APIM's diagnostic logs. This is the only way to distinguish between APIM policy failures and backend failures in production. See [Correlation ID Strategy](../../observability/Correlation-ID-Strategy.md).

**Named Value Hygiene:** Use Named Values for every externalized configuration value (backend URLs, credentials, feature flags). Back all secrets with Azure Key Vault references. Rotate credentials through Key Vault without touching policy XML.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
