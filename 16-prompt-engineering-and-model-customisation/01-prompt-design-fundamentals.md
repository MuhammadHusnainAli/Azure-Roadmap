# Prompt Design Fundamentals

## Why it matters

The prompt is the cheapest, fastest lever you have over model behaviour. Before anyone talks about RAG, fine-tuning or a bigger model, a well-structured prompt usually closes most of the quality gap. Engineers write prompts every day; architects need to know what prompting can and cannot fix so they do not buy an expensive solution to a wording problem; consultants need to explain why "the AI got it wrong" is often a design issue, not a model issue.

## Key concepts

### What a model actually does with a prompt

An LLM predicts the most likely continuation of the text it is given. It has no separate "question answering" mode: a question gets an answer because an answer is the most probable continuation. Everything in prompt design follows from this. You are shaping the probability distribution by controlling what the model sees and in which order.

### Prompt components

Microsoft's prompt engineering guidance breaks a prompt into components. Not every prompt uses all of them.

| Component | What it is | Example |
|---|---|---|
| **Instructions** | What to do | "Summarise the email in three bullet points." |
| **Primary content** | The text the model processes or transforms | The email itself |
| **Examples** | Input and output pairs that show the desired behaviour (few-shot) | Two example emails with their summaries |
| **Cue** | A prefix that jump-starts the output | "Key points:\n- " |
| **Supporting content** | Context that influences output but is not the main target | Today's date, user's preferences, the user's important topics |

### Message roles

Chat-style APIs (Chat Completions and the Responses API, see [Topic 15 file 07](../15-foundry-models-and-azure-openai/07-chat-completions-and-responses-api.md)) send an array of messages with roles:

| Role | Purpose |
|---|---|
| `system` / `developer` | Highest-level instructions for the conversation. On recent OpenAI reasoning models `developer` is functionally the same as `system`; do not send both in one request. In the Responses API you can also use the `instructions` parameter. |
| `user` | The end user's input, plus any content you inject on their behalf |
| `assistant` | Previous model turns, or fabricated turns used as few-shot examples |
| `tool` | Results of a function call ([Topic 15 file 08](../15-foundry-models-and-azure-openai/08-function-calling-and-structured-outputs.md)) |

### The system message

Microsoft Learn describes the system message as the place to define:

- **Role and scope**: what the assistant is and what it may and may not do.
- **Tone and style**.
- **Output contract**: the format the application expects, kept small and stable.
- **Safety constraints**: refusals, non-disclosure, grounding rules.
- **Fallback behaviour**: what to do when the request is ambiguous, out of scope, or lacks information.

A system message **influences** the model; it does not guarantee compliance. Layer it with content filtering, structured outputs and evaluation.

### Zero-shot, one-shot and few-shot

| Style | Use when |
|---|---|
| **Zero-shot** | The task is common and the instruction is unambiguous |
| **One-shot** | The format is unusual and one example shows it |
| **Few-shot** | Labels, tone or edge cases are hard to describe in words |

With chat APIs, put examples as alternating `user`/`assistant` messages after the system message. Examples do not change model weights; they condition only the current request. The order of examples can matter (recency bias), so vary them and evaluate.

### Core techniques from the Microsoft guidance

| Technique | What it does |
|---|---|
| **Start with clear instructions** | State the task before the context. Newer models are less sensitive, but it rarely hurts. |
| **Repeat instructions at the end** | Counters recency bias in long prompts. |
| **Prime the output** | End with a cue such as "Here's a bulleted list:\n- " to force a form. |
| **Add clear syntax** | Use delimiters (`---`), uppercase section names, Markdown or XML tags. Models were trained on lots of Markdown and XML. |
| **Break the task down** | Ask for intermediate steps: extract claims, then generate queries. |
| **Use affordances** | Let the model call tools (search, functions) rather than relying on memory. |
| **Chain of thought** | "Think step by step" for **non-reasoning** models only (see file 02). |
| **Specify output structure** | Ask for citations inline and for fixed structures (entity, relation, entity). |
| **Provide grounding context** | Give the model the source text and tell it to answer only from it. |
| **Give the model an "out"** | "Reply 'not found' if the answer isn't in the text." Reduces fabrication. |

