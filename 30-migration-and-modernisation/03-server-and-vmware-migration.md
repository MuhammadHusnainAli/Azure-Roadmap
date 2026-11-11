# Server and VMware Migration

## Why it matters

Servers are the bulk of most migrations: hundreds or thousands of Windows and Linux VMs on VMware, Hyper-V or physical hardware, often with fragile dependencies and limited documentation. Two paths dominate: **migrate VMs to native Azure VMs** with Azure Migrate (replicate, test, cut over), or **move VMware workloads as-is to Azure VMware Solution (AVS)**, a dedicated VMware vSphere private cloud in Azure, when speed, VMware tooling compatibility or licensing make that attractive. Many programmes combine both: AVS for a fast datacentre exit, then gradual moves from AVS to native Azure services. Executing cutovers safely, with tests and rollback, is what makes migrations boring.

## Key concepts

### Paths

| Path | How | Pros | Cons |
|---|---|---|---|
| **Azure VMs via Azure Migrate (Migrate and Modernize tool)** | Replicate disks to Azure, test migrate, cutover | Native Azure, rightsizing, AHB, path to PaaS | OS/driver conversion; per-VM work; IP and DNS changes |
| **Azure VMware Solution** | Move VMs with **VMware HCX** (vMotion, bulk, replication-assisted vMotion) into a private cloud | Minimal change, same tooling and skills, fast large-scale moves, L2 extension | Dedicated host cost (minimum cluster), still VMware operations; modernisation deferred |
| **Rebuild on Azure** | Deploy fresh from IaC and migrate data | Clean, modern | Most effort |

### Azure Migrate server migration

| Source | Method |
|---|---|
| **VMware** | **Agentless** (uses the discovery appliance, snapshot-based replication) or **agent-based** (Mobility service, replication appliance) |
| **Hyper-V** | Provider agents on Hyper-V hosts |
| **Physical, other clouds (AWS, GCP)** | Agent-based (treated as physical) with replication appliance |
| **Web apps** | Agentless ASP.NET web app migration to App Service at scale (file 05) |

Steps: prepare (landing zone, networking, Entra, permissions), **replicate**, **test migration** into an isolated VNet, fix issues, schedule **cutover** (stop source, final sync, migrate), validate, **complete migration** (clean up replication), decommission.

### Azure VMware Solution

| Concept | Detail |
|---|---|
| **Private cloud** | vSphere clusters on dedicated bare-metal Azure hosts (minimum initial cluster size applies), with vCenter, vSAN, NSX managed by Microsoft |
| **Generations** | Gen 1 (Microsoft-managed ExpressRoute to Azure) and **Gen 2** (hosts connected directly into Azure VNets, simplified networking) |
| **Host SKUs** | AV36, AV36P, AV48, AV52, AV64 (AV64 for expansion or direct deployment in Gen 2, per docs) |
| **HCX** | Migration and network extension tool: bulk migration, vMotion, replication-assisted vMotion, L2 network extension |
| **Integration** | Azure services via VNet (Gen 2) or ExpressRoute Global Reach to on-premises; Azure Backup/partners; Defender for Servers via Arc |
| **Licensing** | VMware licensing is included or brought depending on the current AVS offer; check current terms |

### Cutover patterns

| Concern | Approach |
|---|---|
| **IP addresses** | Re-IP with DNS updates (preferred for native Azure); keep IPs with HCX L2 extension temporarily (AVS) |
| **Downtime** | Minutes for final sync and boot; schedule windows; communicate |
| **Rollback** | Keep source VMs powered off but intact until sign-off; documented rollback steps |
| **Validation** | Smoke tests, monitoring, application owner sign-off, hypercare period |

## How it works on Azure

Agentless VMware migration flow (Azure Migrate):

```text
1. Discovery and assessment complete (file 02); target subscription, VNet, subnets and NSGs ready.
2. Replicate: choose VMs, target size (from assessment), disks (Premium SSD v2/Standard SSD), AHB, availability zone, target VNet.
3. Monitor initial replication and delta syncs.
4. Test migration into an isolated test VNet; run application tests; clean up test resources.
5. Cutover: shut down source VM (option to do so automatically), final delta replication, create Azure VM.
6. Post-migration: install/verify Azure VM agent, extensions (AMA, Defender), backup, tags, update DNS, decommission.
```

