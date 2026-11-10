# Databases as Agent Memory and State

## Why it matters

Language models are stateless: every call starts from nothing. Everything an AI application or agent "remembers" (the conversation so far, the steps of a long task, what it learned about a user last month) lives in a database the application controls. How you store that state determines whether agents can resume after failures, scale across instances, personalise responses, respect privacy and stay within token budgets. This is now a core design topic for AI engineers and architects.

## Key concepts

### Types of agent state

| Type | What it holds | Lifetime | Typical access | Typical store |
|---|---|---|---|---|
| **Conversation history (threads)** | Messages, tool calls and results, per thread | Days to months | Append; read last N turns | Cosmos DB, PostgreSQL, Azure SQL |
| **Working (short-term) memory** | Current task state, scratchpad, recent turns | Minutes to hours | Read and write every turn, low latency | Azure Managed Redis, Cosmos DB with TTL |
| **Checkpoints** | Serialised graph or workflow state at each step | Until the run finishes plus audit | Write per step; read on resume | Cosmos DB, PostgreSQL, Redis, Durable Task Scheduler |
| **Long-term memory** | Facts, preferences, summaries extracted from past interactions | Months to years | Semantic retrieval by relevance | Cosmos DB or PostgreSQL with vectors, Foundry Agent Service memory |
| **Episodic memory** | Records of past tasks and outcomes ("last time we tried X, it failed") | Long | Similarity search | Same as long-term |
| **Shared knowledge** | Documents and organisational knowledge | Long | Retrieval (RAG) | Azure AI Search / Foundry IQ (topic 17) |
| **Operational data** | Orders, tickets, accounts the agent acts on | System of record | Through tools and APIs | Business databases |
| **Audit and traces** | What the agent saw, decided and did | Per retention policy | Append; investigate | Application Insights, Log Analytics, storage |

### Managed options

| Option | What it provides |
|---|---|
| **Foundry Agent Service threads** | Stores conversations and runs for you; in the **standard setup**, they live in your own Cosmos DB account |
| **Foundry Agent Service memory** | Managed long-term memory that extracts, stores and retrieves user-relevant facts across sessions (check current status and scoping options) |
| **Microsoft Agent Framework** | Pluggable chat message stores and context providers (including memory providers) backed by Cosmos DB, Redis and others; checkpointing for workflows |
| **LangGraph checkpointers** | Postgres, Redis and Cosmos DB checkpointer implementations |
| **Durable Functions / Durable Task Scheduler** | Durable orchestration state for long-running, human-in-the-loop agent workflows (topic 10 file 04) |

Use managed options where they meet requirements; build your own when you need specific control over schema, retention, residency or retrieval.

### Data modelling for conversation history in Cosmos DB

```json
{
  "id": "t-7:0042",
  "tenantId": "contoso",
  "userId": "u-42",
  "threadId": "t-7",
  "seq": 42,
  "role": "assistant",
  "content": "Your VPN token has been reset.",
  "toolCalls": [ { "name": "reset_vpn_token", "args": { "user": "u-42" }, "result": "ok" } ],
  "tokens": { "prompt": 812, "completion": 37 },
  "model": "<deployment-name>",
  "ts": "2026-10-15T09:12:05Z",
  "ttl": 7776000
}
```

| Choice | Reason |
|---|---|
| One item per message, not one document per thread | Avoids the 2 MB item limit and write contention |
| Partition key `/tenantId` + `/userId` (hierarchical) | All of a user's threads in one logical partition; tenant isolation |
| `seq` and `ts` | Ordered reads of the last N turns |
| `ttl` | Automatic expiry per retention policy |
| A separate thread summary item | Cheap listing of threads and running summaries |

### Managing context size

Context windows are large but not free. Patterns:

| Pattern | How |
|---|---|
| **Sliding window** | Send only the last N turns |
| **Running summary** | Periodically summarise older turns into a summary item; send summary plus recent turns |
| **Selective recall** | Store memories with embeddings; retrieve only those relevant to the current turn |
| **Tool result trimming** | Store full tool outputs in the database or Blob; send the model a trimmed version or reference |
| **Token accounting** | Record tokens per message; enforce budgets per thread |

### Long-term memory pipeline

```text
After each conversation (or every N turns), asynchronously:
  1. Extract candidate memories with a model: facts, preferences, commitments
     ("prefers answers in Dutch", "manages the Rotterdam warehouse")
  2. De-duplicate and reconcile with existing memories (update, merge, or invalidate)
  3. Store with: userId, text, embedding, source thread, confidence, created/updated, expiry
At the start of each turn:
  4. Retrieve top-k memories by vector similarity to the current message, filtered by userId
  5. Inject them into the system context, clearly labelled as memory
```

Memory must be **user-scoped**, **inspectable** and **deletable**: users and administrators should be able to see and erase what is remembered.

### Checkpoints for resumable agents

Multi-step agents and workflows should checkpoint state after each step so a crash, deployment or long wait (human approval) does not lose progress:

```text
Step 1: plan        → checkpoint {plan}
Step 2: call tool A → checkpoint {plan, A.result}
Step 3: wait for approval (hours) → checkpoint {…, awaiting: approval}
Resume from latest checkpoint on any instance
```

Requirements: idempotent tool calls (or recorded results so they are not repeated), a unique run ID, and concurrency control (optimistic concurrency with ETags) so two workers do not advance the same run.

## How it works on Azure

### Reference design

