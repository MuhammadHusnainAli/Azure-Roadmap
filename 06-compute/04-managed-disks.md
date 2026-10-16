# Managed Disks

> Topic: 06 · Compute · Tracks: E ○ A ● C – · Status checked: 2026-10

## Why it matters

Disks decide the performance of anything stateful on a VM, account for a large share of IaaS cost, and hold the data you most need to protect. Disk type, the VM's own throughput cap, caching and encryption are each a common source of performance complaints and audit findings. The same disks back AKS persistent volumes and database VMs.

## Key concepts

### Managed disks

A managed disk is a block-storage volume that Azure manages for you: you choose the type and size, and Azure handles placement, replication and availability. Each disk is a first-class resource with its own RBAC, tags, snapshots and encryption settings.

**Roles**

| Disk | Notes |
|---|---|
| **OS disk** | Holds the operating system. One per VM |
| **Data disks** | For application data. The number allowed depends on VM size |
| **Temporary disk** | Local to the host, ephemeral. Not a managed disk and not durable (see below) |

### Disk types

| | **Standard HDD** | **Standard SSD** | **Premium SSD** | **Premium SSD v2** | **Ultra Disk** |
|---|---|---|---|---|---|
| Media | HDD | SSD | SSD | SSD | SSD |
| Performance set by | Size | Size | Size tier (P-sizes); tier can be raised independently | **IOPS and throughput set independently of size** | IOPS and throughput set independently, changeable online |
| Max IOPS (approx.) | 2,000 | 6,000 | 20,000 | 80,000 | 400,000 |
| Max throughput (approx.) | 500 MB/s | 750 MB/s | 900 MB/s | 1,200 MB/s | 10,000 MB/s |
| Latency | Variable | Low-ish | Single-digit ms | Sub-millisecond | Sub-millisecond |
| Usable as OS disk | Yes | Yes | Yes | No | No |
| Host caching | Yes | Yes | Yes | No | No |
| Redundancy | LRS | LRS, ZRS | LRS, ZRS | LRS (zonal) | LRS (zonal) |
| Typical use | Backup, archive, infrequent access | Web servers, light dev/test | Production workloads, databases, OS disks | Most performance-sensitive data disks: the best price-performance | Extreme IO: top-tier databases, SAP HANA logs |

Figures change; treat them as orders of magnitude and check the current tables.

Guidance:

- **OS disks:** Premium SSD (or Standard SSD for non-critical VMs), or an **ephemeral OS disk** for stateless instances.
- **Data disks:** **Premium SSD v2** as the default for anything performance-sensitive. You provision capacity, IOPS and throughput separately and pay for each, which usually beats Premium SSD on cost for the same performance.
- **Ultra Disk** when you need more than Premium SSD v2 offers.
- **Standard HDD** only for cold data.

Premium SSD v2 and Ultra Disks are **zonal**, require zone-aware VM placement in most regions, do not support host caching, and have some feature limits compared with Premium SSD. Check region and feature support.

### Performance: three layers of limits

A disk delivers the **minimum** of:

1. what the **disk** is provisioned for;
2. what the **VM size** allows: every size has caps on *uncached* and *cached* disk IOPS and throughput, summed across all attached disks;
3. what the **application and file system** can drive.

A 20,000 IOPS disk on a VM capped at 6,400 uncached IOPS delivers 6,400. This is the most common disk performance mistake. Check both in the size documentation, and watch the VM metrics *VM Uncached IOPS Consumed Percentage* and *Data Disk IOPS Consumed Percentage*.

**Bursting**

| Type | Behaviour |
|---|---|
| **Credit-based** (Premium SSD ≤ 512 GiB, Standard SSD ≤ 1 TiB) | Bursts for short periods (up to 30 minutes) using accumulated credits. Free |
| **On-demand** (Premium SSD > 512 GiB) | Bursts whenever needed beyond provisioned performance, for an additional charge |
| **VM-level bursting** | Some VM sizes can burst their own disk caps |

**Changing performance without resizing**

- **Premium SSD performance tiers:** raise a disk's tier above its size's baseline temporarily (for a batch run or seasonal peak), then lower it.
- **Premium SSD v2 / Ultra:** adjust IOPS and throughput directly.

**Host caching** (Standard and Premium SSD only)

| Setting | Use |
|---|---|
| **ReadOnly** | Read-heavy data disks; reads served from the host's cache |
| **ReadWrite** | OS disks (default). Risky for data unless the application handles write flushing |
| **None** | Write-heavy and log disks; required for Premium SSD v2 and Ultra |

### Redundancy

