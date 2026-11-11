# Model Context Protocol

## Why it matters

Before MCP, every agent framework and every tool needed custom integration code: N frameworks × M tools. The **Model Context Protocol (MCP)** is an open standard that defines how AI applications discover and call tools, read resources and use prompt templates from external servers, so one MCP server works with any MCP client: Foundry Agent Service, Agent Framework, GitHub Copilot, Visual Studio Code, Claude, Copilot Studio and others. Microsoft has adopted MCP across Azure (Foundry toolboxes, Azure MCP Server, Azure AI Search knowledge bases, Work IQ, API Management). Engineers must understand the protocol to build and debug servers; architects must understand its security model, which is where most risk lies.

## Key concepts

### Roles

| Role | What it is | Example |
|---|---|---|
| **Host** | The LLM application the user interacts with | Foundry agent, VS Code, Copilot Studio |
| **Client** | Connector inside the host that talks to one server | The MCP client in Agent Framework |
| **Server** | Service exposing context and capabilities | GitHub MCP server, Azure MCP Server, your Functions-hosted server |

Messages use **JSON-RPC 2.0**.

### Server features

| Feature | Controlled by | Purpose |
|---|---|---|
| **Tools** | Model | Functions the model can call (`tools/list`, `tools/call`), with JSON Schema input and optional output schemas and structured content |
| **Resources** | Application/user | Context and data (files, records) identified by URIs (`resources/list`, `resources/read`, templates) |
| **Prompts** | User | Templated messages and workflows (`prompts/list`, `prompts/get`) |

Client features let the server ask for more input from the user (**elicitation**) and, in earlier revisions, request LLM completions (**sampling**) or the client's file roots (**roots**).

### Transports

| Transport | Use |
|---|---|
| **stdio** | Client launches the server as a local subprocess; newline-delimited messages over standard streams. Local tools, IDEs, CLIs. |
| **Streamable HTTP** | Each message is an HTTP POST to one MCP endpoint; replies are JSON or a request-scoped SSE stream. Remote servers on Azure. |

(The older HTTP+SSE transport was replaced by Streamable HTTP in 2025.)

### Specification versions

MCP is versioned by date. Recent revisions: **2025-06-18**, **2025-11-25**, and **2026-07-28** (current at the time of writing). The 2026-07-28 revision made significant changes:

| Change | Effect |
|---|---|
| **Stateless protocol** | No `initialize` handshake or protocol sessions (`Mcp-Session-Id` removed); every request carries protocol version and client capabilities in `_meta` |
| **`server/discover`** | Servers must advertise supported versions, capabilities and identity |
| **Multi Round-Trip Requests (MRTR)** | Replaces server-initiated requests (sampling, roots, elicitation): servers return `input_required` results and the client retries with the input |
| **`subscriptions/listen`** | One opt-in stream for change notifications (tool list changed, resource updates) |
| **Tasks as an extension** | Long-running asynchronous operations with polling (`tasks/get`, `tasks/update`) |
| **Extensions** | Opt-in modules such as Tasks, **Skills over MCP** and **MCP Apps** (inline interactive UI) |
| **Caching hints** | `ttlMs` and `cacheScope` on list and read results; deterministic tool ordering helps prompt caching |
| **Telemetry** | OpenTelemetry trace context in `_meta` |

Clients and servers negotiate versions, and ecosystems adopt new revisions gradually: check which revision your SDKs, servers and Foundry features support (Agent Framework, for example, has an MCP 2.x migration guide).

### Authorisation (HTTP transports)

| Element | Requirement |
|---|---|
| Model | A protected MCP server is an **OAuth 2.1 resource server**; the MCP client is an OAuth 2.1 client |
| Discovery | **Protected Resource Metadata** (RFC 9728) tells clients which authorisation server to use |
| Audience binding | **Resource indicators** (RFC 8707) so tokens are issued for the specific MCP server |
| Client registration | **Client ID Metadata Documents** (recommended) or Dynamic Client Registration |
| Security | PKCE, issuer validation (`iss`), credentials bound to the issuing authorisation server |

On Azure this maps to **Microsoft Entra ID** as the authorisation server, with tokens audience-scoped to the server's app registration. Foundry adds managed options: key, Entra (agent or project identity), and OAuth identity passthrough (on-behalf-of) through connections and toolboxes ([Topic 18 file 07](../18-agents-with-foundry-agent-service/07-mcp-tools-toolboxes-and-skills.md)).

### Security risks

| Risk | Description | Mitigation |
|---|---|---|
| **Tool poisoning** | Malicious instructions hidden in tool descriptions or results | Vet servers, pin versions, review tool lists, content safety on outputs |
| **Prompt injection via resources** | Retrieved content steers the model | Treat outputs as data; Prompt Shields |
| **Confused deputy / token passthrough** | Server reuses a client token for other services | Audience-bound tokens; never pass through tokens not issued for the server |
| **Excessive permissions** | Server exposes powerful tools | `allowed_tools`, approvals, least-privilege identities |
| **Supply chain** | Compromised third-party servers | Trusted publishers, private catalogues, API Center governance |
| **Local server risk** | stdio servers run code on the user's machine | Only run trusted packages |

## How it works on Azure

