# Agent Optimisation

## Why it matters

Most agents are tuned by hand: someone reads a few bad conversations, edits the instructions, tries three examples, and ships. That loop is slow, subjective and easy to regress. With traces, evaluators and datasets in place (files 02–07), you can make improvement **evidence-driven**: turn production failures into test cases, change one thing at a time (instructions, tool descriptions, skills, model), measure against a baseline, and promote only what wins. Foundry's **agent optimizer** automates much of that loop, generating and scoring candidate configurations for prompt agents and hosted agents. This file covers the loop and the tooling.

## Key concepts

### The improvement loop

```text
Production traces ──► Insights / failing evaluations / user feedback ──► diagnose root cause
       ▲                                                                       │
       │                                                                       ▼
   monitor new version ◄── promote winner ◄── evaluate vs baseline ◄── change: prompt, tool, skill, retrieval, model
                                                      ▲
                                     dataset from traces (+ curated golden set)
```

### Root cause → lever

| Symptom in traces | Likely cause | Lever |
|---|---|---|
| Ignores a rule or format | Instruction ambiguity, conflicting rules | Instruction tuning |
| Wrong tool or wrong parameters | Vague tool or parameter descriptions | Tool-description optimisation |
| Correct tool, misreads result | Output format, missing guidance | Instructions, tool output shaping |
| Good answer missing facts | Retrieval gap | Chunking, query rewriting, index ([Topic 17](../17-rag-and-knowledge/README.md)) |
| Repeated multi-step procedure done inconsistently | Missing reusable procedure | Skill creation or improvement |
| Quality fine, cost/latency high | Over-capable model, long context | Model selection, context trimming |
| Unsafe output or injection success | Guardrail gap | Guardrails, prompt shields ([Topic 23](../23-responsible-ai-security-and-governance/README.md)) |

### Datasets from traces

Foundry can **convert traces into versioned evaluation datasets** (SDK `azure-ai-projects` 2.8.0+). Set `max_samples` (1–1,000) to use **intelligent sampling**, which drops low-signal traffic and uses MinHash to select a diverse, representative set; or supply specific `trace_ids`. Private content is **redacted by default** (`redact_private_content`); disable only when privacy and access requirements allow. The project's managed identity needs Reader on Application Insights. Combine trace-derived data with synthetic data for edge cases you haven't seen yet.

### Foundry agent optimizer

The optimizer runs a closed loop in the cloud: evaluate the **baseline**, generate **candidates**, evaluate each on the same dataset, rank by a **composite score** (0.0–1.0), and mark the best with ★.

| Target | Prompt agent | Hosted agent |
|---|---|---|
| **Instruction tuning** | Yes (selected version is the baseline) | When baseline has `instructions.md` |
| **Skill improvement** | No | When baseline has `skills/` with `SKILL.md` files |
| **Tool optimisation** (function tool and parameter descriptions; not types or required fields) | Yes (tool execution not evaluated, since function tools run client-side) | When baseline has `tools.json` |
| **Model selection** | Choose candidate deployments in the wizard | `model_search_space` in `eval.yaml` |

| Model role | Config | Supported |
|---|---|---|
| **Eval model** | `eval_model` | Any chat model; returns binary scores per criterion |
| **Optimisation model** | `optimization_model` | GPT-5 family (gpt-5 to gpt-5.5), DeepSeek-V4-Pro, DeepSeek-V3.2 |

**Skill optimisation** (Preview) is a separate workflow that learns skills from recorded conversations.

### How to start

| Agent type | Start |
|---|---|
| Prompt agent | Portal: agent → **Optimize** tab → wizard (version, dataset or generate from traces, evaluators, models, cost estimate) → promote a candidate to a new version |
| Hosted agent | Make the agent optimizer-ready (`load_config()` returns baseline or candidate config), then Foundry Toolkit or `azd ai agent optimize` → apply candidate locally → redeploy |

## How it works on Azure

`eval.yaml` for a hosted agent:

```yaml
agent:
  name: support-agent
  kind: hosted
  model: gpt-5-mini
dataset:
  local_uri: ./eval/support-traces.jsonl     # or a registered Foundry dataset generated from traces
evaluators:
  - builtin.task_adherence
  - builtin.tool_call_accuracy
options:
  eval_model: gpt-5-mini
  optimization_model: gpt-5.4
  max_candidates: 4
```

