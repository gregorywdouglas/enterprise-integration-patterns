# Azure Data Factory

## Overview

Azure Data Factory (ADF) is a managed data integration service for building, scheduling, and orchestrating data movement and transformation pipelines at scale. It connects to 90+ data sources (cloud and on-premises) through its connector library, transforms data using mapping data flows (code-free) or compute activities (Databricks, HDInsight, Azure Functions), and orchestrates complex multi-step pipelines with conditional logic, looping, and error handling.

In enterprise integration, ADF complements messaging-based integration patterns by handling bulk data movement, historical data loads, and complex ETL/ELT transformations that are not suited to event-driven Function or Logic App implementations.

---

## Key Capabilities

### Pipeline Activities

ADF pipelines are composed of activities:

| Activity Type | Examples |
|---|---|
| **Data Movement** | Copy Activity — moves data between source and sink |
| **Data Transformation** | Mapping Data Flow, Databricks Notebook, HDInsight Hive, Azure Function |
| **Control Flow** | If Condition, Switch, For Each, Until, Execute Pipeline, Wait |
| **External** | Web Activity (HTTP), Stored Procedure, Lookup, Get Metadata |

The **Copy Activity** is the core workhorse: it moves data from a source dataset to a sink dataset with optional column mapping, data type conversion, and fault tolerance for partial copy failures.

### Integration Runtime

The Integration Runtime (IR) is the compute infrastructure that executes ADF activities:

| IR Type | Location | Use Case |
|---|---|---|
| **Azure IR** | Managed Azure infrastructure | Cloud-to-cloud data movement and transformation |
| **Self-Hosted IR (SHIR)** | On-premises or other cloud | On-premises data source access; private network data movement |
| **Azure-SSIS IR** | Managed Azure VMs | Lift-and-shift SSIS package execution |

For hybrid integration scenarios (on-premises to cloud), the Self-Hosted Integration Runtime is deployed on an on-premises Windows server and connects outbound to ADF. See [On-Premises to Cloud Bridge](../patterns/hybrid/On-Premises-to-Cloud-Bridge.md) and [Hybrid Data Synchronization](../patterns/hybrid/Hybrid-Data-Synchronization.md).

### Mapping Data Flows

Mapping Data Flows provide a code-free data transformation designer that generates Apache Spark jobs executed on managed Spark clusters. Use for:
- Complex column transformations and derived columns
- Aggregations, joins, and union operations
- Schema normalization and denormalization
- Data quality checks and conditional routing (Conditional Split)

Data flows run on auto-provisioned Spark clusters (time-to-live configurable from 0–60 minutes) — cold start is 2–4 minutes. For latency-sensitive pipelines, enable TTL to keep the cluster warm.

### Watermark-Based Incremental Load

The standard ADF pattern for incremental data synchronization:

