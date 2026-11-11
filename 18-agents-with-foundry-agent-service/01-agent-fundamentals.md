# Agent Fundamentals

## Why it matters

"Agent" is the most overused word in AI. To some it means a chatbot, to others an autonomous system that books travel and files expenses. Without a precise definition, teams build agents where a simple RAG app or a few lines of code would do, and miss the cases where an agent genuinely beats a fixed workflow. This file defines what an agent is, how the agent loop works, what it is made of, and when **not** to build one. Everything else in this topic builds on it.

## Key concepts

### A working definition

An **AI agent** is a system that uses a model to **decide what to do next** in pursuit of a goal, by reasoning over instructions and context and **calling tools** to observe or act, repeating until the goal is met or it stops. The key word is *decide*: a fixed sequence of LLM calls in code is a workflow; a model choosing which tool to call, with what arguments, and whether it's done, is an agent.

### Components

| Component | Role | In Foundry Agent Service |
|---|---|---|
| **Model** | Reasons, plans, chooses tools, writes the answer | Any agent-compatible model from the Foundry catalogue |
| **Instructions** | Goal, persona, rules, boundaries, output contract | Agent instructions (system message) |
| **Tools** | Retrieve knowledge, call APIs, run code, take actions | Built-in tools, functions, OpenAPI, MCP, toolboxes (files 05–07) |
| **Knowledge** | Grounding data | File search, Azure AI Search, Foundry IQ ([Topic 17](../17-rag-and-knowledge/README.md)) |
| **Memory and state** | Conversation history and long-term facts | Conversations, memory stores (file 08) |
| **Identity** | Who the agent acts as, what it may access | Microsoft Entra Agent ID, on-behalf-of (file 13) |
| **Guardrails** | Content safety, approvals, limits | Content filters, Prompt Shields, tool approvals |
| **Observability** | Traces, metrics, evaluations | Foundry tracing, Application Insights, evaluators |

### The agent loop

```text
          ┌───────────────────────────────────────────────┐
user ───► │ model reads instructions + context + history  │
          │        │                                      │
          │        ▼                                      │
          │  decide: answer now?  ── yes ──► final answer │
          │        │ no                                   │
          │        ▼                                      │
          │  call tool(s) with arguments                  │
          │        │                                      │
          │        ▼                                      │
          │  tool results appended to context ────────────┘ (repeat)
          └───────────────────────────────────────────────┘
```

Each iteration costs tokens and latency, and each tool call is a chance for error or misuse. Good agent design limits the loop with clear goals, focused tools, maximum iterations and approvals for risky actions.

### Spectrum of autonomy

| Level | Example | Control |
|---|---|---|
| **Single LLM call** | Summarise a document | Fully deterministic flow |
| **RAG** | Answer questions from documents | Fixed retrieve-then-generate |
| **Workflow with LLM steps** | Classify → extract → route ticket | Code decides the path |
| **Tool-using agent** | Support agent that looks up orders and issues refunds under a limit | Model decides tools, code sets limits |
| **Multi-agent system** | Planner delegates to research, writer and reviewer agents | Orchestration plus agent autonomy |
| **Long-running autonomous agent** | Monitors data and acts on triggers | Routines, approvals, strong governance |

Move up the spectrum only when the lower level demonstrably can't do the job.

### Agents vs workflows

| | Workflow | Agent |
|---|---|---|
| Path | Predefined in code or a designer | Chosen by the model at runtime |
| Predictability | High | Lower |
| Handles novel situations | Poorly | Well |
| Testing | Unit tests per step | Scenario evaluations, traces |
| Cost and latency | Predictable | Variable |

Most production systems combine both: deterministic workflows for the parts with known steps, agents for the parts that need judgement.

## How it works on Azure

**Foundry Agent Service** (formerly Azure AI Agent Service) is the managed platform for building, deploying and scaling agents. It provides the agent runtime (conversations, tool calls, lifecycle), toolboxes, model access, observability, identity and security, and publishing to Microsoft 365 and Teams (file 02).

Microsoft's other agent-building surfaces:

| Surface | Use |
|---|---|
| **Foundry Agent Service** | Pro-code and low-code agents on Azure (this topic) |
| **Microsoft Agent Framework** | Open-source SDK for agents and multi-agent workflows in Python and .NET ([Topic 19](../19-agent-frameworks-and-protocols/README.md)) |
| **Copilot Studio** | Low-code agents for Microsoft 365 and business processes ([Topic 24](../Roadmap.md)) |
| **Microsoft 365 Agents SDK** | Agents for Teams and Microsoft 365 channels in code ([Topic 19 file 08](../19-agent-frameworks-and-protocols/08-microsoft-365-agents-sdk-and-toolkit.md)) |

## Design decisions and trade-offs

### When not to build an agent

Microsoft's Cloud Adoption Framework gives two questions to ask first:

1. **Is the task structured or predictable?** If steps are clear, repeatable and rule-based, use regular code or non-generative AI: faster, cheaper, more reliable.
2. **Is it static knowledge retrieval?** If the goal is to answer or summarise from a fixed set of documents without tools or multi-step reasoning, use classic RAG (FAQ bots, document search, simple knowledge assistants).

Only if both answers are "no" does the scenario likely need an agent's reasoning and tool use.

### When an agent adds value

