# Container Apps for AI

> Topic: 07 · Containers and Kubernetes · Tracks: E ● A ● C ○ · Status checked: 2026-10

## Why it matters

Container Apps has become a primary host for AI workloads on Azure: chat back ends, agent services, MCP servers, retrieval pipelines and, with serverless GPUs, self-hosted model inference that scales to zero. It also provides **dynamic sessions**, isolated sandboxes in which agents can run model-generated code safely. For an AI engineer this is often the shortest path from a working prototype to a secure, scalable service.

## Key concepts

### What AI workloads look like on Container Apps

| Workload | Container Apps feature |
|---|---|
| Chat or RAG API calling Foundry Models | Container app with managed identity, HTTP scaling, `minReplicas ≥ 1` |
| Agent service (Microsoft Agent Framework, LangGraph, Semantic Kernel successor code) | Container app; state in Cosmos DB or Redis; Dapr or Service Bus for async work |
| Remote MCP server | Container app with HTTP ingress (Streamable HTTP transport), Easy Auth or API Management in front |
| Document ingestion and indexing | Event-driven job triggered by Blob or Service Bus messages |
| Batch embedding or evaluation runs | Manual or scheduled job with parallelism |
| Open-model inference | Serverless GPU (bursty) or Dedicated GPU profile (steady) |
| Running code that a model wrote | **Dynamic sessions** |

### Serverless GPUs

Serverless GPUs attach NVIDIA GPUs (T4 and A100 classes, depending on region) to the Consumption workload profile.

| Property | Detail |
|---|---|
| Billing | Per second while replicas run; scale to zero means no GPU cost when idle |
| Cold start | Model load dominates. Small models start in tens of seconds; large models take minutes |
| Quota | Requested per subscription and region; not available everywhere |
| Fit | Bursty or intermittent inference, internal tools, evaluation jobs, prototypes |
| Alternative | Dedicated GPU workload profiles for steady traffic, where per-hour pricing wins |

Practices for GPU cold start:

- Keep images small and put weights in a mounted Azure Files share or download them from Blob at start, using **artifact streaming** or caching where available.
- Use an inference server with fast loading (vLLM, Ollama, NVIDIA NIM, Triton).
- Set `minReplicas: 1` during business hours with a cron scale rule if users cannot wait.
- Use **Foundry Models** first; self-host only for the reasons in [topic 06 file 07](../06-compute/07-gpu-and-ai-infrastructure.md).

### Dynamic sessions

Agents increasingly need to run code: to analyse a CSV, plot a chart, transform data or test a snippet. Running model-generated code in your app process is dangerous. **Dynamic sessions** provide fast, isolated, Hyper-V sandboxed environments allocated on demand from a pre-warmed pool.

```text
Agent / app  ──(session identifier, Entra token)──►  Session pool
                                                      ├─ session "user-42"  (sandbox, ~ms to allocate)
                                                      ├─ session "user-77"
                                                      └─ ready pool of warm sandboxes
```

| Session type | Description |
|---|---|
| **Code interpreter sessions** | Microsoft-provided Python (and JavaScript) runtimes with common libraries; execute code and upload or download files through a REST API |
| **Custom container sessions** | Your own image as the sandbox, for custom runtimes or tools |

Key properties:

- Each session is identified by a string you choose (per user or per conversation). Calls with the same identifier reach the same sandbox until it expires after a cooldown period.
- Strong isolation: separate Hyper-V sandbox per session; network egress can be disabled.
- Pool settings: maximum concurrent sessions, ready (pre-warmed) sessions, cooldown period.
- Access with an Entra token; callers need the **Azure ContainerApps Session Executor** role on the pool.
- Integrations exist for LangChain, LlamaIndex, Semantic Kernel and Microsoft Agent Framework, and session pools can be exposed as an MCP server. Check current status of the MCP integration.

Foundry Agent Service also has a built-in code interpreter tool. Use that when building on Foundry agents; use dynamic sessions when you host your own agent or need custom runtimes, tighter control or code execution outside Foundry.

### Hosting MCP servers

A remote MCP server is an HTTP service, so Container Apps fits naturally:

| Concern | Approach |
|---|---|
| Transport | Streamable HTTP on the app's ingress |
| Authentication | Easy Auth with Entra ID, or API Management in front validating tokens (OAuth 2.1 per the MCP authorisation spec) |
| Authorisation to back ends | Managed identity, ideally with on-behalf-of flows for user-specific data |
| Scaling | HTTP scale rule; sessions may need affinity if the server keeps state |
| Discovery and governance | Register in API Center; expose through the API Management MCP features (topic 11) |

