# Hybrid Data Synchronization

## Problem

Enterprise organizations operate data in both on-premises systems (ERP, CRM, legacy databases) and cloud data stores (Azure SQL, Cosmos DB, Azure Data Lake). Applications on each side need access to data that originates on the other side. A cloud-based analytics pipeline needs current customer records from an on-premises Oracle database. An on-premises reporting system needs financial transaction records from a cloud-hosted Cosmos DB.

Batch file exports — the traditional approach — introduce latency (hours or days behind reality), require complex file format negotiation, and are fragile at the file transfer layer. Bidirectional synchronization introduces the risk of update conflicts when the same record can be modified on both sides simultaneously. Without explicit conflict resolution logic, one side silently overwrites the other.

## Solution

Hybrid Data Synchronization uses Change Data Capture (CDC) and event-driven replication to propagate data changes across boundaries with low latency and explicit conflict handling. Changes are captured at the source as they occur and streamed to the destination, rather than being exported in bulk on a schedule.

The pattern has two primary topologies:

**Unidirectional sync (source of record → replica):** One system is the authoritative source; the other receives read-only replicas. No conflict resolution needed. Simplest to implement and operate.

**Bidirectional sync:** Both systems can originate changes. Requires explicit conflict detection (last-write-wins, version vector, or business-rule-based resolution) and careful control of update loops (changes propagated from A to B must not be re-propagated back to A).

On Azure, the primary tools are:
- **Azure Data Factory** with self-hosted integration runtime for scheduled or triggered batch synchronization
- **Azure Database Migration Service** or **Debezium** on Azure Container Apps for continuous CDC from on-premises databases
- **Azure Event Hubs + Azure Stream Analytics** for real-time change stream processing
- **Azure Logic Apps** for event-driven sync with built-in connector support for common on-premises data sources

## When to Use

- Use CDC-based sync when data must be available in the target within minutes of a change at the source.
- Use ADF batch sync for large-volume historical data loads or when near-real-time is not required and batch windows are acceptable.
- Use unidirectional sync wherever business rules allow — bidirectional sync multiplies complexity and conflict risk.
- Avoid bidirectional sync for financial records, audit logs, or compliance data — define a single authoritative system and replicate read replicas to the other side.
- Avoid using database-level replication mechanisms (SQL replication, AlwaysOn) across cloud/on-premises boundaries — they require direct database network connectivity and are fragile across the WAN latency inherent in hybrid connectivity.

## Azure Implementation

### CDC via Debezium on Azure Container Apps

Debezium captures row-level changes from on-premises SQL Server, Oracle, or PostgreSQL and streams them to Azure Event Hubs. From Event Hubs, changes are consumed by Azure Functions or Stream Analytics for transformation and loading into the cloud target:

```yaml
# Debezium SQL Server connector configuration (deployed as Container App)
apiVersion: apps/containerapps/v1
kind: ContainerApp
spec:
  template:
    containers:
      - name: debezium-connector
        image: debezium/connect:2.5
        env:
          - name: BOOTSTRAP_SERVERS
            value: "your-eventhub-namespace.servicebus.windows.net:9093"
          - name: CONFIG_STORAGE_TOPIC
            value: "debezium-configs"
          - name: OFFSET_STORAGE_TOPIC
            value: "debezium-offsets"
          - name: STATUS_STORAGE_TOPIC
            value: "debezium-status"
          - name: CONNECT_SECURITY_PROTOCOL
            value: "SASL_SSL"
          - name: CONNECT_SASL_MECHANISM
            value: "PLAIN"
```

```json
// Debezium SQL Server source connector configuration
{
  "name": "erp-sqlserver-connector",
  "config": {
    "connector.class": "io.debezium.connector.sqlserver.SqlServerConnector",
    "database.hostname": "sql-erp.corp.internal",
    "database.port": "1433",
    "database.user": "debezium_cdc",
    "database.password": "${file:/run/secrets/db-password:password}",
    "database.names": "ERP",
    "table.include.list": "dbo.Customers,dbo.Orders,dbo.OrderItems",
    "database.history.kafka.bootstrap.servers": "your-eventhub-namespace.servicebus.windows.net:9093",
    "database.history.kafka.topic": "dbhistory.ERP",
    "topic.prefix": "erp",
    "snapshot.mode": "initial",
    "decimal.handling.mode": "string",
    "include.schema.changes": "true"
  }
}
```

### Azure Function — CDC Event Consumer and Cloud Target Writer

```csharp
[Function("ERPCustomerSyncProcessor")]
public async Task Run(
    [EventHubTrigger("erp.dbo.Customers", Connection = "EventHubs", ConsumerGroup = "cloud-sync")]
    EventData[] events)
{
    foreach (var eventData in events)
    {
        var changeEvent = JsonSerializer.Deserialize<DebeziumChangeEvent<CustomerRecord>>(
            eventData.EventBody.ToArray())!;

        var after = changeEvent.Payload.After;
        var op    = changeEvent.Payload.Op;  // 'c' = create, 'u' = update, 'd' = delete, 'r' = read (snapshot)

        switch (op)
        {
            case "c" or "u" or "r":
                await _cosmosContainer.UpsertItemAsync(
                    new CustomerDocument
                    {
                        Id           = after.CustomerId.ToString(),
                        CustomerId   = after.CustomerId,
                        FullName     = after.FullName,
                        Email        = after.Email,
                        Status       = after.Status,
                        SyncedAt     = DateTimeOffset.UtcNow,
                        SourceSystem = "ERP",
                        SourceLsn    = changeEvent.Payload.Source.Lsn
                    },
                    new PartitionKey(after.CustomerId.ToString()));
                break;

            case "d":
                // Soft delete — mark as deleted rather than removing the document
                var existing = await _cosmosContainer.ReadItemAsync<CustomerDocument>(
                    after.CustomerId.ToString(),
                    new PartitionKey(after.CustomerId.ToString()));

                existing.Resource.IsDeleted = true;
                existing.Resource.DeletedAt = DateTimeOffset.UtcNow;
                await _cosmosContainer.UpsertItemAsync(existing.Resource,
                    new PartitionKey(after.CustomerId.ToString()));
                break;
        }
    }
}
```