```text
Agent API (Container Apps, managed identity)
 ├─ Azure Managed Redis       working memory, recent turns cache, rate limits     (TTL minutes–hours)
 ├─ Cosmos DB for NoSQL
 │    ├─ messages     (PK tenantId/userId, TTL per policy)
 │    ├─ threads      (summaries, metadata)
 │    ├─ memories     (vector index, DiskANN, PK tenantId/userId)
 │    └─ checkpoints  (PK runId, ETag concurrency)
 ├─ Azure AI Search / Foundry IQ   organisational knowledge
 └─ Business APIs                 operational data via tools
 Change feed → memory extraction job, analytics (Fabric mirroring)
```

PostgreSQL with pgvector, or Azure SQL with vectors, can replace Cosmos DB in the same design when the team prefers relational storage and scale is moderate.

### Retrieving memories (Cosmos DB, Python)

```python
memories = container.query_items(
    query="""SELECT TOP 5 c.text, VectorDistance(c.embedding, @q) AS score
             FROM c WHERE c.userId = @u AND (NOT IS_DEFINED(c.expiresAt) OR c.expiresAt > @now)
             ORDER BY VectorDistance(c.embedding, @q)""",
    parameters=[{"name": "@q", "value": embed([user_message])[0]},
                {"name": "@u", "value": user_id},
                {"name": "@now", "value": now_iso}],
    partition_key=[tenant_id, user_id],
)
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Managed vs own storage | Foundry threads and memory for speed of delivery; own Cosmos DB (standard setup or custom) for control, residency and integration |
| Store for threads | Cosmos DB for scale and multi-region; PostgreSQL or Azure SQL when relational joins with business data matter |
| Working memory | Redis for sub-millisecond, high-frequency access; Cosmos DB with TTL if simplicity outweighs latency |
| Long-term memory | Opt-in or clearly disclosed; scoped per user; with expiry and deletion |
| Summaries | Generate asynchronously to avoid adding latency to the user turn |
| Retention | Define per data type and region; implement with TTL and deletion jobs; support data subject requests |
| Multi-agent systems | Shared state through a database or durable orchestration, not in-process memory |

## Hands-on

1. Using the Cosmos DB account from file 06 (with vector search enabled), create containers `messages` (hierarchical PK `/tenantId`, `/userId`, default TTL 30 days) and `memories` (vector policy on `/embedding`, DiskANN index).
2. Build a minimal chat loop in Python that:
   - writes each user and assistant message to `messages`,
   - sends the model the last 10 messages plus the top 3 memories,
   - after every 5 turns, asks the model to extract up to three durable facts about the user and upserts them into `memories` with embeddings.
3. Restart the program and confirm that the conversation and memories persist.
4. Implement a "forget me" function that deletes all items for a user across both containers, and verify.
5. Extension: add Redis as a cache of the last 10 messages and compare latency.

## Security, cost and reliability notes

- **Security:** conversation history and memories are personal data. Apply encryption, access controls, retention limits and deletion on request.
- **Security:** scope every query by tenant and user at the partition key level. A missing filter leaks another user's memories into a prompt.
- **Security:** treat stored tool results and retrieved memories as untrusted input to the model (prompt injection can persist through memory).
- **Cost:** sending full histories every turn multiplies token costs. Summarise and retrieve selectively.
- **Cost:** TTL removes stale data without paying for delete operations.
- **Reliability:** checkpoint long-running agent work; make tools idempotent; use optimistic concurrency.

## Common mistakes

- Keeping conversation state in application memory, so it is lost on restart or scale-out.
- One ever-growing document per thread.
- Sending the entire history to the model every turn.
- Long-term memory with no way for users to see or delete it.
- Memory queries not filtered by user.
- Agents that repeat side-effecting tool calls after a crash because results were not checkpointed.
- No retention policy for chat logs.

## Check yourself

1. Name five types of agent state and a suitable Azure store for each.
2. Why store one message per item rather than one document per thread in Cosmos DB?
3. What are three techniques to keep context size under control?
4. Describe a long-term memory pipeline.
5. What do checkpoints enable, and what do tools need to be safe with them?
6. Where are Foundry Agent Service threads stored in the standard setup?
7. What privacy obligations apply to agent memory?

<details>
<summary>Answers</summary>

1. Conversation history (Cosmos DB, PostgreSQL), working memory (Managed Redis), checkpoints (Cosmos DB, PostgreSQL, Durable Task Scheduler), long-term memory (Cosmos DB or PostgreSQL with vectors, Foundry memory), shared knowledge (Azure AI Search / Foundry IQ), audit (Application Insights, Log Analytics).
2. To avoid the 2 MB item limit, write contention and expensive rewrites of large documents.
3. Sliding windows, running summaries, selective recall of relevant memories, and trimming tool outputs.
4. Asynchronously extract candidate facts from conversations with a model, reconcile with existing memories, store them with embeddings and metadata per user; at each turn retrieve the most relevant by vector similarity and inject them into the context.
5. Resuming multi-step work after failures, deployments or long waits on any instance. Tools must be idempotent, or their results recorded so they are not re-executed, with concurrency control on the run state.
6. In the customer's own Azure Cosmos DB account.
7. Transparency, user-scoped access, retention limits, the ability to inspect and delete memories (data subject requests), and appropriate protection of personal data.

</details>

## References

- [Agent memory in Foundry Agent Service](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-memory)
- [Foundry Agent Service standard setup](https://learn.microsoft.com/azure/foundry/agents/concepts/standard-agent-setup)
- [Cosmos DB as a store for AI agents](https://learn.microsoft.com/azure/cosmos-db/gen-ai/agentic-memories)
- [Microsoft Agent Framework: chat history and memory](https://learn.microsoft.com/agent-framework/user-guide/agents/agent-memory)
- [LangGraph persistence](https://langchain-ai.github.io/langgraph/concepts/persistence/)
- [Optimistic concurrency in Cosmos DB](https://learn.microsoft.com/azure/cosmos-db/nosql/database-transactions-optimistic-concurrency)