### Agent hosting choices

| Option | When |
|---|---|
| **Foundry Agent Service prompt and workflow agents** | Agents defined declaratively in Foundry; no hosting at all (topic 18) |
| **Foundry hosted agents** | Your agent code in a container, run by Foundry with its identity, memory and tools |
| **Container Apps** | Your agent code with full control of runtime, networking, scaling and dependencies |
| **AKS** | Agent platforms run by a Kubernetes team, many agents, custom controllers |
| **Azure Functions** | Event-driven, short agent steps; durable agents for long-running orchestrations (topic 10) |

## How it works on Azure

### Reference layout for an AI app

```text
Front Door + WAF ─► Container Apps environment (internal, VNet-integrated, zone redundant)
                     ├─ web-ui            (minReplicas 1)
                     ├─ agent-api         (minReplicas 2, managed identity)
                     │     ├─► Foundry Models via private endpoint (keyless)
                     │     ├─► Azure AI Search / Foundry IQ
                     │     ├─► Cosmos DB (threads, memory)
                     │     └─► Session pool (code interpreter)
                     ├─ mcp-tools         (internal ingress, Easy Auth)
                     ├─ ingest-job        (event job on Service Bus)
                     └─ llm-gpu           (serverless GPU, minReplicas 0, internal)
                    Telemetry: OpenTelemetry agent → Application Insights (GenAI traces)
```

### Calling Foundry from the container (keyless)

```python
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import OpenAI

token_provider = get_bearer_token_provider(
    DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default"
)
client = OpenAI(
    base_url="https://<foundry-resource>.openai.azure.com/openai/v1/",
    api_key=token_provider,
)
resp = client.chat.completions.create(
    model="<deployment-name>",
    messages=[{"role": "user", "content": "Summarise our refund policy in two lines."}],
)
print(resp.choices[0].message.content)
```

`DefaultAzureCredential` picks up the app's managed identity. Grant it `Cognitive Services OpenAI User` (or the Foundry user role) on the resource. If you use a user-assigned identity, set `AZURE_CLIENT_ID`.

### Calling a dynamic session

```python
import requests
from azure.identity import DefaultAzureCredential

endpoint = "<pool-management-endpoint>"            # from the session pool resource
token = DefaultAzureCredential().get_token("https://dynamicsessions.io/.default").token
r = requests.post(
    f"{endpoint}/code/execute?api-version=2024-02-02-preview&identifier=conversation-123",
    headers={"Authorization": f"Bearer {token}"},
    json={"properties": {"codeInputType": "inline", "executionType": "synchronous",
                         "code": "import math; print(math.sqrt(1764))"}},
)
print(r.json()["properties"]["stdout"])
```

API versions change; check the current one in the documentation.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Foundry Agent Service or self-hosted agent on Container Apps | Foundry first for standard agents with built-in tools, memory and governance; Container Apps when you need custom frameworks, runtimes or networking |
| Code execution | Never in the app process. Dynamic sessions or Foundry code interpreter |
| Session identifier granularity | Per conversation or per user, never shared across tenants |
| Serverless or Dedicated GPU | Serverless for intermittent use; Dedicated for sustained utilisation; managed Foundry Models before either |
| Long-running agent tasks | Async pattern: accept, enqueue, process in a worker or job, stream progress back (topic 10 file 09) |
| Streaming responses | Server-sent events through ingress work; check idle timeouts on any gateway in front |

## Hands-on

