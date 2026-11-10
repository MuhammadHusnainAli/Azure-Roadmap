# Service Lifecycle, SLAs and Quotas

## Why it matters

Three questions decide whether a design can actually ship: *Is this feature supported for production? What availability can we promise? Can we get the capacity?* In AI projects all three bite harder than usual. New capabilities arrive in preview, model versions retire on a schedule, and model quota is often the real constraint on launch dates.

## Key concepts

### Service lifecycle

| Stage | Meaning | SLA | Support | Use in production? |
|---|---|---|---|---|
| **Private preview** | Invitation only, often under NDA | None | Limited, via product team | No |
| **Public preview** | Open to everyone to evaluate | None | Usually best effort | Only with accepted risk |
| **General availability (GA)** | Fully released | Yes (if the service has one) | Full | Yes |
| **Deprecated** | Still works; no new investment; retirement announced | Until retirement | Until retirement | Plan migration |
| **Retired** | Switched off | – | – | No |

Preview features are governed by the *Supplemental Terms of Use for Microsoft Azure Previews*. In practice that means: no SLA, possible breaking changes, possible withdrawal, regional limits, and sometimes no charge now with pricing introduced at GA.

Two subtleties:

- **GA is per feature, per region, sometimes per SKU.** A service can be GA while a feature inside it is preview, or GA in one region and not yet in another.
- **Preview API versions.** A feature reachable only through a `-preview` API version is a preview feature, whatever the portal label says.

### Retirement policy

For GA services Microsoft generally gives **at least 12 months' notice** before retirement, with shorter notice for some feature-level and security-driven changes. Notices appear in:

- **Azure Service Health → Health advisories** (targeted at subscriptions that use the service).
- **Azure Updates** (filter by *Retirements*).
- **Azure Advisor**, including the service retirement workbook, which lists your affected resources.
- Email to subscription owners and administrators.

### Model lifecycle is faster

AI models do not follow the 12-month pattern. Each model version in Foundry has published **deprecation** and **retirement** dates, and the cycle is short, often 12 to 18 months from release for a GA model and much less for preview models.

| Term | Meaning |
|---|---|
| **Deprecation** | No new deployments of that version for new customers; existing deployments continue |
| **Retirement** | Deployments stop working and return errors |
| **Auto-upgrade** | A deployment setting that moves you to the new default version automatically |

Design consequences:

- Treat the model as a **replaceable dependency**. Keep prompts, evaluations and configuration versioned so you can re-test on a new model quickly.
- Decide the version upgrade policy per deployment: auto-upgrade for convenience, or pinned with a scheduled re-evaluation for control.
- Put model retirement dates in the operations calendar on day one.

Topic 15 covers model lifecycle in full.

### How to track change

