# Evaluation Fundamentals

## Why it matters

"It looked good in the demo" is not evidence. Generative AI outputs vary, can be fluent and wrong, and degrade silently when models, prompts or data change. **Evaluation** turns quality into numbers you can compare across versions, gate releases on, and monitor in production. Getting the fundamentals right (what you evaluate, on what data, judged by whom or what) matters more than any individual metric. This file is the foundation for the rest of the topic.

## Key concepts

### What an evaluation is

An **evaluation** applies one or more **evaluators** to data (or to your app's responses on data) and produces **metrics**, often with explanations. Microsoft's workflow:

1. **Choose what to evaluate**: a single turn or a whole conversation; existing data, generated responses, simulated conversations or production traces.
2. **Select evaluators**: quality, safety, agent and custom metrics that matter for the app.
3. **Run**: once in development, in CI/CD, or on a recurring schedule in production.
4. **Review and improve**: aggregate metrics, individual failures, comparisons between versions, recurring patterns.

### Offline vs online

| | Offline (pre-production) | Online (production) |
|---|---|---|
| Data | Curated test datasets, synthetic data, simulations | Real traffic: traces, sampled responses, user feedback |
| Purpose | Compare versions, gate releases, regression testing | Detect drift, new failure modes, safety incidents |
| Ground truth | Often available (reference answers) | Usually absent (rely on reference-free evaluators, feedback) |
| Foundry feature | Cloud evaluation, CI/CD actions | Continuous evaluation, scheduled evaluation, dashboards |

### Evaluation units

| Level | Scores | Example |
|---|---|---|
| **Turn-level** | One response, using prior messages as context | Was the password-reset guidance correct? |
| **Conversation-level** | A complete multi-turn interaction | Did the agent resolve the sign-in problem? |

The `evaluation_level` setting controls granularity; datasets and evaluators must support the chosen level.

### Evaluation data

| Source | When |
|---|---|
| **Curated dataset** (JSONL, versioned in Foundry) | Stable regression tests, CI gates, version comparisons |
| **Synthetic dataset** | Bootstrap coverage from an agent definition, prompt or reference file; review before trusting |
| **Simulated conversations** | Multi-turn agent behaviour before deployment |
| **Production traces / stored responses** | Evaluate what really happened; convert traces into datasets |
| **Red teaming** | Adversarial safety and security tests (file 09) |

A good dataset covers main intents, edge cases, known failures, "should refuse" and "not in scope" cases, and adversarial inputs, and grows with every production incident.

### Who judges

| Judge | Strengths | Weaknesses |
|---|---|---|
| **Deterministic code** (exact match, regex, JSON schema validation, F1, BLEU/ROUGE, tool-call matching) | Cheap, repeatable, explainable | Only for well-defined outputs |
| **LLM-as-judge** (AI-assisted evaluators, rubric graders) | Scales to open-ended quality; explanations | Cost, variance, bias; must be calibrated (file 04) |
| **Service-based safety evaluators** | Harm, jailbreak, protected material detection | Coverage limited to defined categories |
| **Humans** (SMEs, reviewers, end users) | Ground truth on nuanced quality | Slow, expensive, inconsistent without rubrics |

Use all three: deterministic where possible, LLM judges for scale, humans to calibrate judges and assess high-stakes outputs.

### Human evaluation in Foundry

Foundry supports **human evaluation templates** (Preview) for agents: thumbs up/down, 1–5 sliders, multiple choice and free-text questions answered by reviewers in the agent's preview web app. Results are stored in Application Insights (following its retention) and downloadable as CSV. End-user feedback can also be logged against traces.

### Statistical thinking

- Small differences on small datasets are noise; compare with confidence intervals (the Foundry evaluation GitHub Action reports them).
- Run multiple samples for non-deterministic systems.
- Track distributions and failure counts, not only averages.

## How it works on Azure

| Surface | Use |
|---|---|
| **Foundry portal** | Evaluation wizard; results view; cluster analysis of failures |
| **Foundry SDK** (`azure-ai-projects` + OpenAI-compatible evals API) | Programmatic cloud evaluations on datasets, targets or traces ([Topic 17 file 12](../17-rag-and-knowledge/12-rag-evaluation-and-tuning.md) has a code example) |
| **azd** | Run agent evaluations from the command line |
| **CI/CD** | GitHub Actions and Azure DevOps (file 05) |
| **Continuous evaluation** | Sampled production traffic (file 07) |

Check evaluation region support, rate limits and virtual network support before designing evaluation in regulated environments.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Which metrics** | Pick 3–6 that map to business risk: task completion/adherence, groundedness, relevance, a safety evaluator, format validity, cost/latency. |
| **Thresholds** | Set from baseline and risk appetite; separate "block release" from "investigate". |
| **Dataset size** | Start with 50–100 high-quality cases; grow to hundreds; stratify by intent. |
| **Judge model** | A capable, pinned deployment, separate from the app's model where possible. |
| **Human involvement** | Calibrate LLM judges against human labels; human review for high-impact decisions. |
| **Turn vs conversation** | Turn-level for RAG Q&A; conversation-level for task-oriented agents. |

## Hands-on

1. Write 40 test cases for an agent in JSONL: 25 common intents, 5 edge cases, 5 out-of-scope, 5 adversarial.
2. Upload as a versioned Foundry dataset.
3. Run a portal evaluation with task adherence, relevance, groundedness (if RAG) and a safety evaluator.
4. Review the lowest-scoring cases; classify failures (retrieval, instructions, tool, model).
5. Create a human evaluation template with three questions; have two colleagues review 10 responses; compare with the LLM judge scores.

## Security, cost and reliability notes

- **Evaluation data is sensitive**: production-derived datasets contain user content; restrict and retain appropriately.
- **Cost**: LLM judges cost tokens per row per evaluator; sample and schedule wisely.
- **Judge drift**: pin judge versions; rerun baselines when changing judges.
- **Rate limits**: evaluation services and judge deployments have quotas; plan large runs.

## Common mistakes

| Mistake | Fix |
|---|---|
| Testing on the examples used to write the prompt | Separate development and test sets |
| One aggregate score | Several metrics and failure breakdowns |
| Trusting LLM judges blindly | Calibrate against humans |
| Ignoring "should refuse" cases | Include negative and adversarial cases |
| Comparing tiny differences | Use confidence intervals and larger samples |

## Check yourself

1. What are the four steps of an evaluation workflow?
2. Compare offline and online evaluation.
3. What's the difference between turn-level and conversation-level evaluation?
4. Name four sources of evaluation data.
5. Compare deterministic, LLM-as-judge and human judging.
6. How does Foundry support human evaluation?

<details>
<summary>Answers</summary>

1. Choose what to evaluate, select evaluators, run, review and improve.
2. Offline uses curated/synthetic data before release for comparison and gating; online uses real traffic in production to detect drift and new failures, usually without ground truth.
3. Turn-level scores individual responses with prior context; conversation-level scores the whole interaction (for example task completion).
4. Curated datasets, synthetic datasets, simulated conversations, production traces (plus red-teaming data).
5. Deterministic: cheap and repeatable for well-defined outputs; LLM judges: scalable for open-ended quality but costly and variable; humans: nuanced ground truth but slow and expensive.
6. Human evaluation templates (thumbs, sliders, multiple choice, free text) answered in the preview web app, stored in Application Insights and downloadable as CSV.

</details>

## References

- [Evaluation in Microsoft Foundry](https://learn.microsoft.com/azure/foundry/concepts/evaluation-overview)
- [Observability in generative AI](https://learn.microsoft.com/azure/foundry/concepts/observability)
- [Evaluation datasets](https://learn.microsoft.com/azure/foundry/observability/how-to/evaluation-datasets)
- [Set up human evaluation (Preview)](https://learn.microsoft.com/azure/foundry/observability/how-to/human-evaluation)
- [Cloud evaluation with the Foundry SDK](https://learn.microsoft.com/azure/foundry/observability/how-to/cloud-evaluation)
