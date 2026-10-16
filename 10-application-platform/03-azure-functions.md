# Azure Functions

> Topic: 10 · Application Platform · Tracks: E ● A ● C ○ · Status checked: 2026-10

## Why it matters

Azure Functions runs small pieces of code in response to events: HTTP requests, queue messages, blob uploads, timers, database changes. You write the handler; the platform handles servers, scaling and integration plumbing. Functions is the glue of many Azure solutions and a natural host for AI tasks such as document ingestion, embedding pipelines, agent tools and MCP servers. Choosing the right hosting plan and understanding scaling and cold start are what make it work in production.

## Key concepts

### Triggers and bindings

| Concept | Meaning |
|---|---|
| **Trigger** | What starts the function (exactly one per function) |
| **Input binding** | Declaratively read data when the function runs |
| **Output binding** | Declaratively write data when the function returns |

Common triggers and bindings:

| Service | Trigger | Input | Output |
|---|---|---|---|
| HTTP | ✓ | | ✓ (response) |
| Timer | ✓ | | |
| Blob Storage | ✓ (Event Grid-based recommended) | ✓ | ✓ |
| Queue Storage | ✓ | | ✓ |
| Service Bus | ✓ | | ✓ |
| Event Hubs | ✓ | | ✓ |
| Event Grid | ✓ | | ✓ |
| Cosmos DB | ✓ (change feed) | ✓ | ✓ |
| Azure SQL | ✓ (change tracking) | ✓ | ✓ |
| Durable Functions | Orchestration, activity, entity triggers | | |
| MCP | ✓ (MCP tool trigger) | | |

Bindings support **identity-based connections**: configure `<CONNECTION>__fullyQualifiedNamespace` or `__serviceUri` settings and a managed identity, not connection strings.

### Programming models

| Language | Current model |
|---|---|
| Python | **v2 programming model** (decorators in code) |
| Node.js | **v4 programming model** |
| .NET | **Isolated worker model**. The in-process model reaches end of support on **10 November 2026**; migrate now |
| Java, PowerShell | Supported; custom handlers for any other language (Go, Rust) |

### Hosting plans

| | **Flex Consumption** | Consumption (legacy) | **Premium (Elastic Premium)** | **Dedicated (App Service plan)** | **Container Apps** |
|---|---|---|---|---|---|
| Scale to zero | Yes | Yes | No (minimum 1 instance) | No | Yes |
| Cold start mitigation | **Always-ready instances** | None | Pre-warmed instances | Always on | minReplicas |
| Scale | Fast, per-function scaling, high max instance count | Moderate | Elastic, event-driven | Manual or autoscale | KEDA |
| VNet integration | **Yes** | No | Yes | Yes | Yes |
| Instance memory | Selectable sizes (for example 512 MB, 2 GB, 4 GB) | 1.5 GB | Up to 14 GB | Plan size | Container size |
| Max execution time | Long (HTTP limited by front-end timeout) | 10 min | Unbounded (practically) | Unbounded | Unbounded |
| Billing | Per execution time and memory (on-demand) plus always-ready baseline | Per execution | Per instance-hour | Plan cost | Container Apps pricing |
| OS | Linux | Windows, Linux | Windows, Linux | Windows, Linux | Linux |
| Fit | **Default for new serverless apps** | Existing Windows apps | Constant load needing no cold start, Windows | Reuse existing plans, predictable cost | Functions alongside containers, GPUs |

The **Linux Consumption plan is legacy**: it receives no new features and has an announced retirement (September 2028). Use **Flex Consumption** for new Linux serverless apps. Windows Consumption remains available for now.

### Scaling and concurrency

- The **scale controller** watches event sources (queue length, partitions, HTTP rate) and adds instances.
- **Flex Consumption** scales **per function** for HTTP, Blob (Event Grid) and Durable triggers, and by trigger groups for others.
- **Concurrency** per instance (for example HTTP concurrency, `maxConcurrentCalls` for Service Bus, batch size for queues) determines how much work each instance takes. Dynamic concurrency can tune it.
- Limit maximum scale (`maximumInstanceCount`) to protect downstream databases and model quotas.

