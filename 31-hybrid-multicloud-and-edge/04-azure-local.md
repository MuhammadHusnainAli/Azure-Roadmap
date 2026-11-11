# Azure Local

## Why it matters

Some workloads must run on-premises: production lines that can't stop when the internet drops, retail stores doing local AI inference, hospitals with latency-sensitive systems, defence and government with sovereignty rules, and branch sites with limited connectivity. **Azure Local** (formerly Azure Stack HCI) is Microsoft's distributed infrastructure solution that runs VMs, AKS clusters and Azure services on validated hardware in your locations, managed through Azure Arc with the same portal, ARM templates, policy and security tools as Azure. It supports connected deployments and, for sovereign private clouds, **disconnected operations**. It's licensed per physical core with charges rolled into your Azure subscription.

## Key concepts

### What Azure Local provides

| Capability | Detail |
|---|---|
| **Hyperconverged infrastructure** | Compute (Hyper-V), software-defined storage (Storage Spaces Direct) and networking on validated partner hardware (catalogue and bill of materials) |
| **Arc VMs** | Windows and Linux VMs provisioned and managed from Azure (portal, CLI, ARM/Bicep) |
| **AKS enabled by Azure Arc on Azure Local** | Managed Kubernetes clusters on-premises for containerised and AI workloads |
| **Azure Virtual Desktop on Azure Local** | Desktops close to users or data |
| **Azure management** | Azure Policy, Defender for Cloud, Azure Monitor, Update Manager, RBAC, Azure Backup and Site Recovery integrations |
| **Edge AI** | GPU-capable hardware for local inference; Agents and Tools with Foundry Local, Video Indexer on Arc (file 07) |
| **Deployment sizes** | From single-machine to multi-machine clusters; rack-aware and multi-site designs for resilience |

### Use cases (Microsoft)

- **Local AI inferencing** where data is processed at the source (retail self-checkout, loss prevention, pipeline leak detection).
- **Mission-critical continuity** through network outages (factory production lines, stadium ticketing and access control).
- **Low-latency control systems** (manufacturing execution, quality assurance, financial infrastructure).
- **Sovereignty and regulation** requiring local data and control (defence, intelligence, energy infrastructure).

### Disconnected operations

Deploy and manage Azure Local instances **without a connection to Azure public cloud** to build sovereign private clouds. A local virtual appliance provides an Azure portal-like experience, ARM (subscriptions, resource groups, templates, CLI), RBAC, managed identity, Arc-enabled servers and other supported services locally. It needs extra hardware capacity for the appliance and meets eligibility criteria. Use cases: sovereign/compliance environments, remote or isolated locations, and reduced attack surface.

### Pricing

Per physical core per month on your machines, plus consumption-based charges for Azure services you use (for example AKS, AVD, Defender, monitoring); Azure Hybrid Benefit options apply for Windows Server Datacenter licensing scenarios. Hardware is purchased from partners.

## How it works on Azure

Deployment outline (connected):

1. Choose validated hardware from the catalogue (integrated system, validated nodes or premier solution) sized for workloads and growth.
2. Prepare Active Directory or local identity options per the current deployment guide, networking (management, compute, storage intents), and outbound connectivity (or Arc gateway/proxy).
3. Register machines with Azure Arc; deploy the Azure Local instance from the Azure portal or an ARM template.
4. Create **logical networks**, **VM images** and **storage paths**; deploy Arc VMs.
5. Deploy AKS clusters on Azure Local for container workloads; connect GitOps (file 02).
6. Apply policy, Defender, monitoring (Insights for Azure Local), Update Manager (solution updates) and backup.

Create an Arc VM on Azure Local (CLI sketch; verify parameters in the current docs):

```bash
az stack-hci-vm create --name vm-mes-01 -g rg-azlocal-factory01 \
  --custom-location /subscriptions/<sub>/resourceGroups/rg-azlocal-factory01/providers/Microsoft.ExtendedLocation/customLocations/cl-factory01 \
  --image win2025-datacenter --admin-username azureuser --admin-password "<from Key Vault>" \
  --computer-name vm-mes-01 --nics nic-mes-01 --size Default --os-type Windows --enable-agent true
```

