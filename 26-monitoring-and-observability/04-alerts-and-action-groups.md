# Alerts and Action Groups

## Why it matters

Telemetry nobody looks at doesn't prevent incidents. Alerts turn data into action: they notify the right people, open tickets and trigger automation before customers notice. Badly designed alerting is just as harmful as none: hundreds of noisy notifications train teams to ignore them, and the one that matters gets missed. **Azure Monitor alerts** support metric, log search, activity log, Prometheus and smart detection signals, with **dynamic thresholds** that learn normal behaviour, **action groups** for notifications and automation, and **alert processing rules** for suppression and routing at scale. This file covers designing an alerting strategy that's actionable, owned and cheap to run.

## Key concepts

### Alert types

| Type | Signal | Use |
|---|---|---|
| **Metric alerts** | Platform, custom, Application Insights or log-converted metrics; multi-resource and multi-condition | Fast, cheap alerts on numeric signals (CPU, error rate, latency, 429s) |
| **Log search alerts** | KQL query over Log Analytics at a frequency | Complex conditions, correlation, text patterns; dynamic thresholds (Preview) |
| **Simple log search alerts** | Evaluate each row individually | Event-style conditions |
| **Activity log alerts** | Control-plane events; includes **Service Health** and **Resource Health** alerts | Deletions, policy changes, platform incidents (file 08) |
| **Prometheus alerts** | PromQL rules over managed Prometheus | Kubernetes and container workloads (file 06) |
| **Smart detection** | Application Insights anomaly detection | Failure and performance anomalies |
| **Query-based metric alerts** (Preview) | Prometheus and OpenTelemetry metrics | Metric-style alerting on OTel data |

### Anatomy

| Element | Purpose |
|---|---|
| **Scope** | Resources, resource groups or subscriptions monitored |
| **Condition** | Signal, aggregation, operator, threshold (static or **dynamic**), evaluation frequency and window |
| **Severity** | Sev 0 (critical) to Sev 4 (verbose) |
| **Action group** | Who/what is notified |
| **State** | Fired/resolved (system) and New/Acknowledged/Closed (user response); alerts kept for 30 days |

### Dynamic thresholds

Machine-learned thresholds based on historical patterns (including seasonality), with sensitivity (low, medium, high) and minimum failing periods. Use for signals with variable but regular patterns (traffic, latency), and static thresholds for hard limits (disk 90%, error rate 5%, PTU utilisation 95%).

### Action groups

Reusable sets of notifications and actions: email, SMS, voice, Azure app push, **webhooks/secure webhooks**, **Logic Apps**, **Azure Functions**, **Automation runbooks**, **ITSM** connector, **Event Hubs**. Use **common alert schema** for consistent payloads to downstream tools (PagerDuty, ServiceNow, Teams via Logic Apps).

### Alert processing rules

Applied to fired alerts across scopes to:

- **suppress** notifications during planned maintenance windows (one-off or recurring);
- **add action groups** at scale (for example, route every Sev 0/1 in a subscription to the on-call team);
- filter by resource type, severity, alert rule or context.

### Integration

Fired alerts can start investigations in the **Azure Copilot Observability Agent** and Azure Monitor issues ([Topic 24 file 08](../24-copilot-and-low-code-agents/08-azure-copilot.md)), or trigger the **Azure SRE Agent** ([Topic 24 file 09](../24-copilot-and-low-code-agents/09-azure-sre-agent.md)).

## How it works on Azure

Bicep: action group and a metric alert with dynamic threshold on App Service 5xx:

```bicep
resource ag 'Microsoft.Insights/actionGroups@2023-01-01' = {
  name: 'ag-orders-oncall'
  location: 'global'
  properties: {
    groupShortName: 'orders'
    enabled: true
    emailReceivers: [ { name: 'oncall', emailAddress: 'orders-oncall@contoso.com', useCommonAlertSchema: true } ]
    webhookReceivers: [ { name: 'pagerduty', serviceUri: pagerDutyUri, useCommonAlertSchema: true } ]
  }
}

resource http5xx 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'orders-api-5xx-dynamic'
  location: 'global'
  properties: {
    severity: 1
    enabled: true
    scopes: [ appService.id ]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT15M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.MultipleResourceMultipleMetricCriteria'
      allOf: [ {
        criterionType: 'DynamicThresholdCriterion'
        name: 'http5xx'
        metricName: 'Http5xx'
        operator: 'GreaterThan'
        alertSensitivity: 'Medium'
        failingPeriods: { numberOfEvaluationPeriods: 3, minFailingPeriodsToAlert: 2 }
        timeAggregation: 'Total'
      } ]
    }
    actions: [ { actionGroupId: ag.id } ]
  }
}
```

