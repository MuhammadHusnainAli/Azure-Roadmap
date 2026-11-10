# Function Calling and Structured Outputs

## Why it matters

Language models produce text, but applications need **actions** (look up an order, create a ticket) and **data** (valid JSON for code to parse). **Function calling** (tool calling) lets a model ask your code to run a function with specific arguments. **Structured outputs** guarantee that responses match a JSON Schema. Together they're the foundation of reliable extraction, agents and integrations, and the place where most bugs and security issues in AI apps show up.

## Key concepts

### Function calling flow

```text
1. You send: messages/input + tool definitions (name, description, JSON Schema parameters)
2. Model returns: one or more tool calls (name + JSON arguments), or a normal answer
3. Your code: validates arguments, executes the function, captures results
4. You send: the tool results back (tool messages / function_call_output items)
5. Model returns: a final answer (or more tool calls) → loop until done
```

The model **never executes** anything. It proposes calls; your code decides whether and how to run them.

### Tool definition

```json
{
  "type": "function",
  "name": "get_order_status",
  "description": "Return shipping status for a customer order. Use when the user asks where an order is.",
  "parameters": {
    "type": "object",
    "properties": {
      "order_id": { "type": "string", "description": "Order number, e.g. ORD-1042", "pattern": "^ORD-[0-9]{4,}$" }
    },
    "required": ["order_id"],
    "additionalProperties": false
  },
  "strict": true
}
```

(Chat Completions nests `name`, `description`, `parameters` and `strict` inside a `function` object; the Responses API uses the flat form above.)

| Field | Guidance |
|---|---|
| `name` | Verb-noun, unique, stable |
| `description` | When to use it, and when not to; the model picks tools mostly from descriptions |
| `parameters` | JSON Schema; narrow types, enums, patterns, required fields |
| `strict: true` | Arguments conform exactly to the schema (structured outputs for tools) |

### Controlling tool use

| Option | Effect |
|---|---|
| `tool_choice: "auto"` (default) | Model decides |
| `tool_choice: "required"` | Must call at least one tool |
| `tool_choice: "none"` | Must not call tools |
| Specific function | Force a particular tool |
| `parallel_tool_calls` | Allow (default true on many models) or disallow multiple tool calls in one turn |

**Parallel tool calls** let the model request several independent calls at once (for example weather for three cities), cutting round trips. Your code should run them (often concurrently) and return all results. Some configurations don't support parallel calls (for example certain reasoning models with `reasoning_effort: minimal`), and strict mode with parallel calls can have model-specific limits, so check the model's support table.

### Structured outputs

| Mechanism | Guarantee |
|---|---|
| **JSON mode** (`response_format: {"type": "json_object"}`) | Valid JSON, but **not** schema conformance; you must still instruct the format |
| **Structured outputs** (`json_schema` with `strict: true`) | Output **conforms to your JSON Schema** (constrained decoding) |
| **Strict function calling** (`strict: true` on tools) | Tool arguments conform to the parameter schema |

API shapes:

| API | Where the schema goes |
|---|---|
| Chat Completions | `response_format={"type": "json_schema", "json_schema": {"name": ..., "schema": {...}, "strict": True}}` |
| Responses | `text={"format": {"type": "json_schema", "name": ..., "schema": {...}, "strict": True}}` |
| SDK helpers | `client.chat.completions.parse(..., response_format=PydanticModel)` / `client.responses.parse(..., text_format=PydanticModel)` |

**Schema requirements in strict mode** (OpenAI structured outputs subset; check the Azure docs for current limits):

- Every object must set `additionalProperties: false`.
- All properties must be listed in `required`. Represent optional fields with a union including `null` (for example `"type": ["string", "null"]`).
- A **subset** of JSON Schema is supported: types, enums, `anyOf`, `$ref`/`$defs`, nesting (within depth and size limits); some keywords (for example certain format and length constraints, depending on version) may not be supported.
- The first request with a new schema can take longer while the schema is processed.

### Refusals and incomplete outputs

- When a model refuses for safety reasons with structured outputs, the response carries a **refusal** field instead of schema-conformant content. Handle it explicitly.
- If generation hits `max_tokens` or a content filter, JSON can be incomplete (`finish_reason: "length"` or `"content_filter"`). Check before parsing.

## How it works on Azure

### Function calling loop (Responses API, Python)

