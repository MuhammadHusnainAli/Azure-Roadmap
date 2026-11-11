# Custom Tools

## Why it matters

Built-in tools cover common needs, but real agents must reach **your** systems: order databases, ERP APIs, ticketing, internal microservices, line-of-business workflows. Foundry Agent Service gives several ways to connect them, each with different execution location, authentication model and operational profile: **function calling**, **OpenAPI tools**, **Azure Functions** tools and **managed connectors** (exposed as managed MCP servers, alongside Logic Apps-style connectors). Choosing well determines security (who holds the credentials), latency, reliability and how much glue code you maintain. MCP tools are covered in file 07.

## Key concepts

### Options at a glance

| Option | Where code runs | Who calls the API | Auth | Best for |
|---|---|---|---|---|
| **Function calling** | **Your application** | Your code, after the model returns a `function_call` | Whatever your app uses | Logic that must run in your app's context, or access user session state |
| **OpenAPI tool** | Your API | **Foundry service** calls the API directly from the spec | Anonymous, API key/token (project connection), **project managed identity** | Existing REST APIs with OpenAPI 3.0/3.1 specs |
| **Azure Functions tool** | Azure Functions | Agent places a message on an **input queue**; Function processes and writes to an **output queue** | Managed identity to Storage queues | Asynchronous, serverless custom logic; standard setup only |
| **Managed connectors** (Preview) | Connector Namespace (managed by Foundry) | Foundry, through a **managed MCP server** | OAuth2 (and others) to the SaaS | 1,000+ SaaS and line-of-business connectors |
| **MCP server** | Your or third-party MCP server | Foundry or your hosted agent | Key, Entra, OAuth passthrough | File 07 |
| **A2A** | Another agent | Foundry | Agent auth | File 12 |

### Function calling

- You define a JSON schema (`FunctionTool` with `name`, `description`, `parameters`, optional `strict`).
- The model returns a `function_call` item with arguments; **your code executes** it and returns a `function_call_output` with the `call_id`.
- **Pending runs expire 10 minutes** after creation; submit outputs promptly.
- `strict: true` enforces schema adherence (not supported with the Copilot harness).
- Write descriptions for the model: what the function does, when to use it, units and formats.

### OpenAPI tool

Requirements from the spec:

- OpenAPI **3.0 or 3.1**.
- Every operation needs an **`operationId`** (letters, `-` and `_` only) and a descriptive name.
- Request bodies: `application/json` or `application/json-patch+json`.

Authentication options:

| Auth | Setup | Use when |
|---|---|---|
| **Anonymous** | Connection with `authType: None` or `OpenApiAnonymousAuthDetails` | Public, read-only APIs |
| **API key / bearer token** | Project connection storing the secret (custom keys) | Third-party APIs using keys |
| **Project managed identity** | Connection with `authType: ProjectManagedIdentity`, target URL and **audience** expected by the API; assign the least-privileged role on the target | Azure-hosted APIs secured with Entra ID (APIM, App Service, Functions, Logic Apps) |

Creating connections requires **Foundry Project Manager**; running agents requires **Foundry User**. Microsoft recommends adding OpenAPI tools through a **toolbox** for reuse and central credential management.

### Azure Functions tool (queue-based)

```text
agent ──(AzureFunctionTool)──► input queue ──► Azure Function (your code) ──► output queue ──► agent
```

- Defined with `AzureFunctionTool`, pointing at input and output Storage queues.
- **Flex Consumption** plan recommended: scale to zero, identity-based access, VNet support.
- Supported in **standard agent setup**, not basic.
- Good for longer-running or batch-like operations; adds queue latency.

For synchronous HTTP Functions, expose them with an OpenAPI spec (OpenAPI tool) or as an **MCP server on Functions** ([Topic 19 file 05](../19-agent-frameworks-and-protocols/05-building-and-hosting-mcp-servers-on-azure.md)).

### Managed connectors (managed MCP servers, Preview)

The Foundry Tools Catalog offers **over 1,000 connectors** (SaaS, data and line-of-business systems, the same connector ecosystem as Logic Apps and Power Platform). Adding one makes Foundry provision a **managed MCP server** in your account's **Connector Namespace**:

1. Browse the catalogue.
2. Connect (OAuth2 to the service).
3. Select which actions to expose as tools.
4. Add to the agent or toolbox.

Data handling depends on the publisher tier: Microsoft internal services (SharePoint, Teams, Dynamics 365) stay on Microsoft infrastructure; external and third-party connectors (GitHub, Docusign, Databricks, Box) send data to the provider under its policies; independent publishers have a lower certification bar.

### Logic Apps

Logic Apps can be called from an agent through an OpenAPI tool (with Entra authorisation on the workflow) and, conversely, Logic Apps workflows can call Foundry agents as steps. With Foundry workflows retiring, Logic Apps is Microsoft's suggested low-code orchestration canvas ([Topic 11 file 05](../11-integration-messaging-and-apis/05-azure-logic-apps.md)).

## How it works on Azure

OpenAPI tool with project managed identity, attached directly:

```python
from azure.ai.projects.models import (PromptAgentDefinition, OpenApiTool, OpenApiFunctionDefinition,
    OpenApiProjectConnectionAuthDetails, OpenApiProjectConnectionSecurityScheme)

orders_tool = OpenApiTool(openapi=OpenApiFunctionDefinition(
    name="orders_api",
    description="Look up orders and shipments for Contoso customers.",
    spec=orders_spec,                                       # dict loaded with jsonref
    auth=OpenApiProjectConnectionAuthDetails(
        security_scheme=OpenApiProjectConnectionSecurityScheme(project_connection_id=ORDERS_CONN_ID))))
```

