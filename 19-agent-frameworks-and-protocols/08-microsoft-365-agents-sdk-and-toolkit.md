# Microsoft 365 Agents SDK and Toolkit

## Why it matters

Building the agent's brain (models, tools, workflows) is half the job; delivering it to users in **Microsoft Teams, Microsoft 365 Copilot, Outlook, web chat or third-party channels** is the other half. Foundry can publish agents to Teams and Copilot with a few clicks ([Topic 18 file 12](../18-agents-with-foundry-agent-service/12-publishing-agents.md)), but when you need full control of the conversation experience in code (custom activity handling, Adaptive Cards, proactive messages, multi-channel deployment, your own state), you use the **Microsoft 365 Agents SDK**, with **Microsoft 365 Agents Toolkit** in Visual Studio Code for scaffolding, registration, debugging and deployment. The **Teams SDK** covers Teams-specific agent development. These replace the older Bot Framework SDK and TeamsFx patterns.

## Key concepts

### What the Agents SDK is (and isn't)

The Microsoft 365 Agents SDK is the **plumbing layer** for conversational agents: it receives messages from channels, routes them to your handlers, manages turn state and sends responses back.

| Problem | SDK answer |
|---|---|
| Agent must work in many places (Teams, Copilot, web, Slack, others) | **Channel abstraction**: write logic once; the SDK translates to and from each channel's format |
| AI services change | **AI-agnostic**: plug in Foundry, Agent Framework, OpenAI, Semantic Kernel or any logic |
| Conversation state is hard | **Built-in turn state and storage** management |

It is **not** an AI model, an orchestration engine or a no-code builder; it doesn't decide what the agent says.

Languages: **C#** (.NET 8), **JavaScript** (Node.js 18+), **Python** (3.9–3.11 per the SDK documentation).

### Core concepts

| Concept | Meaning |
|---|---|
| **Activity** | Normalised message or event from a channel (message, conversation update, invoke, typing) |
| **Activity Protocol** | The protocol (shared with Azure Bot Service) used between channels and agents |
| **AgentApplication** | Your agent: routes activities to handlers |
| **TurnContext** | Per-turn context to read the incoming activity and send responses |
| **TurnState / Storage** | Conversation and user state persisted across turns (memory, Blob, Cosmos DB) |
| **CloudAdapter** | Connects the HTTP endpoint (for example `/api/messages`) to the agent and handles auth |
| **Azure Bot Service** | Channel registration and connectivity (Teams, web chat, others) |

### Developer tools