| Option | Copies | Protects against |
|---|---|---|
| **LRS** (locally redundant) | Three, in one data centre (or zone, for zonal disks) | Drive and rack failure |
| **ZRS** (zone-redundant) | Three, across three zones, written synchronously | A zone failure. The disk can be attached to a VM in another zone |

ZRS (Premium SSD and Standard SSD) has slightly higher write latency. It suits applications without their own replication. For clustered databases that replicate at the application layer, zonal LRS disks per replica are the usual choice.

Managed disks are not geo-replicated. Cross-region protection comes from snapshots copied to another region, Azure Backup, Site Recovery, or application replication.

### The temporary disk and ephemeral OS disks

**Temporary disk:** present on sizes with `d` in the name. Local to the physical host, fast, free, and **wiped** on deallocation, resize, redeployment or host failure. Use for page or swap files, tempdb, scratch and caches. Many newer sizes have none, and on the newest generations local disks are raw NVMe devices you must format.

**Ephemeral OS disk:** the OS disk itself placed on local storage (the cache, temporary, or NVMe disk).

| | Managed OS disk | Ephemeral OS disk |
|---|---|---|
| Storage cost | Yes | None |
| Read/write latency | Remote | Local, faster |
| Reimage / reset | Slower | Fast, back to the image state |
| Stop-deallocate | Supported | Not supported in the usual sense: state is lost |
| Persistence | Durable | Lost on redeploy or host failure |

Ideal for stateless scale sets and AKS nodes: faster provisioning and scaling at lower cost.

### Shared disks

A managed disk attached to **several VMs at once** (`maxShares` > 1), using SCSI Persistent Reservations. Azure provides the shared block device; a cluster manager or cluster-aware file system (Windows Server Failover Cluster, Pacemaker with a clustered file system) must coordinate access. Available on Premium SSD, Premium SSD v2, Ultra and Standard SSD. Used for clustered SQL Server failover cluster instances, SAP ASCS and similar legacy clustering.

### Encryption

There are several layers, and they are easy to confuse.

| Layer | What it encrypts | Keys | Notes |
|---|---|---|---|
| **Server-side encryption (SSE) at rest** | Data on the storage cluster | **Platform-managed** by default, or **customer-managed** in Key Vault / Managed HSM via a **disk encryption set** | Always on; cannot be disabled |
| **Encryption at host** | The VM host's local storage: **temporary disk, caches, ephemeral OS disk**, and data in transit between host and storage | Same as the disk's SSE keys | Enable per VM. Closes the gap SSE leaves. **Recommended** |
| **Azure Disk Encryption (ADE)** | Inside the guest, using BitLocker (Windows) or dm-crypt (Linux) | Key Vault | The older approach. It has performance and operational overhead and is scheduled for retirement; move to encryption at host |
| **Confidential disk encryption** | OS disk bound to the VM's vTPM, for confidential VMs | Platform or customer-managed | Protects the disk from the host operator (file 06) |
| **Double encryption at rest** | Two layers of SSE with different keys | Platform plus customer-managed | For specific compliance requirements |

Practical default: **SSE with platform-managed keys plus encryption at host**. Move to **customer-managed keys** through a disk encryption set when policy requires control of key lifecycle and the ability to revoke.

With customer-managed keys: use a user-assigned identity on the disk encryption set, enable automatic key rotation, and protect the vault with purge protection. Losing access to the key means losing access to the disks.

### Snapshots, restore points and backup

| Mechanism | What it is | Use |
|---|---|---|
| **Incremental snapshot** | A point-in-time, read-only copy of one disk, billed only for changed data. Can be copied across regions | Before a risky change; short-term protection; building images |
| **VM restore points** | Multi-disk, application- or crash-consistent snapshots of a VM, grouped in a restore point collection | Consistent point-in-time of a whole VM |
| **Azure Backup** | Policy-driven, scheduled backups with retention, vaults, soft delete, immutability and cross-region restore | The standard backup solution for VMs and disks |
| **Azure Site Recovery** | Continuous replication to another zone or region with orchestrated failover | Disaster recovery |

Snapshots are not backups on their own: no schedule, no retention policy, same blast radius unless copied elsewhere. Topic 28 covers the full picture.

### Other operations

- **Expand** a disk online for most types (no detach needed), then extend the file system in the guest. Disks cannot be shrunk.
- **Convert** between disk types, mostly with the VM deallocated or the disk detached.
- **Attach and detach** data disks on running VMs.
- **Upload and download** VHDs directly to and from managed disks.
- **Disk access / private links** to restrict import and export of disks to a private network, and `networkAccessPolicy` to disable export entirely.

