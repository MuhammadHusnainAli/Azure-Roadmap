# Prompting Reasoning Models

## Why it matters

Most current flagship models in Foundry are **reasoning models**: GPT-6 and the GPT-5.x family, the o-series, and comparable models from other providers. They think before they answer, using hidden reasoning tokens that you pay for. The prompting habits that worked for GPT-4-era models (step-by-step instructions, "think carefully", heavy few-shot scaffolding) can make reasoning models slower, more expensive and sometimes worse. Engineers must know which controls replace those habits; architects must budget for reasoning tokens and latency.

## Key concepts

### Reasoning tokens

Reasoning models generate **reasoning tokens** in addition to input and output tokens. They are used to break the problem apart, compare approaches and discard dead ends. According to Microsoft Learn:

- Reasoning tokens never appear in the message content.
- They occupy context window space.
- They are **billed as output tokens**.
- Usage is reported in `output_tokens_details.reasoning_tokens` (Responses API) or `completion_tokens_details.reasoning_tokens` (Chat Completions).

A request can spend anywhere from a few hundred to tens of thousands of reasoning tokens. If the model hits your cap first, the response returns `status: "incomplete"` with reason `max_output_tokens`, possibly with **no visible answer**, and you still pay for the reasoning. Microsoft recommends reserving at least 25,000 tokens for reasoning and output while you learn a workload.

### Reasoning effort

`reasoning_effort` (Chat Completions) or `reasoning.effort` (Responses API) controls how much the model thinks. Supported values are model-dependent:

| Effort | Best for (Microsoft guidance) |
|---|---|
| `none` | Latency-critical work: voice, fast retrieval, classification. Supported on GPT-5.1 and later. |
| `minimal` | Only the original GPT-5 reasoning models; not GPT-5.1 or later |
| `low` | Tool use, planning, search, multistep decisions where speed and cost matter |
| `medium` | Balanced default for most workloads |
| `high` | Hard reasoning, complex debugging, deep planning |
| `xhigh` | Deep research, asynchronous and long-running agentic work. GPT-6, GPT-5.6, GPT-5.5, GPT-5.4 and `gpt-5.1-codex-max` |
| `max` | Most complex tasks. GPT-6 or GPT-5.6 with the Responses API only |

Defaults vary by model. Notably `gpt-5.1` defaults to `none`, so upgrading from an earlier reasoning model can silently switch reasoning off unless you set it.

The models adapt within a setting: they spend fewer tokens on easy requests and more on hard ones.

### Other controls

| Control | What it does |
|---|---|
| `verbosity` (`low`, `medium`, `high`) | GPT-5 family: how long and detailed the visible answer is, independently of how hard it thinks |
| `reasoning.summary` (`auto`, `concise`, `detailed`) | Returns a summary of the model's reasoning in the Responses API. GPT-5 series do not support `concise`. |
| `reasoning.mode` (`pro`) | GPT-5.6 in the Responses API: pro execution mode for difficult tasks, independent of effort |
| `reasoning.context` (`auto`, `current_turn`, `all_turns`) | Whether earlier reasoning is carried across turns; GPT-5.6 and GPT-6 default to `all_turns` |
| `max_output_tokens` / `max_completion_tokens` | Caps reasoning **plus** visible output. Reasoning models do not accept `max_tokens`. |
| `phase` (`commentary`, `final_answer`) | GPT-5.4 and GPT-5.5 in long tool-heavy runs: marks assistant messages so a preamble isn't mistaken for the final answer |

### Unsupported parameters

Apart from GPT-6 Astra, reasoning models reject `temperature`, `top_p`, `presence_penalty`, `frequency_penalty`, `logprobs`, `top_logprobs`, `logit_bias` and `max_tokens`. Code that sends these will fail after a model swap.

### Do not try to extract raw reasoning

Microsoft's documentation is explicit: chain-of-thought prompting is only for non-reasoning models, and trying to extract raw reasoning by means other than the reasoning summary parameter is unsupported, may breach the Acceptable Use Policy and can lead to throttling or suspension.

## How it works on Azure

### Prompting guidance from Microsoft Learn

Reasoning models work best with:

1. **A clear goal**: what should be true when the task is finished.
2. **Firm constraints**: what must not happen, limits, policies.
3. **An explicit output contract**: format, length, schema.
4. For agentic work, **a definition of done and how to verify it**.

They do **not** need every intermediate step prescribed. Treat `reasoning_effort` as a tuning knob, not the first fix when quality drops.

### What to stop doing

| Old habit | Why to drop it | Instead |
|---|---|---|
| "Think step by step" | The model already reasons internally | State the goal and constraints |
| Long procedural recipes | Over-constrains the planning the model is good at | Describe outcomes and checks |
| Many few-shot examples | Often unnecessary; can anchor the model on surface patterns | Start zero-shot; add one or two examples only for format |
| Asking it to show its reasoning | Unsupported for raw reasoning | Use `reasoning.summary` |
| Tuning temperature | Not supported | Tune effort and verbosity |

### Tool calling and reasoning state

- With the **Responses API**, pass reasoning items back between tool calls, either with `previous_response_id` or by copying all output items into the next `input`. The model then continues its line of reasoning instead of starting over, using fewer tokens.
- If you trim context, keep everything between the last user message and the function call output intact.
- `gpt-5.6` on **Chat Completions** cannot combine function tools with any `reasoning_effort` other than `none`, and defaults to `medium`, so simply sending `tools` fails. Microsoft recommends moving tool-calling to the Responses API.

### Markdown output

