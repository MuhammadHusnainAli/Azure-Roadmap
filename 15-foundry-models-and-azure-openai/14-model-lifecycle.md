# Model Lifecycle

## Why it matters

Models don't live forever. Every model version on Foundry has a published lifecycle: it launches, becomes legacy, is deprecated for new customers, and is eventually retired, after which calls fail with **410 Gone**. Standard deployments may be **auto-upgraded** to a newer version that behaves differently; provisioned deployments are not. Teams that ignore this get outages, silent quality changes or rushed migrations. Lifecycle planning belongs in every AI operating model.

## Key concepts

### Lifecycle stages (Microsoft policy, checked 2026-11)

| Stage | Meaning | New deployments? | Existing deployments work? |
|---|---|---|---|
| **Preview** | Experimental; weights, runtime and API may change; may never reach GA | Yes | Yes |
| **Generally available (GA)** | Production-ready; weights and APIs fixed | Yes | Yes |
| **Legacy** (optional) | Newer models exist; plan migration | Yes, until deprecation | Yes |
| **Deprecated** | Blocked for **new customers**; subscriptions that have ever deployed that model version can still create and manage deployments | Existing customers only | Yes |
| **Retired** | Removed; inference returns **410 Gone** | No | No |

"Existing customer" is evaluated per **subscription**: a new subscription in the same tenant doesn't inherit access to a deprecated model.

### GA timeline (standard 18-month lifecycle)

| Milestone | Timing |
|---|---|
| GA launch | Retirement date set programmatically to **18 months** after launch (visible through the Models API) |
| Deprecated | At **12 months**: existing customers only |
| Replacement in Global Standard | About **90 days** before retirement (replacement chosen about 90–120 days before) |
| Replacement in provisioned regions | About **30 days** before retirement |
| Retired | At **18 months**: 410 Gone |

Variations:

| Provider or case | Lifecycle |
|---|---|
| Anthropic, DeepSeek, Fireworks, Mistral AI GA models | **12 months** |
| xAI Grok GA models from Grok 4.7 | Minimum **six months** |
| Preview models | "Not sooner than" retirement typically 90 days out; force-upgraded to a newer preview or GA (at least **30 days** notice) or retired |
| Embedding models | Extended timelines; check the schedule |
| Azure Government | Typically one version at a time with a 30-day overlap; no Global Standard |
| Security or compliance issues | Emergency retirement with shortened notice is possible |
| Retirement extensions | **Not available**: dates can't be extended |

### Versions, auto-upgrade and `versionUpgradeOption`

For **Global Standard, Data Zone Standard and Standard** (and Priority processing) deployments, set the deployment's `versionUpgradeOption`:

| Value | Behaviour |
|---|---|
| `OnceNewDefaultVersionAvailable` | Upgrade automatically when Microsoft sets a new default version |
| `OnceCurrentVersionExpired` | Stay pinned until your version retires, then upgrade automatically |
| `NoAutoUpgrade` | Never auto-upgrade; the deployment **stops working** at retirement |

At retirement, Microsoft auto-upgrades eligible standard deployments to the declared replacement on a **rolling, region-by-region** schedule (published in the retirement schedule). Example from Microsoft's documentation: gpt-4o version `2024-05-13` retires on **14 January 2027**, with standard deployments auto-upgraded to gpt-5.1 (`2025-11-13`).

**Provisioned deployments are not auto-upgraded.** Migrate them yourself, either **in place** (Azure migrates traffic over a 20–30 minute window with no downtime) or **side by side** (create a new deployment, test, switch traffic, delete the old one). Batch deployments use side-by-side migration.

### Notifications and programmatic checks

| Channel | What you get |
|---|---|
| Email to subscription owners with active deployments | GA retirement notice at least **60 days** ahead; preview at least **30 days** |
| **Azure Service Health** advisories | Filter service "Azure OpenAI Service" and create alert rules |
| **Models API** | `lifecycleStatus`, `deprecation.inference`, `deprecation.fineTune`, per-SKU `deprecationDate` |
| Retirement schedule page | Dates and replacement models |

The API uses different terms from the portal:

| Portal / docs | API `lifecycleStatus` |
|---|---|
| Preview | `Preview` |
| Generally available | `GenerallyAvailable` |
| Deprecated (existing customers only) | `Deprecating` |
| Retired | `Deprecated` |

### Fine-tuned models

Fine-tuned models retire in two phases: **training** retirement (no new fine-tuning) and **deployment** retirement (inference stops). Training usually retires no earlier than the base model; already-trained models stay deployable until deployment retirement. Check the fine-tuned schedule table for each base model.

## How it works on Azure

### Check lifecycle status for models in a region

