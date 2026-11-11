# Requirements and Constraints

## Why it matters

Most architecture failures trace back to requirements: an availability target nobody stated, a data residency rule discovered after go-live, a peak load ten times higher than assumed, a budget that rules out the chosen design. Architects turn vague goals ("it must be fast, secure and scalable") into **measurable requirements**, surface **constraints** that limit options, and make **assumptions** explicit so they can be validated. This is also where consultants add the most value in early engagements: asking the right questions before anyone draws boxes.

## Key concepts

### Requirement types

| Type | Describes | Example |
|---|---|---|
| **Business requirements** | Outcomes and value | Reduce claim handling time by 40% |
| **Functional requirements** | What the system does | Users upload claim documents; the system extracts fields and routes for approval |
| **Non-functional requirements (NFRs) / quality attributes** | How well it does it | 99.9% availability; p95 < 2 s; RPO 15 min; GDPR compliance |
| **Transition requirements** | Migration, cutover, coexistence | Parallel run for one month with the legacy system |

### Quality attributes (NFRs) to capture

| Attribute | Measurable form |
|---|---|
| Availability | SLO per user flow (99.9% monthly) |
| Recoverability | RTO, RPO |
| Performance | Latency percentiles at a given load; throughput |
| Scalability | Peak users/requests; growth rate; burst patterns |
| Security | Authentication methods, data classification, encryption, threat model outcomes |
| Compliance | Regulations, certifications, residency |
| Operability | Deployment frequency, MTTR, monitoring coverage |
| Cost | Budget per month/year; cost per transaction |
| Maintainability | Supported languages/platforms, team skills |
| Usability and accessibility | WCAG level |
| Sustainability | Efficiency targets |
| AI quality | Groundedness, task completion, safety thresholds (Topic 22) |

### Constraints

| Constraint | Example |
|---|---|
| **Technical** | Must integrate with SAP ECC; must use existing ExpressRoute; .NET skills only |
| **Regulatory** | Data stays in the EU; PCI DSS scope reduction |
| **Organisational** | Approved services list; security team mandates private endpoints |
| **Commercial** | Fixed budget; existing licences (AHB); MACC to consume |
| **Time** | Go-live before contract end |
| **Platform** | Landing zone policies, allowed regions and SKUs |

### Assumptions and risks

Assumptions are things believed true but not verified (for example "peak load is 500 requests/second"). Record them with owners and validation plans; unvalidated assumptions become risks.

### Prioritisation

Use **MoSCoW** (Must, Should, Could, Won't) or weighted scoring. Quality attribute **scenarios** make NFRs concrete: "When a zone fails during peak load, the checkout flow continues with < 1% errors and recovers full capacity within 5 minutes."

## How it works on Azure

Requirements workshop agenda (half day):

1. Business goals and success measures.
2. User flows and personas; criticality per flow.
3. Quality attributes with targets (availability, RTO/RPO, performance, security, compliance, cost).
4. Constraints (technical, regulatory, organisational, commercial, time).
5. Assumptions and open questions with owners.
6. Prioritisation.

Translate requirements into Azure design inputs:

| Requirement | Design implication |
|---|---|
| 99.95% availability for checkout | Zone-redundant services, health probes, possibly multi-region |
| RPO 15 min, RTO 1 h | SQL failover group or geo-replication, tested runbooks |
| EU data residency | EU regions, Data Zone EU deployments for AI, policy to deny other regions |
| p95 < 300 ms for EU users | Front Door, caching, regional deployment |
| PCI DSS | Segmentation, tokenisation, WAF, logging, MCSB/PCI controls |
| Budget €10k/month | PaaS and autoscale; commitments for baseline |

Template for an NFR:

```text
ID: NFR-AVAIL-01
Statement: The claim submission flow is available 99.9% per calendar month, measured by synthetic and real-user success rate.
Rationale: Contractual SLA with brokers.
Priority: Must
Verification: Availability tests from 3 regions; SLO dashboard.
Owner: Product owner, claims
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Precision** | Make NFRs measurable and testable; avoid "fast" or "secure" without numbers. |
| **Per flow** | Different flows have different targets. |
| **Challenge** | Push back on unjustified targets (five nines for internal tools). |
| **Traceability** | Link requirements to design decisions (ADRs) and tests. |
| **Living document** | Update as assumptions are validated. |

## Hands-on

1. Run a requirements workshop for a sample use case using the agenda.
2. Write ten NFRs using the template.
3. List constraints and assumptions with owners.
4. Translate each NFR into Azure design implications.
5. Write three quality attribute scenarios.

## Security, cost and reliability notes

- **Security requirements** should come from threat modelling and data classification, not generic lists.
- **Cost constraints** shape architecture as much as technical ones; capture them early.
- **Reliability targets** must be agreed with business owners who understand the cost.

## Common mistakes

| Mistake | Fix |
|---|---|
| Vague NFRs | Measurable targets |
| One target for the whole system | Per flow |
| Undocumented assumptions | Assumptions log |
| Discovering compliance late | Regulatory questions in the first workshop |
| Requirements not linked to design | Traceability to ADRs and tests |

## Check yourself

1. Distinguish business, functional, non-functional and transition requirements.
2. List eight quality attributes.
3. Give examples of four constraint types.
4. Why record assumptions?
5. What is a quality attribute scenario?
6. Translate "EU data residency" into Azure design implications.

<details>
<summary>Answers</summary>

1. Business: outcomes; functional: behaviour; non-functional: quality levels; transition: migration and coexistence needs.
2. Any eight: availability, recoverability, performance, scalability, security, compliance, operability, cost, maintainability, usability/accessibility, sustainability, AI quality.
3. Technical (integration), regulatory (residency), organisational (approved services), commercial (budget), time (deadline), platform (policies).
4. So they can be validated; unvalidated assumptions become risks.
5. A concrete stimulus–response statement making an NFR testable.
6. EU regions only, Data Zone EU AI deployments, Azure Policy denying other regions, backups and DR in EU regions, check service residency.

</details>

## References

- [Well-Architected: design for business requirements](https://learn.microsoft.com/azure/well-architected/reliability/principles)
- [Reliability metrics and targets](https://learn.microsoft.com/azure/well-architected/reliability/metrics)
- [Performance efficiency principles](https://learn.microsoft.com/azure/well-architected/performance-efficiency/principles)
- [Security principles](https://learn.microsoft.com/azure/well-architected/security/principles)
