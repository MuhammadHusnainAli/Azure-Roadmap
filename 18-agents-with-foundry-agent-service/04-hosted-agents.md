# Hosted Agents

## Why it matters

Prompt agents cover a lot, but some agents need **your own code** in the loop: a LangGraph state machine, an Agent Framework multi-agent workflow, custom middleware, a webhook receiver, a voice pipeline, or business logic that must run beside the model. Running that code yourself means owning containers, web servers, scaling, identity, session state and telemetry. **Hosted agents** let you bring the code and framework while Foundry provides the managed endpoint, per-session sandboxes, scaling, Entra identity, state persistence and observability. They are the bridge between "framework of choice" and "enterprise-grade hosting".

## Key concepts

### When to choose hosted agents

Choose hosted agents over prompt agents when you need to:

- **Bring your own code**: Agent Framework, LangGraph, Semantic Kernel, OpenAI Agents SDK, Anthropic Agent SDK, Copilot SDK or custom code.
- **Use custom protocols**: webhooks or non-OpenAI payloads through the **Invocations** protocol.
- **Control compute**: choose CPU and memory for the sandbox.
- **Run stateful workloads**: persist files and state across turns in `$HOME` and `/files`.
- **Run long-lived work resiliently**: preserve in-progress work across process interruptions and replay streams to reconnecting clients.

### How it works

```text
your code (container image or .zip of source) ──► Foundry builds/pulls image
                                                       │
                     Agent Service: dedicated endpoint + Entra agent identity
                                                       │
           request ─► per-session VM-isolated sandbox ($HOME, /files persisted)
                                                       │
           your code calls Foundry models + Toolbox tools + Azure services (agent identity)
```

You own the code inside the sandbox; the platform owns the endpoint, identity, scaling, session state and telemetry around it. You can ship a **container image** (pushed to Azure Container Registry) or a **.zip of source code** that Foundry builds for you. Python and C# are supported.

### Protocols

| Protocol | Payload | Platform manages | Use for |
|---|---|---|---|
| **Responses** | OpenAI-compatible `/responses` | Conversation history, streaming events, background mode | Chat assistants, RAG and tool agents, Teams/M365 (bridged to Activity automatically) |
| **Invocations** | Arbitrary JSON via `/invocations` | Session only; you manage history | Webhooks (GitHub, Stripe, Jira), batch/extraction, AG-UI and custom streaming, protocol bridges |
| **Invocations (WebSocket)** | Bidirectional stream | Session | Real-time voice (Pipecat, LiveKit, Voice Live inside your container) |
| **A2A** | Agent-to-Agent | — | v1.0 GA, v0.3 Preview (file 12) |

Microsoft's advice: **start with Responses**; add Invocations later if needed. One agent can expose several protocols.

Endpoints (no publishing needed for programmatic access):

```text
{project_endpoint}/agents/{name}/endpoint/protocols/openai/responses
{project_endpoint}/agents/{name}/endpoint/protocols/invocations
wss://{account}.services.ai.azure.com/api/projects/{project}/agents/{name}/endpoint/protocols/invocations_ws?api-version=v1
{project_endpoint}/agents/{name}/endpoint/protocols/a2a
```

### Sessions, conversations and state

| Concept | What it is |
|---|---|
| **Session** | Logical session with persisted `$HOME` and `/files`; compute provisioned on demand and restored after idle |
| **Idle timeout** | Configurable 2–240 minutes per version (default 15); compute released when idle; state persisted |
| **Session deletion** | Permanently deleted after 30 days of inactivity |
| **Conversation** | Durable history stored in Foundry (Responses protocol); accessible across playground, API and Teams |
| **State store** | Durable key-value store for app state and framework checkpoints (for example LangGraph or Agent Framework); per-user partitioning; default 30-day item lifetime |

### Compute and scaling

- **Per-session scaling**: each session gets its own VM-isolated sandbox; no replica count or warm pool to manage.
- **Sandbox sizes**: 0.5 vCPU/1 GiB, 1 vCPU/2 GiB, 2 vCPU/4 GiB.
- **Billing**: CPU and memory consumed across all active sessions, plus model and tool usage. Oversizing multiplies cost by concurrency; right-size from Application Insights (raise if sustained peaks exceed about 70%).

### Versions

Each version is an immutable snapshot of image, resources, environment variables and protocols. An endpoint serves **one version at a time** (no traffic splitting for hosted agents). Environment variables carry configuration such as project endpoint and model deployment name.

### Identity

| Identity | Purpose |
|---|---|
| **Agent identity** (Entra, per agent, created at deploy) | Runtime identity for models, tools and downstream Azure services |
| **Project managed identity** | Platform operations (for example pulling from ACR) |

In Microsoft 365 channels, user-invoked requests support **OAuth On-Behalf-Of**; autonomous runs use the agent identity. Assign RBAC to the agent identity for any resource beyond defaults.

## How it works on Azure

Typical developer flow with the Azure Developer CLI and Foundry extensions:

```bash
azd auth login
azd ai agent init --template "https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/responses/01-basic/azure.yaml" --deploy-mode code --src .
azd provision                   # project, model deployment, dependencies
azd ai agent run                # run locally with the Agent Inspector
azd deploy                      # package as .zip, Foundry builds and deploys
azd ai agent invoke "Write a haiku about deploying cloud applications."
azd ai agent monitor --follow   # stream logs
```

