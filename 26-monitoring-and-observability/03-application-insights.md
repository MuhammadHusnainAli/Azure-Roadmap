# Application Insights

## Why it matters

Platform metrics tell you a server is busy; they don't tell you that checkout is slow because a payment API call takes four seconds for customers in one region. **Application Insights** is Azure Monitor's application performance monitoring (APM) feature: it collects requests, dependencies, exceptions, traces, metrics and page views from your code, stitches them into **distributed traces** across services, draws an **application map** of components and dependencies, and runs **availability tests** from around the world. It's now built on **OpenTelemetry**, the vendor-neutral standard, so instrumentation is portable and the same pipeline also carries GenAI traces from Foundry agents ([Topic 22 file 06](../22-evaluation-observability-and-genaiops/06-tracing-with-opentelemetry.md)).

## Key concepts

### Telemetry types

| Type | Table (workspace-based) | Example |
|---|---|---|
| Requests | `AppRequests` | Incoming HTTP request to an API |
| Dependencies | `AppDependencies` | Outgoing SQL, HTTP, Service Bus, Azure OpenAI calls |
| Exceptions | `AppExceptions` | Unhandled errors with stack traces |
| Traces (logs) | `AppTraces` | Application log records |
| Metrics | `AppMetrics` / custom metrics | Business counters, OpenTelemetry metrics |
| Page views, browser timings | `AppPageViews`, `AppBrowserTimings` | Front-end performance |
| Availability results | `AppAvailabilityResults` | Synthetic test outcomes |

Application Insights resources are **workspace-based**: telemetry is stored in a Log Analytics workspace, queryable with KQL alongside other logs.

### Instrumentation options

| Option | Use |
|---|---|
| **Microsoft OpenTelemetry Distro** (`microsoft-opentelemetry` for Python, `@microsoft/opentelemetry` for Node.js, `Microsoft.OpenTelemetry` for .NET) | Recommended for **new** .NET, Node.js and Python apps; exports to Azure Monitor (and other OTLP backends) |
| **Azure Monitor OpenTelemetry Distro** | Existing users need no action; **Java** apps use the Azure Monitor Java agent/distro |
| **Application Insights .NET SDK 3.x** | Upgrade path for SDK 2.x users |
| **Autoinstrumentation** | Codeless for App Service, Functions, AKS (where supported) |
| **Browser SDK** | Client-side page views and user telemetry |

### Key experiences

| Experience | What it shows |
|---|---|
| **Application map** | Components and dependencies with health, call volume, latency and failures |
| **Transaction search / end-to-end transaction details** | One request's full distributed trace across services |
| **Performance and Failures** | Slowest operations, failing dependencies, exception groups |
| **Live metrics** | Real-time stream for deployments and incidents |
| **Availability** | **Standard tests** (single request with TLS certificate validation, custom headers, HTTP verbs) from multiple regions; URL ping tests are deprecated |
| **Usage** | Users, sessions, funnels, retention (browser telemetry) |
| **Smart detection / failure anomalies** | Automatic anomaly alerts |

### Distributed tracing

OpenTelemetry propagates **W3C Trace Context** (`traceparent`) across HTTP and messaging so every span shares an `operation_Id`. Set a meaningful **cloud role name** (`service.name`) per service so the application map is readable.

### Sampling

The distro samples by default (for example, a rate limit of five requests per second in current guidance) to control cost; adjust with sampling overrides. Sampling keeps whole traces together. Don't disable sampling for high-volume apps without a cost plan (file 10).

## How it works on Azure

Python (Flask/FastAPI etc.) with the Microsoft OpenTelemetry Distro:

```bash
pip install microsoft-opentelemetry
export APPLICATIONINSIGHTS_CONNECTION_STRING="<connection string>"   # from the resource; set via app settings in production
export OTEL_SERVICE_NAME="orders-api"
```