The connection (category OpenAPI, `authType: ProjectManagedIdentity`, target `https://api.contoso.com`, audience `api://contoso-orders`) is created once by a project manager; the API validates the token's audience and the project identity's role.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Function calling vs OpenAPI** | Function calling when your app must execute the logic (user context, local state, non-HTTP); OpenAPI when the service can call the API directly and you want less app code. |
| **Who holds credentials** | Prefer project managed identity or toolbox-managed auth; avoid keys in connections where Entra is possible. |
| **Expose an API gateway** | Put API Management in front of internal APIs for rate limits, auth translation and logging; give the agent a narrow, agent-specific API product. |
| **Read vs write operations** | Separate them; require human approval or idempotency keys for writes. |
| **Granularity** | Prefer task-level operations ("get_order_status") over generic CRUD ("query_table"); fewer, clearer tools improve selection. |
| **Sync vs async** | Azure Functions queue tool or background responses for long operations; synchronous tools should return in seconds. |
| **Connectors vs custom** | Use managed connectors for common SaaS; custom APIs for core business logic. |

## Hands-on

Expose an internal API to an agent via OpenAPI with managed identity.

1. Deploy a small API (for example on Azure Container Apps or Functions) with Entra authentication enabled; register an app with identifier URI `api://contoso-orders`.
2. Write an OpenAPI 3.0 spec with `operationId: getOrderStatus` and clear descriptions.
3. In the Foundry project, create an **OpenAPI connection** with **Project managed identity**, target URL and audience `api://contoso-orders`; grant the project managed identity an app role on the API.
4. Create the agent:

```python
import os, jsonref
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import (PromptAgentDefinition, OpenApiTool, OpenApiFunctionDefinition,
    OpenApiProjectConnectionAuthDetails, OpenApiProjectConnectionSecurityScheme)

project = AIProjectClient(endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"], credential=DefaultAzureCredential())
spec = jsonref.loads(open("orders-openapi.json").read())

agent = project.agents.create_version(agent_name="orders-agent", definition=PromptAgentDefinition(
    model="gpt-5-mini",
    instructions="Use the orders API to answer order questions. Never invent order data.",
    tools=[OpenApiTool(openapi=OpenApiFunctionDefinition(
        name="orders_api", spec=spec, description="Contoso order status API",
        auth=OpenApiProjectConnectionAuthDetails(security_scheme=OpenApiProjectConnectionSecurityScheme(
            project_connection_id=os.environ["OPENAPI_CONNECTION_ID"]))))]))

with project.get_openai_client(agent_name=agent.name) as oai:
    print(oai.responses.create(input="What's the status of order 4471?").output_text)
```

5. Check API logs: the caller is the project's managed identity, not a shared key.
6. Compare with a function-calling version (file 01 hands-on) and note where the code runs and who authenticates.

## Security, cost and reliability notes

- **Least privilege**: give tool identities only the roles and scopes needed; separate read and write APIs.
- **Validate inputs server-side**: model-generated arguments are untrusted; APIs must validate and authorise every call.
- **Indirect injection via tool outputs**: API responses containing text can carry instructions; return structured data, not free text, where possible.
- **Third-party connectors**: data leaves Microsoft infrastructure for external providers; review terms and data residency.
- **Reliability**: tool time-outs, retries with idempotency, and graceful error messages the model can act on ("order not found" rather than stack traces).
- **Cost**: tool results become input tokens; return only needed fields.

## Common mistakes

| Mistake | Fix |
|---|---|
| Missing or vague `operationId`s and descriptions | Descriptive, unique IDs and summaries |
| Using API keys where Entra is available | Project managed identity connection |
| Returning huge JSON payloads | Project only required fields |
| Write operations without approval | Require approval or confirmation |
| Not submitting function outputs within 10 minutes | Handle tool execution promptly or use async patterns |
| Azure Functions tool in basic setup | Requires standard setup |

## Check yourself

1. Where does code execute for function calling vs OpenAPI tools?
2. What are the spec requirements for the OpenAPI tool?
3. List the three OpenAPI authentication options and when to use each.
4. How does the Azure Functions tool communicate with the function?
5. What is a managed MCP server in the connectors feature, and where does it run?
6. How long do pending function-calling runs last?
7. Why should APIs exposed to agents validate every input?
8. Which roles are needed to create connections and to run agents?

<details>
<summary>Answers</summary>

1. Function calling: in your application; OpenAPI: the Foundry service calls the API directly.
2. OpenAPI 3.0/3.1, `operationId` on every operation (letters, `-`, `_`), JSON or JSON-patch request bodies.
3. Anonymous (public read-only APIs), API key/token via a project connection (third-party key-based APIs), project managed identity with audience (Entra-protected Azure APIs).
4. Through Azure Storage queues: the agent writes to an input queue; the function processes and writes to an output queue.
5. An MCP server that Foundry provisions from a catalogue connector; it runs in the Foundry account's Connector Namespace.
6. They expire 10 minutes after creation.
7. Arguments are generated by the model and can be wrong or manipulated by prompt injection.
8. Foundry Project Manager to create connections; Foundry User to create and run agents.

</details>

## References

- [Function calling](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/function-calling)
- [OpenAPI tools](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/openapi)
- [Azure Functions with Foundry Agent Service](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/azure-functions)
- [Managed MCP servers from connectors (Preview)](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/connectors)
- [Tool identity and authentication](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/tool-authentication)
- [Tool best practices](https://learn.microsoft.com/azure/foundry/agents/concepts/tool-best-practice)