## How it works on Azure

### Laying out disks for a database VM

| Volume | Disk | Caching | Why |
|---|---|---|---|
| OS | Premium SSD | ReadWrite | Standard |
| Data files | Premium SSD v2, IOPS and throughput sized to the workload | None | Best price-performance |
| Transaction log | Premium SSD v2 or Ultra, sized for write throughput and low latency | None | Sequential synchronous writes |
| tempdb / scratch | Local temporary or NVMe disk | – | Fast and disposable |
| Backups | Blob Storage, not a disk | – | Cheaper, durable, off-VM |

Then choose a **VM size whose uncached caps exceed the sum** of what the disks will be asked for. A managed database service removes this exercise entirely, which is one reason to prefer one (topic 09).

### Disks and AI workloads

| Need | Storage |
|---|---|
| Model weights and container images on inference nodes | Local NVMe or ephemeral OS disk for fast load; pulled from a registry or Blob Storage |
| Training datasets | Blob Storage or Data Lake (often mounted with BlobFuse), **Azure Managed Lustre** or Azure NetApp Files for high-throughput shared access. Not individual managed disks |
| Checkpoints | Blob Storage, so they survive Spot eviction and node loss |
| Scratch during training | Local NVMe on the GPU VM |
| Vector indexes for a self-hosted engine | Premium SSD v2 or local NVMe with replication |
| AKS persistent volumes | Azure Disk CSI driver (Premium SSD v2 for performance), Azure Container Storage, or Azure Files for shared access |

Large GPU sizes come with substantial local NVMe precisely because loading multi-gigabyte model weights from remote disks is slow.

### Monitoring

Key metrics: disk IOPS and throughput consumed percentage, burst credits remaining, queue depth, latency; and on the VM, cached and uncached IOPS and bandwidth consumed percentage. Sustained values near 100% mean throttling. Find out whether the disk or the VM is the limit before paying for a bigger disk.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Disk type for data | Premium SSD v2 unless a feature it lacks is required |
| LRS or ZRS | ZRS for single-instance workloads needing zone resilience. Zonal LRS per replica for software that replicates itself |
| Platform- or customer-managed keys | Platform by default. Customer-managed for policy, accepting the operational duty |
| ADE or encryption at host | Encryption at host |
| Ephemeral or managed OS disk | Ephemeral for stateless, image-based fleets |
| Many small disks or few large | With Premium SSD v2, size for need. Striping several disks is for exceeding a single disk's limits |
| Disk or another storage service | Shared files: Azure Files or NetApp Files. Object data: Blob. Databases: a managed service. Disks are for software that needs a block device |

## Hands-on

```bash
RG=rg-disk-lab; LOC=swedencentral; ZONE=1
az group create -n $RG -l $LOC -o none

# 1. A zonal VM
az vm create -g $RG -n vm-disk --image Ubuntu2404 --size Standard_D4ads_v6 --zone $ZONE \
  --public-ip-address "" --encryption-at-host true \
  --admin-username azureuser --generate-ssh-keys -o none

# 2. What the VM size itself allows
az vm list-skus -l $LOC --size Standard_D4ads_v6 \
  --query "[0].capabilities[?contains(name,'IOPS') || contains(name,'BytesPerSecond') || name=='MaxDataDiskCount'].{Cap:name, Value:value}" -o table

# 3. A Premium SSD v2 data disk with performance set independently of size
az disk create -g $RG -n disk-data-01 --size-gb 64 --sku PremiumV2_LRS --zone $ZONE \
  --disk-iops-read-write 5000 --disk-mbps-read-write 200 -o none
az vm disk attach -g $RG --vm-name vm-disk --name disk-data-01 -o none

# 4. Format, mount, and test throughput
az vm run-command invoke -g $RG -n vm-disk --command-id RunShellScript --scripts '
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
DEV=$(lsblk -dn -o NAME,SIZE | awk "\$2==\"64G\"{print \"/dev/\"\$1}" | head -1)
mkfs.ext4 -q $DEV && mkdir -p /data && mount $DEV /data
apt-get -qq install -y fio >/dev/null 2>&1
fio --name=rand --filename=/data/test --size=2G --bs=8k --rw=randrw --iodepth=64 --direct=1 --runtime=30 --time_based --group_reporting | grep -E "read:|write:"' \
  --query "value[0].message" -o tsv

# 5. Raise performance online, without resizing
az disk update -g $RG -n disk-data-01 --disk-iops-read-write 10000 --disk-mbps-read-write 300 -o none

# 6. Incremental snapshot, then expand the disk
az snapshot create -g $RG -n snap-data-01 --source disk-data-01 --incremental true -o none
az disk update -g $RG -n disk-data-01 --size-gb 128 -o none
az disk show -g $RG -n disk-data-01 --query "{sizeGb:diskSizeGB, iops:diskIOPSReadWrite, mbps:diskMBpsReadWrite, sku:sku.name, encryption:encryption.type}"

az group delete -n $RG --yes --no-wait
```

