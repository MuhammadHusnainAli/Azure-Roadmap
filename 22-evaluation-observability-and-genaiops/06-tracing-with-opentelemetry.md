# Tracing with OpenTelemetry

## Why it matters

When an agent gives a wrong answer, the question is always "why?" Did retrieval return the wrong chunk, did the model ignore the instructions, did a tool time out, did a sub-agent loop five times? Without traces you are guessing from the final output. **Tracing** records every step of a request (model calls, tool calls, retrievals, agent hand-offs) with timing, tokens and, optionally, content. Foundry uses **OpenTelemetry** and its **GenAI semantic conventions**, stores traces in **Application Insights**, and builds evaluation, monitoring and optimisation on top of them. Tracing is the foundation everything else in this topic depends on.

## Key concepts

### OpenTelemetry vocabulary

| Concept | Meaning |
|---|---|
| **Trace** | The full journey of one request through the system |
| **Span** | One operation in the trace (an LLM call, a tool execution) with start/end time, attributes and parent |
| **Attributes** | Key-value metadata on spans (model, tokens, tool name) |
| **Semantic conventions** | Standard attribute and span names so tools interpret data consistently |
| **Exporter** | Sends spans to a backend (Azure Monitor, OTLP collector, console) |
| **Context propagation** | W3C `traceparent`/`tracestate` headers link spans across services |

### GenAI semantic conventions

The OpenTelemetry GenAI conventions (status: **Development**, so attribute names may still change) define spans and attributes for model and agent operations. Microsoft, with Cisco Outshift, contributed multi-agent extensions, integrated into Foundry, Microsoft Agent Framework, LangChain, LangGraph and the OpenAI Agents SDK.

| Type | Name | Purpose |
|---|---|---|
| Span | `invoke_agent` | An agent invocation (child `invoke_agent` spans for sub-agents) |
| Child span | `plan` | Planning or task decomposition |
| Span | `invoke_workflow` | A coordinated multi-agent workflow |
| Child span | `execute_tool` | A tool execution |
| Child span | `create_memory`, `search_memory`, `update_memory`, … | Memory operations |
| Attribute | `gen_ai.tool.definitions` | Tools available to the agent |
| Attribute | `gen_ai.tool.call.arguments` / `gen_ai.tool.call.result` | Tool inputs and outputs |

Chat spans also carry attributes such as the operation name, provider, request/response model, and input/output token usage.

### Server-side vs client-side tracing

| | Server-side | Client-side |
|---|---|---|
| **What** | Foundry records traces for agents it hosts | Your code emits spans via SDK instrumentation |
| **Setup** | Connect Application Insights to the project; no code changes | Install OpenTelemetry packages and instrument |
| **Covers** | Prompt agents and hosted agents (GA); workflows and external agents (Preview) | Your app logic, custom functions, frameworks, non-Foundry calls |
| **Recommended** | Starting point | Add when you need app-level spans or end-to-end correlation |

Traces appear in the Foundry portal **Traces** tab (last 90 days searchable) and in Application Insights. Conversation results show history, tokens, ordered actions and tool calls per conversation.

### Content recording

Prompts, responses and tool arguments are **not** recorded in client-side spans unless you enable content recording (`OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=true` or `enable_content_recording=True`). Microsoft recommends enabling it in development and disabling it in production unless you have a governed reason.

## How it works on Azure

### 1. Connect Application Insights (enables server-side tracing)

Portal: **Build → Agents → Traces → Connect**, or **Project details → Connected resources → Add connection → Application Insights**. Tracing is **off by default**; a Foundry Owner or Foundry Account Owner enables it. Users need **Log Analytics Reader** (plus **Privileged Monitoring Data Reader** if tables are protected) to view traces.

### 2. Client-side tracing with the Foundry SDK

```bash
pip install azure-ai-projects azure-identity opentelemetry-sdk azure-core-tracing-opentelemetry azure-monitor-opentelemetry
```

```python
import os
os.environ["AZURE_EXPERIMENTAL_ENABLE_GENAI_TRACING"] = "true"   # set before creating clients

from azure.ai.projects import AIProjectClient
from azure.ai.projects.telemetry import AIProjectInstrumentor, trace_function
from azure.identity import DefaultAzureCredential
from azure.monitor.opentelemetry import configure_azure_monitor
from opentelemetry import trace

project = AIProjectClient(endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"], credential=DefaultAzureCredential())
configure_azure_monitor(connection_string=project.telemetry.get_application_insights_connection_string())
AIProjectInstrumentor().instrument(enable_content_recording=False)   # keep content out in prod

tracer = trace.get_tracer("support-app")

@trace_function                       # records parameters and return value as span attributes
def lookup_order(order_id: str) -> dict:
    return {"id": order_id, "status": "shipped"}

with tracer.start_as_current_span("handle-ticket") as span:
    span.set_attribute("app.session_id", "s-123")      # pseudonymous ID, not an email
    order = lookup_order("1042")
    with project.get_openai_client(agent_name="support-agent") as client:
        r = client.responses.create(input=f"Customer asks about order {order['id']}")
        print(r.output_text)
```

Notes from the docs:

- `@trace_function` always records parameters and return values, regardless of the content-recording setting; don't decorate functions handling secrets or raw personal data.
- Trace context propagation to Azure OpenAI is on by default when tracing is enabled; **baggage** propagation is off by default and should stay off unless audited.
- A custom `SpanProcessor` can add attributes (session, tenant, app version) to every span.

