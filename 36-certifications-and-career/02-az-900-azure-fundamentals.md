# AZ-900 Azure Fundamentals

## Why it matters

AZ-900 (Microsoft Azure Fundamentals) proves you understand cloud concepts, core Azure services and how Azure is managed and governed. It doesn't expire, it needs no prerequisites, and it gives everyone on a project a shared vocabulary. That matters most for consultants, project managers, pre-sales staff and engineers who come from on-premises or other clouds. For the AI tracks it's a gentle first exam before AI-901 or AZ-104.

## Key concepts

### Skills measured (study guide, as of 20 July 2026)

| Domain | Weight | Topics |
|---|---|---|
| **Describe cloud concepts** | 25–30% | Cloud computing, shared responsibility, public/private/hybrid, consumption model, benefits (HA, scalability, reliability, predictability, security, governance, manageability), IaaS/PaaS/SaaS |
| **Describe Azure architecture and services** | 35–40% | Regions, region pairs, availability zones, sovereign regions, resources, resource groups, subscriptions, management groups; compute and networking; storage; identity, access and security |
| **Describe Azure management and governance** | 30–35% | Cost management, governance and compliance tools, tools for managing and deploying resources, monitoring tools |

### Roadmap mapping

| Exam domain | Roadmap topics |
|---|---|
| Cloud concepts | [01 · Cloud and Azure Fundamentals](../01-cloud-and-azure-fundamentals/README.md) |
| Architecture and services | [01](../01-cloud-and-azure-fundamentals/README.md), [02 · Identity and Access](../02-identity-and-access/README.md), and overviews from [05 Networking](../05-networking/README.md), [06 Compute](../06-compute/README.md), [08 Storage](../08-storage/README.md) |
| Management and governance | [03 · Governance and Landing Zones](../03-governance-and-landing-zones/README.md), [29 · Cost Management](../29-cost-management-and-finops/README.md), [26 · Monitoring](../26-monitoring-and-observability/README.md) (overview level) |

### Concepts candidates often confuse

| Pair | Difference |
|---|---|
| Availability zone vs region pair | Zones are separate datacentres within one region; region pairs are two regions in the same geography used for some platform replication and sequenced updates |
| Azure Policy vs RBAC | Policy controls *what* can be deployed and how it's configured; RBAC controls *who* can act |
| Resource locks vs Policy | Locks prevent deletion or modification of specific resources; Policy evaluates configuration rules |
| Microsoft Entra ID vs Active Directory Domain Services | Cloud identity provider vs on-premises directory |
| Azure Advisor vs Service Health vs Monitor | Recommendations vs Azure platform incidents and maintenance vs telemetry |
| Pricing calculator vs TCO calculator | Estimate Azure costs vs compare on-premises with Azure |

## How it works on Azure

### Four-week study plan (about 4–6 hours a week)

| Week | Focus | Activities |
|---|---|---|
| 1 | Cloud concepts | Topic 01 files; Microsoft Learn path *Describe cloud concepts* |
| 2 | Architecture and core services | Create a free account; deploy a VM, storage account and VNet in one resource group; delete the group |
| 3 | Identity, security, governance | Create Entra users and groups; assign RBAC at resource-group scope; assign a built-in policy (allowed locations); add a delete lock |
| 4 | Cost, tools and revision | Pricing calculator estimate; Cost Management budget; Advisor recommendations; practice assessment until you score consistently above 80% |

### Lab you can do in an hour

```bash
az group create -n rg-az900-lab -l uksouth
az storage account create -n staz900lab$RANDOM -g rg-az900-lab --sku Standard_LRS
az lock create -n no-delete -g rg-az900-lab --lock-type CanNotDelete
az group delete -n rg-az900-lab   # fails because of the lock
az lock delete -n no-delete -g rg-az900-lab
az group delete -n rg-az900-lab --yes
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Take it or skip it?** | Skip it if you're heading straight to AZ-104 and already work in Azure. Take it if you're new to cloud, or if you're in a consulting or pre-sales role. |
| **Self-paced vs instructor-led** | Self-paced Learn paths are enough for most people; Microsoft Virtual Training Days sometimes include an exam offer. |
| **Depth** | Know what each service is for, not how to configure every setting. |

## Hands-on

1. Complete the Microsoft Learn AZ-900 learning paths.
2. Run the one-hour lab above.
3. Build a pricing-calculator estimate for a three-tier web app.
4. Explain shared responsibility for IaaS, PaaS and SaaS to a colleague.
5. Take the practice assessment and review every wrong answer against the docs.

## Security, cost and reliability notes

- Use a free account or sandbox, set a budget alert and delete resources after labs.
- Remember that a lock doesn't stop data-plane operations such as writing blobs.
- Understand the SLA basics: composite SLAs and how zones improve availability.

## Common mistakes

| Mistake | Fix |
|---|---|
| Memorising service lists without understanding | Practise one scenario per service |
| Confusing governance tools | Use the comparison table above |
| Ignoring the cost domain | Build estimates and budgets yourself |
| Over-studying configuration detail | Stay at the "describe" level |

## Check yourself

1. What are the three domains of AZ-900 and their weights?
2. How does an availability zone differ from a region pair?
3. When would you use Azure Policy rather than RBAC?
4. What does the TCO calculator do?
5. Does AZ-900 expire?

<details>
<summary>Answers</summary>

1. Cloud concepts (25–30%), Azure architecture and services (35–40%), Azure management and governance (30–35%).
2. Zones are physically separate datacentres in one region; region pairs are two regions in one geography that are paired for some replication and update sequencing.
3. When you want to enforce or audit resource configuration (for example, allowed locations) regardless of who deploys.
4. It compares the total cost of running workloads on-premises against running them on Azure.
5. No; Fundamentals certifications don't expire.

</details>

## References

- [Exam AZ-900](https://learn.microsoft.com/credentials/certifications/exams/az-900/)
- [AZ-900 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/az-900)
- [Azure Fundamentals certification](https://learn.microsoft.com/credentials/certifications/azure-fundamentals/)
- [Pricing calculator](https://azure.microsoft.com/pricing/calculator/)
