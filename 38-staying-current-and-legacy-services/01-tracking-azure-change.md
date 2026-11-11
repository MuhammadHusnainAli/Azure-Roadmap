# Tracking Azure Change

## Why it matters

Azure ships hundreds of updates a year, and AI services change faster than anything else on the platform. Models retire on fixed schedules. Preview features become GA or disappear. Products are renamed. Exams change. An architecture or proposal that was correct six months ago can now recommend a retired service. Engineers, architects and consultants each need a **repeatable system** for tracking change, both to stay credible and to protect clients from surprise retirements. This file sets out that system.

## Key concepts

### Official sources

| Source | What it covers | How to use it |
|---|---|---|
| **Azure Updates** | Announcements: GA, Preview, retirements, region launches | Filter by product and update type; subscribe via RSS |
| **Microsoft Foundry "What's new"** | Foundry platform, Agent Service, tools, SDKs | Read monthly |
| **Model deprecations and retirements** (Foundry) | Model version retirement dates and upgrade policies | Check before every model choice and quarterly in production |
| **Microsoft Learn "What's new" pages** | Per-service release notes (AI Search, Entra ID, AKS, Defender and more) | Follow for the services you depend on |
| **Azure Advisor service retirement workbook** | Retirements that affect **your** resources | Review monthly per tenant |
| **Azure Service Health** | Health advisories and planned maintenance, including retirement notices for your subscriptions | Configure alerts to the right people |
| **Microsoft 365 Message Center and roadmap** | Copilot, Copilot Studio and Microsoft 365 changes | Admins review weekly |
| **Microsoft Learn credentials pages** | Exam updates, retirements, renewals | Check before booking |
| **Azure SDK release notes and GitHub repos** | Breaking changes in SDKs and samples | Watch the repos you use |
| **Microsoft Build and Microsoft Ignite** (Book of News, session recordings) | Major announcements (Build usually in spring, Ignite usually in autumn) | Read the Book of News; separate announcements from GA dates |
| **Azure Architecture Center** | Updated reference architectures and guides | Check "last updated" dates on architectures you reuse |
| **Microsoft Tech Community and product blogs** | Context and deep dives | Use for detail, verify on Learn |

### Change types and what they mean

| Change | Implication |
|---|---|
| **GA** | Supported with an SLA (where applicable); safe for production after your own testing |
| **Preview** (public or private) | No SLA, may change or be withdrawn, supplemental terms apply ([file 06](06-preview-feature-policy.md)) |
| **Deprecated** | No new deployments or features; plan migration |
| **Retiring** | Fixed date after which the service or feature stops working |
| **Renamed** | Same capability, new name; update documents and code references ([file 02](02-product-rename-map.md)) |
| **Breaking API or SDK change** | New API versions; old versions eventually retired |

## How it works on Azure

### A personal tracking routine

| Cadence | Activity | Time |
|---|---|---|
| Weekly | Skim Azure Updates filtered to your products; Foundry What's new | 30 min |
| Monthly | Model retirements page; release notes for the 5–10 services you depend on; Advisor retirement workbook | 1 h |
| Quarterly | Review your reference architectures and templates against changes; update team wiki | Half a day |
| After Build and Ignite | Read the Book of News; list changes relevant to clients; plan proofs of concept for Preview features | Half a day |

### An organisational routine (for consultants and platform teams)

1. **Inventory** what clients and workloads use (Resource Graph queries and dependency lists).
2. **Alert**: route Service Health retirement notices to platform owners; use Azure Monitor alerts on Service Health.
3. **Triage** monthly: affected resources, date, migration target, effort, owner.
4. **Plan** migrations in the backlog with deadlines at least three months before retirement dates.
5. **Communicate** to clients proactively, as a value-added service.

```kusto
// Resource Graph: list resource types in use, to compare against retirement notices
resources
| summarize count() by type, location
| order by count_ desc
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Breadth vs focus** | Track deeply what you use; skim the rest |
| **Early adoption** | Adopt Preview features in non-critical paths with exit plans |
| **Automation** | Automate retirement detection (Advisor, Service Health alerts) rather than relying on reading |
| **Single source of truth** | Keep a team "platform changes" page with links to official announcements |

## Hands-on

1. Subscribe to Azure Updates RSS filtered to five products you use.
2. Open the Advisor service retirement workbook for a subscription and list affected resources.
3. Create a Service Health alert for health advisories in your subscriptions.
4. Check the model retirements page for every model deployment you own.
5. Write a one-page "what changed this quarter" note for your team.

## Security, cost and reliability notes

- Retired services stop receiving security fixes: unplanned retirements are security risks.
- Model retirements can change behaviour and cost. Evaluate replacement models before auto-upgrade dates.
- Service Health alerts are free. Configure them in every subscription.

## Common mistakes

| Mistake | Fix |
|---|---|
| Relying on blog posts and social media | Verify on Microsoft Learn and Azure Updates |
| Treating Build or Ignite announcements as GA | Check status and dates |
| Discovering retirements when things break | Advisor workbook and Service Health alerts |
| Not checking model retirement dates | Monthly review and auto-upgrade policy decisions |

## Check yourself

1. Which tool shows retirements affecting your own resources?
2. Where do you check model retirement dates?
3. How do Preview and GA differ in support?
4. What's a sensible personal tracking cadence?
5. How should consultants turn change tracking into client value?

<details>
<summary>Answers</summary>

1. The Azure Advisor service retirement workbook (and Service Health advisories).
2. The Foundry model deprecations and retirements page.
3. Preview has no SLA and may change or be withdrawn; GA is fully supported.
4. Weekly skim, monthly deeper review, quarterly architecture refresh, plus after Build and Ignite.
5. Proactive retirement reviews, migration planning and briefings on relevant new capabilities.

</details>

## References

- [Azure Updates](https://azure.microsoft.com/updates/)
- [What's new in Microsoft Foundry](https://learn.microsoft.com/azure/foundry/whats-new-foundry)
- [Model deprecations and retirements](https://learn.microsoft.com/azure/foundry/concepts/model-lifecycle-retirement)
- [Service retirement workbook in Azure Advisor](https://learn.microsoft.com/azure/advisor/advisor-workbook-service-retirement)
- [Azure Service Health overview](https://learn.microsoft.com/azure/service-health/overview)