```bash
azd ai agent invoke "test"                       # check the agent responds
azd ai agent optimize --config eval.yaml         # baseline → candidates → ranked results
```

Typical result:

```text
Candidate        Score   Strategy
baseline         0.81
candidate_1      0.84    system_prompt
candidate_2 ★    0.88    system_prompt, tools
candidate_3      0.86    model: gpt-5-nano
```

Review the diff (instructions and tool descriptions), check per-evaluator scores (not only the composite), run your safety suite and CI gate (file 05), then deploy as a new version with a canary.

### Manual optimisation still matters

Not every change fits the optimizer: retrieval tuning, adding a new tool, splitting into multiple agents, guardrail changes, UX changes. Use the same discipline: one hypothesis, a dataset that exercises it, baseline vs candidate in one evaluation run, and promotion through CI.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Dataset for optimisation** | Representative traces plus a protected golden set the optimizer never sees, to detect overfitting. |
| **Evaluators** | Include safety and policy evaluators so the optimizer can't trade safety for task score. |
| **Side effects** | The optimizer invokes the agent for every task × candidate; point tools at sandboxes or mocks. |
| **Model selection** | Take the cheapest model whose score is within tolerance of the best; check latency too. |
| **Human review** | Review generated instructions for policy, tone and hidden behaviour changes before promotion. |
| **Cadence** | Weekly triage of Insights and failures; monthly optimisation runs; immediate fixes for safety. |

## Hands-on

1. Generate a 200-row dataset from a week of traces with `max_samples=200` and default redaction.
2. Run the portal optimizer for a prompt agent with task adherence and tool call accuracy; include two candidate models.
3. Compare baseline and best candidate per evaluator; read the prompt diff.
4. Run your CI evaluation (file 05) and safety suite against the candidate; promote to a new version.
5. After one week, compare continuous-evaluation scores for the old and new versions.

## Security, cost and reliability notes

- **Side effects**: tool calls execute during optimisation runs (charges, emails, state changes); use non-production tools.
- **Privacy**: trace-derived datasets contain user content; keep redaction on unless approved; restrict dataset access.
- **Cost**: eval model calls = evaluators × tasks × candidates; optimisation model reasons over the full dataset. Review the portal cost estimate.
- **Overfitting**: a candidate tuned on a narrow dataset can regress elsewhere; validate on a held-out set.
- **Auditability**: keep optimisation runs, chosen candidate and reviewer approval with the agent version.

## Common mistakes

| Mistake | Fix |
|---|---|
| Optimising on the same examples you report on | Held-out golden set |
| Composite score only | Inspect per-evaluator results, especially safety |
| Live tools during optimisation | Sandboxes and mocks |
| Auto-promoting candidates | Human review plus CI gate |
| Fixing prompts when retrieval is the problem | Diagnose from traces first |

## Check yourself

1. Describe the trace-driven improvement loop.
2. Map three symptoms to the right optimisation lever.
3. How does intelligent sampling work when generating datasets from traces?
4. What can the agent optimizer improve for prompt agents vs hosted agents?
5. What are the eval model and optimisation model used for?
6. What side-effect risk does optimisation carry?

<details>
<summary>Answers</summary>

1. Collect traces and feedback, find failures (Insights, evaluations), diagnose, change one lever, evaluate candidate vs baseline on a dataset built from traces, promote through CI, monitor.
2. For example: wrong tool → tool descriptions; missing facts → retrieval; high cost with good quality → model selection; rule ignored → instruction tuning.
3. It filters low-signal traffic and uses MinHash to choose a diverse, representative subset up to `max_samples` (1–1,000); private content is redacted by default.
4. Prompt agents: instructions, function tool descriptions, model selection. Hosted agents: those plus skills, driven by `instructions.md`, `skills/`, `tools.json` and `model_search_space`.
5. The eval model scores responses (binary per criterion); the optimisation model generates candidate configurations.
6. The agent is invoked for every task and candidate, so real tools run (charges, data changes, messages); use sandboxes.

</details>

## References

- [What is the agent optimizer?](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-optimizer-overview)
- [Optimize agent instructions, skills, tools and models](https://learn.microsoft.com/azure/foundry/agents/how-to/optimize-agent-targets)
- [Convert agent traces into evaluation datasets](https://learn.microsoft.com/azure/foundry/observability/how-to/traces-to-dataset)
- [Insights in Foundry (Preview)](https://learn.microsoft.com/azure/foundry/observability/how-to/agent-insights)
