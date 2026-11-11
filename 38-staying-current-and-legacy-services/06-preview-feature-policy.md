# Preview Feature Policy

## Why it matters

Many of the most interesting Azure AI capabilities launch in **Preview**, and some stay there for a long time. Clients want them, demos use them, and competitors announce them. But Preview features have no SLA, can change without notice and are sometimes withdrawn. Foundry's visual workflows and several Content Safety features are recent examples of features that retired. Engineers, architects and consultants need a **consistent policy** for evaluating, adopting and communicating Preview features, so innovation doesn't turn into production risk.

## Key concepts

### What Preview means

| Aspect | Preview | GA |
|---|---|---|
| SLA | None | Service SLA where published |
| Terms | **Supplemental Terms of Use for Microsoft Azure Previews** apply | Standard product terms |
| Support | Limited; may be excluded from some support plans | Full support |
| Stability | APIs and behaviour can change; breaking changes possible | Versioned, with deprecation policies |
| Compliance | May not be covered by compliance certifications or data residency commitments | Covered as documented |
| Availability | Limited regions; may require sign-up | Broad regions |
| Future | May become GA, change significantly or be withdrawn | Lifecycle policy applies |

Variants:
- **Private preview**: by invitation, under NDA.
- **Public preview**: open to all.
- **Limited Access**: GA or Preview features that need application and approval, such as some face and voice features.
- **Experimental**: some SDK or framework features.

### Risk tiers for using Preview

| Use | Acceptability |
|---|---|
| Learning, demos, PoCs | Fine, label clearly |
| Internal non-critical tools | Acceptable with an exit plan and owner |
| Customer-facing or regulated production | Generally avoid; requires explicit risk acceptance by the business owner and security, plus mitigations |
| Core platform dependencies | Avoid |

## How it works on Azure

### Evaluation checklist for a Preview feature

| Question | Why it matters |
|---|---|
| What problem does it solve that GA options can't? | Justifies the risk |
| Is there a GA alternative with acceptable trade-offs? | Often the right answer |
| Which regions, quotas and limits apply? | Feasibility |
| Does it process data outside the expected geography or compliance boundary? | Data residency and compliance |
| Is there a published roadmap or GA target? | Planning |
| What happens if it's withdrawn? Exit plan and effort? | Resilience |
| Can it be isolated behind an abstraction or feature flag? | Reversibility |
| Who accepts the risk? | Accountability |

### Engineering practices

- **Abstract** Preview dependencies behind interfaces so they can be swapped.
- **Feature-flag** Preview capabilities and keep a GA fallback path.
- **Pin API versions** and SDK versions; test upgrades.
- **Monitor** release notes and retirement announcements for each Preview feature you use ([file 01](01-tracking-azure-change.md)).
- **Tag resources** (for example `uses-preview=true`) and keep a register.

### Preview register (example)

| Feature | Status | Used in | Purpose | GA alternative | Exit plan | Risk owner | Review date |
|---|---|---|---|---|---|---|---|
| Agentic retrieval at the edge | Preview | Factory pilot | Offline RAG | Cloud RAG over VPN | Fall back to cloud | Ops director | Quarterly |
| Workspaces in Chaos Studio | Preview | Resilience tests | Experiment grouping | Individual experiments | Manual grouping | Platform lead | Quarterly |

### Communicating Preview to clients

- Mark the status in every proposal, design document and demo ("Preview, no SLA").
- Explain the business meaning: "This may change or be withdrawn; we've designed a fallback."
- Record risk acceptance in the decision log (ADR) with named approvers.
- Don't commit fixed-price delivery to Preview behaviour without contingency.
- Revisit status at each milestone; when a feature reaches GA, review the differences.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Adopt now vs wait for GA** | Adopt now when the value is high and the exit plan is cheap; wait when the feature is core or regulated |
| **Pilot scope** | Limit Preview use to pilots with defined success criteria and review dates |
| **Contractual position** | State Preview risks and assumptions in SoWs |
| **Portfolio consistency** | One organisation-wide policy beats case-by-case improvisation |

## Hands-on

1. Write a one-page Preview feature policy for your team or client.
2. Create a Preview register for a current project.
3. Run the evaluation checklist on one Preview feature you want to use.
4. Write an ADR that accepts a Preview risk with an exit plan.
5. Find a feature that moved from Preview to GA recently and list what changed.

## Security, cost and reliability notes

- Preview features may lack private networking, customer-managed keys or compliance coverage. Check before using them with sensitive data.
- Preview pricing can change at GA. Model cost scenarios.
- Without an SLA, design for failure: fallbacks, timeouts and circuit breakers.

## Common mistakes

| Mistake | Fix |
|---|---|
| Building production on Preview without risk acceptance | Formal risk acceptance and exit plan |
| Not labelling Preview in proposals | Mark status everywhere |
| No tracking of Preview dependencies | Register and tags |
| Assuming GA will match Preview | Review differences at GA |

## Check yourself

1. Which terms govern Azure Preview features?
2. Name four ways Preview differs from GA.
3. When is Preview acceptable in production?
4. What engineering practices reduce Preview risk?
5. How should Preview be communicated in a proposal?

<details>
<summary>Answers</summary>

1. The Supplemental Terms of Use for Microsoft Azure Previews.
2. Any four: no SLA, limited support, possible breaking changes, possible withdrawal, limited regions, compliance coverage may be missing.
3. Rarely; only with explicit business and security risk acceptance, mitigations and an exit plan, preferably for non-critical use.
4. Abstraction, feature flags, pinned versions, monitoring announcements, tagging and a register.
5. Clearly labelled with status and implications, risk acceptance recorded, contingency in plans and pricing.

</details>

## References

- [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/)
- [Azure Updates](https://azure.microsoft.com/updates/)
- [What's new in Microsoft Foundry](https://learn.microsoft.com/azure/foundry/whats-new-foundry)
- [Architecture decision records](https://learn.microsoft.com/azure/well-architected/architect-role/architecture-decision-record)