| Azure capability | MCP role |
|---|---|
| Foundry Agent Service MCP tool and **toolboxes** | Client and managed MCP endpoint |
| **Azure MCP Server** | Server for managing Azure resources ([Topic 04 file 10](../04-tooling-and-infrastructure-as-code/10-azure-mcp-server-and-ai-assisted-operations.md)) |
| Azure AI Search knowledge bases | Each knowledge base exposes an MCP endpoint |
| Work IQ, managed connectors | MCP servers over Microsoft 365 and 1,000+ connectors |
| Azure Functions, Container Apps, App Service | Host your own servers (file 05) |
| API Management | Expose REST APIs as MCP servers and govern MCP traffic |
| Azure API Center | Register and discover MCP servers in an organisational catalogue |
| Agent Framework | Local (stdio) and hosted MCP tools; can also expose agents as MCP servers |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **MCP vs OpenAPI vs function calling** | MCP when the tool should be reusable across many hosts and frameworks; OpenAPI when an existing REST API with a spec is enough; function calling for in-app logic. |
| **stdio vs Streamable HTTP** | stdio for local developer tools; Streamable HTTP for shared, remote, governed servers. |
| **Granularity** | Few well-described tools beat dozens of thin wrappers; return structured content. |
| **Spec revision** | Target the revision your clients support; plan for the stateless 2026-07-28 model (no server-side sessions; use explicit handles for state). |
| **Auth** | Entra ID OAuth with audience-scoped tokens; OBO when tools act on user data. |
| **Governance** | Central registry (API Center or Foundry private tool catalogue), approval for new servers, monitoring. |

## Hands-on

Build and test a tiny MCP server with the official Python SDK.

```python
# pip install "mcp[cli]"
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("contoso-orders")

@mcp.tool()
def get_order_status(order_id: str) -> dict:
    """Get the delivery status of a Contoso order by ID."""
    return {"order_id": order_id, "status": "out for delivery", "eta": "17:00"}

@mcp.resource("policy://returns")
def returns_policy() -> str:
    """Contoso returns policy text."""
    return "Unused items can be returned within 30 days."

if __name__ == "__main__":
    mcp.run()             # stdio by default; use transport="streamable-http" for remote
```

1. Run it with the MCP Inspector (`mcp dev server.py`) and call the tool.
2. Add it to VS Code or another MCP client as a local stdio server and ask "where is order 4471?".
3. Switch to Streamable HTTP and connect an Agent Framework agent using its MCP tool support.
4. Review the request messages: tool list, call, result; identify which spec revision the SDK negotiates.
5. File 05 deploys this to Azure with Entra authentication.

## Security, cost and reliability notes

- **Never pass through tokens** that weren't issued for your server; validate audience and issuer.
- **Least privilege**: expose only needed tools; require approvals for side effects; log every call with identity.
- **Data flow**: everything in tool arguments and results passes between host and server; classify data accordingly.
- **Cost**: each tool definition consumes tokens in the model context; use deterministic, concise tool lists and tool search in large catalogues.
- **Reliability**: time-outs (Foundry's synchronous limit is 100 seconds), Tasks extension or background mode for long jobs; stateless servers scale horizontally more easily.

## Common mistakes

| Mistake | Fix |
|---|---|
| Treating MCP servers as trusted code | Vet, pin and monitor them |
| Remote servers without auth | OAuth 2.1 with Entra; audience-bound tokens |
| Relying on server-side sessions in new designs | Use explicit state handles (stateless spec) |
| Exposing every API operation as a tool | Curate task-level tools |
| Ignoring spec version mismatches | Check client and server revision support |

## Check yourself

1. Define host, client and server in MCP.
2. What are the three server features, and who controls each?
3. Compare the stdio and Streamable HTTP transports.
4. Name four changes in the 2026-07-28 specification.
5. Which OAuth standards does MCP authorisation build on?
6. What is tool poisoning, and how do you mitigate it?
7. When would you choose MCP over an OpenAPI tool?
8. Name four Azure services that act as MCP servers or clients.

<details>
<summary>Answers</summary>

1. Host: the LLM application; client: the connector inside the host for one server; server: the service exposing tools, resources and prompts.
2. Tools (model-controlled), resources (application/user-controlled), prompts (user-controlled).
3. stdio runs a local subprocess with newline-delimited messages over standard streams; Streamable HTTP sends each message as an HTTP POST to one endpoint with JSON or SSE responses, suitable for remote servers.
4. Any four: stateless requests without initialize handshake or sessions, `server/discover`, Multi Round-Trip Requests replacing server-initiated requests, `subscriptions/listen`, tasks moved to an extension, extensions such as MCP Apps and Skills over MCP, caching hints, OpenTelemetry context.
5. OAuth 2.1, Protected Resource Metadata (RFC 9728), resource indicators (RFC 8707), Client ID Metadata Documents or Dynamic Client Registration, PKCE.
6. Malicious instructions hidden in tool descriptions or outputs that steer the model; vet and pin servers, review tool lists, restrict tools and apply content safety.
7. When the tool should be reusable across many hosts and frameworks with standard discovery and auth.
8. Any four: Foundry Agent Service/toolboxes, Azure MCP Server, Azure AI Search knowledge bases, Work IQ, managed connectors, API Management, Azure Functions-hosted servers, Agent Framework.

</details>

## References

- [MCP specification (2026-07-28)](https://modelcontextprotocol.io/specification/2026-07-28)
- [Specification changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
- [Transports](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports)
- [Authorization](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [Connect agents to MCP servers (Foundry)](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/model-context-protocol)
- [Local MCP tools in Agent Framework](https://learn.microsoft.com/agent-framework/agents/tools/local-mcp-tools)