### Sampling parameters

For non-reasoning models, `temperature` (0–2) and `top_p` control randomness. Lower temperature (around 0–0.3) suits extraction, classification and legal text; higher (around 0.7+) suits creative writing. Change one, not both. Most current reasoning models (GPT-5 family, o-series) **do not support** `temperature`, `top_p`, penalties or `logit_bias`; you steer them with `reasoning_effort` and `verbosity` instead (file 02).

## How it works on Azure

| Where | What you get |
|---|---|
| **Foundry portal playgrounds** | Chat and agents playgrounds to iterate on system messages, add examples, attach data and compare models side by side. "View code" exports a working Python sample. |
| **Prompt optimiser / generate** | The playground can generate or refine instructions from a short description (check availability for your model and region). |
| **Responses API `instructions`** | System-level text supplied per request without being stored in the conversation history. |
| **Structured outputs** | A JSON schema enforced by the model, which is more reliable than asking for JSON in prose ([Topic 15 file 08](../15-foundry-models-and-azure-openai/08-function-calling-and-structured-outputs.md)). |
| **Safety system messages** | Microsoft publishes tested safety system message templates for grounding, harmful content, jailbreaks and copyright. |
| **Prompty and prompt templates** | File-based prompt assets you can version and test (file 04). |
| **Evaluations** | Foundry evaluators measure groundedness, relevance, coherence and task adherence so prompt changes are measured, not guessed ([Topic 22](../Roadmap.md)). |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **One big system message or modular sections?** | Use clearly delimited sections (`# Role`, `# Rules`, `# Output format`, `# Examples`). Easier to maintain and diff. |
| **Few-shot examples or longer instructions?** | Examples beat paragraphs when behaviour is easier to show than describe. They cost tokens on every call, so cache them (prompt caching rewards a stable prefix) or move to fine-tuning if you need dozens. |
| **JSON in prose or structured outputs?** | Use structured outputs whenever the application parses the result. |
| **Put context in system or user message?** | Stable rules go in system/developer; per-request data (retrieved documents, user profile) goes in the user turn or tool results. This also keeps the cacheable prefix stable. |
| **Negative instructions** | Prefer "Do X" over "Don't do Y". When you must forbid something, say what to do instead. |
| **Prompt length** | Long system messages consume context and can contain contradictions. Delete rules that no evaluation shows are needed. |

## Hands-on

Build a support-ticket classifier that returns a strict structure, using a modular system message and two examples.

```python
# pip install openai azure-identity
from openai import OpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

token_provider = get_bearer_token_provider(
    DefaultAzureCredential(), "https://ai.azure.com/.default"
)
client = OpenAI(
    base_url="https://<resource>.openai.azure.com/openai/v1/",
    api_key=token_provider,
)

SYSTEM = """# Role
You classify customer support tickets for Contoso Bikes.

# Rules
- Choose exactly one category: billing, delivery, product_fault, account, other.
- Set urgency to high only if the customer reports a safety issue or a payment taken twice.
- If the ticket is not about Contoso, use category "other" and urgency "low".

# Output
Return only JSON matching the schema."""

examples = [
    {"role": "user", "content": "My brakes failed on the first ride!"},
    {"role": "assistant", "content": '{"category":"product_fault","urgency":"high"}'},
    {"role": "user", "content": "Where is my order 4471?"},
    {"role": "assistant", "content": '{"category":"delivery","urgency":"low"}'},
]

schema = {
    "type": "json_schema",
    "name": "ticket",
    "strict": True,
    "schema": {
        "type": "object",
        "properties": {
            "category": {"type": "string",
                         "enum": ["billing", "delivery", "product_fault", "account", "other"]},
            "urgency": {"type": "string", "enum": ["low", "medium", "high"]},
        },
        "required": ["category", "urgency"],
        "additionalProperties": False,
    },
}

resp = client.responses.create(
    model="gpt-5-mini",              # your deployment name
    instructions=SYSTEM,
    input=examples + [{"role": "user", "content": "I was charged twice for my helmet."}],
    text={"format": schema},
)
print(resp.output_text)
```

