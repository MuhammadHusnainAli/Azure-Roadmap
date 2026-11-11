# Context Engineering

## Why it matters

Prompt engineering is about wording. **Context engineering** is about deciding *what is in the context window at all* on each model call: instructions, retrieved documents, memory, tool definitions, tool results, and conversation history. In agents and long conversations this is the dominant factor in quality, cost and latency. A model given the wrong 100,000 tokens performs worse than one given the right 5,000. Engineers build the context pipeline; architects decide where state lives (service-managed or application-managed) and what it costs.

## Key concepts

### The context window is a budget

Every call sends the full input again. Without management:

| Problem | Effect |
|---|---|
| **Token limits** | The conversation eventually exceeds the window and the call fails |
| **Cost** | Larger prompts cost more on every turn |
| **Latency** | More input tokens means slower time to first token |
| **Quality ("context rot")** | Irrelevant, stale or contradictory content distracts the model and dilutes the instructions |

### Sources of context

| Source | Examples | Typical control |
|---|---|---|
| **Instructions** | System or developer message, agent instructions | Keep short and stable; put first for caching |
| **Tool definitions** | Function schemas, MCP tool lists | Load only relevant tools; use tool search or toolboxes ([Topic 18 file 07](../18-agents-with-foundry-agent-service/07-mcp-tools-toolboxes-and-skills.md)) |
| **Retrieved knowledge** | Search results, Foundry IQ answers | Retrieve fewer, better chunks; rerank ([Topic 17](../17-rag-and-knowledge/README.md)) |
| **Memory** | User preferences, prior facts | Retrieve only relevant memories ([Topic 18 file 08](../18-agents-with-foundry-agent-service/08-agent-memory.md)) |
| **Conversation history** | Previous turns | Truncate, slide, summarise or compact |
| **Tool results** | API responses, file contents | Trim, project only needed fields, summarise old results |
| **Reasoning items** | Hidden reasoning from previous turns | Keep between tool calls (file 02) |

### The four moves

A common framing for context engineering is: **write** (persist information outside the window: memory, files, scratchpads), **select** (pull back only what is relevant: retrieval, memory search, tool search), **compress** (summarise or compact), and **isolate** (split work across sub-agents so each has a focused window). Microsoft's tooling supports all four, as shown below.

### Who owns the history?

This is the most important architectural question.

| Model | Where history lives | Compaction |
|---|---|---|
| **Service-managed** | Foundry Agent Service conversations, Responses API with `store` enabled (default) and `previous_response_id`, Copilot Studio | The service manages context; client-side compaction strategies have no effect |
| **Application-managed** | Your app holds the message list and sends it each call (`store=false`, or Chat Completions) | You choose truncation, sliding window, summarisation or compaction |

## How it works on Azure

### Responses API

| Feature | What it does |
|---|---|
| `previous_response_id` | Chains turns with service-stored state. Stored responses are retained for 30 days by default and can be deleted by ID. |
| Manual chaining | Carry `response.output` forward in your own `input` array |
| **`/responses/compact`** (`client.responses.compact(...)`) | Explicitly reduces the input context while preserving essential state for later turns |
| **Server-side compaction** | `context_management=[{"type": "compaction", "compact_threshold": N}]` compacts automatically when the threshold is reached. Documented to work with `store=False`. |
| Prompt caching | Discounts a repeated prefix; design the context so the stable part comes first ([Topic 15 file 12](../15-foundry-models-and-azure-openai/12-batch-prompt-caching-and-cost-optimisation.md)) |

Microsoft's guidance: with stateless input-array chaining you can drop items before the most recent compaction item; with `previous_response_id` chaining, do **not** prune manually.

### Microsoft Agent Framework compaction

Agent Framework ([Topic 19 file 01](../19-agent-frameworks-and-protocols/01-microsoft-agent-framework.md)) provides compaction for agents that manage their own in-memory history. The feature is **experimental** in Python and .NET. Messages are grouped into atomic units so a tool call is never separated from its result (which would cause API errors).

| Python strategy | Aggressiveness | Uses an LLM | Best for |
|---|---|---|---|
| `ToolResultCompactionStrategy` | Low | No | Collapsing verbose old tool results into short summaries |
| `SelectiveToolCallCompactionStrategy` | Low–medium | No | Dropping old tool-call groups entirely |
| `SummarizationStrategy` | Medium | Yes | Long conversations where older context still matters |
| `SlidingWindowStrategy` | High | No | Hard limit on groups kept |
| `TruncationStrategy` | High | No | Emergency token backstop |
| `TokenBudgetComposedStrategy` | Configurable | Depends | Running the above in order until a token budget is met |

The harness agent can enable compaction automatically when you set `max_context_window_tokens` and `max_output_tokens`. An `AgentTriggeredCompactionStrategy` even lets the model call a `compact_context` tool itself.

### Other Azure building blocks