| Source | Use it for |
|---|---|
| [Azure Updates](https://azure.microsoft.com/updates/) | Launches, previews, GA, retirements; filterable |
| Service Health in the portal | Incidents, planned maintenance and advisories affecting **your** resources |
| Azure Advisor | Recommendations including upcoming retirements |
| Per-service "What's new" pages on Microsoft Learn | Detail for a specific service (Foundry, AKS, Azure AI Search) |
| Microsoft Build (May) and Ignite (November) | The two big announcement waves each year |

### Service level agreements

An SLA is Microsoft's financially backed commitment for a service, usually expressed as monthly uptime or connectivity.

What an SLA is and is not:

- It is a **commitment with a remedy**, not a prediction. The remedy is a **service credit** (a percentage off that service's bill), which you must **claim**. It never covers your business losses.
- It applies only when you meet the **conditions**. For example, the higher VM SLA requires instances across availability zones.
- It covers the **specific scope defined**, often "connectivity to" or "successful requests to" the service, not that your application works.
- **Free tiers and previews generally have no SLA.**

Downtime allowed per uptime percentage:

| Uptime | Per month (30 days) | Per year |
|---|---|---|
| 99% | 7 h 18 min | 3 d 15 h |
| 99.5% | 3 h 39 min | 1 d 19 h |
| 99.9% | 43 min 49 s | 8 h 46 min |
| 99.95% | 21 min 54 s | 4 h 23 min |
| 99.99% | 4 min 23 s | 52 min 35 s |
| 99.999% | 26 s | 5 min 15 s |

Illustrative SLA patterns (always confirm in the current SLA document, because values change by tier and configuration):

| Configuration | Typical commitment |
|---|---|
| Single VM with premium SSD disks | 99.9% |
| Two or more VMs in an availability set | 99.95% |
| Two or more VMs across availability zones | 99.99% |
| Zone-redundant PaaS tiers | Often 99.99% or higher |
| Many standard PaaS tiers | 99.9% to 99.95% |

### Composite SLA

Your application depends on several services. Its expected availability is a combination of theirs.

**Serial dependencies** (all must work): multiply.

```text
App Service 99.95% × SQL Database 99.99% × Foundry model 99.9%
= 0.9995 × 0.9999 × 0.999
= 0.9984  → 99.84%   (about 70 minutes of downtime a month)
```

Each added dependency lowers the figure. A chain is always weaker than its weakest link.

**Redundant components** (any one is enough): multiply the failure probabilities.

```text
Availability = 1 − (1 − A)ⁿ

Two regions at 99.9% each, independent:
1 − (0.001 × 0.001) = 0.999999  → 99.9999%
```

Then multiply by the availability of whatever routes between them:

```text
Two-region app 99.9999% × global load balancer 99.99% ≈ 99.99%
```

Caveats that matter in real designs:

- Redundancy maths assumes **independent failures**. Shared dependencies (identity, DNS, a single database, a bad deployment pushed to both regions) break that assumption.
- Composite SLA is a **design estimate**, not a contractual figure. Microsoft does not offer an SLA for your composed system.
- Failover takes time. A design with 99.99% on paper and a 30-minute manual failover does not deliver 99.99%.

### SLA, SLO, SLI, RTO, RPO

| Term | Definition | Who sets it |
|---|---|---|
| **SLI** (indicator) | A measured metric: success rate, latency | Engineering |
| **SLO** (objective) | Your internal target for an SLI | You |
| **SLA** (agreement) | A commitment with consequences | Provider to you; you to your customers |
| **RTO** (recovery time objective) | Maximum acceptable time to restore service | Business |
| **RPO** (recovery point objective) | Maximum acceptable data loss, measured in time | Business |

Set your SLO first from business need, then check that the composite SLA of your design can plausibly support it. Topic 28 goes further.

### Quotas and limits

| Term | Meaning | Adjustable? |
|---|---|---|
| **Quota** | A cap on how much of a resource a subscription can consume, usually per region | Usually, by request |
| **Limit** | A fixed constraint of the service | No |

Most quotas are **per subscription, per region**. Common ones:

| Area | Typical quota |
|---|---|
| Compute | vCPUs per region in total, and per VM family; separate spot quota |
| GPU | Per-family vCPU quota, often **zero by default** |
| Networking | Public IPs, VNets, load balancers per region |
| Storage | Storage accounts per region per subscription |
| Foundry Models | Tokens per minute (TPM) per model, per deployment type, per region; provisioned throughput units (PTU) per region |
| Azure AI Search | Services per subscription by tier; indexes and replicas per service by tier |
| Container Apps / App Service | Environments, cores, plans per region |

Request increases from **Quotas** in the portal (or `az quota`), or through a support request. Many are approved automatically; GPU and model quota can take longer or be declined when the region is constrained.

### Quota is not capacity

A quota is permission to ask. **Capacity** is whether hardware is physically available when you ask. You can have quota and still receive an allocation failure (`AllocationFailed`, `SkuNotAvailable`, `ZonalAllocationFailed`) in a busy region.

Ways to get an actual guarantee:

| Mechanism | Guarantees |
|---|---|
| **On-demand capacity reservation** | VM capacity for a size in a region or zone, billed whether used or not |
| **Provisioned throughput (PTU)** | Dedicated model processing capacity with predictable latency |
| **Reserved instances** | A **price discount**. Capacity priority is available in some forms but a reservation alone is primarily financial |

### AI quota in practice

Foundry model quota is assigned per subscription, per region, per model, per deployment type.

- **Standard (pay-as-you-go) deployments:** quota in **tokens per minute**. You divide a model's regional TPM quota among your deployments. A requests-per-minute limit is derived from it. Exceed either and you get **HTTP 429**.
- **Global and Data Zone deployments** have higher default quotas than Regional.
- **Provisioned deployments:** quota in **PTUs**. Having PTU quota does not guarantee capacity exists in the region; check availability before committing to a reservation.
- **New and high-demand models** may require an access request and often have low initial quota.

Planning implications:

1. Estimate peak tokens per minute early. It is the capacity number that matters.
2. Check quota in the target region **before** finalising the design.
3. Request increases well before load testing and go-live.
4. For higher ceilings, spread across deployment types, regions or subscriptions behind a gateway (topic 11 and topic 34).
5. Build clients to handle 429 with backoff, and consider spillover from provisioned to standard.

## How it works on Azure

### Preview adoption checklist

Before using a preview feature in a client solution:

- Is there a GA alternative that meets the need?
- What happens to the solution if the feature changes or is withdrawn?
- Is it available in the target region and compliant with the client's data requirements?
- Is there a stated GA timeline, and does it precede go-live?
- Has the client accepted the risk in writing?
- Is the dependency isolated behind an interface so it can be swapped?

In fast-moving areas such as agents, some preview use is unavoidable. The discipline is in making it a recorded decision.

## Design decisions and trade-offs

- **Each extra nine costs disproportionately more.** Going from 99.9% to 99.99% usually means zone redundancy; to 99.999% means multi-region active-active plus operational maturity. Ask what an hour of downtime actually costs the business.
- **Fewer hard dependencies, higher availability.** Make dependencies soft where you can: cache, queue, degrade gracefully. An AI feature that fails should not take the whole application down.
- **Quota shapes subscription design.** Separate subscriptions give separate quotas and stop one team exhausting another's.
- **Pay-as-you-go vs provisioned for AI.** Standard deployments are flexible but shared and rate-limited. Provisioned gives predictable throughput and latency at a committed cost.
- **Pin or auto-upgrade models.** Regulated or carefully evaluated workloads should pin and upgrade deliberately. Low-risk workloads can auto-upgrade.

## Hands-on

```bash
LOC=swedencentral

# Compute quota and usage
az vm list-usage --location $LOC \
  --query "[?currentValue > \`0\` || contains(name.value,'cores')].{Name:name.localizedValue, Used:currentValue, Limit:limit}" -o table

# GPU family quota (often 0)
az vm list-usage --location $LOC \
  --query "[?contains(name.value,'NC') || contains(name.value,'ND') || contains(name.value,'NV')].{Family:name.localizedValue, Used:currentValue, Limit:limit}" -o table

# Network quota
az network list-usages --location $LOC \
  --query "[?currentValue > \`0\`].{Name:name.localizedValue, Used:currentValue, Limit:limit}" -o table

# Foundry / Azure OpenAI model quota for the region
az cognitiveservices usage list --location $LOC \
  --query "[?limit > \`0\`].{Quota:name.value, Used:currentValue, Limit:limit}" -o table

# Is a VM size restricted in this region for my subscription?
az vm list-skus --location $LOC --size Standard_NC --all \
  --query "[].{Size:name, Restrictions:restrictions[0].reasonCode}" -o table
```

Portal:

1. Open **Quotas** and review Compute and Machine Learning / Foundry quotas. Start (but do not submit) an increase request to see the form.
2. Open **Service Health → Health advisories** and look for retirement notices.
3. Open **Advisor → Workbooks → Service Retirement** and see which of your resources are affected.

**Calculation exercise.** A solution has Front Door (99.99%), App Service across zones (99.95%), Azure AI Search with three replicas (99.9%), a Foundry model deployment (99.9%), and Azure SQL Database zone-redundant (99.995%). All are required.

1. What is the composite availability?
2. The same stack is deployed to a second region with independent data, behind the same Front Door. What is it now?

<details>
<summary>Answers</summary>

1. Regional stack: 0.9995 × 0.999 × 0.999 × 0.99995 ≈ 0.99745. With Front Door: × 0.9999 ≈ 0.99735, about **99.74%** (roughly 1 h 55 min a month).
2. Two regions: 1 − (1 − 0.99745)² ≈ 0.9999935. With Front Door: × 0.9999 ≈ **99.99%**. Front Door is now the limiting factor, and the result assumes the two regions fail independently.

</details>

## Security, cost and reliability notes

- **Security:** retired services and API versions stop receiving fixes. Tracking retirements is a security task as much as an operational one.
- **Cost:** quota is free; capacity guarantees are not. Capacity reservations and PTUs bill whether or not you use them.
- **Cost:** preview features are often free or discounted, then priced at GA. Model the GA price.
- **Reliability:** configure Service Health alerts on every production subscription so incidents and retirements reach a team, not one inbox.

## Common mistakes

- Putting a preview feature on the critical path without a recorded risk decision.
- Quoting a single service's SLA as the application's availability.
- Forgetting the conditions attached to an SLA (zones, replica counts, tier).
- Treating an SLA as insurance. A service credit is a small discount, not compensation.
- Assuming quota means capacity.
- Finding out about model quota during load testing.
- Ignoring model retirement dates until deployments start returning errors.
- Using reservations expecting a capacity guarantee.

## Check yourself

1. What changes between public preview and GA?
2. How much notice does Microsoft typically give before retiring a GA service, and where do you see the notice?
3. Why is model lifecycle handled differently from service lifecycle?
4. Compute the composite SLA of three serial services at 99.9%, 99.95% and 99.99%.
5. What assumption do redundancy calculations rely on, and what commonly breaks it?
6. Distinguish SLA, SLO and SLI.
7. Explain the difference between quota and capacity.
8. In what unit is standard Foundry model quota measured, and what error signals you have exceeded it?
9. How would you guarantee GPU VM capacity for a planned launch?

<details>
<summary>Answers</summary>

1. GA brings an SLA (where offered), full support, production terms and stability commitments. Preview has none of these and may change or be withdrawn.
2. Generally at least 12 months, communicated through Service Health advisories, Azure Updates, Advisor and email.
3. Model versions have short, published deprecation and retirement schedules because models are superseded quickly, so applications must be built to swap versions.
4. 0.999 × 0.9995 × 0.9999 ≈ 0.9984, or 99.84%.
5. That failures are independent. Shared dependencies and correlated causes such as a bad deployment or shared identity and DNS break it.
6. An SLI is a measured metric, an SLO is your internal target for it, and an SLA is a commitment with consequences.
7. Quota is the permitted maximum for your subscription. Capacity is whether the hardware is available at the moment you deploy.
8. Tokens per minute. HTTP 429.
9. Request the quota increase, then create an on-demand capacity reservation for the size and region or zone.

</details>

## References

- [Service Level Agreements for Microsoft Online Services](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services)
- [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/)
- [Azure Updates](https://azure.microsoft.com/updates/)
- [Azure subscription and service limits, quotas, and constraints](https://learn.microsoft.com/azure/azure-resource-manager/management/azure-subscription-service-limits)
- [Azure Quotas overview](https://learn.microsoft.com/azure/quotas/quotas-overview)
- [On-demand capacity reservation](https://learn.microsoft.com/azure/virtual-machines/capacity-reservation-overview)
- [Foundry Models quotas and limits](https://learn.microsoft.com/azure/foundry/openai/quotas-limits)
- [Model deprecations and retirements](https://learn.microsoft.com/azure/foundry/openai/concepts/model-retirements)
- [Service retirement workbook in Azure Advisor](https://learn.microsoft.com/azure/advisor/advisor-workbook-service-retirement)
- [Reliability metrics (Well-Architected Framework)](https://learn.microsoft.com/azure/well-architected/reliability/metrics)
