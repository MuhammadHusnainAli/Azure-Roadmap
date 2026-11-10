# Foundry SDKs and APIs

> Topic: 14 · Microsoft Foundry Platform · Tracks: E ● A ○ C – · Status checked: 2026-11

## Why it matters

Foundry exposes several endpoints and SDKs: the Foundry SDK, the OpenAI SDK, the Anthropic SDK, the Agent Framework and Foundry Tools SDKs. Picking the wrong one leads to missing features, confusing errors or extra latency. Engineers need to know which client to use for which job, how authentication works for each, and how the **OpenAI-compatible v1 API** and the **Responses API** fit in.

## Key concepts

### Endpoints of a Foundry resource

| Endpoint | Format | Used by |
|---|---|---|
| **Project endpoint** | `https://<resource>.services.ai.azure.com/api/projects/<project>` | Foundry SDK: agents, evaluations, connections, datasets, Responses API in project context |
| **OpenAI v1 endpoint** | `https://<resource>.openai.azure.com/openai/v1` | OpenAI SDK: Chat Completions, Responses, embeddings, images, audio, files, batch, fine-tuning |
| **Anthropic endpoint** | `https://<resource>.services.ai.azure.com/anthropic` | Anthropic SDK for Claude models deployed in Foundry |
| **Foundry Tools endpoints** | `https://<resource>.cognitiveservices.azure.com/...` (service-specific) | Speech, Language, Vision, Translator, Content Safety SDKs |

A standalone Azure OpenAI resource exposes only the `/openai/v1` endpoint. A custom subdomain replaces `<resource>` when configured.

### Which SDK when

| SDK | Use it for | Notes |
|---|---|---|
| **Foundry SDK** (`azure-ai-projects`) | Apps using agents, evaluations, connections, datasets, tracing, Foundry-specific tools (file search, code interpreter, web search, memory, SharePoint, Work IQ, Fabric IQ, MCP) | One project endpoint; thin client; provides an OpenAI-compatible client via `get_openai_client()` |
| **Microsoft Agent Framework** | Code-first single and multi-agent systems, hosted agents, workflows (.NET, Python) | Its `foundry` package builds on the Foundry SDK (`FoundryChatClient`) |
| **OpenAI SDK** (`openai`) | Maximum OpenAI compatibility, lowest latency, embeddings, Chat Completions with any model sold by Azure | Point `base_url` at `/openai/v1` |
| **Anthropic SDK** | Claude models in Foundry | Messages API |
| **Foundry Tools SDKs** | A specific AI service (Speech, Language, Vision, Translator, Document Intelligence, Content Safety) | Service-specific clients |

### Foundry SDK versions (checked 2026-11)

| Language | Package (new Foundry) | Classic |
|---|---|---|
| Python | `azure-ai-projects>=2.3.0` (with `openai>=3.0.0`) | `azure-ai-projects==1.0.0` |
| JavaScript/TypeScript | `@azure/ai-projects` 2.x | 1.x |
| .NET | `Azure.AI.Projects` 2.x, `Azure.AI.Projects.Agents`, `Azure.AI.Extensions.OpenAI` | `Azure.AI.Projects` 1.x |
| Java | `com.azure:azure-ai-projects` and `azure-ai-agents` 2.x | — |

Use 2.x with the new Foundry portal and Agent Service; 1.x targets Foundry (classic). In .NET, don't install the preview `Azure.AI.Projects.OpenAI` alongside the GA `Azure.AI.Extensions.OpenAI`.

### Two client types in the Foundry SDK

| Client | Purpose | Examples |
|---|---|---|
| **Project client** (`AIProjectClient`) | Foundry-native operations with no OpenAI equivalent | Connections, project properties, datasets, tracing setup, deployments listing |
| **OpenAI-compatible client** (`project.get_openai_client()`) | Operations that follow OpenAI API shapes | Responses API, agents, evaluations, fine-tuning, files |

Most apps use both: the project client for setup, the OpenAI-compatible client to run models and agents.

### The v1 API

The **Azure OpenAI v1 API** (`/openai/v1/`) is OpenAI-compatible and **doesn't require monthly `api-version` parameters**, unlike the older date-versioned API (`/openai/deployments/{name}/chat/completions?api-version=2024-xx-xx`). You can use the standard `OpenAI` client (not `AzureOpenAI`) with `base_url`, and pass the **deployment name** as `model`. New features land on v1 first. Older `api-version` endpoints still work for existing code (topic 15 file 02).

### Responses API

The **Responses API** is the recommended API for new work. It's stateful (`previous_response_id`, conversations), has built-in tools, background mode, reasoning summaries, and better support for reasoning models and agents. Chat Completions remains supported (topic 15 file 07).

## How it works on Azure

### Foundry SDK (Python)

```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

project = AIProjectClient(
    endpoint="https://<resource>.services.ai.azure.com/api/projects/<project>",
    credential=DefaultAzureCredential())

# Foundry-native operations
for conn in project.connections.list():
    print("connection:", conn.name, conn.type)

# OpenAI-compatible operations in the project context
openai = project.get_openai_client()
resp = openai.responses.create(model="gpt-5-mini", input="Summarise our refund policy in two bullets.")
print(resp.output_text)
```

### OpenAI SDK against the v1 endpoint (Python, keyless)

