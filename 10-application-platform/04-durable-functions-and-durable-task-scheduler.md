# Durable Functions and Durable Task Scheduler

## Why it matters

Real business processes and agent tasks are long-running: they call several services, wait for people to approve things, run steps in parallel, and must survive restarts. Writing that with queues, timers and state tables by hand is error-prone. **Durable Functions** lets you write workflows as ordinary code that the framework checkpoints and replays reliably. The **Durable Task Scheduler** is a managed back end that makes it faster and easier to operate, and the Durable Task SDKs bring the same model to Container Apps and AKS. This is now a common foundation for durable, human-in-the-loop AI agents.

## Key concepts

### Function types

| Type | Role |
|---|---|
| **Client function** | Starts, queries, signals and terminates orchestrations (often an HTTP trigger) |
| **Orchestrator function** | Defines the workflow in code; must be **deterministic** |
| **Activity function** | Does the actual work (call an API, a model, a database); can be non-deterministic |
| **Entity function** | Small, addressable, stateful objects (counters, aggregators, per-user state) with serialised operations |

### How durability works

```text
Orchestrator code runs → schedules activity A → checkpoint (history: "A scheduled") → unloads
Activity A completes → history appended ("A completed: result")
Orchestrator replays from the start, reading results from history instead of re-running A
 → schedules B → checkpoint → …
```

Because orchestrators **replay**, their code must be deterministic:

| Do not, in an orchestrator | Use instead |
|---|---|
| `datetime.now()` | `context.current_utc_datetime` |
| Random numbers, GUIDs | Generate in an activity, or use the deterministic APIs |
| I/O (HTTP, database, model calls) | Activities |
| Sleeps | Durable timers (`context.create_timer`) |
| Infinite loops | `continue_as_new` to reset history |

### Application patterns

| Pattern | Example |
|---|---|
| **Function chaining** | Extract → chunk → embed → index |
| **Fan-out/fan-in** | Summarise 500 documents in parallel, then combine |
| **Async HTTP APIs** | Client gets `202 Accepted` and a status URL; polls until done |
| **Monitor** | Poll an external job until complete, with backoff |
| **Human interaction** | Wait for an approval event with a timeout and escalation |
| **Aggregator (entities)** | Accumulate events per key into stateful entities |

### Storage back ends ("storage providers")

| Back end | Notes |
|---|---|
| Azure Storage (default) | Queues, tables and blobs in a storage account; no extra service; limited throughput and harder to inspect |
| MSSQL | SQL Server or Azure SQL; good for portability and inspection |
| Netherite | High-throughput provider using Event Hubs; being superseded |
| **Durable Task Scheduler** | **Managed, purpose-built back end and recommended choice**: higher throughput, built-in dashboard, Entra authentication, no storage tuning |

### Durable Task Scheduler (DTS)

| Feature | Detail |
|---|---|
| Model | Azure resource (scheduler) with **task hubs**; your app connects with an endpoint and managed identity |
| Dashboard | Web UI to see orchestrations, histories, inputs, outputs and failures, and to manage instances |
| Performance | Much higher orchestration throughput than the Azure Storage provider |
| SKUs | Dedicated (capacity units) and Consumption options (check current pricing and status) |
| Hosting | Durable Functions on any plan, or **Durable Task SDKs** (.NET, Python, Java, JavaScript) in Container Apps, AKS, App Service or anywhere |
| Local dev | Emulator container |

DTS separates the durable engine from Functions, so the same workflow model works in containerised apps.

### Durable agents

Agent loops (plan, call tools, wait for approval, continue) map directly onto orchestrations: each model call and tool call is an activity; approvals are external events; state survives restarts and deployments. Microsoft Agent Framework provides a Durable Functions or Durable Task integration for **durable agents** (check current status), giving long-running agents with checkpointing, retries and human-in-the-loop out of the box.

## How it works on Azure

### Python example: document pipeline with fan-out/fan-in and approval

```python
import azure.functions as func
import azure.durable_functions as df
from datetime import timedelta

app = df.DFApp(http_auth_level=func.AuthLevel.FUNCTION)

@app.route(route="review/start", methods=["POST"])
@app.durable_client_input(client_name="client")
async def start(req: func.HttpRequest, client) -> func.HttpResponse:
    instance_id = await client.start_new("review_orchestrator", client_input=req.get_json())
    return client.create_check_status_response(req, instance_id)   # 202 + status URLs

@app.orchestration_trigger(context_name="ctx")
def review_orchestrator(ctx: df.DurableOrchestrationContext):
    docs = ctx.get_input()["documents"]

    # Fan-out: summarise each document in parallel
    summaries = yield ctx.task_all([ctx.call_activity("summarise", d) for d in docs])

    # Fan-in: combine
    report = yield ctx.call_activity("combine", summaries)

    # Human in the loop: wait up to 3 days for approval
    deadline = ctx.current_utc_datetime + timedelta(days=3)
    approval = ctx.wait_for_external_event("Approval")
    timeout = ctx.create_timer(deadline)
    winner = yield ctx.task_any([approval, timeout])

    if winner == approval and approval.result.get("approved"):
        timeout.cancel()
        yield ctx.call_activity("publish", report)
        return {"status": "published"}
    return {"status": "rejected_or_timed_out"}

@app.activity_trigger(input_name="doc")
def summarise(doc: dict) -> str:
    # call a Foundry model here (keyless), with retries
    return f"summary of {doc['name']}"

@app.activity_trigger(input_name="summaries")
def combine(summaries: list) -> str:
    return "\n".join(summaries)

@app.activity_trigger(input_name="report")
def publish(report: str) -> None:
    ...
```