### ADF Pipeline for Bulk Sync with Watermark

For large historical loads or when CDC is not available, ADF pipelines with a watermark-based incremental load pattern provide efficient bulk sync:

```json
{
  "name": "IncrementalCustomerSync",
  "type": "Microsoft.DataFactory/factories/pipelines",
  "properties": {
    "activities": [
      {
        "name": "GetLastWatermark",
        "type": "Lookup",
        "typeProperties": {
          "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": "SELECT MAX(SyncedAt) AS LastWatermark FROM SyncWatermarks WHERE TableName = 'Customers'"
          }
        }
      },
      {
        "name": "CopyIncrementalData",
        "type": "Copy",
        "dependsOn": [{ "activity": "GetLastWatermark", "dependencyConditions": ["Succeeded"] }],
        "typeProperties": {
          "source": {
            "type": "SqlServerSource",
            "sqlReaderQuery": {
              "value": "@concat('SELECT * FROM dbo.Customers WHERE ModifiedAt > ''', activity('GetLastWatermark').output.firstRow.LastWatermark, '''')"
            }
          },
          "sink": { "type": "CosmosDbSink", "writeBehavior": "upsert" }
        }
      },
      {
        "name": "UpdateWatermark",
        "type": "SqlServerStoredProcedure",
        "dependsOn": [{ "activity": "CopyIncrementalData", "dependencyConditions": ["Succeeded"] }],
        "typeProperties": {
          "storedProcedureName": "usp_UpdateSyncWatermark",
          "storedProcedureParameters": {
            "TableName": "Customers",
            "NewWatermark": { "value": "@pipeline().TriggerTime", "type": "String" }
          }
        }
      }
    ]
  }
}
```

### Conflict Resolution for Bidirectional Sync

For bidirectional synchronization, implement explicit conflict detection using a version vector or timestamp comparison:

```csharp
public async Task<SyncDecision> ResolveConflictAsync(
    CustomerRecord onPremRecord,
    CustomerDocument cloudRecord)
{
    // Last-write-wins based on modification timestamp
    if (onPremRecord.ModifiedAt > cloudRecord.ModifiedAt)
        return SyncDecision.ApplyOnPremToCloud;

    if (cloudRecord.ModifiedAt > onPremRecord.ModifiedAt)
        return SyncDecision.ApplyCloudToOnPrem;

    // Simultaneous modification — apply business rules
    // e.g., cloud modifications to contact fields win; on-prem wins for financial fields
    return SyncDecision.MergeFields;
}
```

### Loop Prevention

In bidirectional sync, a change propagated from A to B must not trigger a sync event back to A. Use a sync source marker to identify records modified by the sync process itself:

```sql
-- On-premises: skip records where sync flag is set
UPDATE dbo.Customers
SET FullName = @FullName, Email = @Email, SyncSource = 'AzureCloud', SyncedAt = GETUTCDATE()
WHERE CustomerId = @CustomerId AND SyncSource <> 'AzureCloud'
-- Trigger-based CDC should filter out rows where SyncSource = 'AzureCloud'
```

## Key Considerations

**CDC Prerequisite: Change Tracking / Supplemental Logging:** CDC requires the source database to be configured for change capture. SQL Server requires SQL Server Agent to be running and the database to be enabled for CDC (ALTER DATABASE ... SET CHANGE_TRACKING ON). Oracle requires supplemental logging enabled at the table level. This is a DBA-level database configuration change — plan for it in the database change management process.

**Initial Snapshot Consistency:** When starting CDC for the first time, Debezium performs an initial snapshot of the current table state before switching to change stream mode. Ensure the snapshot is consistent: no data writes should occur during the snapshot, or use the database's snapshot isolation level to produce a consistent read. Initial snapshots of large tables (millions of rows) can take hours — schedule them during maintenance windows.

**Schema Evolution:** When the source table schema changes (column added, renamed, or type changed), the Debezium connector may pause or emit changed schema events. Have a process for handling schema evolution: update the target schema, deploy the updated consumer, then resume the connector. Schema changes are the most common cause of CDC pipeline outages.

**Data Volume and Event Hub Throughput:** Debezium publishes one message per row change. For high-write-rate tables (e.g., transaction logs, audit tables), the Event Hubs throughput unit requirement can be substantial. Estimate the peak change rate, calculate the message throughput, and size Event Hubs partition count accordingly (1 throughput unit = 1 MB/s inbound).

**Monitoring:** Alert on Debezium connector lag (the difference between the CDC log position being processed and the current database log position) as a key health signal. A growing lag means the connector or consumer is falling behind. Alert on sync failure counts from the Azure Function consumer.

---

*Part of the [Enterprise Integration Patterns](../../README.md) library by Cheops Consulting Services.*
