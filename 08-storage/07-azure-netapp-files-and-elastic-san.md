# Azure NetApp Files and Elastic SAN

## Why it matters

Some workloads need more than Azure Files or managed disks can give: SAP HANA, large Oracle databases, high-performance Linux file workloads, VMware datastores, or hundreds of VMs sharing a pool of block storage. Azure NetApp Files brings enterprise NAS performance and features as a first-party Azure service; Azure Elastic SAN brings a cloud-native storage area network. Architects and consultants meet these in migration and modernisation conversations, where picking the right one can make or break performance and cost.

## Key concepts

### Azure NetApp Files (ANF)

A first-party Azure service, built on NetApp ONTAP technology and sold and supported by Microsoft.

```text
NetApp account (regional, holds AD connection and policies)
 └─ Capacity pool (service level, provisioned size; QoS auto or manual)
      ├─ Volume "sapdata"   NFSv4.1   4 TiB
      ├─ Volume "home"      SMB       2 TiB
      └─ Volume "shared"    dual-protocol (NFS + SMB)
```

| Property | Detail |
|---|---|
| Protocols | NFSv3, NFSv4.1, SMB 3.x, dual-protocol |
| Service levels | **Standard**, **Premium**, **Ultra** (throughput per provisioned TiB rises by level), and **Flexible** (throughput provisioned independently of capacity) |
| Performance | Sub-millisecond latency; very high throughput, especially with **large volumes** (hundreds of TiB) |
| Snapshots | Instant, space-efficient; restore files or whole volumes; snapshot policies |
| Replication | **Cross-region** and **cross-zone replication** for DR |
| Backup | ANF backup to Azure storage, policy-based |
| Cool access | Moves cold blocks to cheaper storage automatically |
| Networking | Volumes live in a **delegated subnet** in your VNet; accessed privately only |
| Identity | AD DS or Entra Domain Services for SMB and Kerberos NFS; LDAP for NFS identities |
| Certified for | SAP HANA, Oracle, and many enterprise applications |

Common uses:

- **SAP HANA and SAP application** shared file systems and databases.
- **Oracle and SQL Server** databases on VMs needing high throughput.
- **Azure VMware Solution** datastores (NFS).
- **High-performance computing and EDA** (electronic design automation) workloads.
- **Virtual desktop profiles** at large scale.
- Lift-and-shift of on-premises NetApp ONTAP workloads.
- **AI and analytics** needing a fast shared POSIX file system (also via Object REST API access in newer capabilities; check status).

### Azure Elastic SAN

A cloud-native, fully managed storage area network presenting **block volumes over iSCSI**.

```text
Elastic SAN (provisioned base units + capacity units)
 └─ Volume group  (network rules: allowed VNets/private endpoints, encryption settings)
      ├─ Volume 1  (iSCSI target)  → VM, AKS (via CSI), AVS
      └─ Volume 2
```

| Property | Detail |
|---|---|
| Protocol | iSCSI |
| Provisioning | **Base capacity** (includes IOPS and throughput) plus additional **capacity-only** units; performance shared across volumes |
| Redundancy | LRS or ZRS |
| Snapshots | Volume snapshots; export to managed disk snapshots |
| Access | Private endpoints or service endpoints |
| Clients | VMs (Windows, Linux), AKS (through Azure Container Storage or iSCSI CSI), Azure VMware Solution datastores |

Elastic SAN makes sense when you have **many volumes** that together need a large pool of performance, especially when migrating from an on-premises SAN. Pooled performance is often cheaper than provisioning each managed disk individually for peak.

### Choosing among storage services

| Need | Best fit |
|---|---|
| General file shares, profiles, lift-and-shift SMB or NFS | Azure Files |
| Highest file performance, enterprise NAS features, SAP, Oracle | Azure NetApp Files |
| Block storage for a single VM | Managed disks (topic 06 file 04) |
| Large pool of block storage for many VMs or clusters, SAN migration | Elastic SAN |
| Parallel file system for HPC and AI training at extreme scale | Azure Managed Lustre (file 08) |
| Objects, data lakes, datasets | Blob Storage and Data Lake |
| VMware datastores in AVS | ANF, Elastic SAN, or vSAN |

## How it works on Azure

### ANF provisioning flow

1. Register the `Microsoft.NetApp` resource provider; check regional and quota availability.
2. Create a **NetApp account**; for SMB, add an Active Directory connection.
3. Create a **delegated subnet** (`Microsoft.NetApp/volumes`).
4. Create a **capacity pool** with a service level and size.
5. Create **volumes** with protocol, size (quota), export policy and snapshot policy.
6. Mount from clients over the VNet.

```bash
az netappfiles account create -g $RG -n anf-acct -l $LOC
az netappfiles pool create -g $RG --account-name anf-acct -n pool-premium -l $LOC --size 2 --service-level Premium
az netappfiles volume create -g $RG --account-name anf-acct --pool-name pool-premium -n vol1 -l $LOC \
  --service-level Premium --usage-threshold 100 --file-path vol1 \
  --vnet vnet-app --subnet snet-anf --protocol-types NFSv4.1 --allowed-clients 10.20.0.0/16 --rule-index 1
```

