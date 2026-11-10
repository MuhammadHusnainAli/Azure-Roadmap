# Functions for AI

## Why it matters

Agents need tools, and tools need somewhere to run. Azure Functions has become one of the most practical places to host them: as **remote MCP servers**, as tool back ends for Foundry Agent Service, and as event-driven pipelines that feed knowledge bases. Functions provides an MCP trigger, bindings for AI services, serverless scaling and built-in authentication. For AI engineers this is often the fastest path from "the agent needs to look up an order" to a secure, scalable tool in production.

## Key concepts

### Ways Functions serve AI workloads

| Role | How |
|---|---|
| **Remote MCP server** | MCP tool trigger extension: each function becomes an MCP tool; Functions hosts the MCP endpoint |
| **Self-hosted MCP server** | Run an MCP server built with an official MCP SDK (Python, TypeScript, .NET, Java) on Functions as a custom handler-style app (check current status) |
| **Foundry Agent Service tools** | OpenAPI tools pointing at HTTP functions; Azure Functions tool via queues; or MCP tools pointing at your Functions MCP endpoint |
| **AI bindings** | Extension bindings for text completion, embeddings, semantic search and assistants (check the current status; many teams now call the SDK directly) |
| **Ingestion pipelines** | Blob/Event Grid triggers → extract, chunk, embed, index |
| **Event-driven AI** | Classify, summarise or route incoming messages (Service Bus, Event Hubs) |
| **Durable agents** | Long-running agent workflows with Durable Functions (file 04) |

### MCP in one paragraph

The **Model Context Protocol** (MCP) is an open protocol that lets AI applications discover and call **tools**, read **resources** and use **prompts** from servers in a standard way. Remote servers use the **Streamable HTTP** transport (older servers used Server-Sent Events). Authorisation follows OAuth 2.1 conventions. Topic 19 covers MCP in depth; here we focus on hosting.

### The MCP tool trigger

```python
import json
import azure.functions as func

app = func.FunctionApp()

tool_properties = json.dumps([
    {"propertyName": "orderId", "propertyType": "string", "description": "The order number, e.g. ORD-1042"},
])

@app.generic_trigger(
    arg_name="context",
    type="mcpToolTrigger",
    toolName="get_order_status",
    description="Return the shipping status of a customer order.",
    toolProperties=tool_properties,
)
def get_order_status(context: str) -> str:
    args = json.loads(context)["arguments"]
    order_id = args["orderId"]
    # look up the order with a managed identity (SQL, Cosmos DB, an internal API)
    return json.dumps({"orderId": order_id, "status": "shipped", "eta": "2026-10-18"})
```

The Functions host exposes an MCP endpoint at `/runtime/webhooks/mcp` (Streamable HTTP; an SSE endpoint may also be available for older clients). MCP clients (GitHub Copilot in VS Code, Foundry agents, Microsoft Agent Framework, Claude and others) list and call the tools. Decorator names and endpoint paths evolve with the extension; check the current documentation.

### Authenticating MCP servers on Functions

| Option | Notes |
|---|---|
| Function keys (`x-functions-key`) | Default; shared secret; fine for prototypes only |
| **Built-in authentication (Easy Auth) with Entra ID** | OAuth-based protection; Functions can advertise protected resource metadata so MCP clients discover how to authenticate (check current status) |
| **API Management in front** | Central token validation, rate limits, logging, and exposure as an MCP server in APIM; register in API Center for discovery |
| Network isolation | Private endpoints and VNet integration for internal-only tools |