Log search alert query (failed logins spike by account):

```kusto
SigninLogs
| where ResultType != 0
| summarize failures = count() by UserPrincipalName, bin(TimeGenerated, 5m)
| where failures > 20
```

Maintenance suppression with an alert processing rule:

```bash
az monitor alert-processing-rule create -g rg-mon -n suppress-weekend-patching \
  --rule-type RemoveAllActionGroups --scopes /subscriptions/<sub-id>/resourceGroups/rg-app \
  --schedule-recurrence-type Weekly --schedule-recurrence Saturday \
  --schedule-start-time 22:00:00 --schedule-end-time 23:59:59 --description "Weekly patch window"
```

**Azure Monitor Baseline Alerts (AMBA)** provides policy-driven, recommended alert definitions for platform landing zones and common services; a good starting point.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Symptoms over causes** | Alert on user-facing symptoms (error rate, latency, availability) at high severity; causes (CPU) at lower severity or dashboards. |
| **Severity model** | Agree Sev definitions and response expectations; page only Sev 0–1. |
| **Metric vs log** | Metric alerts where possible (cheaper, faster); log alerts for complex conditions. |
| **Ownership** | Every alert maps to an owning team's action group. |
| **Scale** | Deploy alerts as code (Bicep/Terraform/AMBA policies); processing rules for routing. |
| **Noise** | Review alert volume monthly; tune or delete alerts nobody acts on. |

## Hands-on

1. Create an action group with email and a Logic App posting to Teams using the common alert schema.
2. Create a dynamic-threshold metric alert on App Service 5xx and a static alert on response time.
3. Create a log search alert on failed sign-ins.
4. Create an alert processing rule suppressing actions during a weekly maintenance window.
5. Deploy AMBA policies to a test management group and review the alerts created.

## Security, cost and reliability notes

- **Webhook security**: use secure webhooks (Entra auth) for sensitive endpoints.
- **Cost**: log search alerts are billed per rule and frequency; metric alerts per time series monitored.
- **Single point of failure**: don't route all paging through one email inbox; use on-call tooling.
- **Test alerts**: regularly verify action groups (test action group feature).

## Common mistakes

| Mistake | Fix |
|---|---|
| Alerting on every metric | Symptom-based, owned alerts |
| Static thresholds for seasonal traffic | Dynamic thresholds |
| Disabling alerts during maintenance manually | Alert processing rules |
| One action group for everything | Per-team and per-severity routing |
| Alerts configured by hand in the portal | Alerts as code |

## Check yourself

1. Name six alert types and a use for each.
2. When should you use dynamic thresholds?
3. What can action groups do?
4. What are alert processing rules for?
5. How long are fired alerts retained?
6. What is AMBA?

<details>
<summary>Answers</summary>

1. Metric (error rate), log search (complex KQL conditions), simple log search (per-row events), activity log (deletions, Service/Resource Health), Prometheus (Kubernetes), smart detection (anomalies), query-based metric alerts (OTel/Prometheus, Preview).
2. For signals with variable but regular patterns where static thresholds are noisy.
3. Notify by email, SMS, voice, push; call webhooks, Logic Apps, Functions, runbooks, ITSM and Event Hubs.
4. Suppressing notifications (maintenance) and adding action groups or filters at scale to fired alerts.
5. 30 days.
6. Azure Monitor Baseline Alerts: policy-driven recommended alert definitions for landing zones and services.

</details>

## References

- [What are Azure Monitor alerts?](https://learn.microsoft.com/azure/azure-monitor/alerts/alerts-overview)
- [Dynamic thresholds](https://learn.microsoft.com/azure/azure-monitor/alerts/alerts-dynamic-thresholds)
- [Action groups](https://learn.microsoft.com/azure/azure-monitor/alerts/action-groups)
- [Alert processing rules](https://learn.microsoft.com/azure/azure-monitor/alerts/alerts-processing-rules)
- [Azure Monitor Baseline Alerts](https://azure.github.io/azure-monitor-baseline-alerts/)