```python
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import OpenAI

tp = get_bearer_token_provider(DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default")
client = OpenAI(base_url="https://<resource>.openai.azure.com/openai/v1/", api_key=tp)

emb = client.embeddings.create(model="text-embedding-3-small", input=["hello", "world"])
chat = client.chat.completions.create(model="<deployment>", messages=[{"role": "user", "content": "Hi"}])
```

### JavaScript (Foundry SDK)

```javascript
import { AIProjectClient } from "@azure/ai-projects";
import { DefaultAzureCredential } from "@azure/identity";

const project = new AIProjectClient(
  "https://<resource>.services.ai.azure.com/api/projects/<project>",
  new DefaultAzureCredential());
const openai = await project.getOpenAIClient();
const r = await openai.responses.create({ model: "gpt-5-mini", input: "Hello from JS" });
console.log(r.output_text);
```

Method names differ slightly by language and SDK version; check the language's quickstart.

### Microsoft Agent Framework (Python sketch)

```python
# pip install agent-framework  (check the package name and extras for Foundry in the docs)
# The Foundry integration uses a FoundryChatClient over the project endpoint's Responses API.
```

Topic 19 covers Agent Framework code in depth.

### REST

Every SDK call maps to REST. Useful for debugging and for languages without an SDK:

```bash
TOKEN=$(az account get-access-token --resource https://cognitiveservices.azure.com --query accessToken -o tsv)
curl -s https://<resource>.openai.azure.com/openai/v1/responses \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"model":"gpt-5-mini","input":"Say hello"}'
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Foundry SDK vs OpenAI SDK | Foundry SDK when you use agents, evaluations, connections or Foundry tools; OpenAI SDK for plain model calls, embeddings and lowest latency. Both can coexist |
| Responses vs Chat Completions | Responses for new apps, reasoning models, tools and stateful conversations; Chat Completions for existing code and maximum portability |
| v1 vs date-versioned API | v1 for new code; migrate older `AzureOpenAI` + `api-version` code when convenient |
| SDK vs Agent Framework | Foundry SDK for straightforward agent use; Agent Framework for complex orchestration, multi-agent workflows and portability |
| Keys vs Entra ID | Entra ID with `DefaultAzureCredential` (or `ManagedIdentityCredential` in production) |

## Hands-on

1. Install `azure-ai-projects>=2.3.0` and `openai`, run the Foundry SDK sample, and list your project's connections.
2. Call the same deployment with the OpenAI SDK on `/openai/v1` and compare latency with the project-client path.
3. Use `curl` with an Entra token against `/openai/v1/responses`.
4. Port a legacy snippet that uses `AzureOpenAI(api_version="2024-xx-xx")` to the v1 `OpenAI(base_url=...)` pattern.
5. If you have Claude deployed, call it via the Anthropic SDK endpoint.

## Security, cost and reliability notes

- **Security:** never ship keys in client-side code; use backend services with managed identities.
- **Security:** pin SDK versions in production and review release notes; preview features in 2.x SDKs may change.
- **Cost:** SDK choice doesn't change token price, but the Responses API's stored conversations and built-in tools have their own costs (storage, tool calls).
- **Reliability:** SDKs include retry policies for transient errors and 429s. Configure them rather than writing your own loops, and set timeouts.

## Common mistakes

- Using the classic 1.x SDK with the new Foundry portal or agents.
- Mixing `AzureOpenAI` with `api-version` and the v1 `base_url` pattern in the same codebase without understanding the difference.
- Passing the model name instead of the **deployment name** as `model`.
- Using the project endpoint for operations only available at the resource endpoint (for example some Foundry Tools APIs).
- Hard-coding endpoints instead of configuration.

## Check yourself

1. What are the four main endpoints of a Foundry resource?
2. When would you use the OpenAI SDK instead of the Foundry SDK?
3. What are the two client types in the Foundry SDK?
4. What's special about the v1 API?
5. Which Python package version targets the new Foundry?
6. What value goes into the `model` parameter?

<details>
<summary>Answers</summary>

1. The project endpoint (`services.ai.azure.com/api/projects/...`), the OpenAI v1 endpoint (`openai.azure.com/openai/v1`), the Anthropic endpoint (`services.ai.azure.com/anthropic`) and Foundry Tools endpoints (`cognitiveservices.azure.com`).
2. For maximum OpenAI compatibility, lowest latency, embeddings, or Chat Completions against models sold by Azure, when you don't need Foundry-specific features.
3. The project client (`AIProjectClient`) for Foundry-native operations, and the OpenAI-compatible client (`get_openai_client()`) for Responses, agents, evaluations and fine-tuning.
4. It's OpenAI-compatible and doesn't need monthly `api-version` parameters; you can use the standard OpenAI client with a `base_url`.
5. `azure-ai-projects` 2.x (2.3.0 or later).
6. The deployment name.

</details>

## References

- [Get started with Foundry SDKs and endpoints](https://learn.microsoft.com/azure/foundry/how-to/develop/sdk-overview)
- [Azure AI Projects client library for Python](https://learn.microsoft.com/python/api/overview/azure/ai-projects-readme)
- [Azure OpenAI v1 API lifecycle](https://learn.microsoft.com/azure/foundry/openai/api-version-lifecycle)
- [Responses API](https://learn.microsoft.com/azure/foundry/openai/how-to/responses)
- [Microsoft Agent Framework overview](https://learn.microsoft.com/agent-framework/overview/agent-framework-overview)
- [Prepare your development environment](https://learn.microsoft.com/azure/foundry/how-to/develop/install-cli-sdk)
