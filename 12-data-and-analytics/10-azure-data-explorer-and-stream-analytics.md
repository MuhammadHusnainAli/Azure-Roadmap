# Azure Data Explorer and Stream Analytics

## Why it matters

Telemetry (logs, metrics, IoT readings, clickstreams, AI traces) arrives continuously and in huge volumes. Two Azure PaaS services specialise in it. **Azure Data Explorer (ADX)** stores and queries time-series and log data at scale with KQL. **Azure Stream Analytics (ASA)** processes events in flight with SQL-like queries. Fabric Real-Time Intelligence (file 05) now offers the same Kusto engine as a SaaS service, so architects should know when the standalone PaaS services are still the better fit.

## Key concepts

### Azure Data Explorer

| Concept | Detail |
|---|---|
| **Cluster** | Compute (engine nodes) plus storage, sized by SKU and instance count; optimised autoscale |
| **Database / table** | Columnar, compressed, automatically indexed (including full-text on strings) |
| **Ingestion** | Queued (batched, default), **streaming ingestion** (low latency, small batches), from Event Hubs, IoT Hub, Event Grid (blob notifications), Kafka, Logstash, SDKs, LightIngest |
| **Policies** | **Retention** (soft delete), **caching** (hot data on SSD), **update policies**, partitioning, row-level security |
| **Materialized views** | Always-up-to-date aggregations |
| **External tables** | Query data in ADLS, SQL and other sources without ingesting |
| **Follower databases / database shortcuts** | Read-only attachment of a database from another cluster (or Fabric eventhouse), for compute isolation |
| **KQL** | Same language as Log Analytics, Sentinel and Fabric Eventhouse |
| **Free cluster** | A free, limited cluster for learning and prototyping (check current limits) |

ADX is the engine behind Azure Monitor Logs, Microsoft Sentinel, Defender advanced hunting and Fabric Eventhouse.

### Azure Stream Analytics

| Concept | Detail |
|---|---|
| **Job** | Inputs → query → outputs, running continuously |
| **Inputs** | Streaming: Event Hubs, IoT Hub, Blob/ADLS Gen2, Kafka. **Reference data**: Blob/ADLS or Azure SQL Database (slowly changing lookup data) |
| **Query** | SQL dialect with temporal semantics: `TIMESTAMP BY`, windowing, temporal joins, `LAG`, pattern matching (`MATCH_RECOGNIZE`), geospatial functions, built-in **anomaly detection** functions, JavaScript and C# UDFs |
| **Outputs** | Event Hubs, Service Bus, Blob/ADLS (Parquet, Delta), Azure SQL, Cosmos DB, Azure Functions, Power BI, Azure Data Explorer, Fabric lakehouse, PostgreSQL and more |
| **Scale** | **Streaming units (SU V2)**, with parallelism through partitioned inputs and `PARTITION BY` |
| **Hosting** | Standard (multitenant) jobs, or a **Stream Analytics cluster** (dedicated, VNet support) |
| **No-code editor** | Build ASA jobs visually from the Event Hubs portal experience (for example capture to Delta or Parquet) |
| **Delivery guarantees** | Exactly-once processing within the job. End-to-end exactly-once depends on the output (some are at-least-once) |

### Windowing functions

| Window | Behaviour | Example |
|---|---|---|
| **Tumbling** | Fixed-size, non-overlapping | Count per 5 minutes |
| **Hopping** | Fixed-size, overlapping by a hop | 10-minute average every 1 minute |
| **Sliding** | Emits when the content changes, considering events within the window | Alert when 3+ errors in any 10 seconds |
| **Session** | Groups events that arrive close together, with timeout and max duration | User sessions |
| **Snapshot** | Groups events with the same timestamp | Simultaneous events |

```sql
-- Per-deployment 429 rate in 1-minute tumbling windows, flag spikes
SELECT
    System.Timestamp() AS WindowEnd,
    Deployment,
    COUNT(*) AS Calls,
    SUM(CASE WHEN StatusCode = 429 THEN 1 ELSE 0 END) AS Throttled
INTO [alerts-eventhub]
FROM [apim-logs] TIMESTAMP BY EventTime
GROUP BY Deployment, TumblingWindow(minute, 1)
HAVING SUM(CASE WHEN StatusCode = 429 THEN 1 ELSE 0 END) > 50
```

## How it works on Azure

### Typical telemetry architecture

```text
Devices / apps / APIM ──► Event Hubs ──┬─► Stream Analytics (filter, window, enrich with reference data)
                                        │        ├─► alerts → Event Hubs / Functions / Logic Apps
                                        │        └─► aggregates → SQL / Power BI / lakehouse
                                        └─► Azure Data Explorer (raw + update policies, 30–90 days hot, years cold)
                                                 └─► dashboards, Grafana, KQL, ML on time series
```

### Create an ADX cluster and stream from Event Hubs (CLI and KQL)

```bash
az kusto cluster create -g rg-telemetry -n adxcontoso -l westeurope \
  --sku name="Standard_E8ads_v5" tier="Standard" capacity=2
az kusto database create -g rg-telemetry --cluster-name adxcontoso --database-name telemetry \
  --read-write-database soft-delete-period=P365D hot-cache-period=P31D location=westeurope
```