### 3. Frameworks

Microsoft Agent Framework emits GenAI spans natively; LangChain/LangGraph and the OpenAI Agents SDK integrate through the documented tracing integrations ([Topic 19](../19-agent-frameworks-and-protocols/README.md)).

### 4. Local development

The **Foundry Toolkit for VS Code** runs a local OTLP collector and trace viewer (no Application Insights needed); the Aspire Dashboard is another OTLP viewer. Use a `ConsoleSpanExporter` for quick checks.

### 5. Query traces in Application Insights

```kusto
dependencies
| where timestamp > ago(1d)
| where customDimensions["gen_ai.operation.name"] == "chat"
| extend model = tostring(customDimensions["gen_ai.response.model"]),
         inTok = toint(customDimensions["gen_ai.usage.input_tokens"]),
         outTok = toint(customDimensions["gen_ai.usage.output_tokens"])
| summarize calls=count(), p95_ms=percentile(duration, 95), tokens=sum(inTok + outTok) by model
```

Attribute names follow the Development-status conventions, so verify them against your own telemetry before building alerts.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Application Insights topology** | One per environment (dev/test/prod); workspace-based, in the same region as the project where possible. |
| **Content recording** | On in dev; off in prod, or on with redaction, restricted access and short retention when evaluation of real conversations is needed. |
| **Sampling** | Keep 100% for low-volume agents; sample high-volume apps but keep all error traces. Continuous evaluation needs enough traced content to sample from. |
| **Correlation** | Propagate trace context through your API, gateway and tools; add session and app-version attributes. |
| **Entra-authenticated ingestion** | Prefer over connection-string-only ingestion where supported (Preview for Foundry agent traces). |
| **Third-party backends** | Any OTLP backend works; Foundry portal features (evaluation from traces, dashboard) need Application Insights. |

## Hands-on

1. Connect Application Insights to a dev Foundry project; run a prompt agent and view its server-side trace.
2. Add client-side tracing with the code above; confirm your `handle-ticket` span is the parent of the agent spans.
3. Toggle content recording and compare what is captured.
4. Write the KQL query above; build a workbook tile for p95 latency and tokens by model.
5. Use the Foundry Toolkit locally to trace a multi-agent Agent Framework workflow and identify the slowest `execute_tool` span.

## Security, cost and reliability notes

- **Traces are Customer Data**: they can include prompts, outputs, tool arguments and personal data. Restrict Log Analytics Reader, use protected tables and set retention ([Topic 23 file 07](../23-responsible-ai-security-and-governance/07-data-privacy-residency-and-abuse-monitoring.md)).
- **Never put secrets** in prompts, tool arguments, span attributes or baggage.
- **Inform users** about data collection where required; this is the customer's responsibility.
- **Cost**: Application Insights ingestion and retention are billed; content recording increases volume significantly. Use sampling, daily caps and tiered retention.
- **Disabling**: disconnecting Application Insights stops new collection; existing data follows the workspace retention.

## Common mistakes

| Mistake | Fix |
|---|---|
| Enabling tracing after the first incident | Connect Application Insights on day one |
| Content recording on in production by default | Off by default; governed exceptions |
| Users' emails as session IDs | Pseudonymous IDs |
| Broken parent/child links across services | Propagate W3C trace context |
| Building alerts on attribute names without checking | Conventions are Development status; validate |

## Check yourself

1. Define trace, span, attribute and exporter.
2. Name four GenAI/multi-agent span types from the semantic conventions.
3. Compare server-side and client-side tracing in Foundry.
4. How do you enable client-side tracing in Python, and how is content recording controlled?
5. What roles do users need to view traces?
6. Why should content recording usually be off in production?

<details>
<summary>Answers</summary>

1. Trace: one request's journey; span: one operation with timing and parent; attribute: key-value metadata; exporter: sends spans to a backend.
2. `invoke_agent`, `plan`, `invoke_workflow`, `execute_tool` (plus memory operation spans).
3. Server-side: automatic for Foundry-hosted agents once Application Insights is connected; client-side: SDK instrumentation in your code for app-level and non-Foundry spans.
4. Set `AZURE_EXPERIMENTAL_ENABLE_GENAI_TRACING=true`, configure Azure Monitor with the project's connection string and call `AIProjectInstrumentor().instrument()`; content via `enable_content_recording` or `OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT` (default off).
5. Access to the Foundry project and read access to Application Insights/Log Analytics (for example Log Analytics Reader; Privileged Monitoring Data Reader for protected tables).
6. Content contains user data and possibly personal or confidential information; it increases privacy risk and ingestion cost.

</details>

## References

- [Agent tracing overview](https://learn.microsoft.com/azure/foundry/observability/concepts/trace-agent-concept)
- [Set up tracing in Microsoft Foundry](https://learn.microsoft.com/azure/foundry/observability/how-to/trace-agent-setup)
- [Configure client-side tracing](https://learn.microsoft.com/azure/foundry/observability/how-to/trace-agent-client-side)
- [Tracing integrations for agent frameworks](https://learn.microsoft.com/azure/foundry/observability/how-to/trace-agent-framework)
- [Tracing and data handling](https://learn.microsoft.com/azure/foundry/observability/concepts/trace-data)
- [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
