# Chat Completions and Responses API

## Why it matters

Almost every AI application calls one of two APIs. **Chat Completions** is the long-standing, stateless messages API supported by nearly every model and framework. The **Responses API** is the newer, stateful API designed for reasoning models, tools and agents, and it's what Foundry Agent Service builds on. Knowing how messages, streaming, state, tools and background mode work in each lets you build responsive, reliable apps and choose the right API for new work.

## Key concepts

### Side by side

| Capability | Chat Completions | Responses API |
|---|---|---|
| Endpoint (v1) | `/openai/v1/chat/completions` | `/openai/v1/responses` |
| Input | `messages` array (system/developer, user, assistant, tool) | `input` (string or list of items: messages, files, images, tool outputs) plus `instructions` |
| State | **Stateless**: you resend history every turn | **Stateful** option: `previous_response_id` chains turns; stored responses retrievable; or stateless with `store=false` |
| Output | `choices[].message` | `output` items (messages, reasoning, tool calls); `output_text` convenience |
| Built-in tools | Function calling (your code) | Function calling plus built-in tools: code interpreter, file search, image generation, remote **MCP**, computer use (model-dependent), web search where available |
| Reasoning models | Supported, with limits (for example tool calling restrictions on some models; `max_completion_tokens`) | First-class: reasoning summaries, encrypted reasoning, `reasoning.effort`, full tool support |
| Long tasks | Synchronous only | **Background mode** (`background: true`) with polling, cancellation and resumable streaming |
| Context management | You trim history | `compact` and server-side **compaction** options |
| Portability | Broadest (many providers and frameworks) | OpenAI and Azure OpenAI; growing support in other providers and Foundry |

Microsoft recommends the Responses API for new work, especially with reasoning models and tools. Chat Completions remains fully supported.

### Messages and roles

| Role | Purpose |
|---|---|
| `system` / `developer` | Instructions and rules. For reasoning models `developer` is equivalent to `system`; don't send both |
| `user` | End-user input (text, images, audio where supported) |
| `assistant` | Prior model output |
| `tool` | Results of your function calls (Chat Completions); Responses uses `function_call_output` items |

### Streaming

Streaming returns tokens as they're generated, which cuts **perceived** latency:

| API | How |
|---|---|
| Chat Completions | `stream=True` → chunks with `choices[].delta`; final chunk has `finish_reason` (use `stream_options={"include_usage": True}` for token counts) |
| Responses | `stream=True` → typed events such as `response.created`, `response.output_text.delta`, `response.completed`, and `error` |

If an error occurs mid-stream in Responses, the HTTP status is still 200 and an `error` event is emitted. You aren't charged for tokens generated in failed streams. Front ends usually relay streams with Server-Sent Events, or with SignalR or Web PubSub for many clients (topic 10).

### Stateful responses

| Mechanism | Use |
|---|---|
| `previous_response_id` | Chain a new turn to a stored response; the service supplies prior context (you still pay for the context tokens) |
| Stored responses | Retained **30 days by default**; retrieve with `GET /responses/{id}`, delete with `DELETE /responses/{id}` |
| `store=false` (stateless) | Nothing stored; you pass prior output items yourself. For reasoning models, include returned **encrypted reasoning** items to keep reasoning context |
| Conversations | A conversation object can hold items across many responses (check current API surface) |
| Compaction | `responses.compact` or server-side compaction with a threshold to shrink long contexts |

### Background mode

For long-running reasoning or agentic tasks (minutes), set `background: true`:

| Step | Detail |
|---|---|
| Start | Request returns immediately with an ID and status `queued` |
| Poll | `GET /responses/{id}` while status is `queued` or `in_progress` |
| Cancel | `POST /responses/{id}/cancel` (idempotent) |
| Stream | Set `background` and `stream` to true; resume after a dropped connection with `GET ...?stream=true&starting_after=<sequence_number>` |
| Limits | Requires `store=true` (stateless not supported); resume only if the original request streamed; higher time-to-first-token than synchronous calls |

## How it works on Azure

### Chat Completions with streaming (Python, keyless)

```python
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import OpenAI

tp = get_bearer_token_provider(DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default")
client = OpenAI(base_url="https://<resource>.openai.azure.com/openai/v1/", api_key=tp)

stream = client.chat.completions.create(
    model="<chat-deployment>",
    messages=[{"role": "system", "content": "You are a concise support assistant."},
              {"role": "user", "content": "How do I reset my VPN token?"}],
    stream=True,
    stream_options={"include_usage": True})

for chunk in stream:
    if chunk.choices and chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
    if chunk.usage:
        print("\n", chunk.usage)
```

### Responses API: stateful multi-turn