| Signal | Example |
|---|---|
| Steps vary by case and need judgement | Triage of complex customer issues |
| Multiple systems must be queried and combined | "Why is this order late?" across ERP, carrier API and email |
| Actions follow from reasoning | Rebooking, refund under policy, raising a ticket |
| Conversation needs clarification | Gathering missing details before acting |
| Long tasks with sub-goals | Research reports, data investigations |

### Key design questions

| Question | Why it matters |
|---|---|
| What is the goal and the definition of done? | Prevents endless loops and vague output |
| Which actions can the agent take, and which need approval? | Safety and accountability |
| Whose identity does it act under? | Data access and audit |
| What does a failure look like, and what's the fallback? | Reliability |
| How will you evaluate it? | You can't improve what you can't measure |
| What is the cost budget per task? | Agents can call tools and models many times |

## Hands-on

Build the smallest possible agent: a prompt agent with one function tool, using the Foundry SDK.

```python
# pip install "azure-ai-projects>=2.0.0" azure-identity
import json
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import PromptAgentDefinition, FunctionTool

project = AIProjectClient(endpoint="https://<acct>.services.ai.azure.com/api/projects/<proj>",
                          credential=DefaultAzureCredential())

get_order = FunctionTool(
    name="get_order_status",
    description="Get the delivery status of an order by its ID.",
    parameters={"type": "object",
                "properties": {"order_id": {"type": "string"}},
                "required": ["order_id"], "additionalProperties": False},
    strict=True)

agent = project.agents.create_version(
    agent_name="order-helper",
    definition=PromptAgentDefinition(
        model="gpt-5-mini",
        instructions="Help customers with order status. Always look up the order before answering. "
                     "If no order ID is given, ask for it.",
        tools=[get_order]))

oai = project.get_openai_client(agent_name=agent.name)
conv = oai.conversations.create()
r = oai.responses.create(conversation=conv.id, input="Where is my order 4471?")

# the agent loop, client side: run requested tools and send results back
# (submit outputs promptly: pending runs expire after 10 minutes)
while any(i.type == "function_call" for i in r.output):
    outputs = []
    for call in (i for i in r.output if i.type == "function_call"):
        args = json.loads(call.arguments)
        result = {"order_id": args["order_id"], "status": "out for delivery", "eta": "today 17:00"}
        outputs.append({"type": "function_call_output", "call_id": call.call_id, "output": json.dumps(result)})
    r = oai.responses.create(conversation=conv.id, input=outputs)
print(r.output_text)
```

Then ask "Where's my order?" without an ID and confirm it asks for one. Open the trace in the Foundry portal and identify each loop iteration.

## Security, cost and reliability notes

- **Excessive agency** is the top agent risk: limit tools to what the task needs, scope their permissions, and require approval for irreversible actions.
- **Prompt injection** through tool results and documents can hijack the loop; enable content filters and Prompt Shields, and treat tool outputs as data.
- **Unbounded loops** cost money; set maximum tool calls or iterations and time-outs.
- **Non-determinism**: the same input can produce different paths; evaluate on scenario sets and monitor in production.
- **Accountability**: log every tool call with the identity used.

## Common mistakes

| Mistake | Fix |
|---|---|
| Building an agent for a fixed process | Use code or a workflow |
| Building an agent for FAQ answers | Use classic RAG |
| Giving one agent 40 tools | Focus tools; split into specialised agents if needed |
| No definition of done | State completion criteria in instructions |
| Testing with a few happy-path chats | Scenario-based evaluations and red teaming |
| Agent uses a shared admin identity | Dedicated agent identity, least privilege, OBO where user context matters |

## Check yourself

1. What distinguishes an agent from a workflow that calls an LLM?
2. Name six components of an agent.
3. Describe one iteration of the agent loop.
4. What two questions does the Cloud Adoption Framework ask before choosing an agent?
5. Give three signals that an agent adds value.
6. Why should you define "done" in an agent's instructions?
7. What is "excessive agency", and how do you limit it?
8. Who runs the tool in the hands-on example, the service or the client?

<details>
<summary>Answers</summary>

1. In an agent the model decides at runtime which tools to call and when it's finished; in a workflow the path is fixed in code or a designer.
2. Any six: model, instructions, tools, knowledge, memory/state, identity, guardrails, observability.
3. The model reads instructions, context and history; decides to answer or call tools; tool results are appended to context; it repeats until it answers.
4. Is the task structured or predictable (then use code or non-generative AI)? Is it static knowledge retrieval (then use classic RAG)?
5. Any three: variable steps needing judgement, multiple systems to combine, actions following reasoning, need for clarification, long tasks with sub-goals.
6. It gives the model a stopping condition and prevents endless or incomplete loops.
7. An agent having more capability or permission than its task needs; limit tools and scopes, and require approval for risky actions.
8. The client: function tools return a `function_call` item; your code executes it and sends `function_call_output` back.

</details>

## References

- [What is Foundry Agent Service?](https://learn.microsoft.com/azure/foundry/agents/overview)
- [Agents, conversations and responses (runtime components)](https://learn.microsoft.com/azure/foundry/agents/concepts/runtime-components)
- [AI agent adoption: business strategy and plan (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/business-strategy-plan)
- [AI agent orchestration patterns (Azure Architecture Center)](https://learn.microsoft.com/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [Function calling tool](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/function-calling)