Some older reasoning models (`o1`, `o3-mini`) do not produce Markdown by default. Starting the developer message with `Formatting re-enabled` increases the likelihood of Markdown output.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Reasoning model or not?** | Use a non-reasoning or `none` setting for high-volume, low-ambiguity tasks (classification, extraction, chit-chat). Use reasoning for planning, multi-step tool use, maths, code and judgement. |
| **Which effort level?** | Start at `low` or `medium`. Raise only when evaluation shows a gain worth the latency and cost. |
| **Effort vs verbosity** | Effort changes thinking; verbosity changes answer length. A support bot might want `medium` effort and `low` verbosity. |
| **Token caps** | Set caps to protect cost, but generously enough to avoid empty `incomplete` responses. Monitor the reasoning-token distribution. |
| **Synchronous or background** | Long `xhigh`/`max` or `o3-pro` tasks should use background mode to avoid timeouts ([Topic 15 file 07](../15-foundry-models-and-azure-openai/07-chat-completions-and-responses-api.md)). |
| **Router** | The model router can choose a reasoning or non-reasoning model per request ([Topic 15 file 06](../15-foundry-models-and-azure-openai/06-model-router-and-model-selection.md)). |

## Hands-on

Compare effort levels on the same task and measure cost.

```python
from openai import OpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

client = OpenAI(
    base_url="https://<resource>.openai.azure.com/openai/v1/",
    api_key=get_bearer_token_provider(DefaultAzureCredential(), "https://ai.azure.com/.default"),
)

TASK = """Goal: produce a rollback plan for the database migration below.
Constraints: no downtime longer than 5 minutes; customer data must not be lost.
Done when: every forward step has a matching rollback step and a verification check.
Output: a Markdown table with columns Step, Rollback, Check.

<migration>
1. Add nullable column orders.region
2. Backfill region from customers table
3. Make region NOT NULL
4. Drop customers.legacy_region
</migration>"""

for effort in ["low", "medium", "high"]:
    r = client.responses.create(
        model="gpt-5.5",                 # your deployment name
        input=TASK,
        reasoning={"effort": effort, "summary": "auto"},
        text={"verbosity": "low"},
        max_output_tokens=25000,
    )
    if r.status == "incomplete":
        print(effort, "INCOMPLETE:", r.incomplete_details)
        continue
    u = r.usage
    print(effort, u.output_tokens_details.reasoning_tokens, u.output_tokens)
```

Then score the three outputs against a checklist (every step has a rollback, step 4 is identified as irreversible without a backup). Decide which effort level is "good enough" and record the token difference.

## Security, cost and reliability notes

- **Cost hides in reasoning tokens.** A short answer can cost more than a long one. Track reasoning tokens per request in telemetry and alert on outliers.
- **Latency is variable.** Higher effort increases time to first token. Use streaming, background mode, or a lower effort for interactive paths.
- **Handle `incomplete` explicitly.** Never treat an empty output as a valid answer.
- **Upgrade risks.** Defaults differ by model version (for example `gpt-5.1` defaulting to `none`), and parameters such as `minimal` or `temperature` may be rejected. Pin the parameters you depend on and run regression tests before swapping deployments.
- **Multi-turn input grows.** With `all_turns`, earlier reasoning items are resent; use prompt caching to reduce repeated input cost.

## Common mistakes

| Mistake | Fix |
|---|---|
| Copying GPT-4 prompts with "think step by step" | Rewrite as goal, constraints, output contract |
| Setting effort to `high` everywhere | Evaluate; most tasks are fine at `low`/`medium` |
| Cap too low, empty answers | Reserve ample tokens; check `status` |
| Dropping reasoning items between tool calls | Use `previous_response_id` or pass all output items |
| Function tools on Chat Completions with GPT-5.6 | Use the Responses API |
| Asking the model to reveal its chain of thought | Use reasoning summaries |

## Check yourself

1. How are reasoning tokens billed, and where do you see the count?
2. What happens when a reasoning model hits `max_output_tokens` before answering?
3. What is the difference between `reasoning_effort` and `verbosity`?
4. Which effort value would you choose for a voice assistant, and why?
5. Why should you pass reasoning items back between tool calls?
6. Name four parameters most reasoning models reject.
7. What does Microsoft's prompting guidance say reasoning models need instead of step-by-step instructions?
8. Why might an app behave differently after upgrading a deployment to `gpt-5.1`?

<details>
<summary>Answers</summary>

1. As output tokens; `output_tokens_details.reasoning_tokens` in the Responses API, `completion_tokens_details.reasoning_tokens` in Chat Completions.
2. It returns `status: incomplete` with reason `max_output_tokens`, possibly with no visible output, and you still pay for input and reasoning tokens.
3. Effort controls how much the model thinks; verbosity controls how long the visible answer is.
4. `none` (or the lowest supported), because voice is latency-critical and rarely benefits from extended reasoning.
5. So the model continues its existing reasoning rather than starting over, which reaches a good answer in fewer tokens.
6. Any four of `temperature`, `top_p`, `presence_penalty`, `frequency_penalty`, `logprobs`, `top_logprobs`, `logit_bias`, `max_tokens`.
7. A clear goal, firm constraints, an explicit output contract, and for agentic work a definition of done and how to verify it.
8. `gpt-5.1` defaults `reasoning_effort` to `none`, so reasoning may stop unless you set it explicitly.

</details>

## References

- [Azure OpenAI reasoning models](https://learn.microsoft.com/azure/foundry/openai/how-to/reasoning)
- [Responses API](https://learn.microsoft.com/azure/foundry/openai/how-to/responses)
- [Prompt caching](https://learn.microsoft.com/azure/foundry/openai/how-to/prompt-caching)
- [Upgrade from Chat Completions to the Responses API](https://learn.microsoft.com/azure/developer/ai/how-to/azure-openai-to-responses)
- [Prompt engineering techniques](https://learn.microsoft.com/azure/foundry/openai/concepts/prompt-engineering)
