# Reliability Fundamentals

## Why it matters

"It must never go down" isn't a requirement; it's a wish with an unlimited budget. Real reliability engineering starts by agreeing how reliable each workload needs to be, how much data loss and downtime the business can tolerate, and which failures are likely, then designing and paying for exactly that. Over-engineering wastes money and adds complexity (which itself causes outages); under-engineering surprises the business during the first regional incident. The vocabulary in this file (SLOs, RTO, RPO, failure mode analysis, redundancy versus recoverability) is what architects, consultants and business stakeholders use to make those trade-offs explicitly, following the Azure Well-Architected Framework reliability pillar.

## Key concepts

### Targets

| Term | Meaning | Example |
|---|---|---|
| **SLA** | Contractual commitment from a provider (Azure service SLAs) with service credits | App Service 99.95% |
| **SLO** | Your internal target for a user journey | 99.9% of checkout requests succeed within 1 s per 30 days |
| **SLI** | The measured indicator | Ratio of successful checkout requests |
| **Error budget** | Allowed unreliability (100% − SLO) | 0.1% ≈ 43 minutes/month |
| **RTO** (recovery time objective) | Maximum acceptable time to restore service | 4 hours |
| **RPO** (recovery point objective) | Maximum acceptable data loss measured in time | 15 minutes |
| **MTTR / MTBF** | Mean time to recover / between failures | Operational metrics |

Composite SLA: dependent services multiply (99.95% × 99.99% × 99.9% ≈ 99.84%); redundancy across independent paths raises availability. Your workload SLO can't exceed what its critical dependencies and design support.

### Availability in time

| Availability | Downtime per 30 days |
|---|---|
| 99% | 7.2 hours |
| 99.9% | 43.2 minutes |
| 99.95% | 21.6 minutes |
| 99.99% | 4.3 minutes |

### Criticality tiers

| Tier | Example | Typical targets | Design |
|---|---|---|---|
| **Mission-critical** | Payments, emergency services | 99.99%, RTO minutes, RPO ≈ 0 | Multi-region active-active, zone redundancy everywhere |
| **Business-critical** | Customer portal, ERP | 99.9–99.95%, RTO 1–4 h, RPO ≤ 15 min | Zone-redundant primary + warm/cold secondary region |
| **Business-operational** | Internal tools | 99.5–99.9%, RTO 8–24 h, RPO hours | Zone redundancy, backup and restore |
| **Administrative** | Dev/test, reporting | Best effort, RTO days | Backup only |

### Redundancy vs recoverability

| Approach | Protects against | Doesn't protect against |
|---|---|---|
| **Redundancy** (zones, replicas, multi-region) | Hardware, zone and region failures | Data corruption, deletion, ransomware, bad deployments (they replicate!) |
| **Recoverability** (backups, point-in-time restore, immutable copies) | Corruption, deletion, ransomware, logical errors | Requires restore time (RTO) |

You need both: replication keeps you running; backups let you go back in time.

### Failure mode analysis (FMA)

Identify components and dependencies, list how each can fail (outage, latency, throttling, misconfiguration, expired certificate, quota exhaustion, bad deployment), assess likelihood and impact, define detection and mitigation. Include non-technical dependencies (DNS, identity, certificates, third-party APIs, people).

### Resilience patterns

Retries with exponential backoff and jitter; timeouts; circuit breakers; bulkheads; queue-based load levelling; idempotency; graceful degradation; health endpoint monitoring; throttling and rate limiting; compensating transactions.

## How it works on Azure

Reliability workflow (Well-Architected):

1. **Identify user and system flows** and rate their criticality.
2. **Define SLOs, RTO and RPO** per flow with business owners.
3. **Run FMA** on each flow's components and dependencies.
4. **Design** redundancy (zones, regions) and recoverability (backups, PITR) to meet targets (files 02–06).
5. **Implement health modelling and monitoring** with SLIs and alerts (Topic 26).
6. **Test**: chaos experiments, zone-down drills, DR drills (file 07).
7. **Operate**: incident management, post-incident reviews, error budget policy (file 08).

SLI query (Application Insights) for a checkout SLO:

```kusto
AppRequests
| where TimeGenerated > ago(30d) and Name == "POST /checkout"
| summarize total = count(), good = countif(Success == true and DurationMs < 1000)
| extend sli = 100.0 * good / total, slo = 99.9, budgetUsedPct = 100.0 * (total - good) / (total * 0.001)
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Targets per flow** | Different flows within one app can have different SLOs (checkout vs reporting). |
| **Cost vs reliability** | Each extra nine roughly multiplies cost and complexity; justify with business impact. |
| **Error budget policy** | When the budget is exhausted, prioritise reliability work over features. |
| **Dependencies** | Know each dependency's SLA and failure behaviour; design fallbacks. |
| **Simplicity** | Simpler architectures fail less; avoid unnecessary multi-region complexity. |
| **Shared responsibility** | Azure provides resilient infrastructure; you configure zone redundancy, replication, backups and tested recovery. |

## Hands-on

1. List the user flows of a workload; assign criticality, SLO, RTO and RPO with a business owner.
2. Calculate the composite SLA of the critical flow's dependencies.
3. Run an FMA workshop: table of components, failure modes, impact, detection, mitigation.
4. Implement the SLI query and an error-budget burn alert.
5. Identify where the design relies on redundancy only and add recoverability.

## Security, cost and reliability notes

- **Security incidents are reliability incidents**: ransomware and credential compromise need recoverability (immutable backups).
- **Cost**: multi-region active-active can double or triple cost; tier workloads honestly.
- **Change is the top cause of outages**: safe deployment practices are a reliability control (Topic 27 file 07).

## Common mistakes

| Mistake | Fix |
|---|---|
| "Five nines" for everything | Tier by business impact |
| Relying on replication as backup | Add point-in-time and immutable backups |
| SLOs on infrastructure metrics only | User-journey SLIs |
| Ignoring dependencies (DNS, identity, third parties) | Include them in FMA |
| Untested DR plans | Regular drills |

## Check yourself

1. Define SLA, SLO, SLI and error budget.
2. What are RTO and RPO?
3. How do you calculate a composite SLA?
4. Why do you need both redundancy and recoverability?
5. What does failure mode analysis involve?
6. Name five resilience patterns.

<details>
<summary>Answers</summary>

1. SLA: provider's contractual commitment; SLO: internal target; SLI: measured indicator; error budget: allowed unreliability (100% − SLO).
2. RTO: maximum acceptable downtime; RPO: maximum acceptable data loss in time.
3. Multiply availabilities of serially dependent services; redundancy across independent paths improves it.
4. Redundancy handles infrastructure failures but replicates corruption and deletion; recoverability lets you restore to a good point.
5. Mapping components and dependencies, listing failure modes, assessing impact and likelihood, defining detection and mitigation.
6. Any five: retries with backoff, timeouts, circuit breakers, bulkheads, queue-based load levelling, idempotency, graceful degradation, health endpoints, throttling.

</details>

## References

- [Reliability metrics (Well-Architected)](https://learn.microsoft.com/azure/well-architected/reliability/metrics)
- [Failure mode analysis](https://learn.microsoft.com/azure/well-architected/reliability/failure-mode-analysis)
- [Reliability in Azure](https://learn.microsoft.com/azure/reliability/overview)
- [Reliability design patterns](https://learn.microsoft.com/azure/well-architected/reliability/design-patterns)
