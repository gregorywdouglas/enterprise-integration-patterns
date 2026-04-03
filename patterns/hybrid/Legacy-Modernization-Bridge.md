# Legacy Modernization Bridge

## Problem

Enterprise organizations operating BizTalk Server, legacy ESB platforms (IBM MQ, TIBCO, MuleSoft on-premises), mainframe-based integration, or custom middleware face a strategic modernization challenge. Full replacement of these systems is risky, expensive, and time-consuming — existing integrations represent years of business logic, transformation rules, and partner agreements embedded in platform-specific configurations. Yet maintaining legacy middleware while simultaneously building new cloud capabilities creates a dual-platform operational burden and limits the ability to adopt modern integration patterns.

The most common failure mode is the "big bang" migration: attempting to migrate all integrations simultaneously from the legacy platform to Azure, which stalls when complexity is underestimated and risks a prolonged parallel operation period that never ends.

## Solution

The Legacy Modernization Bridge wraps legacy integration systems in API facades that present modern, RESTful interfaces to new consumers, enabling incremental migration without full replacement. The legacy system continues to operate as the underlying orchestration engine; the bridge translates between the legacy platform's protocol and the modern API surface.

This creates three migration phases:
1. **Facade phase:** Legacy system runs unchanged; the bridge exposes its capabilities through a modern API. New consumers use the API, not the legacy platform.
2. **Selective migration phase:** Individual integrations are migrated from the legacy platform to Azure (Functions, Logic Apps, Service Bus), one at a time. The facade routes those routes to new implementations; legacy routes for unmigrated integrations remain unchanged.
3. **Decommission phase:** Once all integrations are migrated, the legacy platform is decommissioned. The facade contracts remain unchanged for consumers.

On Azure, the facade is implemented in Azure API Management (exposing the REST contract) and Azure Functions or Logic Apps (handling BizTalk/ESB interaction and protocol translation).

## When to Use

- Use when a legacy integration platform hosts dozens or hundreds of integrations that cannot be migrated simultaneously.
- Use when existing consumers of the legacy platform cannot be changed and require a stable interface contract during migration.
- Use as the organizing principle for any BizTalk Server modernization engagement — avoid direct BizTalk decommission without a facade strategy.
- Avoid when the legacy system hosts fewer than 10–15 integrations — at that scale, direct replacement is usually faster and less risky than implementing a bridge.
- Avoid using the bridge as a long-term architecture — it is a migration vehicle with a planned decommission date, not a permanent pattern.

## Azure Implementation

### BizTalk Facade via APIM + Azure Functions

BizTalk Server exposes integrations through WCF endpoints or HTTP adapters. The bridge translates REST API calls to BizTalk HTTP or WCF calls:

```xml
<!-- APIM: expose modern REST endpoint for legacy BizTalk integration -->
<!-- Inbound: transform REST JSON to BizTalk-compatible XML -->
<inbound>
    <base />
    <set-header name="Content-Type" exists-action="override">
        <value>application/xml</value>
    </set-header>
    <set-body>@{
        var body = context.Request.Body.As<JObject>();
        return $@"<?xml version=""1.0"" encoding=""utf-8""?>
<ns0:OrderRequest xmlns:ns0=""http://YourCompany.BizTalk.Schemas.OrderRequest"">
    <OrderId>{body["orderId"]}</OrderId>
    <CustomerId>{body["customerId"]}</CustomerId>
    <TotalAmount>{body["totalAmount"]}</TotalAmount>
    <Currency>{body["currency"]}</Currency>
    <RequestDate>{DateTimeOffset.UtcNow:O}</RequestDate>
</ns0:OrderRequest>";
    }</set-body>
    <set-backend-service base-url="http://biztalk-server.corp.internal:8080/BizTalkHttpHandler" />
</inbound>
<!-- Outbound: transform BizTalk XML response to REST JSON -->
<outbound>
    <base />
    <set-header name="Content-Type" exists-action="override">
        <value>application/json</value>
    </set-header>
    <set-body>@{
        var xml     = context.Response.Body.As<XDocument>();
        var ns      = XNamespace.Get("http://YourCompany.BizTalk.Schemas.OrderResponse");
        var response = xml.Root;
        return new JObject(
            new JProperty("orderId",     response?.Element(ns + "OrderId")?.Value),
            new JProperty("status",      response?.Element(ns + "Status")?.Value),
            new JProperty("confirmedAt", response?.Element(ns + "ConfirmedDate")?.Value)
        ).ToString();
    }</set-body>
</outbound>
```

### Route Arbitration — Legacy vs. Migrated Implementation

As individual integrations are migrated from BizTalk to Azure, the facade must route calls to the appropriate implementation based on a routing configuration:

```xml
<!-- APIM: conditional routing based on migration status flag in Named Value -->
<inbound>
    <base />
    <choose>
        <when condition="@(context.Variables.GetValueOrDefault<bool>("OrderIntegration.Migrated", false))">
            <!-- Route to new Azure Function implementation -->
            <set-backend-service base-url="https://order-integration-func.azurewebsites.net/api" />
        </when>
        <otherwise>
            <!-- Route to legacy BizTalk endpoint -->
            <set-backend-service base-url="http://biztalk-server.corp.internal:8080/BizTalkHttpHandler" />
            <!-- Apply legacy protocol transformation -->
            <set-header name="Content-Type" exists-action="override">
                <value>application/xml</value>
            </set-header>
        </otherwise>
    </choose>
</inbound>
```

