# Fabric Real-Time Intelligence

## Why it matters

Some data loses its value within seconds: fraud signals, machine telemetry, clickstreams, logistics positions, and the token usage and latency of your AI apps. **Real-Time Intelligence (RTI)** is the Fabric workload for data in motion. It ingests streams with **Eventstreams**, stores and queries them with **Eventhouses** using **KQL**, visualises them in **Real-Time Dashboards**, and acts on patterns with **Activator**. All of it is centred on the **Real-Time hub**. For AI teams, RTI is a natural home for model and agent telemetry, and Fabric data agents can query Eventhouse data in natural language.

## Key concepts

### Components

| Component | Role |
|---|---|
| **Real-Time hub** | Tenant-wide catalogue of streams and events: data streams, Fabric events, Azure events, and connectors to sources |
| **Eventstream** | No-code stream ingestion, transformation and routing (sources → processing → destinations) |
| **Eventhouse** | A container of **KQL databases** optimised for time-series, logs and event data, with automatic indexing and partitioning |
| **KQL database** | Tables, functions, materialized views, update policies; data can also be made available in OneLake as Delta |
| **KQL queryset** | Saved KQL queries and results, shareable |
| **Real-Time Dashboard** | Low-latency dashboards built on KQL queries, with auto-refresh |
| **Activator** | No-code detection of conditions in data, triggering actions (email, Teams, Power Automate, pipelines, notebooks, user data functions) |
| **Anomaly detection, maps, digital twin builder** | Additional RTI capabilities (check current status of each) |

### Eventstreams

| Part | Examples |
|---|---|
| **Sources** | Azure Event Hubs, Azure IoT Hub, Event Grid namespaces, custom endpoint (Event Hubs-, AMQP- or Kafka-compatible), Kafka (Confluent, Amazon MSK), Google Pub/Sub, Amazon Kinesis, CDC from Azure SQL, PostgreSQL, MySQL and Cosmos DB, Fabric workspace and OneLake events, Azure Blob events, sample data |
| **Processing** | Filter, manage fields, aggregate (windowed), group by, union, expand, join; **SQL operator** for code-based transforms |
| **Destinations** | Eventhouse (KQL database), lakehouse (Delta), Activator, derived stream, custom endpoint, Spark notebook |

The source list grows often. Check the current Eventstream sources page, and note that some sources and operators are still in preview.

### Eventhouse and KQL

KQL (Kusto Query Language) is a read-only, pipe-based query language built for log and time-series analysis. The same engine powers **Azure Data Explorer**, **Azure Monitor Logs**, **Microsoft Sentinel** and **Microsoft Defender** advanced hunting, so the skill transfers widely.

```kusto
// Token usage and latency by model over the last hour, 5-minute bins
LlmCalls
| where Timestamp > ago(1h)
| summarize Calls = count(),
            TotalTokens = sum(PromptTokens + CompletionTokens),
            P95LatencyMs = percentile(LatencyMs, 95)
          by Model, bin(Timestamp, 5m)
| order by Timestamp asc
| render timechart
```

| KQL concept | Use |
|---|---|
| `where`, `project`, `extend`, `summarize`, `join`, `top`, `render` | Core operators |
| `bin()`, `ago()`, `make-series` | Time bucketing and series |
| `series_decompose_anomalies()` | Built-in anomaly detection on time series |
| **Update policies** | Transform data on ingestion from a raw table into a curated table (a bronze → silver pattern inside Kusto) |
| **Materialized views** | Continuously maintained aggregations (for example latest value per device) |
| **Functions** | Reusable, parameterised queries; also used by data agents |
| **Caching and retention policies** | Hot cache (SSD) vs total retention; tune for cost and performance |
| **OneLake availability** | Expose KQL tables as Delta in OneLake for Spark, SQL and Power BI |
| **Shortcuts / external tables** | Query OneLake or ADLS data in place with `external_table()` |

### Activator

Activator watches data (from Eventstreams, Power BI visuals, Real-Time Dashboards or Fabric events) and fires **rules** when conditions hold for objects (for example "any device whose temperature is above 80 for 5 minutes"). Actions include email, Teams messages, Power Automate flows, Fabric pipelines, notebooks and functions. It's the "act" in the ingest → analyse → act loop.

## How it works on Azure

### Typical architecture

```text
Devices / apps / APIM logs ──► Event Hubs (or IoT Hub, Kafka, CDC)
                                   │
                         Eventstream (filter, enrich, aggregate)
                     ┌─────────────┼────────────────────┐
                     ▼             ▼                    ▼
               Eventhouse     Lakehouse (Delta)      Activator → Teams / pipeline / Logic App
               (KQL DB)        for batch + ML
                     │
        Real-Time Dashboard · Power BI · Fabric data agent (NL2KQL) · OneLake availability
```

### AI observability example

Send API Management or application telemetry about model calls (model, deployment, tokens, latency, status, consumer) to Event Hubs, route it through an Eventstream into an Eventhouse, then:

- build a Real-Time Dashboard of tokens per consumer and p95 latency;
- create an Activator rule: "if 429s for a deployment exceed 50 in 5 minutes, post to the on-call Teams channel";
- let operations staff ask a Fabric data agent "which app used the most tokens yesterday?" (answered with NL2KQL).