The VM needs an outbound path to install `fio`; skip that line if it has none and compare the configured values instead. Compare the fio result with both the disk's provisioned IOPS and the VM's uncached cap from step 2, and note which one limited you.

Exercise: a workload needs 30,000 IOPS and 400 MB/s on 500 GiB. Compare the monthly price of one Premium SSD at the size that reaches that performance against a Premium SSD v2 provisioned exactly, and pick a VM size able to deliver it.

## Security, cost and reliability notes

- **Security:** without encryption at host, the temporary disk and caches are not covered by your key choices. Enable it by policy.
- **Security:** disk export (generating a SAS for a disk or snapshot) is a data exfiltration path. Restrict the action with RBAC and set the network access policy to deny or private-link only.
- **Security:** snapshots contain the full data. Protect and expire them.
- **Cost:** disks bill for **provisioned** size (and provisioned IOPS and throughput on v2 and Ultra), not used space, and continue billing when the VM is deallocated.
- **Cost:** unattached disks and forgotten snapshots are classic waste. Query for `diskState == 'Unattached'` regularly.
- **Reliability:** LRS disks do not survive a zone outage. Design at the application or ZRS level.
- **Reliability:** test restores. A backup that has never been restored is an assumption.

## Common mistakes

- Oversized disks on a VM that cannot deliver their performance.
- Data on the temporary disk.
- Premium SSD for large data volumes where Premium SSD v2 would cost less for more performance.
- ReadWrite caching on database log disks.
- Azure Disk Encryption on new builds.
- Customer-managed keys without purge protection or a rotation plan.
- Snapshots treated as the backup strategy.
- Orphaned disks left after VM deletion.
- Individual disks for shared training data that needed a file or object store.
- Expecting to shrink a disk.

## Check yourself

1. Which disk type lets you set IOPS and throughput independently of size, at the best price-performance?
2. What three layers limit the performance a disk delivers?
3. What does encryption at host cover that server-side encryption alone does not?
4. When is an ephemeral OS disk a good choice?
5. LRS versus ZRS for managed disks: what is the difference?
6. Why are snapshots not a backup strategy by themselves?
7. Which host caching setting suits a transaction log disk?
8. Where should training datasets and checkpoints live?

<details>
<summary>Answers</summary>

1. Premium SSD v2 (Ultra Disk also allows it, at higher performance and price).
2. The disk's provisioned performance, the VM size's cached and uncached disk caps, and the application or file system.
3. The temporary disk, host caches and ephemeral OS disk, and data flowing between the host and storage.
4. For stateless, image-based instances such as scale set and AKS nodes, where fast provisioning and reimaging matter and local state can be lost.
5. LRS keeps three copies in one data centre or zone; ZRS replicates synchronously across three zones and survives a zone failure.
6. They have no scheduling or retention policy and, unless copied elsewhere, share the fate of the source.
7. None.
8. In Blob Storage or a high-throughput shared file system such as Azure Managed Lustre, not on individual managed disks; checkpoints in durable object storage so they survive eviction.

</details>

## References

- [Introduction to Azure managed disks](https://learn.microsoft.com/azure/virtual-machines/managed-disks-overview)
- [Azure managed disk types](https://learn.microsoft.com/azure/virtual-machines/disks-types)
- [Virtual machine and disk performance](https://learn.microsoft.com/azure/virtual-machines/disks-performance)
- [Managed disk bursting](https://learn.microsoft.com/azure/virtual-machines/disk-bursting)
- [Redundancy options for managed disks](https://learn.microsoft.com/azure/virtual-machines/disks-redundancy)
- [Overview of managed disk encryption options](https://learn.microsoft.com/azure/virtual-machines/disk-encryption-overview)
- [Encryption at host](https://learn.microsoft.com/azure/virtual-machines/disk-encryption#encryption-at-host---end-to-end-encryption-for-your-vm-data)
- [Ephemeral OS disks](https://learn.microsoft.com/azure/virtual-machines/ephemeral-os-disks)
- [Share an Azure managed disk](https://learn.microsoft.com/azure/virtual-machines/disks-shared)
- [Create an incremental snapshot](https://learn.microsoft.com/azure/virtual-machines/disks-incremental-snapshots)