```python
import json
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import OpenAI

tp = get_bearer_token_provider(DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default")
client = OpenAI(base_url="https://<resource>.openai.azure.com/openai/v1/", api_key=tp)

tools = [{
    "type": "function", "name": "get_order_status", "strict": True,
    "description": "Return shipping status for an order.",
    "parameters": {"type": "object",
                   "properties": {"order_id": {"type": "string"}},
                   "required": ["order_id"], "additionalProperties": False}}]

def get_order_status(order_id: str) -> dict:
    if not order_id.startswith("ORD-"):                 # validate even with strict schemas
        return {"error": "invalid order id"}
    return {"order_id": order_id, "status": "shipped", "eta": "2026-11-14"}   # real lookup with managed identity

items = [{"role": "user", "content": "Where are orders ORD-1042 and ORD-2077?"}]
resp = client.responses.create(model="<deployment>", input=items, tools=tools)

while any(o.type == "function_call" for o in resp.output):
    items += resp.output                                  # keep the model's tool call items (and reasoning items)
    for call in (o for o in resp.output if o.type == "function_call"):
        args = json.loads(call.arguments)
        result = get_order_status(**args)
        items.append({"type": "function_call_output", "call_id": call.call_id, "output": json.dumps(result)})
    resp = client.responses.create(model="<deployment>", input=items, tools=tools)

print(resp.output_text)
```

### Structured extraction with Pydantic

```python
from pydantic import BaseModel
from typing import Literal, Optional

class Invoice(BaseModel):
    supplier: str
    invoice_number: str
    currency: Literal["GBP", "EUR", "USD"]
    total: float
    due_date: Optional[str]          # becomes ["string","null"] and stays required in the schema

parsed = client.responses.parse(
    model="<deployment>",
    input=[{"role": "system", "content": "Extract invoice fields. Use null when a field is missing."},
           {"role": "user", "content": invoice_text}],
    text_format=Invoice)

invoice = parsed.output_parsed       # an Invoice instance, or handle refusal / incomplete output
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Structured outputs vs JSON mode | Structured outputs (strict) whenever code consumes the output; JSON mode only for models without strict support |
| Few large tools vs many small tools | Small, single-purpose tools with clear descriptions; but keep the tool count manageable (many tools increase tokens and confusion) |
| Read vs write tools | Separate them; require confirmation or approval for side-effecting tools |
| Where tools run | Your app, Azure Functions, or MCP servers behind API Management (topic 10 file 05, topic 11 file 07) |
| Parallel calls | Enable for independent lookups; disable when calls have ordering dependencies or side effects |
| Validation | Validate arguments server-side even with strict schemas (business rules, authorisation, ranges) |

## Hands-on

1. Implement the order-status loop and ask about two orders; confirm the model issues parallel calls.
2. Add a `create_refund` tool and require user confirmation before executing it.
3. Extract fields from 20 invoices using `responses.parse` with a Pydantic model; measure field accuracy.
4. Remove `additionalProperties: false` from a strict schema and observe the API error.
5. Force `tool_choice` to a specific function and observe behaviour on an unrelated question.

## Security, cost and reliability notes

- **Security:** treat tool arguments as untrusted input. Prompt injection can make the model call tools with malicious arguments. Validate, authorise per user, and use least-privilege identities for tool back ends.
- **Security:** don't return secrets or excessive data in tool results; they go into the model's context and possibly to the user.
- **Cost:** tool definitions are input tokens on every call; large schemas and many tools add cost (cacheable as a prefix).
- **Reliability:** implement idempotency for side-effecting tools, timeouts, and clear error results the model can reason about.

## Common mistakes

- Executing tool calls without validation or authorisation.
- Vague tool descriptions, so the model picks the wrong tool.
- Using JSON mode and assuming schema conformance.
- Not handling refusals and truncated JSON.
- Losing reasoning items between tool-call turns with reasoning models (pass back the full output items).

## Check yourself

1. Who executes a function in function calling?
2. What's the difference between JSON mode and structured outputs?
3. What two schema rules are required in strict mode?
4. How do you represent an optional field in a strict schema?
5. What are parallel tool calls, and when should you disable them?
6. Name three security controls for tool calling.

<details>
<summary>Answers</summary>

1. Your application code; the model only proposes calls with arguments.
2. JSON mode guarantees syntactically valid JSON only; structured outputs (strict JSON Schema) guarantee conformance to your schema.
3. `additionalProperties: false` on every object, and every property listed in `required`.
4. Keep it in `required` but allow `null` (for example `"type": ["string", "null"]`).
5. Multiple tool calls returned in one turn for independent operations; disable them when calls depend on each other or have side effects that must be ordered.
6. Server-side argument validation, per-user authorisation and least-privilege identities for tools, approvals for side-effecting actions (also: limit data returned, log tool calls, defend against prompt injection).

</details>

## References

- [Function calling with Azure OpenAI](https://learn.microsoft.com/azure/foundry/openai/how-to/function-calling)
- [Structured outputs](https://learn.microsoft.com/azure/foundry/openai/how-to/structured-outputs)
- [JSON mode](https://learn.microsoft.com/azure/foundry/openai/how-to/json-mode)
- [Responses API](https://learn.microsoft.com/azure/foundry/openai/how-to/responses)
- [Reasoning models: tool calling](https://learn.microsoft.com/azure/foundry/openai/how-to/reasoning)
- [Prompt injection and Prompt Shields](https://learn.microsoft.com/azure/ai-services/content-safety/concepts/jailbreak-detection)
