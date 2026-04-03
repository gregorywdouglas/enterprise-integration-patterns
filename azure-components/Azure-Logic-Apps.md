# Azure Logic Apps

## Overview

Azure Logic Apps is a cloud-based workflow automation service that orchestrates business processes, system integrations, and data flows using a visual designer with a JSON workflow definition underneath. It provides 400+ managed connectors for common enterprise systems (SAP, Salesforce, ServiceNow, Dynamics 365, Office 365, SharePoint) and supports both stateful and stateless workflow types.

Logic Apps Standard (the current platform generation, formerly "single-tenant Logic Apps") runs on Azure App Service infrastructure, supporting VNet integration, managed identity, and private endpoint connectivity. Logic Apps Consumption (multi-tenant, legacy) uses a shared infrastructure model and is better suited for simple automation workflows than enterprise integration.

---

## Key Capabilities

### Workflow Types

**Stateful workflows:** Persist intermediate state, inputs, and outputs in Azure Storage. Support long-running operations, human-in-the-loop patterns, and correlation of events across extended time periods. Can run for days or years.

**Stateless workflows:** In-memory only; no state persistence. Faster execution (sub-second for simple workflows). No run history unless explicitly configured. Suitable for synchronous request-response workflows with predictable short execution times.

### Connectors

Logic Apps connectors abstract the complexity of connecting to external systems:

| Category | Examples |
|---|---|
| **Azure Services** | Service Bus, Event Grid, Event Hubs, Blob Storage, Table Storage, Cosmos DB, SQL Database |
| **On-Premises (via OPDG)** | SQL Server, Oracle DB, SAP, File System, IBM MQ, BizTalk |
| **SaaS** | Salesforce, ServiceNow, Dynamics 365, Workday, NetSuite |
| **Collaboration** | Office 365, SharePoint, Teams |
| **HTTP/API** | HTTP, SOAP, AS2, X12 (EDI), EDIFACT |
| **AI** | Azure OpenAI, Azure Cognitive Services |

On-premises connectors require the On-Premises Data Gateway (OPDG) installed on a server with network access to the on-premises system. See [On-Premises to Cloud Bridge](../patterns/hybrid/On-Premises-to-Cloud-Bridge.md).

### Built-in Connectors (Standard Only)

Logic Apps Standard includes built-in (in-process) connectors for Service Bus, Event Hubs, SQL, HTTP, Azure Functions, and others. Built-in connectors are faster (no HTTP hop to the managed API) and support managed identity natively.

### Workflow Definition Language

Logic Apps workflows are defined in JSON (Workflow Definition Language). Visual designer generates and reads this JSON. Workflows can be authored in code:

```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "triggers": {
      "When_a_message_is_received": {
        "type": "ServiceBus",
        "inputs": {
          "parameters": {
            "entityName": "orders-queue",
            "sessionId": ""
          },
          "serviceProviderConfiguration": {
            "connectionName": "serviceBus",
            "operationId":    "receiveQueueMessages",
            "serviceProviderId": "/serviceProviders/serviceBus"
          }
        },
        "recurrence": { "frequency": "Second", "interval": 10 }
      }
    },
    "actions": {
      "Parse_Order_Message": {
        "type": "ParseJson",
        "inputs": {
          "content": "@triggerBody()?['contentData']",
          "schema": { "$ref": "#/definitions/OrderMessage" }
        }
      },
      "Condition_Is_High_Value": {
        "type": "If",
        "expression": {
          "greater": ["@body('Parse_Order_Message')?['totalAmount']", 10000]
        },
        "actions": {
          "Route_to_Premium_Processing": {
            "type": "ServiceBus",
            "inputs": {
              "parameters": {
                "entityName": "premium-orders-queue",
                "message": {
                  "contentData": "@body('Parse_Order_Message')",
                  "contentType": "application/json",
                  "correlationId": "@triggerBody()?['correlationId']"
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### Error Handling

Logic Apps supports error handling at the action level:

```json
"On_Failure_Notify": {
  "type": "ServiceBus",
  "runAfter": {
    "Call_Backend_API": ["Failed", "TimedOut"]
  },
  "inputs": {
    "parameters": {
      "entityName": "error-queue",
      "message": {
        "contentData": {
          "workflowRunId": "@workflow().run.name",
          "failedAction":  "Call_Backend_API",
          "error":         "@actions('Call_Backend_API').error",
          "correlationId": "@triggerBody()?['correlationId']"
        }
      }
    }
  }
}
```

### Retry Policies

Configure retry policies per action:

```json
"retryPolicy": {
  "type":     "exponential",
  "count":    4,
  "interval": "PT5S",
  "minimumInterval": "PT5S",
  "maximumInterval": "PT1H"
}
```

---

## Logic Apps vs. Azure Functions Decision Guide

| Scenario | Use Logic Apps | Use Azure Functions |
|---|---|---|
| Visual workflow design requirement | ✓ | |
| 400+ connector ecosystem needed | ✓ | |
| On-premises data sources via OPDG | ✓ | |
| EDI / B2B processing (AS2, X12) | ✓ | |
| Complex code / custom algorithms | | ✓ |
| High-throughput, low-latency processing | | ✓ |
| Custom SDK integrations | | ✓ |
| Unit-testable business logic | | ✓ |
| Long-running orchestration with wait states | Both | Durable Functions |

Logic Apps and Functions are complementary. A common pattern: Logic Apps orchestrates the workflow (connector-based steps, conditional routing), calling Azure Functions for compute-intensive or code-intensive steps.

---

## Common Use in Integration Patterns

| Pattern | Logic Apps Role |
|---|---|
| [On-Premises to Cloud Bridge](../patterns/hybrid/On-Premises-to-Cloud-Bridge.md) | On-premises connector workflows via OPDG |
| [Legacy Modernization Bridge](../patterns/hybrid/Legacy-Modernization-Bridge.md) | BizTalk replacement for routing/transformation workflows |
| [Dead-Letter Recovery](../patterns/event-driven/Dead-Letter-Recovery.md) | Orchestrate manual DLQ review and replay |
| [Saga Pattern](../patterns/event-driven/Saga-Pattern.md) | Choreography-based saga step with connector actions |
| [Hybrid Data Synchronization](../patterns/hybrid/Hybrid-Data-Synchronization.md) | Trigger data sync workflows from blob or queue events |

---

## Configuration Considerations

### Standard vs. Consumption

| Feature | Standard | Consumption |
|---|---|---|
| VNet integration | Yes | No |
| Private endpoints | Yes | No |
| Managed identity | Yes | Limited |
| Stateless workflows | Yes | No |
| Built-in connectors | Yes | No |
| Pricing model | App Service plan | Per-execution |
| CI/CD deployment | Yes (zip deploy, GitHub Actions) | ARM template |

Use **Logic Apps Standard** for all production enterprise integration workloads requiring VNet connectivity or managed identity.

### Managed Identity Configuration

```bicep
resource logicApp 'Microsoft.Web/sites@2023-01-01' = {
  name: 'la-integration-prod'
  kind: 'workflowapp'
  identity: { type: 'SystemAssigned' }
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      appSettings: [
        // Service Bus via managed identity
        { name: 'ServiceBus__fullyQualifiedNamespace'
          value: '${serviceBusNamespace.name}.servicebus.windows.net' }
      ]
    }
  }
}
```

### Service Limits (Standard)

| Limit | Value |
|---|---|
| Max workflow actions | 500 |
| Max workflow run duration | 90 days (stateful) |
| Max parallel branches | 10 |
| Max for-each iterations | 100,000 (default configurable) |
| Max request body | 100 MB (with chunking) |
| Max response body | 100 MB |
| Stateless workflow max duration | 5 minutes |

---

*Part of the [Enterprise Integration Patterns](../README.md) library by Cheops Consulting Services.*