```python
first = client.responses.create(
    model="<deployment>",
    instructions="You are an HR policy assistant. Cite policy section numbers.",
    input="How many days of parental leave do I get?")
print(first.output_text)

follow_up = client.responses.create(
    model="<deployment>",
    previous_response_id=first.id,          # service carries the context forward
    input="And can I split it into two blocks?")
print(follow_up.output_text)
```

### Responses API: background task

```python
import time
job = client.responses.create(
    model="<reasoning-deployment>",
    input="Analyse these 40 contract clauses for termination risks and produce a table.",
    reasoning={"effort": "high"},
    background=True)

while job.status in {"queued", "in_progress"}:
    time.sleep(5)
    job = client.responses.retrieve(job.id)

print(job.status)
print(job.output_text)
```

### Responses API: remote MCP tool with approvals

```python
r = client.responses.create(
    model="<deployment>",
    tools=[{"type": "mcp", "server_label": "orders",
            "server_url": "https://apim-contoso.azure-api.net/orders-mcp/mcp",
            "headers": {"Authorization": "Bearer <token>"},
            "require_approval": "always"}],
    input="What's the status of order ORD-1042?")
# If approval is required, the output contains an mcp_approval_request item;
# send a new response with an mcp_approval_response item (approve: true/false) to continue.
```

By default the Responses API asks for approval before sending data to a remote MCP server. The MCP tool works only in the Responses API.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| API for new apps | Responses, especially with reasoning models, tools, MCP or long tasks |
| Keep Chat Completions when | You need maximum framework or provider portability, or are maintaining existing code |
| Stateful vs stateless | Stateful (`previous_response_id`) for simplicity; stateless (`store=false` + encrypted reasoning) when you can't store data in the service (for example strict retention rules) |
| Streaming | Stream for interactive UIs; don't stream for backend batch work |
| Background mode | For tasks that may exceed HTTP timeouts; combine with a job record and notifications |
| History growth | Trim, summarise or compact; long histories increase cost and latency every turn |

## Hands-on

1. Implement the same three-turn chat with Chat Completions (resending history) and with Responses (`previous_response_id`); compare code and token usage.
2. Stream a response to the console and to a simple web page using Server-Sent Events.
3. Run a background reasoning task, poll it, then cancel a second one mid-run.
4. Use `store=false` with a reasoning model and pass encrypted reasoning items between turns.
5. Connect a remote MCP server (for example your Functions MCP server from topic 10) and approve a tool call.

## Security, cost and reliability notes

- **Security:** stored responses retain data for 30 days by default. Use `store=false` or delete responses when your data-retention policy requires it.
- **Security:** review data sent to remote MCP servers; keep approvals on for external servers and log approvals.
- **Cost:** chained or resent history is billed as input every turn; reasoning and long outputs dominate cost; cached prefixes reduce it (file 12).
- **Reliability:** handle mid-stream `error` events, 429 retries and timeouts; use background mode for long tasks instead of long-held HTTP connections.

## Common mistakes

- Resending ever-growing histories without trimming or compaction.
- Sending both `system` and `developer` messages to reasoning models.
- Ignoring that a 200 streaming response can still contain an `error` event.
- Using background mode with `store=false` (not supported).
- Leaving MCP approvals off for third-party servers.

## Check yourself

1. Name three differences between Chat Completions and the Responses API.
2. How do you continue a conversation statefully in Responses?
3. How long are stored responses retained by default?
4. What are the steps and limitations of background mode?
5. How do you keep reasoning context across turns without storing responses?
6. What does the Responses API do by default before calling a remote MCP server?

<details>
<summary>Answers</summary>

1. Responses can be stateful (`previous_response_id`), has built-in tools (code interpreter, file search, MCP, image generation) and supports background mode and reasoning summaries; Chat Completions is stateless with function calling only and broader portability.
2. Pass the previous response's ID as `previous_response_id`.
3. 30 days.
4. Send with `background: true`, receive an ID with status `queued`, poll until a terminal state, optionally cancel or stream and resume with `starting_after`. It requires `store=true`, resume only works if streaming was requested, and time to first token is higher.
5. Use `store=false` and pass back the returned encrypted reasoning items with the other output items in the next request's input.
6. It requires explicit approval, returning an `mcp_approval_request` item that you answer with an `mcp_approval_response`.

</details>

## References

- [Responses API](https://learn.microsoft.com/azure/foundry/openai/how-to/responses)
- [Chat completions how-to](https://learn.microsoft.com/azure/foundry/openai/how-to/chatgpt)
- [Azure OpenAI reasoning models](https://learn.microsoft.com/azure/foundry/openai/how-to/reasoning)
- [Upgrade from Chat Completions to Responses](https://learn.microsoft.com/azure/developer/ai/how-to/azure-openai-to-responses)
- [Azure OpenAI v1 API reference](https://learn.microsoft.com/azure/foundry/openai/reference-preview-latest)
- [Stream LLM responses with Web PubSub](https://learn.microsoft.com/azure/azure-web-pubsub/overview)
