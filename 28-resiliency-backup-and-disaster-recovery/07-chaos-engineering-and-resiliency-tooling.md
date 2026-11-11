# Chaos Engineering and Resiliency Tooling

## Why it matters

Every architecture diagram shows zone redundancy and failover arrows; far fewer have been proven under real failure. Retries that amplify load, health probes that check the wrong thing, a "zone-redundant" setting missed on one dependency, DNS caching that delays failover, a runbook that references a deleted script: these are found either in a controlled experiment or during a real outage. **Chaos engineering** deliberately injects faults to test hypotheses about resilience, and **resiliency tooling** helps you see posture and gaps continuously. On Azure that means **Azure Chaos Studio** (Workspaces and Scenarios, Preview), **Azure Infrastructure Resiliency Manager** (Preview) with zone resilience goals, recommendations, recovery plans and **Availability Zone Down Drills**, plus Advisor reliability recommendations and Well-Architected reviews.

## Key concepts

### Chaos engineering method

1. **Define steady state** with SLIs (success rate, latency, throughput).
2. **Hypothesis**: "If zone 1 fails, the checkout success rate stays above 99.5% and recovers within 2 minutes."
3. **Design the experiment**: fault, targets, duration, blast radius, abort conditions.
4. **Run** (staging first, under load), observe telemetry.
5. **Analyse and fix**; repeat; automate as regression tests.

### Azure Chaos Studio

| Element | Detail |
|---|---|
| **Workspaces** (Preview) | Connect to a scope (subscription, resource group or service group), discover resources and recommend scenarios |
| **Scenarios** (Preview) | Preconfigured resilience tests: **Compute Zone Down**, DNS outage, networking outages, database failovers, cache stampedes, messaging disruptions; customisable in the scenario designer |
| **Faults** | Service-direct (VM shutdown, VMSS zone down, AKS Chaos Mesh faults, Cosmos DB failover, NSG rule changes, Key Vault deny access) and agent-based (CPU/memory pressure, network latency/loss, kill process) |
| **Safety** | Managed identity with explicit permissions per target; stop experiments; scoped targets |

### Azure Infrastructure Resiliency Manager (Preview)

A global, nonregional experience within Resiliency in Azure, organised around three journeys:

| Journey | Capabilities |
|---|---|
| **Start Resilient** | **Azure Copilot Resiliency Agent** reviews designs (natural language, diagrams or IaC) and generates resilient templates; also available in GitHub Copilot (limited preview) |
| **Get Resilient** | Resource resilience posture without setup; application discovery into **service groups**; assign **zone resilience goals**; recommendations (also surfaced in Advisor) with portal or agent-assisted remediation |
| **Stay Resilient** | Reassess service groups; **recovery orchestration plans** for resources needing active failover (zonal failover, Preview); **Availability Zone Down Drills** that simulate zone outages on supported resources |

Free during preview. Drill and plan configuration alone doesn't guarantee recovery: validate with drills and update configuration.

### Other tooling

| Tool | Use |
|---|---|
| **Azure Advisor reliability** | Recommendations and reliability workbook |
| **Well-Architected Review** | Assessment against the reliability pillar |
| **Azure Monitor** | SLIs, alerts, dashboards during experiments |
| **Azure Load Testing** | Realistic load during chaos (Topic 27 file 08) |
| **Resiliency reviews / game days** | Cross-team exercises using scenarios |

## How it works on Azure

Zone-down experiment on a VM Scale Set behind a load balancer, running while a load test drives traffic:

```text
Steady state: AppRequests success ≥ 99.5%, p95 < 800 ms
Experiment (Chaos Studio): VMSS shutdown fault, zone 1, 10 minutes, abort if success < 95% for 2 minutes
Observe: Application Insights availability, Load Balancer health probe status, VMSS instance counts, autoscale actions
Expected: traffic shifts to zones 2–3; autoscale adds capacity; success ≥ 99.5% after 2 minutes
```

Chaos experiment identity permissions (example): the experiment's managed identity needs **Virtual Machine Contributor** on the target VMSS to shut down instances; nothing broader.

