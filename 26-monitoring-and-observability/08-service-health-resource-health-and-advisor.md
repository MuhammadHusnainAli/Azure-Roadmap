# Service Health, Resource Health and Advisor

## Why it matters

Not every incident is your fault. Sometimes an Azure region has a networking issue, a storage scale unit degrades, a host is being repaired, or planned maintenance will reboot your VMs next Tuesday. Teams that can't distinguish platform problems from their own waste hours debugging healthy code, miss chances to fail over, and get surprised by retirements. **Azure Service Health** and **Resource Health** tell you what the platform is doing to the services and resources you use, and **Azure Advisor** turns telemetry and configuration into prioritised recommendations across reliability, security, performance, cost and operational excellence. Wiring these into alerts and governance is cheap and high-value.

## Key concepts

### Health services

| Service | Scope | Tells you |
|---|---|---|
| **Azure status** (status page) | Global, all services and regions | Widespread outages; public page |
| **Service Health** | Your subscriptions, services and regions | **Service issues** (outages), **planned maintenance**, **health advisories** (feature changes, required actions), **security advisories**, **billing updates**; root cause analyses (post-incident reviews) |
| **Resource Health** | Individual resources | Current and historical availability (Available, Unavailable, Degraded, Unknown) and whether causes were platform- or user-initiated |

Service Health is personalised: you see only events affecting services and regions you use, with tracking IDs, impacted resources, updates and RCAs. **Retirements** are surfaced as health advisories; many also appear in Advisor and in Azure Update Manager/feeds.

### Resource Health states

| State | Meaning |
|---|---|
| **Available** | No issues detected |
| **Unavailable** | Platform or non-platform event affecting the resource |
| **Degraded** | Reduced performance |
| **Unknown** | No signal for more than 10 minutes (may still be running) |

Resource Health also shows **planned** vs **unplanned** events and recommended actions.

### Azure Advisor

| Category | Examples |
|---|---|
| **Reliability** | Enable zone redundancy, configure backup, use availability zones, soft delete |
| **Security** | Recommendations from Defender for Cloud |
| **Performance** | SQL index suggestions, Redis sizing, Premium storage |
| **Cost** | Rightsize or shut down underused VMs, reservations and savings plans, idle resources |
| **Operational excellence** | Service retirements, deprecated APIs, best-practice configuration |

**Advisor Score** aggregates how well you follow recommendations per category; **workbooks** in Advisor provide reliability and cost reviews; recommendations can be exported, filtered, postponed or dismissed, and acted on through Azure Resource Graph and automation. Configure CPU thresholds for rightsizing recommendations.

## How it works on Azure

Service Health alert (activity log alert) for outages and maintenance in the regions and services you use:

```bicep
resource shAlert 'Microsoft.Insights/activityLogAlerts@2020-10-01' = {
  name: 'service-health-prod'
  location: 'global'
  properties: {
    enabled: true
    scopes: [ subscription().id ]
    condition: {
      allOf: [
        { field: 'category', equals: 'ServiceHealth' }
        { anyOf: [
            { field: 'properties.incidentType', equals: 'Incident' }
            { field: 'properties.incidentType', equals: 'Maintenance' }
            { field: 'properties.incidentType', equals: 'ActionRequired' }
            { field: 'properties.incidentType', equals: 'Security' } ] }
        { field: 'properties.impactedServices[*].ImpactedRegions[*].RegionName', containsAny: [ 'West Europe', 'North Europe' ] }
      ]
    }
    actions: { actionGroups: [ { actionGroupId: platformAg.id } ] }
  }
}
```

Resource Health alerts for critical resources (condition `category == 'ResourceHealth'` with `currentHealthStatus` Unavailable/Degraded and cause `PlatformInitiated`) route to the owning team.

Query Advisor recommendations at scale:

```kusto
advisorresources
| where type == "microsoft.advisor/recommendations"
| extend category = tostring(properties.category), impact = tostring(properties.impact),
         problem = tostring(properties.shortDescription.problem)
| summarize count() by category, impact, problem
| order by category asc, impact asc
```

Query Service Health events (including retirements) with Resource Graph:

```kusto
servicehealthresources
| where type =~ "microsoft.resourcehealth/events"
| extend eventType = tostring(properties.EventType), title = tostring(properties.Title), status = tostring(properties.Status)
| where eventType in ("HealthAdvisory", "PlannedMaintenance")
| project title, eventType, status, impactStart = todatetime(properties.ImpactStartTime)
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Who receives Service Health** | Platform/operations team for all; app owners for their regions and services; avoid emailing everyone. |
| **Incident triage** | Check Service Health and Resource Health first during incidents; link tracking IDs in your incident record. |
| **Retirement management** | Weekly review of health advisories and Advisor operational excellence; assign owners and dates. |
| **Advisor governance** | Use Advisor Score in monthly reviews; automate common fixes; postpone with reason rather than ignoring. |
| **Multi-region** | Use Service Health signals to trigger documented failover decisions (Topic 28). |

## Hands-on

1. Create Service Health alerts for your subscriptions and regions using the Bicep above.
2. Create a Resource Health alert for a production database.
3. Review the Service Health history and download an RCA (post-incident review) for a past event.
4. Run the Advisor Resource Graph query and build a workbook of high-impact recommendations by category.
5. List upcoming retirements from Service Health advisories and assign owners.

## Security, cost and reliability notes

- **Security advisories** should reach the security team, not only operations.
- **Cost**: Service Health and Advisor are free; activity log alerts are free.
- **Reliability**: platform-initiated unavailability is a trigger for your failover runbooks.
- **Don't over-trust "Available"**: Resource Health is platform-level; your app can still be broken.

## Common mistakes

| Mistake | Fix |
|---|---|
| Learning about outages from customers | Service Health alerts |
| Debugging app code during a platform incident | Check Service/Resource Health first |
| Ignoring retirement advisories | Tracked review with owners |
| Dismissing Advisor recommendations without reason | Postpone with justification |
| Service Health alerts to a shared inbox | Route to on-call tooling |

## Check yourself

1. Compare Azure status, Service Health and Resource Health.
2. What event types does Service Health report?
3. What are the Resource Health states?
4. Name Advisor's five categories with an example each.
5. How do you query Advisor recommendations across subscriptions?
6. How should Service Health feed incident response?

<details>
<summary>Answers</summary>

1. Azure status: global public view; Service Health: personalised events for your services and regions; Resource Health: availability of individual resources.
2. Service issues, planned maintenance, health advisories, security advisories, billing updates (plus RCAs).
3. Available, Unavailable, Degraded, Unknown.
4. Reliability (zone redundancy), security (Defender recommendations), performance (SQL indexes), cost (rightsizing), operational excellence (retirements).
5. Azure Resource Graph `advisorresources`.
6. Alerts route to on-call; check health during triage; link tracking IDs; trigger failover decisions for platform outages.

</details>

## References

- [What is Azure Service Health?](https://learn.microsoft.com/azure/service-health/overview)
- [Service Health overview](https://learn.microsoft.com/azure/service-health/service-health-overview)
- [Resource Health overview](https://learn.microsoft.com/azure/service-health/resource-health-overview)
- [Create Service Health alerts](https://learn.microsoft.com/azure/service-health/alerts-activity-log-service-notifications-portal)
- [Azure Advisor overview](https://learn.microsoft.com/azure/advisor/advisor-overview)
- [Advisor score](https://learn.microsoft.com/azure/advisor/advisor-score)
