# GenAIOps Pipeline

## Why it matters

Generative AI apps change constantly: prompts, models, retrieval settings, tools and agent instructions. Each change can silently reduce quality or safety. Without automated evaluation, teams either ship regressions or slow down to manual testing. In this capstone you build a **GenAIOps pipeline** with evaluation gates, AI red teaming and continuous monitoring. It turns the RAG assistant (capstone 02) or the multi-agent process (capstone 03) into a product that can ship safely every week.

**Brief:** The Fabrikam HR assistant from capstone 02 now gets changes from three developers. The HR director requires that no release reduces answer quality or safety, and that production quality is monitored with alerts.

## Key concepts

### Requirements

| # | Requirement |
|---|---|
| R1 | Prompts, agent definitions, retrieval settings and evaluation datasets versioned in Git |
| R2 | Pull request pipeline: unit tests, lint, IaC validation, **offline evaluation** on a curated dataset |
| R3 | Quality gate: block merge if groundedness, relevance or task adherence drop beyond thresholds compared with the main branch baseline |
| R4 | Safety gate: AI red teaming scan (for example the AI Red Teaming Agent) with an attack success rate threshold |
| R5 | Deploy to staging and then production with approvals; support rollback to the previous version |
| R6 | Continuous evaluation on sampled production traffic; alerts on quality drops, safety events, latency and cost |
| R7 | Feedback loop: thumbs-down and failed cases flow into the evaluation dataset |
| R8 | Dashboard showing quality trend per release |

### Skills practised

Topics [22](../22-evaluation-observability-and-genaiops/README.md), [27](../27-devops-and-platform-engineering/README.md), [04](../04-tooling-and-infrastructure-as-code/README.md), [26](../26-monitoring-and-observability/README.md) and [23](../23-responsible-ai-security-and-governance/README.md). It's also excellent preparation for [AI-300](../36-certifications-and-career/07-ai-300-mlops-engineer.md).

## How it works on Azure

### Pipeline

```text
PR opened
 ├─ unit tests, lint, bicep what-if
 ├─ deploy ephemeral or shared dev environment (azd / Bicep)
 ├─ offline evaluation (Foundry evaluation via GitHub Action or SDK) on eval/golden.jsonl
 │     compare to baseline from main → post results as PR comment → fail if thresholds breached
 └─ red teaming scan (selected risk categories and attack strategies) → fail if ASR > threshold
Merge to main
 ├─ deploy staging → smoke tests + evaluation on staging
 ├─ approval (environment protection)
 └─ deploy production (new version / revision, traffic shift) → record release in dashboard
Production
 ├─ tracing to Application Insights (OpenTelemetry)
 ├─ continuous evaluation on sampled traces
 └─ alerts: groundedness < x, safety events > y, p95 latency, token cost per day
```

### Gate thresholds (example; set from your baseline)

| Metric | Gate |
|---|---|
| Groundedness (1–5) | Mean ≥ 4.0 and no drop > 0.2 versus baseline |
| Relevance (1–5) | Mean ≥ 4.0 |
| Retrieval recall at 5 | ≥ 0.85 |
| Task adherence (agents) | Pass rate ≥ 90% |
| Red teaming attack success rate | ≤ 2% on selected categories |
| Cost per request | No increase > 15% without approval |

### Milestones

| Milestone | Deliverable |
|---|---|
| M1 | Golden dataset (100–200 cases) with ownership and refresh process |
| M2 | Evaluation script locally; baseline results committed |
| M3 | PR pipeline with evaluation gate and PR comment |
| M4 | Red teaming job and threshold |
| M5 | Staging and production deployment with approvals and rollback |
| M6 | Continuous evaluation and alerts in production |
| M7 | Feedback-to-dataset loop and release quality dashboard |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Dataset size vs pipeline time | Smaller smoke set on every PR; full set nightly or before release |
| LLM-as-judge cost | Use a cost-effective judge model; check judge agreement with human labels |
| Absolute vs relative thresholds | Combine: absolute floors plus "no regression versus baseline" |
| Red teaming frequency | Smoke scan per PR or release; deeper scans on schedule and for major changes |
| Non-determinism | Run evaluations with fixed settings and enough cases; accept small variance bands |

## Hands-on

1. Complete milestones M1–M7.
2. Open a PR that deliberately degrades the prompt and show the gate blocking it.
3. Open a PR that weakens a guardrail and show the red teaming gate failing.
4. Simulate a production quality drop and show the alert firing.
5. Write the runbook for responding to a quality alert.

## Security, cost and reliability notes

- Evaluation datasets can contain sensitive data: store them securely, use synthetic data where possible.
- Use OIDC federation for pipelines; the evaluation identity needs only the access it requires.
- Budget for evaluation and red teaming tokens; they can rival production cost for small apps.
- Keep the previous version deployable for fast rollback.

## Common mistakes

| Mistake | Fix |
|---|---|
| Evaluating only before launch | Gates on every change plus continuous evaluation |
| Thresholds picked arbitrarily | Derive from baseline and business tolerance |
| Datasets that never change | Feed production failures back in |
| Red teaming once | Automate and schedule |

## Check yourself

1. What versioned assets does a GenAIOps pipeline manage?
2. Why combine absolute thresholds with regression checks?
3. What does a red teaming gate measure?
4. How do production failures improve future releases?
5. How do you manage evaluation cost and time?

<details>
<summary>Answers</summary>

1. Code, prompts, agent definitions, retrieval settings, IaC and evaluation datasets.
2. Absolute floors guarantee minimum quality; regression checks stop gradual decline even above the floor.
3. Attack success rate across risk categories and attack strategies.
4. Failed and thumbs-down cases are reviewed and added to the evaluation dataset.
5. Smoke sets per PR and full sets nightly, cost-effective judge models, and sampling in production.

</details>

## References

- [Run evaluations in CI/CD (GitHub Actions)](https://learn.microsoft.com/azure/foundry/how-to/evaluation-github-action)
- [Run AI red teaming scans](https://learn.microsoft.com/azure/foundry/how-to/develop/run-scans-ai-red-teaming-agent)
- [AI Red Teaming Agent](https://learn.microsoft.com/azure/foundry/concepts/ai-red-teaming-agent)
- [Observability in generative AI](https://learn.microsoft.com/azure/foundry/concepts/observability)
