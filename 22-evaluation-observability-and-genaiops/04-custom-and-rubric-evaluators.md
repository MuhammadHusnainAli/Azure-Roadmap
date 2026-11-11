# Custom and Rubric Evaluators

## Why it matters

Built-in evaluators measure generic qualities, but your business cares about specific things: does the support agent offer a refund only within policy, does the sales assistant avoid pricing promises, does the summary include the three mandatory sections, is the tone right for the brand? **Custom evaluators** encode those criteria as code, LLM judge prompts or your own endpoints; **rubric evaluators** let you define weighted criteria that an LLM judge applies consistently. Microsoft recommends rubric evaluators as the **primary measure of agent quality**, paired with built-in safety and groundedness evaluators. Getting them right (and calibrated) is what makes evaluation trustworthy.

## Key concepts

### Custom evaluator types (Preview)

| | Code-based | Prompt-based | Endpoint-based |
|---|---|---|---|
| **How** | Python `grade(sample, item)` with deterministic logic | A judge prompt instructs an LLM to score | Your HTTP endpoint receives evaluation data and returns scores |
| **Best for** | Rules, keywords, format, schema and length checks | Subjective quality, tone, semantic criteria | Proprietary models, complex pipelines, logic needing network access |
| **Output** | Float 0.0–1.0 (higher is better) | JSON `{result, reason}`: ordinal, continuous or binary | JSON with `score`, `reason`, `status`, optional properties |

Code-based details: dataset fields via `item.get("response")`; responses generated from a model/agent target via `item.get("sample", {}).get("output_text")`. If `grade()` raises or times out, the item scores 0.0 and is marked as an error, so code defensively. Custom evaluators can be added to the project's **evaluator catalogue** and reused in runs.

### Azure OpenAI graders

| Grader | Use |
|---|---|
| **String Checker** | Exact match, contains, starts-with, regex-like checks |
| **Text Similarity** | Semantic or lexical similarity to a reference |
| **Model Labeler** | LLM classifies output into your labels (for example compliant / non-compliant) |
| **Model Scorer** | LLM produces a numeric score in your range with your guidelines |

These are the same grader types used for reinforcement fine-tuning ([Topic 16 file 06](../16-prompt-engineering-and-model-customisation/06-fine-tuning-in-foundry.md)).

### Rubric evaluators (Preview)

A **rubric** is a set of scoring **dimensions**, each with a description and a **weight**; the LLM judge scores applicable criteria and returns a weighted average normalised to **0–1** with per-dimension reasoning. Works for single responses and multi-turn conversations.

| Field | Meaning |
|---|---|
| `id` | Stable slug preserved across versions |
| `description` | What the criterion measures |
| `weight` | Relative importance (auto-generation gives one decisive criterion 8–10, others 1–6) |
| `always_applicable` | Score every time instead of judging applicability |

**Auto-generation** (recommended): the service drafts a rubric from a Foundry agent's instructions or description, a pasted system prompt, or reference documents, optionally grounded in **production traces**. You review and edit before use. Supported judge models include the GPT-5.x family (for example `gpt-5.4-mini` as the cost/performance balance).

### LLM-as-judge design

| Practice | Why |
|---|---|
| One criterion per question; explicit scale anchors ("1 = …, 5 = …") | Reduces ambiguity |
| Ask for reasoning before the score | Improves consistency and debuggability |
| Provide the needed context (instructions, retrieved docs, tool outputs) | Judge can't assess what it can't see |
| Pin the judge model and version | Comparable results over time |
| Use a different or stronger model than the app where possible | Reduces self-preference bias |
| Binary or 3-point scales for gates | More reliable than fine-grained scales |
| Few-shot examples of good and bad answers | Anchors judgement |

Known judge biases: position, verbosity (prefers longer answers), self-preference, leniency.

### Calibration

