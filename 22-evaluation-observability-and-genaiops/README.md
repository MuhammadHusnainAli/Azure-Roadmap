# 22 · Evaluation, Observability and GenAIOps

> Tracks: E ● A ● C ○ · Part D · AI on Azure

Generative AI applications and agents don't have a single accuracy number, behave non-deterministically, and can degrade without any code changing. This topic covers the practices and Microsoft Foundry capabilities that take them from experiment to production and keep them healthy: the GenAIOps lifecycle; evaluation fundamentals; built-in, custom and rubric evaluators and how to calibrate LLM judges; evaluation gates in GitHub Actions and Azure DevOps; tracing with OpenTelemetry GenAI semantic conventions and Application Insights; continuous evaluation, monitoring dashboards and Insights; trace-driven agent optimisation with the agent optimizer; automated red teaming with the AI Red Teaming Agent and PyRIT; and measuring business value so agents earn their keep.

## What you will be able to do

- Map an AI app or agent onto the GenAIOps lifecycle with artefacts, owners and gates.
- Design offline and online evaluation with the right data and judges.
- Select built-in evaluators for RAG, agents and safety, and write custom and rubric evaluators.
- Calibrate LLM-as-judge evaluators against human labels.
- Gate releases on quality and safety regressions in CI/CD with keyless auth.
- Instrument agents with OpenTelemetry and analyse traces in Foundry and Application Insights.
- Run continuous and scheduled evaluation with dashboards and alerts on quality, safety, latency and cost.
- Improve agents from production traces with datasets, Insights and the agent optimizer.
- Run automated red teaming scans and interpret Attack Success Rate.
- Define business KPIs, baselines and an ROI model for an agent.

## Prerequisites

[Topic 16](../16-prompt-engineering-and-model-customisation/README.md) (prompts), [Topic 17](../17-rag-and-knowledge/README.md) (RAG, especially file 12), and [Topic 18](../18-agents-with-foundry-agent-service/README.md) (Foundry agents). [Topic 21](../21-azure-machine-learning-and-mlops/README.md) file 09 helps for CI/CD patterns.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [GenAIOps Lifecycle](01-genaiops-lifecycle.md) | 35 min | Lifecycle map, definition of done |
| 02 | [Evaluation Fundamentals](02-evaluation-fundamentals.md) | 40 min | Test dataset, first evaluation |
| 03 | [Built-In Evaluators](03-built-in-evaluators.md) | 45 min | Agent and safety evaluators |
| 04 | [Custom and Rubric Evaluators](04-custom-and-rubric-evaluators.md) | 45 min | Code grader, rubric, calibration |
| 05 | [Evaluation in CI/CD](05-evaluation-in-ci-cd.md) | 50 min | GitHub Action gate with OIDC |
| 06 | [Tracing with OpenTelemetry](06-tracing-with-opentelemetry.md) | 45 min | Server- and client-side tracing |
| 07 | [Continuous Evaluation and Monitoring](07-continuous-evaluation-and-monitoring.md) | 45 min | Evaluation rule, alerts |
| 08 | [Agent Optimisation](08-agent-optimisation.md) | 40 min | Traces to dataset, optimizer run |
| 09 | [AI Red Teaming](09-ai-red-teaming.md) | 50 min | Local and cloud scans |
| 10 | [Measuring Business Value](10-measuring-business-value.md) | 35 min | KPIs, ROI model |

**AI engineers:** all of it, with 03–07 in depth. **Architects:** 01, 02, 05, 06, 07 and 09 for platform design. **Consultants:** 01, 02, 09 (concepts) and 10.

## Completion checklist

- [ ] I can explain how GenAIOps differs from MLOps and list the artefacts to version.
- [ ] I have built a stratified evaluation dataset including adversarial and out-of-scope cases.
- [ ] I have run an evaluation with agent, RAG and safety evaluators and interpreted the failures.
- [ ] I have written a code-based evaluator and a rubric, and measured judge agreement with humans.
- [ ] I have a CI workflow that compares a candidate agent version against the baseline and blocks regressions.
- [ ] I have server-side and client-side traces in Application Insights and can query them with KQL.
- [ ] I have configured continuous evaluation, a scheduled evaluation and alerts.
- [ ] I have generated a dataset from traces and run an optimisation, reviewing the result before promotion.
- [ ] I have run a red teaming scan, applied a mitigation and compared ASR.
- [ ] I can present an agent's KPIs, baseline and ROI to a business stakeholder.

## Maps to

- **AI-103**: evaluate, monitor and optimise AI apps and agents; implement responsible AI safety testing.
- **AI-300**: GenAIOps alongside MLOps (evaluation, monitoring, CI/CD).
- **AB-100** (adjacent): measuring agent business value.

## Next

23 · [Responsible AI, AI Security and Governance](../23-responsible-ai-security-and-governance/README.md)
