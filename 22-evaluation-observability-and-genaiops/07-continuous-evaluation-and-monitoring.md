# Continuous Evaluation and Monitoring

## Why it matters

Passing pre-production evaluation proves an agent worked on your test set on the day you tested it. Production is different: users ask things nobody anticipated, retrieved content changes, tool APIs slow down, a model version is updated, and costs creep up as conversations get longer. **Continuous evaluation** scores a sample of real traffic with the same evaluators you used in CI, and **monitoring** watches operational signals (latency, success rate, tokens, cost) alongside those quality and safety scores, raising alerts before customers or auditors notice. Foundry's **Agent Monitoring Dashboard**, evaluation rules, scheduled evaluations and **Insights** provide this on top of Application Insights.

## Key concepts

### What to monitor

| Category | Signals | Typical threshold to investigate |
|---|---|---|
| **Operational** | Run success rate, errors, latency (p50/p95), throttling (429s) | Success < 95%; latency > 10 s (Microsoft's guidance for the dashboard) |
| **Cost** | Tokens per run, tokens per day, tool calls per run, cost per task | Trend increase > 20% week on week |
| **Quality** | Task adherence, intent resolution, groundedness, relevance, rubric score on sampled traffic | Drop vs baseline |
| **Safety and security** | Harm evaluators, indirect attack, sensitive data leakage, scheduled red-team results | Any high-severity finding |
| **User signals** | Thumbs up/down, escalations to humans, abandonment | Rising negative rate |

### Agent Monitoring Dashboard (Preview)

In the Foundry portal: **Build → Agents → (agent) → Monitor**. Summary cards (token usage, latency, run success rate, evaluation metrics, red-teaming results) plus charts for the selected time range. The **Monitor settings** gear configures:

| Setting | Purpose |
|---|---|
| **Recurring evaluations** (Preview) | Scheduled or continuous evaluation of agent traffic |
| **Red team scans** (Preview) | Scheduled adversarial tests (file 09) |
| **Alerts** (Preview) | Latency, token usage, evaluation score or red-team finding alerts |

Data lives in the connected Application Insights resource; retention and billing follow its configuration.

### Continuous vs scheduled evaluation

| | Continuous | Scheduled |
|---|---|---|
| **Trigger** | Evaluation rule on each completed response (`RESPONSE_COMPLETED`) | Recurrence (for example, daily at 09:00) |
| **Data** | Live responses, sampled | Traces in a time window, filtered by agent, up to `max_traces` |
| **Rate control** | `max_hourly_runs` (default 100 per hour) | Schedule frequency and trace cap |
| **Sampling (portal)** | Random or **Intelligent** sampling of live traffic | Window-based |
| **Best for** | Near-real-time safety and quality signals | Daily/weekly trend reports, heavier evaluators |

The Foundry project's **managed identity** needs the **Foundry User** role to run continuous evaluation rules.

### Insights in Foundry (Preview)

Insights analyses production traces with a GPT-5-class judge to group **recurring behaviour** into reviewable findings, each with representative traces, affected agent versions, a likely cause and a recommended next action. The first scan uses a 7-day lookback. It complements known-condition evaluators by surfacing failure patterns you didn't know to test. Prerequisites include Monitoring Reader on Application Insights for the user and project managed identity (plus Privileged Monitoring Data Reader if the GenAI content table is protected).

## How it works on Azure

### Continuous evaluation rule

```python
import os
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import (
    EvaluationRule, ContinuousEvaluationRuleAction, EvaluationRuleFilter, EvaluationRuleEventType,
)

project = AIProjectClient(endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"], credential=DefaultAzureCredential())
oai = project.get_openai_client()

ev = oai.evals.create(
    name="support-agent-online",
    data_source_config={"type": "azure_ai_source", "scenario": "responses"},
    testing_criteria=[
        {"type": "azure_ai_evaluator", "name": "violence", "evaluator_name": "builtin.violence"},
        {"type": "azure_ai_evaluator", "name": "xpia", "evaluator_name": "builtin.indirect_attack"},
        {"type": "azure_ai_evaluator", "name": "task_adherence", "evaluator_name": "builtin.task_adherence",
         "initialization_parameters": {"deployment_name": "gpt-5-mini"}},
    ],
)

project.evaluation_rules.create_or_update(
    id="support-agent-continuous",
    evaluation_rule=EvaluationRule(
        display_name="Support agent continuous evaluation",
        action=ContinuousEvaluationRuleAction(eval_id=ev.id, max_hourly_runs=60),
        event_type=EvaluationRuleEventType.RESPONSE_COMPLETED,
        filter=EvaluationRuleFilter(agent_name="support-agent"),
        enabled=True,
    ),
)
```

### Scheduled evaluation over traces

```python
from azure.ai.projects.models import Schedule, RecurrenceTrigger, DailyRecurrenceSchedule, EvaluationScheduleTask

run = {
    "eval_id": ev.id,
    "name": "daily-trace-eval",
    "data_source": {
        "type": "azure_ai_trace_data_source_preview",
        "trace_source": {"type": "agent_filter", "agent_name": "support-agent", "max_traces": 500},
    },
}
project.beta.schedules.create_or_update(
    schedule_id="support-agent-daily",
    schedule=Schedule(display_name="Daily trace evaluation", enabled=True,
                      trigger=RecurrenceTrigger(interval=1, schedule=DailyRecurrenceSchedule(hours=[6])),
                      task=EvaluationScheduleTask(eval_id=ev.id, eval_run=run)),
)
```

(The Microsoft sample also sets `start_time` and `end_time` on `trace_source`; check the current sample for the exact window semantics.)

### Azure Monitor alerts and workbooks

Beyond the dashboard's built-in alerts, use Application Insights data for log search alerts:

```kusto
dependencies
| where timestamp > ago(15m) and customDimensions["gen_ai.operation.name"] == "chat"
| summarize failures = countif(success == false), total = count()
| extend failure_rate = todouble(failures) / total
| where failure_rate > 0.05
```

Also use Azure OpenAI/Foundry platform metrics (requests, throttling, tokens, provisioned utilisation) and API Management gateway token metrics ([Topic 11 file 07](../11-integration-messaging-and-apis/07-api-management-as-ai-gateway.md)) for cost allocation per app or team. Route alerts through action groups to the owning team.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Sampling rate** | Enough samples per day for statistically meaningful trends (hundreds), within judge budget. Use intelligent sampling to favour informative traffic. |
| **Which evaluators online** | Cheap, reference-free and safety-critical: safety, XPIA, task adherence, groundedness. Heavier rubric evaluators on daily schedules. |
| **Alert levels** | Page for safety and outage; ticket for quality trends; weekly review for cost. |
| **Baselines** | Compare against the pre-production score for the same agent version; reset on version change. |
| **Content needed** | Continuous evaluation needs response content; govern access to evaluation results like traces. |
| **Separate monitoring per version** | Tag traces with agent version to compare canary and stable. |
| **Dashboards** | Foundry dashboard for agent owners; Azure Monitor workbooks or Grafana for platform-wide views. |

## Hands-on

1. Open the Monitor tab for a test agent; generate traffic and review tokens, latency and success rate.
2. Create a continuous evaluation rule with violence, indirect attack and task adherence; send 50 requests and view scores on the dashboard.
3. Create a daily scheduled evaluation over traces with a rubric evaluator.
4. Configure a dashboard alert on latency and a log search alert on failure rate.
5. Run an Insights scan; pick one Insight, confirm it with traces and decide the next action (new test case, prompt fix or owner escalation).

## Security, cost and reliability notes

- **Evaluation results include content** from real conversations; apply the same access control and retention as traces.
- **Cost drivers**: judge tokens per sampled response × evaluators; Application Insights ingestion; Insights scans. Cap with `max_hourly_runs`, sampling and schedules.
- **Hourly limit reached**: increase `max_hourly_runs` or accept sampling; don't silently lose safety coverage.
- **Preview**: dashboard, alerts, recurring evaluations and Insights are Preview; keep Azure Monitor alerts as a GA backstop.
- **Region**: safety evaluators run only in supported regions; check before designing monitoring for data-resident workloads.

## Common mistakes

| Mistake | Fix |
|---|---|
| Monitoring only latency and errors | Add quality, safety and cost signals |
| Evaluating 100% of traffic with expensive judges | Sample and schedule |
| Alerts with no owner | Route to the agent's owning team via action groups |
| No version tagging | Tag agent version to compare releases |
| Insights never acted on | Weekly triage turning findings into tests or fixes (file 08) |

## Check yourself

1. List the five signal categories for monitoring AI agents.
2. What does the Agent Monitoring Dashboard show, and what can its settings configure?
3. Compare continuous and scheduled evaluation.
4. What controls the rate of continuous evaluation, and what is the default?
5. What do Insights in Foundry provide?
6. Which identity needs which role for continuous evaluation?

<details>
<summary>Answers</summary>

1. Operational, cost, quality, safety/security, user signals.
2. Token usage, latency, run success rate, evaluation metrics and red-teaming results; settings configure recurring evaluations, red team scans and alerts (all Preview).
3. Continuous: evaluation rule triggered on completed responses, sampled live traffic, near real time. Scheduled: recurrence over traces in a window with a trace cap, for trends and heavier evaluators.
4. `max_hourly_runs` on the rule action; default 100 runs per hour (plus portal sampling choices).
5. Grouped recurring behaviours from production traces with evidence, affected versions, likely cause and recommended action, using a GPT-5-class judge and a 7-day initial lookback.
6. The Foundry project's managed identity needs the Foundry User role.

</details>

## References

- [Monitor agents with the Agent Monitoring Dashboard](https://learn.microsoft.com/azure/foundry/observability/how-to/how-to-monitor-agents-dashboard)
- [Insights in Foundry (Preview)](https://learn.microsoft.com/azure/foundry/observability/how-to/agent-insights)
- [Observability in generative AI](https://learn.microsoft.com/azure/foundry/concepts/observability)
- [Log search alerts in Azure Monitor](https://learn.microsoft.com/azure/azure-monitor/alerts/alerts-types)
- [Log end-user feedback](https://learn.microsoft.com/azure/foundry/observability/how-to/log-end-user-feedback)