Pool size is in TiB; minimum pool and volume sizes apply, so even a lab costs real money. Check current minimums.

### Performance model

- ANF volume throughput limit = provisioned size × throughput per TiB for the service level (auto QoS), or set per volume from the pool's total (manual QoS), or provisioned directly with the Flexible level.
- Elastic SAN performance comes from base units and is shared across volumes; each volume also has its own maximums.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| ANF vs Azure Files | ANF for sub-millisecond latency, very high throughput, certified enterprise workloads, NAS features; Azure Files for everything else at lower cost |
| ANF service level | Start from required throughput, not capacity. Flexible lets you avoid over-provisioning capacity to get throughput |
| Elastic SAN vs managed disks | Elastic SAN when consolidating many volumes and pooled performance is cheaper; managed disks for individual VMs and simplicity |
| DR | ANF cross-region replication; Elastic SAN snapshots exported to disks or application-level replication |
| Cost control | Right-size pools and volumes; use ANF cool access for cold data; reserved capacity where available |

## Hands-on

ANF and Elastic SAN have minimum sizes that make even short labs costly. Use paper exercises unless you have budget.

1. **Sizing exercise:** a workload needs 2 TiB of data and 1,600 MiB/s throughput. Using current per-TiB throughput for Standard, Premium and Ultra (from the documentation), compute the capacity you must provision at each level to meet the throughput, and the monthly cost. Then compute the Flexible service level option. Which is cheapest?
2. **Elastic SAN exercise:** 40 VMs each need a 512 GiB data disk with peak 5,000 IOPS, but only 25% peak at the same time. Compare 40 Premium SSD v2 disks provisioned for peak with one Elastic SAN sized for aggregate peak.
3. If you have budget, create an Elastic SAN with one volume, connect it to a Linux VM using the iSCSI commands from the portal's "Connect" blade, format and mount it, then delete everything.

```bash
az extension add -n elastic-san -y
az elastic-san create -g $RG -n esan1 -l $LOC --base-size-tib 1 --extended-capacity-size-tib 0 --sku "{name:Premium_LRS,tier:Premium}"
az elastic-san volume-group create -g $RG -e esan1 -n vg1
az elastic-san volume create -g $RG -e esan1 -v vg1 -n vol1 --size-gib 100
```

## Security, cost and reliability notes

- **Security:** both services are private by design (delegated subnet for ANF, private endpoints for Elastic SAN). Use NSGs and export policies to restrict clients.
- **Security:** ANF supports encryption at rest with platform or customer-managed keys, Kerberos for NFSv4.1 and SMB encryption in transit.
- **Cost:** you pay for provisioned capacity, not used. Over-provisioning for throughput is the main waste; Flexible ANF and Elastic SAN pooling address it.
- **Reliability:** use availability zone placement and cross-zone or cross-region replication for ANF; ZRS for Elastic SAN where supported.

## Common mistakes

- Choosing ANF for a general team file share where Azure Files suffices.
- Sizing ANF by capacity alone and missing throughput targets.
- Forgetting the delegated subnet requirement and having to replan the VNet.
- Treating snapshots as backups without replicating or backing them up elsewhere.
- Using Elastic SAN for a handful of VMs where managed disks are simpler.

## Check yourself

1. What is the hierarchy of resources in Azure NetApp Files?
2. How does the Flexible service level differ from Standard, Premium and Ultra?
3. Name three workloads where ANF is typically chosen.
4. What protocol does Elastic SAN use, and what is its main economic advantage?
5. When would you choose managed disks over Elastic SAN?
6. How are ANF volumes accessed from the network?

<details>
<summary>Answers</summary>

1. NetApp account, capacity pool, volume.
2. Throughput is provisioned independently of capacity, instead of being a fixed amount per provisioned TiB.
3. Any three of: SAP HANA, Oracle or SQL Server databases, Azure VMware Solution datastores, HPC and EDA, large virtual desktop profile stores, lift-and-shift NetApp workloads.
4. iSCSI. Performance is pooled across many volumes, so you can provision for aggregate peak rather than each volume's peak.
5. For a small number of VMs, for simplicity, or when each disk needs independent performance guarantees.
6. Privately, through a delegated subnet in your virtual network; there is no public endpoint.

</details>

## References

- [Azure NetApp Files documentation](https://learn.microsoft.com/azure/azure-netapp-files/)
- [ANF service levels](https://learn.microsoft.com/azure/azure-netapp-files/azure-netapp-files-service-levels)
- [ANF performance considerations](https://learn.microsoft.com/azure/azure-netapp-files/azure-netapp-files-performance-considerations)
- [ANF cross-region replication](https://learn.microsoft.com/azure/azure-netapp-files/cross-region-replication-introduction)
- [Azure Elastic SAN introduction](https://learn.microsoft.com/azure/storage/elastic-san/elastic-san-introduction)
- [Plan for an Elastic SAN](https://learn.microsoft.com/azure/storage/elastic-san/elastic-san-planning)
- [Storage options for SAP on Azure](https://learn.microsoft.com/azure/sap/workloads/planning-guide-storage)