```json
{
  "name": "IncrementalCopyPipeline",
  "properties": {
    "activities": [
      {
        "name": "LookupLastWatermark",
        "type": "Lookup",
        "typeProperties": {
          "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": "SELECT Watermark FROM WatermarkTable WHERE TableName = 'Customers'"
          },
          "dataset": { "referenceName": "WatermarkDataset", "type": "DatasetReference" }
        }
      },
      {
        "name": "LookupCurrentWatermark",
        "type": "Lookup",
        "typeProperties": {
          "source": {
            "type": "SqlServerSource",
            "sqlReaderQuery": "SELECT MAX(ModifiedAt) AS CurrentWatermark FROM dbo.Customers"
          },
          "dataset": { "referenceName": "OnPremCustomersDataset", "type": "DatasetReference" }
        }
      },
      {
        "name": "CopyIncrementalData",
        "type": "Copy",
        "dependsOn": [
          { "activity": "LookupLastWatermark", "dependencyConditions": ["Succeeded"] },
          { "activity": "LookupCurrentWatermark", "dependencyConditions": ["Succeeded"] }
        ],
        "typeProperties": {
          "source": {
            "type": "SqlServerSource",
            "sqlReaderQuery": {
              "value": "@concat('SELECT * FROM dbo.Customers WHERE ModifiedAt > ''', activity('LookupLastWatermark').output.firstRow.Watermark, ''' AND ModifiedAt <= ''', activity('LookupCurrentWatermark').output.firstRow.CurrentWatermark, '''')"
            }
          },
          "sink": { "type": "CosmosDbSink", "writeBehavior": "upsert" },
          "parallelCopies": 4,
          "enableStaging": false
        }
      },
      {
        "name": "UpdateWatermark",
        "type": "SqlServerStoredProcedure",
        "dependsOn": [{ "activity": "CopyIncrementalData", "dependencyConditions": ["Succeeded"] }],
        "typeProperties": {
          "storedProcedureName": "usp_UpdateWatermark",
          "storedProcedureParameters": {
            "LastModifiedTime": { "value": "@activity('LookupCurrentWatermark').output.firstRow.CurrentWatermark", "type": "DateTime" },
            "TableName":        { "value": "Customers", "type": "String" }
          }
        }
      }
    ]
  }
}
```

### Pipeline Triggers

| Trigger Type | Description |
|---|---|
| **Schedule** | Run at fixed times (daily, hourly, cron expression) |
| **Tumbling Window** | Recurring fixed-size time windows with dependency support |
| **Event-based (Storage)** | Fire on blob creation/deletion in Azure Storage |
| **Event-based (Custom)** | Fire on Event Grid custom events |
| **Manual** | API-triggered execution |

---

## Common Use in Integration Patterns

| Pattern | ADF Role |
|---|---|
| [Hybrid Data Synchronization](../patterns/hybrid/Hybrid-Data-Synchronization.md) | Bulk copy, watermark-based incremental load |
| [On-Premises to Cloud Bridge](../patterns/hybrid/On-Premises-to-Cloud-Bridge.md) | SHIR-based on-premises data extraction |

---

## Configuration Considerations

### Managed Identity Authentication

ADF uses system-assigned managed identity to authenticate to Azure services. Grant ADF's managed identity the appropriate RBAC roles:

```bicep
// Grant ADF identity read access to source storage
resource adfStorageRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(dataFactory.id, storageAccount.id, 'reader')
  scope: storageAccount
  properties: {
    roleDefinitionId: subscriptionResourceId(
      'Microsoft.Authorization/roleDefinitions',
      '2a2b9908-6ea1-4ae2-8e65-a410df84e7d1')  // Storage Blob Data Reader
    principalId:   dataFactory.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

### SHIR High Availability

Deploy SHIR in multi-node configuration for high availability:

```powershell
# Register additional SHIR nodes using the same authentication key
Set-AzDataFactoryV2IntegrationRuntime -ResourceGroupName "rg-integration" `
  -DataFactoryName "adf-integration-prod" `
  -Name "shir-onprem" `
  -SharedIntegrationRuntimeResourceId "/subscriptions/.../shir-primary"
```

### Copy Activity Performance Tuning

| Setting | Guidance |
|---|---|
| `parallelCopies` | Degree of parallelism per copy activity (default: auto) |
| `dataIntegrationUnits` | Compute allocation for Azure IR (2–256 DIUs; default: auto) |
| Staging | Use staging for large copy operations (source → Blob → sink) |
| Partition option | Use dynamic range partitioning for parallel SQL reads |

### Service Limits

| Limit | Value |
|---|---|
| Max pipeline activities | 40 per pipeline |
| Max pipelines per factory | 5,000 |
| Max concurrent pipeline runs | 10,000 |
| Max copy throughput | 5 GB/s (Azure IR) |
| Max SHIR nodes per IR | 4 |
| Max linked services | 5,000 |

---

*Part of the [Enterprise Integration Patterns](../README.md) library by Cheops Consulting Services.*