```python
import logging
from microsoft.opentelemetry import use_microsoft_opentelemetry

use_microsoft_opentelemetry(enable_azure_monitor=True, logger_name="orders")
logger = logging.getLogger("orders")
logger.setLevel(logging.INFO)

# Instrumented libraries (HTTP, requests, DB clients) now emit spans automatically
logger.info("Order service started")
```

Custom span for business logic:

```python
from opentelemetry import trace
tracer = trace.get_tracer("orders")

def place_order(order):
    with tracer.start_as_current_span("place_order") as span:
        span.set_attribute("order.items", len(order.items))
        ...
```

Create a workspace-based resource and a standard availability test with Bicep (simplified):

```bicep
resource appi 'Microsoft.Insights/components@2020-02-02' = {
  name: 'appi-orders-prod'
  location: resourceGroup().location
  kind: 'web'
  properties: { Application_Type: 'web', WorkspaceResourceId: law.id, DisableLocalAuth: true }
}
```

`DisableLocalAuth: true` requires Entra-authenticated ingestion (managed identity), preventing anyone with the connection string from sending spoofed telemetry.

KQL: slowest dependencies

```kusto
AppDependencies
| where TimeGenerated > ago(1h)
| summarize p95 = percentile(DurationMs, 95), calls = count(), failures = countif(Success == false) by Target, Type
| order by p95 desc
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Resource per app per environment** | One Application Insights resource per application per environment; shared workspace allowed. |
| **Distro choice** | Microsoft OpenTelemetry Distro for new apps; keep existing Azure Monitor distro working; migrate SDK 2.x. |
| **Sampling** | Default sampling for high-volume; higher rates for low-volume critical services. |
| **Entra ingestion** | Disable local auth in production. |
| **Front-end telemetry** | Add the browser SDK for user experience; respect consent and privacy. |
| **Multi-vendor** | OpenTelemetry lets you export to other backends too. |

## Hands-on

1. Instrument two services (web front end and API) with the distro; set service names; view the application map.
2. Trace one request end to end in transaction details.
3. Add a custom span and attribute; query it with KQL.
4. Create a standard availability test from three regions with SSL expiry check and an alert.
5. Disable local auth and switch ingestion to managed identity.

## Security, cost and reliability notes

- **PII in telemetry**: avoid logging personal data; use telemetry processors to scrub.
- **Cost**: high-volume traces and logs; sampling and log levels matter.
- **Overhead**: instrumentation adds small latency; benchmark critical paths.
- **Availability tests** come from Microsoft locations; allow them through WAF/firewalls.

## Common mistakes

| Mistake | Fix |
|---|---|
| No cloud role names | Set `service.name` |
| Logging at DEBUG in production | INFO or above; dynamic log levels |
| One resource for all environments | Separate per environment |
| Using deprecated URL ping tests | Standard tests |
| Connection strings in source code | App settings/environment variables, Entra ingestion |

## Check yourself

1. List five Application Insights telemetry types.
2. Which instrumentation is recommended for a new Python app?
3. What does the application map show?
4. How does distributed tracing correlate spans?
5. What do standard availability tests check?
6. Why disable local authentication?

<details>
<summary>Answers</summary>

1. Any five: requests, dependencies, exceptions, traces, metrics, page views, availability results.
2. The Microsoft OpenTelemetry Distro (`pip install microsoft-opentelemetry`).
3. Components and dependencies with health, volume, latency and failures.
4. W3C Trace Context propagation so spans share an operation (trace) ID.
5. Endpoint availability and response time from multiple regions, with TLS certificate validation and custom request options.
6. To require Entra-authenticated ingestion and stop spoofed telemetry from anyone holding the connection string.

</details>

## References

- [Application Insights overview](https://learn.microsoft.com/azure/azure-monitor/app/app-insights-overview)
- [Enable OpenTelemetry](https://learn.microsoft.com/azure/azure-monitor/app/opentelemetry-enable)
- [Application map](https://learn.microsoft.com/azure/azure-monitor/app/app-map)
- [Availability tests](https://learn.microsoft.com/azure/azure-monitor/app/availability)
- [Sampling](https://learn.microsoft.com/azure/azure-monitor/app/sampling)