| Tool | Purpose |
|---|---|
| **Microsoft 365 Agents Toolkit** (formerly Teams Toolkit), VS Code | Templates; automated registration of app IDs, bots and Entra apps; multiple environments (dev/test/prod); Bicep for Azure hosting; debug tunnels; run and debug in Teams, Outlook and Copilot |
| **Microsoft 365 Agents Playground** (formerly Teams App Test Tool) | Chat with your agent locally and see Adaptive Cards as they'd appear in Teams, without a Microsoft 365 tenant |
| **Teams SDK** (formerly Teams AI Library) and **Teams developer CLI** | Recommended for new Teams-specific agents and apps (GA for JavaScript and C#, Python in developer preview at time of writing) |
| **Teams Developer Portal** | Configure, publish and manage Teams apps |

**TeamsFx SDK is deprecated** (community-only support); don't use it for new development.

### Choosing how to build a Microsoft 365 agent

| Option | Who | Brain | Channel layer |
|---|---|---|---|
| **Copilot Studio** | Makers, low-code | Copilot Studio | Built in |
| **Declarative agent for Microsoft 365 Copilot** | Developers extending Copilot | Microsoft 365 Copilot's orchestrator | Copilot |
| **Foundry agent published to Teams/Copilot** | Pro-code, minimal channel work | Foundry Agent Service | Foundry publishing (Activity Protocol) |
| **Microsoft 365 Agents SDK (custom engine agent)** | Pro-code, full control | Your choice (Foundry, Agent Framework, others) | Agents SDK |
| **Teams SDK** | Teams-centric experiences | Your choice | Teams SDK |

Low-code and Copilot extensibility options are covered in [Topic 24](../Roadmap.md).

## How it works on Azure

Minimal Python agent with the Agents SDK (from the official quickstart):

```python
# pip install microsoft-agents-hosting-aiohttp
from microsoft_agents.hosting.core import AgentApplication, TurnState, TurnContext, MemoryStorage
from microsoft_agents.hosting.aiohttp import CloudAdapter
from start_server import start_server          # small aiohttp helper from the quickstart

AGENT_APP = AgentApplication[TurnState](storage=MemoryStorage(), adapter=CloudAdapter())

async def _help(context: TurnContext, _: TurnState):
    await context.send_activity("Welcome! Ask me about your orders.")

AGENT_APP.conversation_update("membersAdded")(_help)
AGENT_APP.message("/help")(_help)

@AGENT_APP.activity("message")
async def on_message(context: TurnContext, _):
    # call your Foundry agent here (see below) and return its answer
    await context.send_activity(f"you said: {context.activity.text}")

if __name__ == "__main__":
    start_server(AGENT_APP, None)               # listens on localhost:3978
```

Connecting the channel layer to a Foundry agent inside `on_message`:

```python
from azure.identity.aio import DefaultAzureCredential
from azure.ai.projects.aio import AIProjectClient

async def ask_foundry(question: str, conversation_id: str | None) -> str:
    async with AIProjectClient(endpoint=PROJECT_ENDPOINT, credential=DefaultAzureCredential()) as project:
        oai = project.get_openai_client(agent_name="order-helper")
        r = await oai.responses.create(input=question, conversation=conversation_id)
        return r.output_text
```

Deployment: host on Azure App Service, Container Apps or Functions; register with **Azure Bot Service** (Entra app identity, preferably user-assigned managed identity or single-tenant app); enable channels (Teams, Microsoft 365 Copilot, web chat); package a Teams app manifest. The Agents Toolkit automates these steps with Bicep.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Foundry publishing vs Agents SDK** | Foundry publishing when the default experience is enough; Agents SDK when you need custom handling (Adaptive Cards, proactive messages, SSO flows, multiple channels, non-Microsoft channels). |
| **Agents SDK vs Teams SDK** | Agents SDK for multi-channel and Copilot custom engine agents; Teams SDK for Teams-specific capabilities. |
| **Where state lives** | Memory storage only for local testing; Cosmos DB or Blob for production; or rely on Foundry conversations and store only the mapping. |
| **Identity** | Use SSO and on-behalf-of to call Foundry and Graph as the user where data is user-specific. |
| **Bot Framework migration** | Move existing Bot Framework SDK bots to the Agents SDK; reuse business logic. |
| **Language** | Follow team skills; check Python version support in the SDK documentation. |

## Hands-on

1. Install **Microsoft 365 Agents Toolkit** in VS Code; create a new agent from a template (Python or TypeScript).
2. Run it locally in the **Agents Playground**; send `/help` and a message.
3. Replace the echo with a call to a Foundry agent (`ask_foundry` above) and pass the Teams conversation ID mapping to a Foundry conversation.
4. Return an **Adaptive Card** with the order status and a "Track parcel" button.
5. Use the Toolkit to provision Azure resources (App Service + Bot Service) for a `dev` environment and debug in Teams.
6. Compare with publishing the same Foundry agent directly from the Foundry portal: list what you gained and what you now maintain.

## Security, cost and reliability notes

- **Authentication**: secure `/api/messages` with the CloudAdapter's auth configuration; avoid anonymous mode outside local testing.
- **Secrets**: use managed identity or federated credentials for the bot's Entra app; never store app secrets in code.
- **Data residency**: messages flow through Microsoft 365 and Bot Service; align with your compliance requirements.
- **State storage**: protect conversation state stores (they hold user content); set retention.
- **Cost**: hosting (App Service/Container Apps), Bot Service (premium channels), storage, plus the AI backend.
- **Reliability**: handle Teams retries and timeouts; acknowledge quickly and use proactive messages for long operations.

## Common mistakes

| Mistake | Fix |
|---|---|
| Starting new Teams projects on TeamsFx or Bot Framework SDK | Agents SDK or Teams SDK |
| Putting AI logic inside channel handlers | Keep handlers thin; call Foundry or Agent Framework |
| `MemoryStorage` in production | Durable storage |
| Anonymous endpoints in production | Configure authentication |
| Building a custom channel layer when Foundry publishing suffices | Publish from Foundry |

## Check yourself

1. What three problems does the Microsoft 365 Agents SDK solve?
2. What is the SDK explicitly not?
3. Define Activity, AgentApplication, TurnContext and CloudAdapter.
4. What does Microsoft 365 Agents Toolkit automate?
5. What is the Agents Playground used for?
6. What is the status of TeamsFx?
7. When would you use the Agents SDK instead of Foundry's built-in Teams publishing?
8. Where should conversation state be stored in production?

<details>
<summary>Answers</summary>

1. Working across multiple channels, staying AI-agnostic, and managing conversation state.
2. Not an AI model, orchestration engine or no-code builder.
3. Activity: a normalised channel message or event; AgentApplication: your agent routing activities to handlers; TurnContext: per-turn context to read and respond; CloudAdapter: connects the HTTP endpoint to the agent and handles auth.
4. Project templates, registration of app IDs/bots/Entra apps, environment configurations, Bicep-based Azure provisioning, debug tunnels and running in Teams/Outlook/Copilot.
5. Testing agents locally, including Adaptive Cards, without a Microsoft 365 tenant or sideloading.
6. Deprecated; community-only support; not for new development.
7. When you need custom conversation handling, Adaptive Cards, proactive messages, multiple or non-Microsoft channels, or your own state.
8. Durable storage such as Cosmos DB or Blob Storage (or Foundry conversations with a stored mapping).

</details>

## References

- [What is the Microsoft 365 Agents SDK?](https://learn.microsoft.com/microsoft-365/agents-sdk/agents-sdk-overview)
- [Quickstart: create and test a basic agent](https://learn.microsoft.com/microsoft-365/agents-sdk/quickstart)
- [Microsoft 365 Agents Toolkit overview](https://learn.microsoft.com/microsoftteams/platform/toolkit/agents-toolkit-fundamentals)
- [Teams SDK](https://learn.microsoft.com/microsoftteams/platform/teams-sdk/welcome)
- [Publish Foundry agents to Microsoft Copilot and Teams](https://learn.microsoft.com/azure/foundry/agents/how-to/publish-copilot)