```kusto
.create table Telemetry (DeviceId: string, Ts: datetime, Temperature: real)
.create table Telemetry ingestion json mapping 'tmap'
'[{"column":"DeviceId","path":"$.deviceId"},{"column":"Ts","path":"$.ts"},{"column":"Temperature","path":"$.temp"}]'

// Then create an Event Hubs data connection (portal/CLI) using the cluster's managed identity
Telemetry
| where Ts > ago(1d)
| make-series AvgTemp = avg(Temperature) on Ts step 5m by DeviceId
| extend anomalies = series_decompose_anomalies(AvgTemp, 2.5)
```

Check the `az kusto` parameter names against the current CLI reference; some are set through extension-specific shorthand.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **ADX vs Fabric Eventhouse** | Same engine and KQL. Choose Fabric Eventhouse for SaaS simplicity, OneLake integration, Activator and Fabric capacity billing. Choose ADX for PaaS control (SKU, VNet injection/private endpoints, managed identity on Azure resources), Azure-native billing, or existing ADX estates and ISV multi-tenant scenarios |
| **Stream Analytics vs Fabric Eventstreams** | Eventstreams for no-code routing into Fabric; ASA for complex temporal SQL (pattern matching, temporal joins, geospatial), many Azure outputs and mature production workloads |
| **Stream Analytics vs Spark Structured Streaming** | ASA for SQL-based, low-ops stream processing; Spark (Databricks or Fabric) for complex code, ML in the stream, or joining with large lakehouse data |
| **Process in stream vs query at rest** | Stream processing for immediate actions and reductions; ADX for interactive, ad hoc analysis of everything. Many designs do both |
| **ASA dedicated cluster** | When jobs need VNet access to private resources or predictable isolated capacity |

## Hands-on

1. Create a free ADX cluster (or an Eventhouse), ingest sample data, and run time-series KQL with `make-series` and `series_decompose_anomalies`.
2. Create an Event Hub and a Stream Analytics job with a tumbling-window query writing to a Blob output; send test events with a Python producer.
3. Add **reference data** from a CSV in Blob (device → site) and join it in the query.
4. Replace the Blob output with ADX and watch data arrive.
5. Compare the same pipeline built as a Fabric Eventstream → Eventhouse and note the differences in setup and cost model.

## Security, cost and reliability notes

- **Security:** use managed identities for ASA inputs and outputs and ADX data connections; disable local auth on Event Hubs.
- **Security:** ADX supports private endpoints, disk encryption with CMK and row-level security; use database-level roles mapped to Entra groups.
- **Cost:** ADX costs are driven by cluster size; tune hot cache (SSD) vs retention, use optimised autoscale, and stop dev clusters. ASA costs are driven by streaming units running 24×7, so stop unused jobs.
- **Reliability:** ASA jobs checkpoint and recover automatically; plan for late and out-of-order events with event ordering policies.
- **Reliability:** ADX supports availability zones; for disaster recovery, ingest into clusters in two regions or use follower and replication patterns as documented.

## Common mistakes

- Using `System.Timestamp()` without `TIMESTAMP BY`, so windows use arrival time instead of event time.
- Too few input partitions, so the ASA job can't scale.
- Keeping all ADX data in hot cache.
- Building heavy parsing into every KQL query instead of update policies.
- Running a standalone ADX cluster for a team that is fully on Fabric, or the reverse, without a reason.

## Check yourself

1. Name the five windowing functions in Stream Analytics.
2. What is reference data in Stream Analytics?
3. Which Microsoft services share the ADX engine?
4. When would you choose ADX over a Fabric Eventhouse?
5. What does a hot cache policy control?
6. Why use `TIMESTAMP BY`?

<details>
<summary>Answers</summary>

1. Tumbling, hopping, sliding, session and snapshot.
2. Slowly changing lookup data (from Blob/ADLS or Azure SQL) joined with streaming data to enrich events.
3. Azure Monitor Logs (Log Analytics), Microsoft Sentinel, Defender advanced hunting, and Fabric Eventhouse / Real-Time Intelligence.
4. When you need PaaS control (SKUs, networking, Azure-native billing), have an existing ADX estate, or build multi-tenant ISV solutions outside Fabric.
5. How much recent data is kept on local SSD for fast queries, as opposed to total retention in cheaper storage.
6. To process events by their own event time (not arrival time), so windows and temporal logic are correct with late or out-of-order data.

</details>

## References

- [What is Azure Data Explorer?](https://learn.microsoft.com/azure/data-explorer/data-explorer-overview)
- [Ingestion overview](https://learn.microsoft.com/azure/data-explorer/ingest-data-overview)
- [Cache policy](https://learn.microsoft.com/kusto/management/cache-policy)
- [Free Azure Data Explorer cluster](https://learn.microsoft.com/azure/data-explorer/start-for-free)
- [What is Azure Stream Analytics?](https://learn.microsoft.com/azure/stream-analytics/stream-analytics-introduction)
- [Windowing functions](https://learn.microsoft.com/azure/stream-analytics/stream-analytics-window-functions)
- [Reference data inputs](https://learn.microsoft.com/azure/stream-analytics/stream-analytics-use-reference-data)
- [Stream Analytics outputs](https://learn.microsoft.com/azure/stream-analytics/stream-analytics-define-outputs)
- [Fabric Real-Time Intelligence compared with Azure Data Explorer](https://learn.microsoft.com/fabric/real-time-intelligence/real-time-intelligence-compare)