### Cold start

The delay when a new instance starts: allocate, load runtime, load your code and dependencies, initialise clients.

| Mitigation | How |
|---|---|
| Always-ready instances (Flex) or pre-warmed (Premium) | Keep warm capacity |
| Smaller deployment packages, fewer imports at module load | Faster startup |
| Initialise clients once, outside the handler | Reuse connections |
| Avoid heavy frameworks for simple functions | Less to load |
| Keep HTTP-critical paths on warm plans | Latency-sensitive APIs |

### Reliability semantics

| Trigger type | Retry behaviour |
|---|---|
| Queue Storage, Service Bus | Built-in redelivery; poison queues or dead-letter after max attempts |
| Event Hubs, Cosmos DB change feed | Checkpointing; configure **retry policies** to avoid skipping events on failure |
| HTTP | Client retries |
| Timer | Next schedule; catch-up optional |

Functions may run more than once for the same event: make handlers **idempotent**.

## How it works on Azure

### Python v2 example: HTTP and queue functions with identity-based connections

```python
import json, logging
import azure.functions as func

app = func.FunctionApp(http_auth_level=func.AuthLevel.FUNCTION)

@app.route(route="ingest", methods=["POST"])
@app.queue_output(arg_name="msg", queue_name="ingest", connection="Storage")
def ingest(req: func.HttpRequest, msg: func.Out[str]) -> func.HttpResponse:
    body = req.get_json()
    msg.set(json.dumps({"blob": body["blob"]}))
    return func.HttpResponse("queued", status_code=202)

@app.queue_trigger(arg_name="m", queue_name="ingest", connection="Storage")
def process(m: func.QueueMessage) -> None:
    job = json.loads(m.get_body())
    logging.info("processing %s (dequeue count %s)", job["blob"], m.dequeue_count)
```

App settings for identity-based connection (no connection string):

```text
Storage__queueServiceUri = https://<account>.queue.core.windows.net
Storage__credential      = managedidentity
```

Grant the function app's identity `Storage Queue Data Contributor`.

### Create a Flex Consumption app

```bash
az functionapp create -g $RG -n $FUNC --storage-account $SA \
  --flexconsumption-location $LOC --runtime python --runtime-version 3.12 \
  --instance-memory 2048 --assign-identity [system]
az functionapp scale config always-ready set -g $RG -n $FUNC --settings http=1
az functionapp scale config set -g $RG -n $FUNC --maximum-instance-count 40
```

The host storage account can itself be accessed by managed identity (`AzureWebJobsStorage__accountName`).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Functions vs Container Apps vs App Service | Functions for event-driven, short tasks with bindings; Container Apps for containerised services and long jobs; App Service for full web apps |
| Plan | Flex Consumption by default; Premium for Windows or constant load; Dedicated to reuse spare App Service capacity; Container Apps to co-locate with containers or use GPUs |
| HTTP APIs on Functions | Fine for lightweight APIs; front with API Management for many APIs; mind cold start |
| Long tasks | Do not block HTTP; enqueue and process asynchronously, or use Durable Functions |
| Downstream protection | Cap max instances and concurrency to respect database and model quotas |
| Organisation | Group functions that deploy and scale together in one app; separate apps for different scaling or security needs |

## Hands-on

```bash
# Prerequisites: Azure Functions Core Tools v4, Python 3.12
func init fnlab --python -m V2 && cd fnlab
func new --name hello --template "HTTP trigger" --authlevel anonymous
func start           # test locally at http://localhost:7071/api/hello?name=you

RG=rg-func-lab; LOC=swedencentral; SA=stfunc$RANDOM; FUNC=func-lab-$RANDOM
az group create -n $RG -l $LOC -o none
az storage account create -g $RG -n $SA -l $LOC --sku Standard_LRS -o none
az functionapp create -g $RG -n $FUNC --storage-account $SA --flexconsumption-location $LOC \
  --runtime python --runtime-version 3.12 -o none
func azure functionapp publish $FUNC
curl -s "https://$FUNC.azurewebsites.net/api/hello?name=azure"

cd .. && az group delete -n $RG --yes --no-wait
```