Create an AKS cluster on Azure Local:

```bash
az aksarc create -g rg-azlocal-factory01 -n aks-factory01 \
  --custom-location cl-factory01 --vnet-ids <logical-network-id> \
  --generate-ssh-keys --node-count 3 --control-plane-count 1
```

Fleet view: Azure Local instances across sites appear in the portal with health, updates and compliance; Resource Graph queries summarise versions and update status.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Azure Local vs Azure region** | Use Azure regions unless latency, connectivity resilience, data locality or sovereignty require on-premises. |
| **Cluster size** | Single-machine for small sites (accepting lower resilience); 2+ machines for HA; rack-aware for larger sites. |
| **Connected vs disconnected** | Connected for full Azure services and simpler operations; disconnected only for strict sovereignty or isolation needs. |
| **VMs vs AKS** | VMs for legacy apps; AKS for containerised and AI workloads with GitOps. |
| **Hardware** | Validated partner solutions; GPUs for AI inference; plan capacity for updates (N+1). |
| **Operations at scale** | Standardise site templates (ARM/Bicep), GitOps, Update Manager rings across sites. |

## Hands-on

1. Review the Azure Local hardware catalogue and size a two-node cluster for 20 VMs and an AKS cluster.
2. Deploy Azure Local in a lab (nested virtualisation evaluation guides exist) and register it with Arc.
3. Create a logical network, image and Arc VM from the portal or CLI.
4. Deploy AKS on Azure Local and apply a Flux configuration.
5. Assess whether a sovereign scenario needs disconnected operations; list required services.

## Security, cost and reliability notes

- **Secured-core and security defaults**: Azure Local ships with security baselines, BitLocker, WDAC and drift control; keep them enabled.
- **Physical security** of sites matters; plan for theft and tampering.
- **Cost**: hardware, per-core subscription, Windows Server guest licensing, Azure services.
- **Resilience**: updates need capacity headroom; plan backup (Azure Backup/MABS or partners) and DR.

## Common mistakes

| Mistake | Fix |
|---|---|
| Choosing Azure Local when an Azure region would do | Justify on latency, resilience, locality or sovereignty |
| Non-validated hardware | Use the catalogue |
| Snowflake sites | Standard templates and GitOps |
| No capacity headroom for updates | N+1 sizing |
| Assuming all Azure services run disconnected | Check supported services list |

## Check yourself

1. What is Azure Local, and what was it called before?
2. Name four use cases.
3. What runs on Azure Local?
4. What do disconnected operations provide?
5. How is Azure Local priced?
6. When would you choose AKS on Azure Local over VMs?

<details>
<summary>Answers</summary>

1. Microsoft's distributed infrastructure solution extending Azure capabilities to customer-owned environments; formerly Azure Stack HCI.
2. Local AI inferencing, continuity during network outages, low-latency control systems, sovereignty/regulatory requirements.
3. Arc VMs, AKS clusters, Azure Virtual Desktop, edge AI extensions, with Azure management services.
4. Deployment and management without Azure public cloud connectivity using a local appliance with portal, ARM, RBAC, managed identity and Arc-enabled servers support.
5. Per physical core per month plus consumption for Azure services used.
6. For containerised and AI workloads where you want Kubernetes, GitOps and portability.

</details>

## References

- [What is Azure Local?](https://learn.microsoft.com/azure/azure-local/overview)
- [Disconnected operations for Azure Local](https://learn.microsoft.com/azure/azure-local/manage/disconnected-operations-overview)
- [AKS enabled by Azure Arc on Azure Local](https://learn.microsoft.com/azure/aks/aksarc/aks-overview)
- [Azure Local system requirements](https://learn.microsoft.com/azure/azure-local/concepts/system-requirements-23h2)