### Create a table with an update policy (KQL)

```kusto
.create table RawLlmEvents (Payload: dynamic, EnqueuedTime: datetime)

.create table LlmCalls (Timestamp: datetime, Model: string, PromptTokens: int,
                        CompletionTokens: int, LatencyMs: real, Consumer: string)

.create function ParseLlm() {
    RawLlmEvents
    | project Timestamp = todatetime(Payload.time),
              Model = tostring(Payload.model),
              PromptTokens = toint(Payload.usage.prompt_tokens),
              CompletionTokens = toint(Payload.usage.completion_tokens),
              LatencyMs = toreal(Payload.latencyMs),
              Consumer = tostring(Payload.subscriptionId)
}

.alter table LlmCalls policy update
@'[{"Source":"RawLlmEvents","Query":"ParseLlm()","IsEnabled":true,"IsTransactional":true}]'
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Fabric RTI vs Azure Data Explorer | Same engine. Choose Fabric RTI for SaaS simplicity, OneLake integration and capacity billing; Azure Data Explorer for standalone PaaS control, Azure-native networking and billing, or existing ADX estates (file 10) |
| Eventstream vs direct ingestion | Eventstream for no-code routing to multiple destinations and light transforms; direct Event Hubs → Eventhouse ingestion (data connection) for the simplest high-volume path |
| Eventhouse vs lakehouse for events | Eventhouse for interactive, sub-second queries over recent, high-volume time-series; lakehouse for batch analytics and ML over long history. Use OneLake availability to have both without copying |
| Hot cache and retention | Keep hot cache to the period users query interactively (for example 7–30 days); total retention per compliance needs |
| Activator vs Logic Apps / Functions | Activator for no-code, data-condition-driven alerts and actions; Logic Apps or Functions for complex, multi-step integration responses |

## Hands-on

1. Create an Eventhouse and an Eventstream with the **sample data** source (for example bicycle rentals). Route it to a KQL table.
2. Write KQL to summarise by 5-minute bins and render a time chart; pin it to a Real-Time Dashboard.
3. Add an Activator rule that emails you when a station has fewer than two bikes.
4. Turn on **OneLake availability** for the table and query it from a Spark notebook.
5. Optional: route Event Hubs data from a Python producer (topic 11, file 04) through an Eventstream with a filter and aggregation.

## Security, cost and reliability notes

- **Security:** Eventhouse supports database-level roles and row-level security policies. Restrict who can see raw logs that may contain prompts or PII.
- **Security:** connect to Azure sources (Event Hubs) with managed identity or Entra-based connections, and prefer private network options for sources where available.
- **Cost:** Eventstreams and Eventhouses consume CUs continuously while running. Eventhouse has an **always-on** option (minimum consumption) for guaranteed latency; without it, it can scale down when idle. Tune hot cache and retention to control storage.
- **Reliability:** RTI isn't delayed in the first throttling stage, but it's rejected when capacity is badly overloaded. Isolate critical real-time workloads on their own capacity.
- **Reliability:** use Event Hubs upstream as a durable buffer with retention, so you can replay if downstream processing fails.

## Common mistakes

- Using Real-Time Intelligence for data that is only analysed daily.
- Keeping all data in hot cache "just in case".
- Writing complex parsing logic in every query instead of update policies or functions.
- Alerting on every event rather than on conditions held over time, which causes alert storms.
- Logging full prompts and completions without data-protection review.

## Check yourself

1. Name the main components of Real-Time Intelligence and what each does.
2. Which other Microsoft services use KQL?
3. What is an update policy used for?
4. When would you store events in an Eventhouse rather than a lakehouse?
5. What does Activator do?
6. How can Spark access Eventhouse data without copying it?

<details>
<summary>Answers</summary>

1. Real-Time hub (catalogue), Eventstream (ingest, transform, route), Eventhouse/KQL database (store and query), Real-Time Dashboards (visualise), Activator (detect and act).
2. Azure Data Explorer, Azure Monitor Logs (Log Analytics), Microsoft Sentinel, Microsoft Defender advanced hunting.
3. Transforming data automatically at ingestion from a source table into a target table.
4. When you need fast interactive queries over high-volume, recent time-series or log data.
5. It monitors data streams and items for conditions and triggers actions such as alerts, flows, pipelines or notebooks.
6. By enabling OneLake availability on the KQL table (exposed as Delta) or through shortcuts.

</details>

## References

- [What is Real-Time Intelligence?](https://learn.microsoft.com/fabric/real-time-intelligence/overview)
- [Eventstreams overview](https://learn.microsoft.com/fabric/real-time-intelligence/event-streams/overview)
- [Eventhouse overview](https://learn.microsoft.com/fabric/real-time-intelligence/eventhouse)
- [Kusto Query Language overview](https://learn.microsoft.com/kusto/query/)
- [Update policy](https://learn.microsoft.com/kusto/management/update-policy)
- [What is Activator?](https://learn.microsoft.com/fabric/real-time-intelligence/data-activator/activator-introduction)
- [Real-Time hub](https://learn.microsoft.com/fabric/real-time-hub/real-time-hub-overview)
- [Eventhouse OneLake availability](https://learn.microsoft.com/fabric/real-time-intelligence/event-house-onelake-availability)