Downstream, the function should use its **managed identity**, or **on-behalf-of** the calling user when tools must respect user permissions (for example reading a user's mailbox or documents).

### Tool design for agents

| Principle | Why |
|---|---|
| **Clear names and descriptions** | The model chooses tools from their descriptions |
| **Narrow, typed parameters** | Reduces mistakes and injection surface |
| **Least privilege** | Each tool's identity can only do what that tool needs |
| **Read and write separation** | Side-effecting tools (refund, send email) separate from read tools; require confirmation or approval |
| **Idempotency keys** | Retries and repeated agent calls must not duplicate actions |
| **Bounded outputs** | Return concise, structured results; large results as references |
| **Validation** | Validate every argument server-side; never trust the model's input |
| **Observability** | Log tool calls with correlation IDs (OpenTelemetry), without logging sensitive payloads |

### Ingestion pipeline on Functions

```text
Blob created (Event Grid) → function: extract text (Content Understanding / Document Intelligence)
  → chunk → embed (Foundry embeddings, batched, keyless)
  → push to Azure AI Search (or upsert into Cosmos DB / PostgreSQL vectors)
Blob deleted → function: delete from index
Failures → retry policy → poison queue → alert
```

For large volumes, enqueue per-document work and let a queue-triggered function scale, capped to your embedding quota.

## How it works on Azure

### Calling Foundry Models from a function (SDK, keyless)

```python
import azure.functions as func
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import OpenAI

_tp = get_bearer_token_provider(DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default")
_client = OpenAI(base_url="https://<foundry-resource>.openai.azure.com/openai/v1/", api_key=_tp)   # created once

app = func.FunctionApp()

@app.service_bus_queue_trigger(arg_name="msg", queue_name="tickets", connection="ServiceBus")
def classify(msg: func.ServiceBusMessage) -> None:
    text = msg.get_body().decode()
    r = _client.chat.completions.create(
        model="<deployment>",
        messages=[{"role": "system", "content": "Classify the ticket as billing, technical or other. Reply with one word."},
                  {"role": "user", "content": text}],
        max_tokens=3,
    )
    category = r.choices[0].message.content.strip().lower()
    ...
```

### Connecting a Foundry agent to the Functions MCP server

In Foundry Agent Service, add an **MCP tool** with the server URL `https://<func>.azurewebsites.net/runtime/webhooks/mcp` and the required authentication (a key header for prototypes; Entra-based auth for production). Configure **approval** requirements for side-effecting tools. The agent can then call `get_order_status` during a conversation.

### Deploying with azd templates

Microsoft publishes `azd` templates for remote MCP servers on Functions (Python, TypeScript, .NET, Java), including Flex Consumption, VNet and managed identity options:

```bash
azd init --template remote-mcp-functions-python
azd up
```

Template names change; search the Azure Samples gallery for "remote MCP functions".

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Functions vs Container Apps for MCP servers | Functions for tool-per-function, event-driven, minimal code; Container Apps for MCP servers built as full apps with SDKs, state or custom runtimes |
| MCP vs OpenAPI tools | MCP for reuse across many agent clients and discovery; OpenAPI where the agent platform or existing APIs favour it. APIM can expose existing APIs as MCP |
| Auth | Entra-based (Easy Auth or APIM) for anything beyond a prototype |
| User context | On-behalf-of flows when tools access user-scoped data; managed identity for system data |
| Plan | Flex Consumption with always-ready instances for interactive tools; avoid cold starts in the middle of agent turns |
| AI bindings vs SDK | SDK calls give more control and keep up with model APIs; bindings for simple cases if their status suits you |

## Hands-on

1. Create a Functions project from the remote MCP template (or add the MCP trigger example to a Python v2 app) and run it locally with `func start`.
2. In Visual Studio Code, add the local server (`http://localhost:7071/runtime/webhooks/mcp`) to your MCP configuration and ask GitHub Copilot agent mode to call `get_order_status`.
3. Deploy to Flex Consumption with `azd up`. Connect to the remote endpoint using the function key header.
4. Enable Easy Auth with Entra ID, then connect again with an OAuth-capable client.
5. Add a second, side-effecting tool (`request_refund`) and require approval for it in your agent configuration.

## Security, cost and reliability notes

- **Security:** MCP servers are powerful: they let a model act. Authenticate every call, authorise per tool, validate inputs, and log actions.
- **Security:** prompt injection can make an agent call tools maliciously. Keep side-effecting tools behind approvals and least-privilege identities.
- **Security:** never return secrets or excessive data from tools; responses go into the model's context and possibly to the user.
- **Cost:** model and embedding calls dominate; batch, cache, and cap concurrency.
- **Reliability:** tool timeouts should be short and well defined; return structured errors the agent can understand.
- **Reliability:** idempotency keys for actions, since agents and clients retry.

## Common mistakes

- Function keys as the only protection for a production MCP server.
- One tool identity with broad access to everything.
- Vague tool descriptions, so the model calls the wrong tool.
- Returning huge payloads that fill the model's context.
- Creating the OpenAI client per invocation.
- No approval step on tools that spend money or send messages.

## Check yourself

1. What does the MCP tool trigger do?
2. Name three ways to secure a remote MCP server on Functions.
3. Why should side-effecting tools be separated and gated?
4. When would you use on-behalf-of authentication in a tool?
5. Describe an event-driven ingestion pipeline on Functions.
6. When would you host an MCP server on Container Apps instead of Functions?

<details>
<summary>Answers</summary>

1. It exposes a function as an MCP tool, with name, description and typed properties, through the Functions host's MCP endpoint.
2. Function keys (prototypes only), Easy Auth with Entra ID (OAuth), API Management in front with token validation and policies, plus private networking.
3. Because models can be manipulated or make mistakes; gating with approvals and least privilege limits damage from wrong or malicious calls.
4. When the tool must act with the calling user's permissions, such as reading their documents or mailbox, so access is trimmed to that user.
5. Blob created events trigger a function that extracts text, chunks, embeds with a Foundry model and indexes into Azure AI Search; delete events remove documents; failures go to a poison queue with alerts.
6. When the server is a full application built with an MCP SDK, keeps state or sessions, needs custom runtimes, or should run alongside other containerised services.

</details>

## References

- [MCP tool trigger for Azure Functions](https://learn.microsoft.com/azure/azure-functions/functions-bindings-mcp)
- [Remote MCP servers on Azure Functions](https://learn.microsoft.com/azure/azure-functions/scenario-custom-remote-mcp-server)
- [Host MCP servers built with official SDKs on Functions](https://learn.microsoft.com/azure/azure-functions/self-hosted-mcp-servers)
- [Model Context Protocol specification](https://modelcontextprotocol.io/)
- [MCP tools in Foundry Agent Service](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/model-context-protocol)
- [Expose APIs as MCP servers in API Management](https://learn.microsoft.com/azure/api-management/export-rest-mcp-server)
- [Azure OpenAI extension for Azure Functions](https://learn.microsoft.com/azure/azure-functions/functions-bindings-openai)