Approve from anywhere (a Teams card, an admin UI):

```bash
curl -X POST "$BASE/runtime/webhooks/durabletask/instances/<id>/raiseEvent/Approval?code=<key>" \
  -H "Content-Type: application/json" -d '{"approved": true, "by": "alex@contoso.com"}'
```

### Configuring the Durable Task Scheduler back end

```json
// host.json (excerpt)
{
  "version": "2.0",
  "extensions": {
    "durableTask": {
      "hubName": "%TASKHUB_NAME%",
      "storageProvider": {
        "type": "azureManaged",
        "connectionStringName": "DURABLE_TASK_SCHEDULER_CONNECTION_STRING"
      }
    }
  }
}
```

```text
DURABLE_TASK_SCHEDULER_CONNECTION_STRING = Endpoint=https://<scheduler>.durabletask.io;Authentication=ManagedIdentity
TASKHUB_NAME = default
```

Grant the app's identity the **Durable Task Data Contributor** role on the scheduler or task hub.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Durable Functions vs Logic Apps | Durable Functions for code-first workflows owned by developers; Logic Apps for low-code integration with many connectors (topic 11) |
| Durable Functions vs Durable Task SDKs | Functions when the app is already on Functions; SDKs with DTS for containerised apps on Container Apps or AKS |
| Back end | Durable Task Scheduler for new apps; Azure Storage provider only for small, cost-sensitive workloads |
| Orchestration size | Keep histories small: pass references (blob URLs) not large payloads; use sub-orchestrations and `continue_as_new` |
| Versioning | Running orchestrations replay old code; use orchestration versioning or side-by-side deployments when changing logic |
| Agents | Durable orchestration for long-running or approval-gated agent tasks; plain request-response for short chat turns |

## Hands-on

1. Create a local Functions project with the Python Durable Functions template, run it with the DTS emulator (`docker run -p 8080:8080 -p 8082:8082 mcr.microsoft.com/dts/dts-emulator:latest`), and open the dashboard on port 8082.
2. Implement the orchestration above with fake activities that sleep for a few seconds.
3. Start an instance, watch fan-out in the dashboard, then raise the `Approval` event and confirm completion.
4. Start another instance and let the timer expire (shorten it to two minutes for testing).
5. Deploy to Azure with a Durable Task Scheduler resource and managed identity.

## Security, cost and reliability notes

- **Security:** the HTTP management APIs (status, raise event, terminate) are protected by keys or auth; do not expose them publicly without authentication. Validate who is allowed to approve.
- **Security:** orchestration inputs and outputs are persisted; avoid putting secrets or unnecessary personal data in them.
- **Cost:** activities are billed like normal functions; DTS has its own pricing; histories in storage grow without purging. Purge completed instances by policy.
- **Reliability:** activities can run more than once; make them idempotent. Use retry options on activity calls for transient failures.
- **Reliability:** non-deterministic orchestrator code causes replay failures that are hard to debug.

## Common mistakes

- Calling HTTP, databases or models directly from the orchestrator.
- Using `datetime.now()` or random values in orchestrators.
- Passing large documents through orchestration inputs and outputs.
- Changing orchestrator code while instances are running, breaking replay.
- Never purging old instance history.
- Building approvals with polling loops instead of external events and durable timers.

## Check yourself

1. What are the four Durable Functions function types?
2. Why must orchestrator code be deterministic?
3. How do you implement a human approval with a timeout?
4. What does the Durable Task Scheduler add over the Azure Storage provider?
5. When would you use the Durable Task SDKs instead of Durable Functions?
6. Why are durable orchestrations a good fit for long-running AI agents?

<details>
<summary>Answers</summary>

1. Client, orchestrator, activity and entity functions.
2. Orchestrators are replayed from history to rebuild state; any non-deterministic behaviour would produce different decisions on replay.
3. Wait for an external event and a durable timer together with `task_any`; whichever completes first decides the path, and the timer is cancelled if approval arrives.
4. A managed, high-throughput back end with a dashboard for monitoring and managing orchestrations, Entra authentication, and no storage tuning.
5. When the application runs in containers on Container Apps, AKS or elsewhere rather than on Azure Functions.
6. Agent steps (model calls, tool calls) become checkpointed activities, approvals become external events, and state survives failures, restarts and long waits.

</details>

## References

- [Durable Functions overview](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-overview)
- [Orchestrator code constraints](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-code-constraints)
- [Durable Task Scheduler](https://learn.microsoft.com/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler)
- [Durable Task SDKs](https://learn.microsoft.com/azure/azure-functions/durable/durable-task-scheduler/durable-task-scheduler-sdks)
- [Human interaction pattern](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-phone-verification)
- [Orchestration versioning](https://learn.microsoft.com/azure/azure-functions/durable/durable-functions-versioning)
- [Durable agents with Microsoft Agent Framework](https://learn.microsoft.com/agent-framework/user-guide/agents/durable-agents)