The routing flag (`OrderIntegration.Migrated`) is a Named Value in APIM backed by a feature flag service or a simple Azure App Configuration value, enabling instantaneous cutover without redeployment.

### Azure Function — MQ Bridge (IBM MQ to Azure Service Bus)

For BizTalk/ESB integrations that use IBM MQ internally, a bridge Function translates between MQ and Service Bus:

```csharp
[Function("MQBridgeInbound")]
public async Task Run(
    [ServiceBusTrigger("inbound-queue", Connection = "ServiceBus")]
    ServiceBusReceivedMessage message,
    ServiceBusMessageActions messageActions)
{
    try
    {
        var correlationId = message.CorrelationId ?? Guid.NewGuid().ToString();

        // Convert Service Bus message to MQ message
        using var mqManager       = new MQQueueManager("INTEGRATION.QM");
        using var mqQueue         = mqManager.AccessQueue("LEGACY.ORDER.REQUEST",
            MQC.MQOO_OUTPUT | MQC.MQOO_FAIL_IF_QUIESCING);

        var mqMessage             = new MQMessage();
        mqMessage.CorrelationIdAsString = correlationId;
        mqMessage.Format          = MQC.MQFMT_STRING;
        mqMessage.CharacterSet    = 1208;  // UTF-8

        // Transform Service Bus payload to MQ legacy format
        var payload = JsonSerializer.Deserialize<OrderRequest>(message.Body)!;
        mqMessage.WriteString(TransformToLegacyFormat(payload));

        var putMessageOptions    = new MQPutMessageOptions();
        mqQueue.Put(mqMessage, putMessageOptions);

        await messageActions.CompleteMessageAsync(message);

        _logger.LogInformation("Message {CorrelationId} bridged to IBM MQ successfully", correlationId);
    }
    catch (MQException mqEx)
    {
        _logger.LogError(mqEx, "IBM MQ error {ReasonCode} bridging message", mqEx.ReasonCode);
        await messageActions.AbandonMessageAsync(message,
            new Dictionary<string, object> { ["MQReasonCode"] = mqEx.ReasonCode });
    }
}
```

### Mainframe CICS/IMS Bridge via Azure Logic Apps

For mainframe-hosted integrations, Azure Logic Apps with the IBM 3270 connector or the IBM Host Files connector can call CICS transactions or read mainframe datasets:

```json
{
  "type": "ApiConnection",
  "inputs": {
    "host": {
      "connection": { "name": "@parameters('$connections')['ibm3270']['connectionId']" }
    },
    "method": "post",
    "path": "/ExecuteQuery",
    "body": {
      "TN3270Credentials": {
        "Server": "mainframe.corp.internal",
        "Port": 23,
        "LogonType": "ExplicitCredentials",
        "LogonName": "SVCLOGAP",
        "LogonPassword": "@parameters('mainframePassword')"
      },
      "Actions": [
        { "Type": "Wait", "WaitForText": "CICS MENU", "TimeoutSeconds": 10 },
        { "Type": "SendKey", "Key": "ORDRINQ" },
        { "Type": "SendKey", "Key": "Enter" },
        { "Type": "GetScreenContent", "StartRow": 5, "StartColumn": 10, "Length": 80 }
      ]
    }
  }
}
```

For modern mainframes with CICS web services enabled, use Logic Apps HTTP connector with SOAP policies in APIM.

### Migration Tracking Dashboard

Maintain a migration registry that tracks each legacy integration's migration status:

```kql
// Application Insights: query integration routing telemetry to track migration progress
customEvents
| where name == "IntegrationRouted"
| extend integrationName = tostring(customDimensions["IntegrationName"])
| extend routedTo = tostring(customDimensions["Target"])  // "legacy" or "azure"
| summarize LegacyRoutes = countif(routedTo == "legacy"),
            AzureRoutes  = countif(routedTo == "azure")
    by integrationName, bin(timestamp, 1d)
| order by integrationName, timestamp
```

## Key Considerations

**Feature Flag Governance:** The routing flags that control legacy-vs-Azure routing are operational controls with production impact. Manage them through Azure App Configuration with feature flag support, not through APIM Named Values directly. App Configuration provides change history, staged rollout (percentage-based routing), and access control. Never manually flip routing flags in the portal without a documented change record.

**Parallel Operation Period:** During migration, both the legacy system and the new Azure implementation are maintained simultaneously. This doubles the operational burden for the affected integration. Keep the parallel period short (days to weeks, not months) by validating the Azure implementation thoroughly before cutover, not after.

**Legacy System Documentation:** The most common obstacle in legacy modernization is discovering that the legacy integration's business logic is undocumented and understood only by the original developer. Before migrating any integration from BizTalk or ESB, document the transformation logic, routing rules, exception handling, and downstream dependencies by reading the existing artifacts. Budget time for this discovery work.

**Testing the Bridge:** The bridge itself must be tested independently of both the legacy system and the migrated Azure implementation. Integration tests should verify that: (a) the REST API contract exposed by APIM is correct, (b) the transformation to legacy format is accurate, (c) the legacy system response is correctly transformed back to the REST response, and (d) errors from the legacy system are surfaced correctly through the bridge.

**Decommission Planning:** Define the decommission date for the legacy platform at the start of the project, not after migration is complete. A decommission date creates urgency, prevents migration stall, and enables license and infrastructure cost planning. Treat the decommission date as a firm commitment with executive sponsorship.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
