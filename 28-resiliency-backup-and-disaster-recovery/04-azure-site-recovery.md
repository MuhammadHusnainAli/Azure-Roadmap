# Azure Site Recovery

## Why it matters

Restoring hundreds of VMs from backup after a regional disaster can take days; many business-critical systems need minutes or hours. **Azure Site Recovery (ASR)** continuously replicates VMs and physical servers to a secondary location and orchestrates failover with **recovery plans**, so a whole multi-tier application can come up in the right order, with scripts, in the target region. It covers Azure-to-Azure DR, on-premises VMware/Hyper-V/physical servers to Azure, and replication from Azure Extended Zones (Preview). Crucially, it supports **test failovers** into isolated networks without affecting production, which is how DR plans become proven rather than theoretical.

## Key concepts

### Scenarios

| Scenario | Notes |
|---|---|
| **Azure VM → another Azure region** | Agent (Mobility service extension) replicates disks continuously; recovery points every few minutes; crash-consistent and app-consistent points |
| **Azure VM zone-to-zone** | DR between availability zones in the same region where cross-region isn't possible (data residency) |
| **VMware VMs / physical servers → Azure** | Azure Site Recovery replication appliance (modernised, replacing the configuration server) |
| **Hyper-V VMs → Azure** | Hyper-V provider/agent |
| **Azure Extended Zones → region** | Preview |
| **Migration** | ASR can migrate, but Azure Migrate is the recommended migration tool (Topic 30) |

**High churn** support for Azure VMs handles data change rates up to 100 MB/s (some configurations up to 500 MB/s).

### Core concepts

| Concept | Meaning |
|---|---|
| **Replication policy** | Recovery point retention and app-consistent snapshot frequency |
| **Recovery point** | Crash-consistent (frequent) or app-consistent (VSS/pre-post scripts) |
| **Target configuration** | Target region, resource group, VNet/subnet, availability zone or set, disk types, capacity reservation |
| **Recovery plan** | Groups of machines failed over in order with manual actions and Azure Automation scripts (for example update DNS, start app tier after DB) |
| **Test failover** | Bring up machines in an isolated test VNet from a recovery point; validate; clean up |
| **Planned failover** | Zero data loss migration-style failover (source shut down first) |
| **Unplanned failover** | Disaster failover from latest or chosen recovery point |
| **Commit / reprotect / failback** | Finalise, start reverse replication, return to primary later |

### RTO and RPO

RPO depends on replication lag (typically seconds to minutes for Azure-to-Azure); RTO depends on recovery plan steps, VM boot times and application startup. Measure both in test failovers.

### What ASR doesn't do

It replicates VMs, not PaaS services. PaaS components need their own geo-replication (SQL failover groups, Storage GRS, Cosmos DB multi-region, Key Vault, App Service redeployment) coordinated in the recovery plan or runbooks. Infrastructure Resiliency Manager adds recovery orchestration plans for zonal failover (Preview, file 07).

## How it works on Azure

Enable Azure-to-Azure replication (high level):

1. Create a Recovery Services vault **in the target region**.
2. Enable replication for VMs: choose target region, resource group, VNet, zones, replication policy (for example 24-hour retention, app-consistent every 4 hours).
3. Pre-create target networking (subnets, NSGs, private DNS, firewall) via IaC; configure network mapping.
4. Use **capacity reservation groups** in the target region for critical VM sizes.
5. Build a **recovery plan**: Group 1 domain controllers/DNS (if IaaS), Group 2 databases, Group 3 app servers, Group 4 web servers; add Automation runbooks to update Front Door/Traffic Manager, connection strings or DNS.
6. Run **test failover** into an isolated VNet quarterly; document RTO/RPO achieved.

CLI example to start a test failover of a recovery plan:

```bash
az site-recovery recovery-plan test-failover -g rg-dr-weu --vault-name rsv-dr-neu -n rp-erp \
  --failover-direction PrimaryToRecovery \
  --network-id /subscriptions/<sub>/resourceGroups/rg-dr-neu/providers/Microsoft.Network/virtualNetworks/vnet-dr-test \
  --network-type VmNetworkAsInput
```

(The `site-recovery` CLI is an extension; parameter names can change, so verify with `az site-recovery recovery-plan test-failover --help`.)

Monitor replication health, RPO and test failover status in the vault and in Azure Business Continuity Center; alert on replication errors.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **ASR vs redeploy-from-IaC** | ASR for stateful or legacy VMs that can't be rebuilt quickly; IaC redeploy for stateless, immutable infrastructure. |
| **Cross-region vs zone-to-zone** | Cross-region for regional disasters; zone-to-zone where residency forbids leaving the region (zones already help availability). |
| **App consistency** | App-consistent points for databases on VMs; crash-consistent for stateless tiers. |
| **Networking** | Same IP ranges in DR (non-overlapping with active primary if connected) or re-IP with DNS updates; decide early. |
| **Capacity** | Capacity reservations for critical sizes; DR without capacity is not DR. |
| **Backups too** | ASR is not a backup; combine with Azure Backup for point-in-time recovery. |

## Hands-on

1. Replicate a two-tier VM application to a secondary region.
2. Create a recovery plan with ordered groups and a runbook that updates a Traffic Manager endpoint.
3. Run a test failover into an isolated VNet; validate the app; clean up.
4. Measure RPO from the replication health view and RTO from the test.
5. Document the failback procedure.

## Security, cost and reliability notes

- **Isolation of test failovers** prevents duplicate machines conflicting with production (domain, IPs).
- **Security baseline** in the DR region (policies, Defender, Key Vault, private endpoints) must match production.
- **Cost**: per protected instance plus replica storage and snapshots; compute only after failover (plus capacity reservations if used).
- **Drift**: changes in primary networking or new VMs must be reflected in DR configuration; automate checks.

## Common mistakes

| Mistake | Fix |
|---|---|
| Replicating VMs but not PaaS dependencies | Plan PaaS geo-replication in the recovery plan |
| Never testing failover | Quarterly test failovers |
| No capacity in target region | Capacity reservations |
| Vault in the same region as source | Vault in target region |
| Treating ASR as backup | Combine with Azure Backup |

## Check yourself

1. Which scenarios does ASR support?
2. What is a recovery plan, and what can it include?
3. How does a test failover work?
4. Compare planned and unplanned failover.
5. What does ASR not protect?
6. Why use capacity reservations with ASR?

<details>
<summary>Answers</summary>

1. Azure VM cross-region and zone-to-zone, VMware/physical and Hyper-V to Azure, Extended Zones to region (Preview); migration (though Azure Migrate is preferred).
2. An ordered grouping of machines for failover with manual actions and Automation runbooks.
3. Machines start from a recovery point in an isolated network without affecting production replication; you validate and clean up.
4. Planned: zero data loss, source shut down first; unplanned: disaster failover from latest or selected recovery point with possible data loss.
5. PaaS services and point-in-time recovery from corruption; those need geo-replication and backups.
6. To guarantee VM capacity in the target region during a regional event.

</details>

## References

- [About Site Recovery](https://learn.microsoft.com/azure/site-recovery/site-recovery-overview)
- [Azure-to-Azure disaster recovery architecture](https://learn.microsoft.com/azure/site-recovery/azure-to-azure-architecture)
- [Recovery plans](https://learn.microsoft.com/azure/site-recovery/recovery-plan-overview)
- [Test failover to Azure](https://learn.microsoft.com/azure/site-recovery/azure-to-azure-tutorial-dr-drill)