Then:

1. Remove the examples and compare accuracy on 20 tickets.
2. Delete the urgency rule and observe how often "high" appears.
3. Move the rules to the end of the prompt and check whether results change for your model.

Record each variant and its accuracy: that table is the start of prompt management (file 04).

## Security, cost and reliability notes

- **Prompt injection.** Anything in the user turn or retrieved content can contain instructions. Delimit untrusted content, tell the model to treat it as data, and enable **Prompt Shields** in Azure AI Content Safety ([Topic 20 file 09](../20-foundry-tools/09-azure-ai-content-safety.md)).
- **Do not put secrets in prompts.** System messages can be extracted by determined users. Never include keys, internal URLs or personal data you would not show the user.
- **Cost scales with every token on every call.** A 3,000-token system message at one million calls is three billion input tokens. Keep the stable prefix at the start so **prompt caching** applies ([Topic 15 file 12](../15-foundry-models-and-azure-openai/12-batch-prompt-caching-and-cost-optimisation.md)).
- **Model upgrades change behaviour.** Prompts tuned for one model version can regress on the next. Keep a regression set (file 04).

## Common mistakes

| Mistake | Fix |
|---|---|
| Vague tasks ("make it better") | State audience, length, format and success criteria |
| Contradictory rules ("be brief", "be comprehensive") | Prioritise explicitly |
| Asking for JSON in prose and parsing with regex | Use structured outputs |
| Chain-of-thought prompts on reasoning models | Let the model reason internally; use `reasoning_effort` |
| Tuning temperature on a model that ignores it | Check the model's supported parameters |
| No fallback instruction | Tell the model exactly what to say when it cannot answer |
| Judging a prompt on three manual tries | Evaluate on a fixed dataset |

## Check yourself

1. Name the five prompt components in Microsoft's guidance.
2. What should a system message contain, according to the Foundry design checklist?
3. Where do few-shot examples go when using a chat API?
4. Why does "repeat instructions at the end" help?
5. Why is structured outputs preferable to asking for JSON in the prompt?
6. Which sampling parameters do most reasoning models reject?
7. How do you keep a long system message cheap at high volume?
8. What is the main security risk of putting retrieved documents into the prompt?

<details>
<summary>Answers</summary>

1. Instructions, primary content, examples, cue and supporting content.
2. The assistant's role and expected outcome, boundaries, the output format, a "when unsure" policy, and it must then be tested and iterated.
3. As alternating user and assistant messages after the system message.
4. Models show recency bias: content near the end has more influence, so restating the task after long content keeps it in focus.
5. The schema is enforced during generation, so the output always parses and matches the required keys and enums.
6. `temperature`, `top_p`, `presence_penalty`, `frequency_penalty`, `logprobs`, `top_logprobs`, `logit_bias` and `max_tokens`.
7. Keep it stable and at the start of the request so prompt caching applies, and remove rules that evaluations show are unnecessary.
8. Indirect prompt injection: the documents may contain instructions the model follows. Delimit them, treat as data, and use Prompt Shields.

</details>

## References

- [Prompt engineering techniques](https://learn.microsoft.com/azure/foundry/openai/concepts/prompt-engineering)
- [System message design](https://learn.microsoft.com/azure/foundry/openai/concepts/advanced-prompt-engineering)
- [Safety system messages](https://learn.microsoft.com/azure/foundry/openai/concepts/system-message)
- [Safety system message templates](https://learn.microsoft.com/azure/foundry/openai/concepts/safety-system-message-templates)
- [Azure OpenAI reasoning models](https://learn.microsoft.com/azure/foundry/openai/how-to/reasoning)
- [Responses API](https://learn.microsoft.com/azure/foundry/openai/how-to/responses)