```bash
RG=rg-aca-ai; LOC=swedencentral; ENV=aca-ai-env
az group create -n $RG -l $LOC -o none
az containerapp env create -g $RG -n $ENV -l $LOC --enable-workload-profiles -o none

# 1. A Python code interpreter session pool
az containerapp sessionpool create -g $RG -n code-pool -l $LOC \
  --container-type PythonLTS --max-sessions 20 --ready-sessions 2 \
  --cooldown-period 300 --network-status EgressDisabled -o none

POOL_ID=$(az containerapp sessionpool show -g $RG -n code-pool --query id -o tsv)
ENDPOINT=$(az containerapp sessionpool show -g $RG -n code-pool --query properties.poolManagementEndpoint -o tsv)

# 2. Grant yourself the session executor role and run code
az role assignment create --assignee $(az ad signed-in-user show --query id -o tsv) \
  --role "Azure ContainerApps Session Executor" --scope $POOL_ID -o none
TOKEN=$(az account get-access-token --resource https://dynamicsessions.io --query accessToken -o tsv)
curl -s -X POST "$ENDPOINT/code/execute?api-version=2024-02-02-preview&identifier=demo-1" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"properties":{"codeInputType":"inline","executionType":"synchronous","code":"import platform; print(platform.platform()); print(sum(range(10**6)))"}}' | jq .properties

# 3. (If you have serverless GPU quota) add a GPU profile and deploy a small model server
# az containerapp env workload-profile add -g $RG -n $ENV --workload-profile-name gpu \
#   --workload-profile-type Consumption-GPU-NC8as-T4
# az containerapp create -g $RG -n llm --environment $ENV --workload-profile-name gpu \
#   --image <your-acr>/ollama-phi:1.0 --ingress internal --target-port 11434 \
#   --min-replicas 0 --max-replicas 1 --cpu 8 --memory 56Gi

az group delete -n $RG --yes --no-wait
```

Try sending code with a network call (`import urllib.request; urllib.request.urlopen("https://example.com")`) and confirm egress is blocked.

## Security, cost and reliability notes

- **Security:** treat all model-generated code as hostile. Isolate it in sessions with egress disabled, and never pass secrets into a session.
- **Security:** one session identifier per user or conversation. Reusing identifiers leaks files and state between users.
- **Security:** authenticate MCP servers. An unauthenticated MCP endpoint is a remote code or data access endpoint for anyone who finds it.
- **Cost:** the biggest AI cost is usually tokens, not containers. Instrument token usage per request.
- **Cost:** GPU profiles left at `minReplicas: 1` cost money 24/7. Use schedules.
- **Reliability:** set request timeouts and retries with backoff for model calls; handle `429` throttling from Foundry with the `retry-after` header.
- **Reliability:** long LLM calls and streaming need generous timeouts at ingress and any gateway.

## Common mistakes

- Executing model-written code with `exec()` in the API container.
- Sharing one session across all users.
- Hosting a self-built model on a GPU when a Foundry model would do.
- Using API keys stored as secrets instead of managed identity.
- Scale to zero on the main chat API, so every first user waits for a cold start.
- No authentication on internal MCP servers because "it is internal".

## Check yourself

1. What are dynamic sessions and why do agents need them?
2. Which role does a caller need to execute code in a session pool?
3. How does the session identifier affect isolation?
4. When are serverless GPUs a better fit than Dedicated GPU profiles?
5. How should a container app authenticate to Foundry Models?
6. Name three ways to host an agent on Azure, and when Container Apps is the right one.
7. What must you add in front of a remote MCP server on Container Apps?

<details>
<summary>Answers</summary>

1. On-demand, Hyper-V isolated sandboxes from a pre-warmed pool. Agents need them to run untrusted, model-generated code without risking the host application.
2. Azure ContainerApps Session Executor on the session pool.
3. Calls with the same identifier reach the same sandbox and share files and state; different identifiers are isolated. Use one per user or conversation.
4. For bursty or intermittent workloads, where scale to zero avoids paying for idle GPUs.
5. With its managed identity and `DefaultAzureCredential`, after granting the identity a Foundry or OpenAI user role. No keys.
6. Foundry Agent Service (prompt, workflow, hosted agents), Container Apps, AKS, Azure Functions. Container Apps fits when you need your own framework and runtime with control of networking and scaling but not Kubernetes itself.
7. Authentication and authorisation: Easy Auth with Entra ID or API Management validating OAuth tokens, plus ideally a WAF or private ingress.

</details>

## References

- [Serverless GPUs in Azure Container Apps](https://learn.microsoft.com/azure/container-apps/gpu-serverless-overview)
- [Dynamic sessions](https://learn.microsoft.com/azure/container-apps/sessions)
- [Code interpreter sessions](https://learn.microsoft.com/azure/container-apps/sessions-code-interpreter)
- [Custom container sessions](https://learn.microsoft.com/azure/container-apps/sessions-custom-container)
- [AI integration with Container Apps](https://learn.microsoft.com/azure/container-apps/ai-integration)
- [Deploy an MCP server to Container Apps](https://learn.microsoft.com/azure/container-apps/tutorial-mcp-server-python)
- [Foundry hosted agents](https://learn.microsoft.com/azure/foundry/agents/concepts/hosted-agents)