| Need | Service |
|---|---|
| Selecting knowledge | Azure AI Search, Foundry IQ, agentic retrieval ([Topic 17](../17-rag-and-knowledge/README.md)) |
| Long-term memory | Foundry Agent Service memory; Cosmos DB, Azure Managed Redis, PostgreSQL as memory stores ([Topic 09 file 10](../09-databases/10-databases-as-agent-memory-and-state.md)) |
| Selecting tools | Foundry toolboxes and tool search; MCP |
| Isolating work | Multi-agent workflows ([Topic 18 file 09](../18-agents-with-foundry-agent-service/09-multi-agent-workflows.md)) |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Service-managed vs application-managed history** | Service-managed is simpler and keeps reasoning state. Application-managed gives full control over compaction, data residency of history and portability between models. |
| **Truncate vs summarise** | Truncation is free and predictable but forgets. Summarisation preserves facts but costs a model call and can drop or distort details. Use a cheap model for summaries. |
| **How many retrieved chunks?** | More chunks rarely beat better chunks. Use semantic ranking and a top 3–10 cutoff, then measure groundedness. |
| **All tools or a subset?** | Dozens of tool schemas cost tokens and confuse selection. Group tools per task or use tool search. |
| **Where to put dynamic content** | Stable instructions and tool definitions first; volatile retrieval and user content last. This maximises cache hits. |
| **One agent with a huge context or several focused ones** | Sub-agents with clean windows often beat a single agent with everything, at the cost of orchestration complexity. |

## Hands-on

**Part A: server-side compaction with the Responses API.**

```python
from openai import OpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

client = OpenAI(
    base_url="https://<resource>.openai.azure.com/openai/v1/",
    api_key=get_bearer_token_provider(DefaultAzureCredential(), "https://ai.azure.com/.default"),
)

conversation = [{"role": "user", "content": "Let's plan a 3-day trip to Lisbon."}]
for turn in range(30):
    r = client.responses.create(
        model="gpt-5.5",                    # your deployment
        input=conversation,
        store=False,
        context_management=[{"type": "compaction", "compact_threshold": 20000}],
    )
    conversation += r.output                # carry output items forward
    print(turn, r.usage.input_tokens)
    conversation.append({"role": "user", "content": "Add another activity for day 2."})
```

Watch `input_tokens`: it grows, then drops after compaction.

**Part B: Agent Framework compaction pipeline (experimental).**

```python
from agent_framework import (
    Agent, CharacterEstimatorTokenizer, TokenBudgetComposedStrategy,
    ToolResultCompactionStrategy, SummarizationStrategy, SlidingWindowStrategy,
)

tokenizer = CharacterEstimatorTokenizer()
strategy = TokenBudgetComposedStrategy(
    token_budget=16_000,
    tokenizer=tokenizer,
    strategies=[
        ToolResultCompactionStrategy(keep_last_tool_call_groups=1),
        SummarizationStrategy(client=summariser_client, target_count=4, threshold=2),
        SlidingWindowStrategy(keep_last_groups=20),
    ],
)
agent = Agent(client=client, instructions="You are a travel planner.",
              compaction_strategy=strategy, tokenizer=tokenizer)
```

Compare answer quality after 30 turns with and without the pipeline, using a question that depends on something said in turn 1.

## Security, cost and reliability notes

- **Summaries can leak.** A summariser sees tool arguments and results, which may contain sensitive data. Microsoft's guidance: trust the summariser client as much as the primary model.
- **Retention.** Stored responses persist (30 days by default). For regulated data, use `store=false` and manage history yourself, or delete stored responses.
- **Injection persistence.** A malicious instruction inside a tool result can survive into summaries and memory. Sanitise before writing to memory.
- **Measure cost per turn**, not per call. Context growth makes later turns far more expensive.
- **Don't break tool-call pairs.** Removing a tool call without its result (or the reverse) causes API errors; use group-aware strategies.

## Common mistakes

| Mistake | Fix |
|---|---|
| Sending the whole history forever | Compact, slide or summarise |
| Configuring client-side compaction on a service-managed agent | It has no effect; manage history client-side or rely on the service |
| Stuffing 50 retrieved chunks "to be safe" | Rerank and cut; measure groundedness |
| Registering every tool on every agent | Scope tools per agent or use tool search |
| Putting timestamps or user IDs at the start of the prompt | Move volatile content to the end to keep the cache prefix stable |
| Pruning manually while using `previous_response_id` | Don't; the service owns that state |

## Check yourself

1. What is the difference between prompt engineering and context engineering?
2. Name four sources of content in an agent's context window.
3. Why does client-side compaction have no effect on Foundry Agent Service agents?
4. How do you enable server-side compaction in the Responses API?
5. Why are tool calls and tool results compacted as one unit?
6. When would you prefer summarisation over truncation?
7. How does context ordering affect prompt caching?
8. What security concern applies to summarisation strategies?

<details>
<summary>Answers</summary>

1. Prompt engineering is about how instructions are worded; context engineering decides what information is in the window on each call and in what order.
2. Any four of instructions, tool definitions, retrieved knowledge, memory, conversation history, tool results, reasoning items.
3. Their context is managed server-side by the service, so the application doesn't hold or send the history.
4. Pass `context_management=[{"type": "compaction", "compact_threshold": N}]` (documented with `store=False`), or call `/responses/compact` explicitly.
5. Removing one without the other produces an invalid message sequence and API errors.
6. When older turns contain facts or decisions the model still needs and you can afford an extra (cheap) model call.
7. Caching applies to an identical prefix, so stable instructions and tool definitions should come first and variable content last.
8. The summariser receives tool arguments and results that may be sensitive, so it must be as trusted as the primary model.

</details>

## References

- [Responses API (Azure OpenAI)](https://learn.microsoft.com/azure/foundry/openai/how-to/responses)
- [Compaction in Microsoft Agent Framework](https://learn.microsoft.com/agent-framework/agents/conversations/compaction)
- [Prompt caching](https://learn.microsoft.com/azure/foundry/openai/how-to/prompt-caching)
- [Azure OpenAI reasoning models](https://learn.microsoft.com/azure/foundry/openai/how-to/reasoning)
- [Prompt engineering techniques](https://learn.microsoft.com/azure/foundry/openai/concepts/prompt-engineering)