Post-migration hardening checklist applied by policy and automation: Azure Monitor Agent and DCRs, Defender for Servers, Azure Backup policy, Update Manager schedule, tags, no public IPs, disk encryption at host, boot diagnostics.

AVS pattern for a datacentre exit:

```text
On-premises vSphere ── HCX (bulk migration, L2 extension) ── AVS private cloud (Gen 2) ── Azure VNets (hub)
Phase 1: move all VMs to AVS within months; keep IPs via L2 extension.
Phase 2: remove L2 extensions, re-IP gradually.
Phase 3: modernise selected workloads from AVS to native PaaS/VMs.
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Native VMs vs AVS** | Native for long-term cost and modernisation; AVS for speed, scale, VMware dependencies (appliances, specific vSphere features) or licensing transition. |
| **Agentless vs agent-based** | Agentless for VMware simplicity; agent-based for physical/other clouds or when agentless limits apply. |
| **Rightsizing at migration** | Use performance-based sizes; avoid copying oversized on-premises VMs. |
| **Disks** | Choose SSD types by measured IOPS/throughput; consider Premium SSD v2. |
| **Zones and DR** | Place VMs in zones; add backup and Site Recovery as part of migration (Topic 28). |
| **Licensing** | Apply AHB; consider Extended Security Updates for older Windows/SQL versions on Azure. |

## Hands-on

1. Replicate a test VMware or Hyper-V VM with Azure Migrate; run a test migration into an isolated VNet.
2. Perform a cutover and complete the migration; apply the post-migration checklist via policy.
3. Design an AVS Gen 2 private cloud for 300 VMs: cluster sizing, networking, HCX approach.
4. Write a cutover runbook with rollback steps.
5. Compare 3-year costs of native VMs vs AVS for the same estate using assessments.

## Security, cost and reliability notes

- **Security baseline on arrival**: Defender, AMA, backup, encryption, no public IPs.
- **Replication traffic**: plan bandwidth; use ExpressRoute/VPN; secure replication appliances.
- **Cost**: replication storage during migration; AVS hosts are billed whether used or not; plan reservations.
- **Reliability**: rehosted single VMs are not zone-resilient by default; redesign critical tiers.

## Common mistakes

| Mistake | Fix |
|---|---|
| Skipping test migrations | Always test in isolation |
| Same VM sizes as on-premises | Performance-based rightsizing |
| Permanent L2 extensions | Plan re-IP and removal |
| Forgetting extensions/backup after cutover | Policy-driven post-migration baseline |
| Deleting sources immediately | Keep until sign-off and rollback window ends |

## Check yourself

1. Compare migrating to Azure VMs with moving to AVS.
2. What are the agentless and agent-based migration options?
3. Describe the migration steps from replication to completion.
4. What does HCX provide?
5. How do you handle IP addresses during migration?
6. What should be applied immediately after cutover?

<details>
<summary>Answers</summary>

1. Azure VMs: native, rightsized, path to PaaS, more per-VM change. AVS: minimal change and fast with VMware tooling, but dedicated host cost and continued VMware operations.
2. Agentless for VMware using the appliance; agent-based with the Mobility service for VMware, physical and other clouds; Hyper-V uses host providers.
3. Prepare, replicate, test migration, fix, cutover with final sync, validate, complete migration, decommission.
4. Bulk migration, vMotion, replication-assisted vMotion and L2 network extension to AVS.
5. Re-IP with DNS updates for native Azure; temporarily retain IPs with HCX L2 extension for AVS.
6. Monitoring agent and DCRs, Defender, backup, updates, tags, encryption, network hardening, DNS updates.

</details>

## References

- [Migrate VMware VMs (agentless)](https://learn.microsoft.com/azure/migrate/tutorial-migrate-vmware)
- [Migrate Hyper-V VMs](https://learn.microsoft.com/azure/migrate/tutorial-migrate-hyper-v)
- [Migrate physical or other-cloud servers](https://learn.microsoft.com/azure/migrate/tutorial-migrate-physical-virtual-machines)
- [Prepare for migration](https://learn.microsoft.com/azure/migrate/prepare-for-migration)
- [What is Azure VMware Solution?](https://learn.microsoft.com/azure/azure-vmware/introduction)
- [Configure VMware HCX](https://learn.microsoft.com/azure/azure-vmware/configure-vmware-hcx)
