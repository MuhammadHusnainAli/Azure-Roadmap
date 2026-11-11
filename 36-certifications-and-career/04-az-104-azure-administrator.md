# AZ-104 Azure Administrator

## Why it matters

AZ-104 (Microsoft Azure Administrator) is the most widely recognised hands-on Azure certification. It proves you can implement, manage and monitor identity, governance, storage, compute, networking and backup. It's also the **prerequisite for the Azure Solutions Architect Expert** (AZ-305) and one of the prerequisites accepted for the DevOps Engineer Expert (AZ-400). For AI engineers and architects it closes a common gap: AI workloads fail in production more often because of networking, identity and platform mistakes than because of model choice.

## Key concepts

### Skills measured (study guide)

| Domain | Weight | Sub-areas |
|---|---|---|
| **Manage Azure identities and governance** | 20–25% | Entra users and groups; access to Azure resources (RBAC); subscriptions and governance (Policy, locks, tags, management groups, cost) |
| **Implement and manage storage** | 15–20% | Storage access (SAS, keys, identity-based access, firewalls); storage accounts (redundancy, object replication, encryption); Azure Files and Blob (tiers, lifecycle, soft delete, versioning) |
| **Deploy and manage Azure compute resources** | 20–25% | ARM templates and Bicep; VMs (sizes, disks, availability sets and zones, scale sets); containers (ACR, Container Instances, Container Apps); App Service |
| **Implement and manage virtual networking** | 15–20% | VNets, subnets, peering, routes; NSGs, ASGs, Bastion, service and private endpoints; DNS and load balancing |
| **Monitor and maintain Azure resources** | 10–15% | Azure Monitor metrics, logs, alerts, Network Watcher; Backup and Site Recovery |

The skills-measured date on the AZ-104 study guide is older than on most Azure exams. Check the exam page for any announced update before booking.

### Roadmap mapping

| Exam domain | Roadmap topics |
|---|---|
| Identities and governance | [02 · Identity and Access](../02-identity-and-access/README.md), [03 · Governance and Landing Zones](../03-governance-and-landing-zones/README.md) |
| ARM and Bicep | [04 · Tooling and Infrastructure as Code](../04-tooling-and-infrastructure-as-code/README.md) |
| Networking | [05 · Networking](../05-networking/README.md) |
| Compute | [06 · Compute](../06-compute/README.md), [07 · Containers and Kubernetes](../07-containers-and-kubernetes/README.md) (ACR, ACI, Container Apps) |
| Storage | [08 · Storage](../08-storage/README.md) |
| App Service | [10 · Application Platform](../10-application-platform/README.md) |
| Monitor, backup and recovery | [26 · Monitoring and Observability](../26-monitoring-and-observability/README.md), [28 · Resiliency, Backup and DR](../28-resiliency-backup-and-disaster-recovery/README.md) |

## How it works on Azure

### Eight-week study plan (about 6–8 hours a week)

| Week | Focus | Lab |
|---|---|---|
| 1 | Entra ID, RBAC, custom roles | Create users, groups and a custom role; test scope inheritance |
| 2 | Governance | Management groups, Policy initiatives, tags, locks, budgets |
| 3 | Storage | Storage account with private endpoint, SAS vs identity access, lifecycle rules, file share with snapshots |
| 4 | IaC and VMs | Deploy a VM with Bicep; resize; add data disks; scale set with autoscale |
| 5 | Containers and App Service | Push to ACR; run on ACI and Container Apps; App Service with slots and scaling |
| 6 | Networking I | Hub-spoke peering, UDRs, NSG and ASG rules, Bastion |
| 7 | Networking II and DNS | Private DNS zones, private endpoints, Load Balancer vs Application Gateway, Network Watcher diagnostics |
| 8 | Monitor and backup; revision | Alerts and action groups, Log Analytics queries, VM backup and restore, Site Recovery test failover; practice assessment |

### Exam technique

- Expect scenario questions that ask for the *minimum* change or the *least administrative effort*.
- Know the scope of each control: management group, subscription, resource group, resource.
- Learn the effective-rules tools: IP flow verify, effective routes and effective security rules.
- Learn the portal navigation as well as the CLI. Some questions describe portal blades.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **AZ-104 before AI-103?** | Architects should take AZ-104 early because AZ-305 requires it. AI engineers can take AI-103 first and AZ-104 later. |
| **Lab budget** | Labs that use Bastion, Application Gateway, Firewall and Site Recovery cost money. Deploy them, test them, then delete them the same day. |
| **CLI vs portal** | Practise both. The exam uses both perspectives. |

## Hands-on

1. Build a hub-spoke network with Bastion and a private endpoint to storage.
2. Assign a policy that denies public network access on storage accounts and test it.
3. Deploy a VM scale set with Bicep and configure autoscale.
4. Back up a VM, restore a single file, and run a Site Recovery test failover.
5. Write three KQL queries against VM and activity logs, and create an alert.

## Security, cost and reliability notes

- Practise least privilege: test that a Reader can't change resources and that a Contributor can't assign roles.
- Set subscription budgets with alerts before you start labs.
- Learn redundancy options (LRS, ZRS, GRS, GZRS) and when each meets an availability requirement.

## Common mistakes

| Mistake | Fix |
|---|---|
| Studying only theory | Hands-on labs every week |
| Confusing service endpoints and private endpoints | Build both and compare the traffic path and DNS |
| Forgetting scope inheritance | Test RBAC and Policy at different scopes |
| Leaving expensive lab resources running | Delete resource groups after each session |

## Check yourself

1. List the five AZ-104 domains.
2. Which Expert certifications accept AZ-104 as a prerequisite?
3. How do service endpoints and private endpoints differ?
4. Which tool shows the effective security rules on a NIC?
5. How do you test DR for a VM without affecting production?

<details>
<summary>Answers</summary>

1. Identities and governance; storage; compute; virtual networking; monitor and maintain.
2. Azure Solutions Architect Expert (AZ-305) and DevOps Engineer Expert (AZ-400).
3. Service endpoints extend the VNet identity to a service's public endpoint; private endpoints give the service a private IP inside your VNet.
4. Network Watcher effective security rules (or the NIC's effective security rules view).
5. Run a Site Recovery test failover into an isolated network.

</details>

## References

- [Exam AZ-104](https://learn.microsoft.com/credentials/certifications/exams/az-104/)
- [AZ-104 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/az-104)
- [Azure Administrator Associate certification](https://learn.microsoft.com/credentials/certifications/azure-administrator/)
- [Azure Network Watcher overview](https://learn.microsoft.com/azure/network-watcher/network-watcher-overview)