Extensions: add the queue example above with identity-based connections; send 500 messages and watch instance counts in Application Insights Live Metrics; then set the maximum instance count to 2 and observe the backlog drain more slowly.

## Security, cost and reliability notes

- **Security:** identity-based connections everywhere; Key Vault references for remaining secrets; disable function keys for internal APIs and use Easy Auth or API Management instead (function keys are shared secrets).
- **Security:** VNet integration (Flex, Premium, Dedicated) for private dependencies; private endpoints for inbound.
- **Cost:** Flex Consumption bills on execution and memory; always-ready instances add a baseline. Premium bills for at least one instance all the time.
- **Cost:** unbounded scale can multiply downstream costs (database RUs, model tokens). Set maximums.
- **Reliability:** idempotent handlers, retry policies, poison and dead-letter handling, and Application Insights alerts on failures.

## Common mistakes

- Starting new apps on the Linux Consumption plan.
- Staying on the .NET in-process model past its end of support.
- Connection strings with account keys instead of identity-based connections.
- Long-running HTTP functions that hit timeouts; use async patterns.
- Creating clients (HTTP, database, OpenAI) inside the handler on every call.
- Unlimited scale overwhelming a database or exhausting model quota.
- Non-idempotent handlers producing duplicates on retry.

## Check yourself

1. What is the difference between a trigger and a binding?
2. Why is Flex Consumption the default plan for new apps?
3. What happens to the Linux Consumption plan and the .NET in-process model?
4. Name three ways to reduce cold start.
5. Why should you cap maximum instances?
6. How do identity-based connections for bindings work?
7. Why must function handlers be idempotent?

<details>
<summary>Answers</summary>

1. A trigger starts the function (one per function); bindings declaratively read input or write output.
2. It scales to zero, supports VNet integration, scales fast per function, offers always-ready instances to reduce cold start, and has selectable instance memory.
3. Linux Consumption is legacy with an announced retirement; the .NET in-process model reaches end of support on 10 November 2026. Migrate to Flex Consumption and the isolated worker model.
4. Always-ready or pre-warmed instances; smaller packages and fewer imports; initialising clients once outside the handler.
5. To protect downstream systems (databases, APIs, model quotas) and control cost.
6. Settings such as `<name>__fullyQualifiedNamespace` or `__serviceUri` tell the binding where to connect, and the function app's managed identity, granted data roles, authenticates.
7. Events can be delivered or processed more than once due to retries and at-least-once semantics.

</details>

## References

- [Azure Functions overview](https://learn.microsoft.com/azure/azure-functions/functions-overview)
- [Hosting options](https://learn.microsoft.com/azure/azure-functions/functions-scale)
- [Flex Consumption plan](https://learn.microsoft.com/azure/azure-functions/flex-consumption-plan)
- [Triggers and bindings](https://learn.microsoft.com/azure/azure-functions/functions-triggers-bindings)
- [Identity-based connections](https://learn.microsoft.com/azure/azure-functions/functions-reference#configure-an-identity-based-connection)
- [Python developer guide (v2 model)](https://learn.microsoft.com/azure/azure-functions/functions-reference-python)
- [Migrate .NET apps to the isolated worker model](https://learn.microsoft.com/azure/azure-functions/migrate-dotnet-to-isolated-model)
- [Reliable event processing](https://learn.microsoft.com/azure/azure-functions/functions-reliable-event-processing)
- [Linux Consumption plan retirement](https://learn.microsoft.com/azure/azure-functions/consumption-plan)