```bash
az rest --method get \
  --url "https://management.azure.com/subscriptions/<sub>/providers/Microsoft.CognitiveServices/locations/swedencentral/models?api-version=2024-10-01" \
  --query "value[?model.name=='gpt-4o'].{version:model.version, status:model.lifecycleStatus, retire:model.deprecation.inference}" -o table
```

### Pin or control upgrades (Bicep)

```bicep
resource chat 'Microsoft.CognitiveServices/accounts/deployments@2025-06-01' = {
  parent: foundry
  name: 'chat-prod'
  sku: { name: 'GlobalStandard', capacity: 200 }
  properties: {
    model: { format: 'OpenAI', name: 'gpt-5-mini', version: '2025-08-07' }
    versionUpgradeOption: 'OnceCurrentVersionExpired'   // stay pinned until retirement
  }
}
```

### Lifecycle runbook

```text
Quarterly:
  - Export all deployments (model, version, type, upgrade option) across subscriptions (Resource Graph / Models API)
  - Flag versions within 6 months of retirement; record replacement candidates
Per migration:
  1. Shortlist replacement models (don't wait for the official replacement)
  2. Run the evaluation suite (quality, safety, latency, cost) on candidates vs current
  3. Adapt prompts, schemas and parameters (reasoning models reject some sampling params)
  4. Deploy side by side; canary a percentage of traffic via the gateway
  5. Switch; monitor; delete the old deployment
  6. For PTU: confirm quota and capacity for the target model before migrating
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Auto-upgrade policy | `OnceCurrentVersionExpired` for production apps with evaluation gates (predictable, upgrade at retirement); `OnceNewDefaultVersionAvailable` for low-risk internal tools; `NoAutoUpgrade` only with a strong migration process (or you'll get an outage) |
| Preview models | Avoid in production; they can be force-upgraded with 30 days' notice |
| Partner models | Shorter lifecycles mean more frequent migrations; budget evaluation effort accordingly |
| Abstraction | Keep model and version in configuration, prompts versioned, and evaluations automated so migrations are routine |

## Hands-on

1. Query the Models API for three models you use and note lifecycle status and retirement dates.
2. Set up an Azure Service Health alert for "Azure OpenAI Service" advisories.
3. Change a test deployment's `versionUpgradeOption` with Bicep and verify the property.
4. Run your evaluation set against your current model version and a newer one; document the differences.
5. Write a migration plan for one provisioned deployment (in-place vs side-by-side).

## Security, cost and reliability notes

- **Reliability:** a retired model returns 410. Track retirements as operational risks with owners and dates.
- **Reliability:** auto-upgrades change behaviour (format, tone, refusals, tool use). Re-run evaluations after upgrades and monitor quality metrics.
- **Cost:** newer models can have different pricing (for example long-context pricing, cache-write charges); update cost models.
- **Compliance:** record model versions used for decisions in regulated processes; upgrades may need change approval.

## Common mistakes

- `NoAutoUpgrade` with no migration plan, followed by a production outage at retirement.
- Waiting for Microsoft's replacement announcement (90–120 days before retirement) to start evaluating.
- Assuming provisioned deployments upgrade automatically.
- Building production on preview models.
- Misreading the API: `Deprecated` in the API means **retired**.

## Check yourself

1. What are the five lifecycle stages?
2. How long is the standard GA lifecycle, and when are models deprecated?
3. What are the three `versionUpgradeOption` values?
4. Which deployment types are auto-upgraded at retirement, and which aren't?
5. What does an API `lifecycleStatus` of `Deprecating` mean?
6. How much notice do you get for GA and preview retirements?
7. Which providers have 12-month lifecycles?

<details>
<summary>Answers</summary>

1. Preview, generally available, legacy (optional), deprecated, retired.
2. 18 months from launch; deprecated (existing customers only) at 12 months.
3. `OnceNewDefaultVersionAvailable`, `OnceCurrentVersionExpired`, `NoAutoUpgrade`.
4. Global Standard, Data Zone Standard, Standard (and Priority processing) are auto-upgraded; provisioned deployments aren't, and batch uses side-by-side migration.
5. The model is deprecated in portal terms: still serving existing customers but blocked for new customers.
6. At least 60 days for GA models and at least 30 days for preview models (plus the published schedule).
7. Anthropic, DeepSeek, Fireworks and Mistral AI.

</details>

## References

- [Foundry Models lifecycle and support policy](https://learn.microsoft.com/azure/foundry/openai/concepts/model-retirements)
- [Model retirement schedule](https://learn.microsoft.com/azure/foundry/openai/concepts/model-retirement-schedule)
- [Working with models: upgrade configuration](https://learn.microsoft.com/azure/foundry/openai/how-to/working-with-models)
- [Model migration process](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/model-migration)
- [Models API reference](https://learn.microsoft.com/rest/api/aiservices/accountmanagement/models)
- [Service Health alerts](https://learn.microsoft.com/azure/service-health/alerts-activity-log-service-notifications-portal)