1. Have SMEs label 50–100 examples with the same rubric.
2. Run the judge on the same examples.
3. Measure agreement (accuracy for pass/fail, correlation or Cohen's kappa for scales).
4. Inspect disagreements; refine rubric wording or examples; repeat until agreement is acceptable.
5. Re-calibrate when the judge model or rubric changes, and spot-check periodically.

## How it works on Azure

Code-based evaluator for mandatory sections:

```python
def grade(sample: dict, item: dict) -> float:
    try:
        text = (item.get("sample", {}) or {}).get("output_text") or item.get("response", "")
        required = ["Impact", "Timeline", "Root cause", "Actions"]
        found = sum(1 for h in required if h.lower() in text.lower())
        return found / len(required)
    except Exception:
        return 0.0
```

Prompt-based evaluator (judge prompt sketch):

```text
You are grading a customer-support reply for Contoso.
Criterion: refunds may only be offered for orders within 30 days and under £200.
Reply under review: {{sample.output_text}}
Order context: {{item.order_context}}
Return JSON: {"result": true|false, "reason": "<one sentence>"}
true = the reply complies with the refund policy (or doesn't mention refunds).
```

Rubric example:

| id | description | weight |
|---|---|---|
| `resolves-issue` | Correctly resolves the customer's stated problem or gives the right next step | 9 |
| `policy-compliance` | Offers only actions allowed by refund and warranty policy | 6 |
| `verifies-identity` | Asks for order ID before account-specific actions | 4 |
| `tone` | Polite, concise, no blame | 2 |
| `general-quality` (always applicable) | Clear, accurate, well structured | 3 |

Create evaluators in the Foundry portal (**Evaluations → Evaluator catalogue**), then select them in evaluation runs, CI actions or continuous evaluation.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Code vs LLM judge** | Code for anything checkable deterministically; LLM for semantics and tone. Combine both. |
| **Rubric vs many single evaluators** | Rubric for overall agent quality aligned with your definition of good; single evaluators for diagnostics. |
| **Endpoint-based** | When scoring needs your proprietary model or systems (for example, checking against a CRM). Secure the endpoint. |
| **Gate design** | Use calibrated binary criteria for blocking gates; weighted rubric scores for trend monitoring. |
| **Ownership** | Business SMEs own rubric content; engineers own implementation and calibration. |

## Hands-on

1. Write the mandatory-sections code evaluator and the refund-policy prompt evaluator; add both to the project catalogue.
2. Auto-generate a rubric from your agent's instructions plus a policy document; edit weights.
3. Label 40 responses with two colleagues using the rubric; run the rubric evaluator on the same responses; compute agreement.
4. Fix the two criteria with lowest agreement (wording, anchors, examples); re-run.
5. Use the calibrated evaluators in a CI gate (file 05).

## Security, cost and reliability notes

- **Code evaluators run in a managed sandbox** with package and time limits; don't embed secrets.
- **Endpoint evaluators** receive evaluation data (possibly user content); secure with Entra and private networking.
- **Judge cost**: rubric and prompt evaluators call LLMs per item; use smaller recommended judges where calibration allows.
- **Gaming**: if teams optimise against a rubric, it can drift from real quality; refresh with human review.

## Common mistakes

| Mistake | Fix |
|---|---|
| Vague criteria ("is it good?") | Specific, observable criteria with anchors |
| 1–10 scales for gates | Binary or 3-point |
| No calibration against humans | Measure agreement before trusting |
| Judge sees response but not context | Map instructions, docs and tool outputs |
| Changing the judge model without re-baselining | Pin and re-baseline |

## Check yourself

1. Compare code-based, prompt-based and endpoint-based custom evaluators.
2. What must a code-based `grade()` function return, and what happens on error?
3. What is a rubric evaluator, and how is its score computed?
4. How can rubrics be auto-generated?
5. List four LLM-as-judge design practices.
6. Describe the calibration process.

<details>
<summary>Answers</summary>

1. Code-based: deterministic Python returning 0.0–1.0; prompt-based: LLM judge returning result and reason; endpoint-based: your HTTP service returning score, reason and status.
2. A float between 0.0 and 1.0; exceptions or time-outs score 0.0 and are marked as errors.
3. An LLM judge scores weighted criteria and returns a weighted average normalised to 0–1 with per-criterion reasoning.
4. From a Foundry agent's instructions/description, a pasted system prompt or reference files, optionally grounded with production traces, using a supported judge model.
5. Any four: one criterion per question, explicit scale anchors, reasoning before score, provide context, pin judge model, use a different/stronger judge, binary scales for gates, few-shot examples.
6. SMEs label a sample, run the judge, measure agreement, analyse disagreements, refine and repeat; recalibrate when judge or rubric changes.

</details>

## References

- [Custom evaluators (Preview)](https://learn.microsoft.com/azure/foundry/concepts/evaluation-evaluators/custom-evaluators)
- [Rubric evaluators (Preview)](https://learn.microsoft.com/azure/foundry/concepts/evaluation-evaluators/rubric-evaluators)
- [Azure OpenAI graders](https://learn.microsoft.com/azure/foundry/concepts/evaluation-evaluators/azure-openai-graders)
- [Built-in evaluators reference](https://learn.microsoft.com/azure/foundry/concepts/built-in-evaluators)