The agent definition lives in `azure.yaml` (service `host: azure.ai.agent`, `kind: hosted`), including protocols, environment variables and `container.resources` (CPU and memory).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Hosted vs prompt** | Hosted only when you need code, a framework, custom protocols or compute control. More flexibility means more code to test and maintain. |
| **Hosted vs Azure Container Apps / AKS** | Hosted agents give agent identity, Foundry tools, conversations and publishing out of the box; Container Apps/AKS give full infrastructure control ([Topic 07](../07-containers-and-kubernetes/README.md)). |
| **Responses vs Invocations** | Responses for conversational agents; Invocations for webhooks and structured jobs. |
| **Image vs .zip** | .zip for simplicity (Foundry builds); image for custom OS dependencies and supply-chain control. |
| **Sandbox size** | Start small (0.5 vCPU) and measure; cost scales with concurrent sessions. |
| **Idle timeout** | Short for cost; longer for interactive tasks with expensive warm-up. |
| **Tools** | Use a Foundry **Toolbox** (file 07) for platform tools and user-delegated MCP access instead of wiring credentials in code. |

## Hands-on

The official minimal hosted agent (Microsoft Agent Framework + Responses protocol):

```python
# main.py
import os
from agent_framework import Agent
from agent_framework.foundry import FoundryChatClient
from agent_framework_foundry_hosting import ResponsesHostServer
from azure.identity import DefaultAzureCredential

def main():
    client = FoundryChatClient(
        project_endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"],
        model=os.environ["AZURE_AI_MODEL_DEPLOYMENT_NAME"],
        credential=DefaultAzureCredential(),
    )
    agent = Agent(
        client=client,
        instructions="You are a friendly assistant. Keep your answers brief.",
        default_options={"store": False},   # history is managed by the hosting platform
    )
    ResponsesHostServer(agent).run()

if __name__ == "__main__":
    main()
```

Steps:

1. Initialise from the template with `azd ai agent init` (above) and run locally with `azd ai agent run`.
2. Add a Python function tool to the agent and test it in the Agent Inspector.
3. `azd deploy`, then call the agent from Python with the project client:

```python
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient

project = AIProjectClient(endpoint="https://<acct>.services.ai.azure.com/api/projects/<proj>",
                          credential=DefaultAzureCredential())
with project.get_openai_client(agent_name="agent-framework-agent-basic-responses") as oai:
    print(oai.responses.create(input="Hello!").output_text)
```

4. Open the linked Application Insights resource and check CPU and memory per session; adjust `container.resources` in the next version.

## Security, cost and reliability notes

- **No secrets in images or environment variables**: use managed identity and connections (Key Vault connection for secrets).
- **Treat uploads as untrusted**: `$HOME` is upload-writable; don't load code, config or trusted instructions from upload paths.
- **Third-party frameworks and models** are used at your own risk; review data flows outside your compliance boundary.
- **Private networking**: hosted agents support BYO VNet; plan delegated subnet size for concurrent sessions.
- **429s**: `session_quota_exceeded` (too many active sessions) and `regional_session_quota_exceeded`; terminate unused sessions, back off, or request quota.
- **Cost**: CPU × memory × active session time, plus models and tools; idle timeout directly affects cost.

## Common mistakes

| Mistake | Fix |
|---|---|
| Hosting simple configuration agents as containers | Use prompt agents |
| Oversized sandboxes | Measure and right-size per version |
| Storing durable state only in memory | Use `$HOME`, `/files` or the state store |
| Expecting traffic splitting between hosted versions | One version per endpoint; test before switching |
| Forgetting RBAC for the agent identity | Assign roles on downstream resources |
| Building custom tool auth in code | Use a Toolbox with managed auth and OBO |

## Check yourself

1. Give four reasons to choose a hosted agent over a prompt agent.
2. Compare the Responses and Invocations protocols.
3. What is persisted when a session goes idle, and how long can idle timeouts be?
4. How does hosted agent billing work, and why does oversizing matter?
5. What does a hosted agent version capture?
6. Which two identities are involved, and what does each do?
7. What does `azd deploy` do for a code-mode hosted agent?
8. Where should durable framework checkpoints be stored?

<details>
<summary>Answers</summary>

1. Any four: own code/framework, custom protocols (webhooks, non-OpenAI payloads), compute control, stateful workloads with persisted files, resilient long-running work.
2. Responses is OpenAI-compatible with platform-managed history, streaming and background mode; Invocations accepts arbitrary JSON, and you manage sessions and streaming.
3. `$HOME` and `/files` are persisted and restored on resume; idle timeout is 2–240 minutes (default 15).
4. CPU and memory consumed across all active sessions plus model and tool usage; sizes apply per session, so oversizing multiplies cost by concurrency.
5. Container image, resource allocation, environment variables and protocol configuration (immutable).
6. The per-agent Entra agent identity (runtime access to models, tools, Azure services) and the project managed identity (platform operations such as pulling images).
7. Packages the source as a .zip, uploads it, Foundry builds the image remotely and deploys the agent.
8. In the durable state store (or `$HOME`), not process memory.

</details>

## References

- [What are hosted agents?](https://learn.microsoft.com/azure/foundry/agents/concepts/hosted-agents)
- [Quickstart: deploy your first hosted agent](https://learn.microsoft.com/azure/foundry/agents/quickstarts/quickstart-hosted-agent)
- [Hosted agent samples (foundry-samples)](https://github.com/microsoft-foundry/foundry-samples/tree/main/samples/python/hosted-agents)
- [What is Foundry Agent Service?](https://learn.microsoft.com/azure/foundry/agents/overview)
- [Limits, quotas and regions](https://learn.microsoft.com/azure/foundry/agents/concepts/limits-quotas-regions)
