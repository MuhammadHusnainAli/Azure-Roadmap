# Large Language Models

> Topic: 13 · AI and Generative AI Foundations · Tracks: E ● A ● C ● · Status checked: 2026-11

## Why it matters

Large language models (LLMs) are the engine of chat assistants, RAG systems, copilots and agents. Using them well means understanding **tokens** (how you're billed and limited), **context windows** (how much the model can consider at once), **sampling parameters** (how deterministic or creative output is), **reasoning models** (which think before answering) and the **limitations** that drive almost every design decision in topics 16–23: hallucination, knowledge cut-offs, prompt injection and non-determinism.

## Key concepts

### Tokens

LLMs read and write **tokens**: sub-word pieces produced by a tokenizer (for OpenAI models, byte-pair encoding such as `o200k_base`).

| Rule of thumb (English) | Value |
|---|---|
| 1 token | About 4 characters, or about ¾ of a word |
| 100 tokens | About 75 words |
| 1 page of prose | About 500–700 tokens |

Other languages, code and unusual strings use more tokens per word. Count with the real tokenizer when cost matters:

```python
import tiktoken
enc = tiktoken.get_encoding("o200k_base")       # used by GPT-4o and later OpenAI models
text = "Microsoft Foundry hosts many model families."
print(len(enc.encode(text)), "tokens")
```

Tokens matter because:

- **Pricing** is per input token and per output token (output usually costs more), with discounts for **cached input** tokens.
- **Limits** (context window, max output, quotas in tokens per minute) are all in tokens.
- **Latency** grows with input length (time to first token) and output length (generation time).

### Context window

The **context window** is the maximum number of tokens a model can consider in one request: system prompt + conversation history + retrieved documents + tool definitions + the output, including any **reasoning tokens**.

| Example (Azure OpenAI in Foundry, checked 2026-11) | Context window | Max output |
|---|---|---|
| GPT-6 and GPT-5.5/5.6 models | 1,050,000 tokens (input up to 922,000) | 128,000 |
| GPT-5 / GPT-5.1 family (most) | 400,000 tokens (input up to 272,000) | 128,000 |
| o3 / o4-mini | 200,000 input | 100,000 |

Model limits change with each release, so always check the model's page. Input and output limits aren't additive: everything shares one budget.

Large windows don't mean you should fill them. Long contexts cost more, respond more slowly, and models can miss information buried in the middle ("lost in the middle"). Retrieval (RAG) and context engineering (topic 16) usually beat dumping everything in.

### How generation works

The model repeatedly predicts a probability distribution over the next token, **samples** one, appends it, and repeats until a stop condition. It doesn't look facts up; it generates plausible continuations.

### Sampling and generation parameters

| Parameter | Effect | Typical guidance |
|---|---|---|
| `temperature` (0–2) | Scales randomness. Low means focused and repeatable; high means diverse | 0–0.3 for extraction, classification, code; 0.7–1.0 for creative writing |
| `top_p` (nucleus sampling) | Samples only from the smallest set of tokens whose probabilities sum to p | Adjust temperature **or** top_p, not both |
| `max_tokens` / `max_completion_tokens` / `max_output_tokens` | Cap on generated tokens (name depends on API and model) | Always set it, to control cost and runaway output |
| `stop` | Sequences that end generation | Useful for structured formats |
| `presence_penalty`, `frequency_penalty` (−2 to 2) | Discourage repeating topics or tokens | Small values if output is repetitive |
| `seed` | Best-effort reproducibility | Not a guarantee of identical outputs |
| `logprobs` | Return token probabilities | Confidence signals, classification scoring |
| `response_format` / structured outputs | Force JSON matching a schema | Use for anything parsed by code (topic 15) |

**Reasoning models don't support most sampling parameters.** On Azure OpenAI, most reasoning models reject `temperature`, `top_p`, `presence_penalty`, `frequency_penalty`, `logprobs`, `top_logprobs`, `logit_bias` and `max_tokens`. Use `max_completion_tokens` (Chat Completions) or `max_output_tokens` (Responses API). A few newer models accept some of these, so check the model's API support table.

### Reasoning models

Reasoning models (OpenAI o-series, GPT-5 and later reasoning models, and other providers' "thinking" models) spend **reasoning tokens** working through a problem before answering.

| Aspect | Detail (Azure OpenAI, checked 2026-11) |
|---|---|
| Reasoning tokens | Not shown in the output, but they occupy context and are **billed as output tokens** |
| `reasoning_effort` | Controls how much the model thinks. Values are model-dependent: `none`, `minimal`, `low`, `medium`, `high`, `xhigh`, `max`. For example, `minimal` applies only to the original GPT-5 reasoning models; `xhigh` to GPT-6, GPT-5.6, GPT-5.5, GPT-5.4 and gpt-5.1-codex-max; `max` to GPT-6 and GPT-5.6 with the Responses API; and newer models accept `none` for no reasoning |
| Reasoning summaries | With the Responses API you can request a **summary** of the reasoning (`auto`, `concise` or `detailed`, model-dependent). Raw chain of thought isn't exposed |
| Developer messages | `role: "developer"` is equivalent to a system message for these models; don't send both |
| Tool calling | Prefer the **Responses API** for tool calling with reasoning models. Some recent models only support tools on Chat Completions when `reasoning_effort` is `none` |
| When to use | Multi-step problems: planning, maths, code, complex analysis, agentic tasks. Use non-reasoning settings for simple, latency-sensitive tasks |

Higher effort usually improves hard-task quality at the cost of latency and tokens. Tune it with evaluations, not intuition.

### Chat roles

| Role | Purpose |
|---|---|
| `system` / `developer` | Instructions, persona, rules, output format (highest priority) |
| `user` | The end user's input |
| `assistant` | Previous model outputs (conversation history) |
| `tool` | Results of tool or function calls returned to the model |

Models are **stateless** at the API level: each request must contain the history needed, unless you use a stateful API such as the **Responses API** with `previous_response_id` or conversations (topic 15).

### Limitations you must design for

| Limitation | What happens | Mitigation |
|---|---|---|
| **Hallucination** | Confident, plausible but false statements or citations | Ground with RAG and tools, require citations, evaluate groundedness, allow "I don't know" |
| **Knowledge cut-off** | No knowledge after the training data date | Retrieval, web grounding, tools |
| **Non-determinism** | Different outputs for the same input | Low temperature, structured outputs, validation, evaluation sets |
| **Prompt injection** | Instructions hidden in user input or retrieved documents hijack behaviour | Separate data from instructions, Prompt Shields, least-privilege tools, human approval (topic 23) |
| **Arithmetic and exact recall** | Errors in precise calculation and long lists | Code interpreter, calculators, databases as tools |
| **Context limits and dilution** | Missing information in long contexts | Retrieval, summarisation, context engineering |
| **Bias and harmful content** | Reflecting training data biases | Content filters, evaluation, responsible AI practices |
| **Sycophancy** | Agreeing with the user's wrong premise | Instructions to challenge assumptions, evaluation |
| **Cost and latency** | Large models are slow and costly | Smaller models, caching, routing, batch |

## How it works on Azure

### Call a model with keyless authentication (Python)

```python
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import OpenAI

token_provider = get_bearer_token_provider(DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default")
client = OpenAI(base_url="https://<resource>.openai.azure.com/openai/v1/", api_key=token_provider)

# Non-reasoning style call (deployment of a chat model)
r = client.chat.completions.create(
    model="<chat-deployment>",
    messages=[{"role": "system", "content": "You extract dates. Reply with ISO 8601 only."},
              {"role": "user", "content": "The contract started on the third of March 2025."}],
    temperature=0, max_tokens=20)
print(r.choices[0].message.content, r.usage)

# Reasoning model via the Responses API
resp = client.responses.create(
    model="<reasoning-deployment>",
    input="Plan a three-step migration of 40 ADF pipelines to Fabric, with risks.",
    reasoning={"effort": "medium", "summary": "auto"},
    max_output_tokens=4000)
print(resp.output_text)
print(resp.usage)   # includes reasoning tokens in output token details
```

For a Foundry resource, the identity needs the **Foundry User** role (formerly *Azure AI User*). For a standalone Azure OpenAI resource, use **Cognitive Services OpenAI User**. Topic 14 covers roles and endpoints in detail.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Reasoning vs non-reasoning | Reasoning for complex, multi-step, high-value tasks; non-reasoning (or `reasoning_effort: none`) for simple, high-volume, latency-sensitive tasks |
| Temperature | Low for deterministic business tasks; higher only for ideation |
| Long context vs RAG | RAG for large or changing corpora and for permission trimming; long context for a few documents that must be read in full |
| Output length | Always cap output; ask for concise formats; structured outputs for machine consumption |
| Chat Completions vs Responses API | Responses API for new work, especially reasoning models, tools and stateful conversations (topic 15) |

## Hands-on

1. Count tokens for the same paragraph in English, another language and JSON with `tiktoken`, and compare.
2. Call a chat model at temperature 0 and 1.2 five times each with the same creative prompt; compare variability.
3. Send `temperature=0.2` to a reasoning model deployment and note the error. Rewrite the call with `reasoning_effort` and `max_completion_tokens`.
4. Ask a reasoning model a multi-step problem at `low` and `high` effort; compare quality, latency and `usage` (reasoning tokens).
5. Ask a model about an event after its training cut-off and observe the hallucination or refusal. Then supply a source document in the prompt and ask again.

## Security, cost and reliability notes

- **Security:** treat all model output as untrusted input to your systems: validate it before executing code, SQL or actions.
- **Security:** never put secrets in prompts; prompts may be logged and can be extracted through prompt injection.
- **Cost:** output and reasoning tokens usually cost more than input. Cap them, and cache repeated prefixes (prompt caching).
- **Reliability:** handle 429 (rate limit) responses with retries and backoff, and handle incomplete responses when token caps are reached.

## Common mistakes

- Estimating cost from words instead of tokens, and forgetting reasoning tokens.
- Sending sampling parameters to reasoning models.
- Filling the context window instead of retrieving what's relevant.
- Treating model output as fact without grounding or validation.
- No `max_tokens` cap, so one bad prompt produces a huge bill.

## Check yourself

1. Roughly how many tokens is 1,500 English words?
2. What counts towards the context window?
3. What does temperature control, and what value suits data extraction?
4. Which parameters do most reasoning models not support on Azure OpenAI?
5. How are reasoning tokens billed?
6. Name four LLM limitations and a mitigation for each.
7. Why should you avoid filling a 1-million-token context window by default?

<details>
<summary>Answers</summary>

1. About 2,000 tokens (1,500 ÷ 0.75).
2. Everything in the request and response: system/developer instructions, conversation history, retrieved content, tool definitions and results, the generated output, and reasoning tokens.
3. Randomness of token sampling; 0 to 0.2 for extraction.
4. `temperature`, `top_p`, `presence_penalty`, `frequency_penalty`, `logprobs`, `top_logprobs`, `logit_bias` and `max_tokens` (use `max_completion_tokens` or `max_output_tokens` instead).
5. As output tokens, even though they don't appear in the response.
6. Hallucination → grounding with RAG and citations; knowledge cut-off → retrieval or web grounding; prompt injection → Prompt Shields, least privilege and approvals; non-determinism → low temperature, structured outputs and evaluation (others: arithmetic → tools; cost → smaller models and caching).
7. It increases cost and latency, and models can miss information in very long contexts; retrieving only relevant content is usually more accurate and cheaper.

</details>

## References

- [Azure OpenAI reasoning models](https://learn.microsoft.com/azure/foundry/openai/how-to/reasoning)
- [Foundry Models sold by Azure (context windows and capabilities)](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure)
- [Chat completions how-to](https://learn.microsoft.com/azure/foundry/openai/how-to/chatgpt)
- [Responses API](https://learn.microsoft.com/azure/foundry/openai/how-to/responses)
- [Prompt engineering techniques](https://learn.microsoft.com/azure/foundry/openai/concepts/prompt-engineering)
- [Introduction to large language models (training)](https://learn.microsoft.com/training/modules/introduction-large-language-models/)
- [tiktoken (OpenAI tokenizer)](https://github.com/openai/tiktoken)