Infrastructure Resiliency Manager workflow:

1. Discover the application and create a **service group**.
2. Assign a **zone resilience goal**; review posture and recommendations (for example "Storage account is LRS", "SQL database not zone redundant").
3. Remediate via portal or Resiliency Agent assistance; reassess.
4. Define an **Availability Zone Down Drill** for supported resources; execute from the portal; review results.
5. Configure a recovery orchestration plan for resources needing active failover.

Record findings in a resilience review: hypothesis, outcome, root cause of any failure, fix, owner, re-test date.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Where to start** | Staging with production-like topology and load; production only with mature observability, abort conditions and stakeholder agreement. |
| **Which faults first** | Most likely and most impactful: zone loss, dependency latency/timeouts, throttling, DNS, certificate expiry, bad configuration. |
| **Cadence** | Quarterly game days; automated experiments in CI for critical services. |
| **Preview tooling** | Chaos Studio Workspaces/Scenarios and IRM are Preview; use for insight, validate before relying on them as compliance evidence. |
| **Ownership** | Service teams own experiments; platform provides tooling and templates. |
| **AI workloads** | Include model throttling and region failover scenarios via gateway configuration (file 06). |

## Hands-on

1. Create a Chaos Studio workspace for a staging resource group and run the Compute Zone Down scenario under load.
2. Run an agent-based network latency fault on an API's dependency; verify timeouts and circuit breakers.
3. In Infrastructure Resiliency Manager, create a service group, assign a zone resilience goal and remediate two recommendations.
4. Execute an Availability Zone Down Drill on a supported resource.
5. Write a resilience review report with findings and owners.

## Security, cost and reliability notes

- **Permissions**: experiments can cause outages; restrict who can create and run them; least-privilege experiment identities.
- **Abort conditions**: always define and test stop mechanisms.
- **Communication**: inform on-call and stakeholders before experiments.
- **Cost**: experiments themselves are cheap; load tests and extra capacity cost more.

## Common mistakes

| Mistake | Fix |
|---|---|
| Chaos without a hypothesis or steady state | Define SLIs and expectations |
| Running in production first | Staging first, then carefully |
| Experiments without load | Combine with load testing |
| No follow-up on findings | Owners and re-test dates |
| Broad permissions for experiment identities | Target-specific roles |

## Check yourself

1. Describe the chaos engineering method.
2. What are Chaos Studio workspaces and scenarios?
3. Name three Chaos Studio fault types.
4. What are the three journeys in Infrastructure Resiliency Manager?
5. What are Availability Zone Down Drills?
6. Why combine chaos experiments with load tests?

<details>
<summary>Answers</summary>

1. Define steady state, form a hypothesis, design an experiment with blast radius and abort conditions, run, observe, analyse and fix, repeat.
2. Workspaces connect to a scope, discover resources and recommend tests; scenarios are preconfigured resilience tests such as Compute Zone Down (both Preview).
3. Any three: VM/VMSS shutdown or zone down, AKS Chaos Mesh faults, Cosmos DB failover, NSG changes, Key Vault deny, CPU/memory pressure, network latency, kill process.
4. Start Resilient, Get Resilient, Stay Resilient.
5. Simulated zone outages on supported resources to validate cross-zone resilience before a real outage.
6. Many resilience issues (capacity, retry storms, failover timing) only appear under realistic load.

</details>

## References

- [What is Azure Chaos Studio?](https://learn.microsoft.com/azure/chaos-studio/chaos-studio-overview)
- [Infrastructure Resiliency Manager overview](https://learn.microsoft.com/azure/resiliency/infrastructure-resiliency-manager-overview)
- [Availability Zone Down Drills](https://learn.microsoft.com/azure/resiliency/availability-zone-down-drills-about)
- [Resiliency goals and recommendations](https://learn.microsoft.com/azure/resiliency/goals-recommendations-about)
- [Reliability testing strategy (Well-Architected)](https://learn.microsoft.com/azure/well-architected/reliability/testing-strategy)
