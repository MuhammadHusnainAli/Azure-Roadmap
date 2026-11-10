# Data Transfer

## Why it matters

Every migration, every new data platform and every AI project starts by moving data into Azure, and many end by moving results somewhere else. The right tool depends on volume, bandwidth, frequency and whether the source is online. Choosing badly turns a two-day copy into a two-month project, or produces an unexpected egress bill. Consultants are often asked "how long will it take to move our data?"; the answer comes from the arithmetic in this file.

## Key concepts

### The arithmetic

```text
Transfer time (days) ≈ data size (TB) × 8,000,000 ÷ (bandwidth (Mbps) × utilisation × 86,400)
```

| Data | 100 Mbps at 80% | 1 Gbps at 80% | 10 Gbps at 80% |
|---|---|---|---|
| 1 TB | ~1.2 days | ~2.8 hours | ~17 minutes |
| 10 TB | ~12 days | ~1.2 days | ~2.8 hours |
| 100 TB | ~116 days | ~12 days | ~1.2 days |
| 1 PB | ~3 years | ~116 days | ~12 days |

If the network transfer takes longer than shipping a device (roughly one to two weeks end to end), use offline transfer.

### Tool selection

| Tool | Type | Best for |
|---|---|---|
| **AzCopy** | CLI, online | Scripted copies to and from Blob, Data Lake and Files; server-to-server copies between accounts and from S3 or Google Cloud Storage |
| **Azure Storage Explorer** | Desktop GUI (uses AzCopy) | Ad hoc browsing, upload, download, permissions |
| **Azure Storage Mover** | Managed migration service with agents | Migrating NFS and SMB file shares to Blob and Azure Files, with managed jobs, monitoring, and cloud-to-cloud moves (for example from AWS S3; check current status) |
| **Azure File Sync** | Continuous sync | Windows file servers to Azure Files (file 04) |
| **Azure Data Factory / Fabric Data Factory** | Data integration pipelines | Recurring, transformed, database and SaaS sources (topic 12) |
| **Azure Data Box family** | Offline physical devices | Tens of TB to petabytes, limited bandwidth |
| **Azure Import/Export** | Your own disks shipped | Niche; Data Box is usually simpler |
| **Object replication** | Continuous, account to account | Ongoing copies between storage accounts (file 02) |
| `az storage` / SDKs | CLI and code | Small, programmatic transfers |
| **Database Migration Service** | Database-aware | Databases (topic 09) |

### AzCopy essentials

| Feature | Detail |
|---|---|
| Auth | `azcopy login` (Entra, including managed identity), or SAS |
| Server-to-server | `azcopy copy` between URLs copies within Azure without passing data through the client |
| Sync | `azcopy sync` copies only changed files (by last modified time or hash) |
| Performance | Parallelism (`AZCOPY_CONCURRENCY_VALUE`), throughput cap (`--cap-mbps`), automatic retries, resumable jobs (`azcopy jobs resume`) |
| Other clouds | Copy from Amazon S3 and Google Cloud Storage directly into Blob |
| Preservation | `--preserve-smb-info`, `--preserve-permissions` for Files; `--put-md5` for integrity checks |

### Azure Data Box family

| Device | Capacity (approximate, usable) | Notes |
|---|---|---|
| **Data Box Disk** | SSDs of several TB each, a few per order | Small offline transfers |
| **Data Box** (current generation) | Larger capacity options (on the order of 100 TB to over 500 TB per device, depending on model) | Most common; faster interfaces in newer models |
| **Data Box Heavy** | Petabyte-scale (where still offered) | Check availability; newer Data Box models have replaced many uses |

Flow: order in the portal → device shipped → copy data over SMB or NFS locally → ship back → data uploaded to your storage account → device securely erased (NIST guidelines). Data on the device is AES-256 encrypted. Data Box also supports **export** from Azure.

Models, capacities and regional availability change; check the current Data Box documentation.

### Online vs offline decision

```text
Is the source online and the network adequate (transfer ≤ ~1–2 weeks)?
 ├─ Yes → One-time or recurring?
 │        ├─ One-time files → AzCopy (scripts) / Storage Mover (managed file share migration)
 │        ├─ Ongoing file server → Azure File Sync
 │        ├─ Pipelines, transformation, databases → Data Factory / DMS
 │        └─ Account to account ongoing → Object replication
 └─ No  → Data Box (then online sync of the delta)
```

The common hybrid: Data Box for the bulk, then AzCopy sync or Storage Mover for changes made while the device was in transit, then cutover.

## How it works on Azure

### AzCopy examples

```bash
azcopy login --tenant-id <tenant>                       # Entra auth; or: azcopy login --identity

# Upload a directory recursively to Blob
azcopy copy "./dataset" "https://$SA.blob.core.windows.net/raw/" --recursive --put-md5

# Incremental sync of local folder to a container
azcopy sync "./dataset" "https://$SA.blob.core.windows.net/raw/dataset" --recursive --delete-destination=false

# Server-to-server between accounts (no data through your machine)
azcopy copy "https://src.blob.core.windows.net/raw/" "https://dst.blob.core.windows.net/raw/" --recursive

# From Amazon S3 (needs AWS credentials in environment variables)
azcopy copy "https://s3.amazonaws.com/my-bucket/data/" "https://$SA.blob.core.windows.net/raw/s3-data/" --recursive

# Resume a failed job
azcopy jobs list
azcopy jobs resume <job-id>
```

### Storage Mover

```text
Storage Mover resource (regional)
 ├─ Agents: VMs deployed on-premises near the source, registered with Azure Arc
 ├─ Endpoints: source (NFS/SMB share, S3 bucket) and target (Blob container, Azure file share)
 └─ Projects → Job definitions → Job runs (monitored, repeatable for deltas)
```

It suits migration programmes with many shares, where you want central management, repeatable runs and reporting rather than ad hoc scripts.

### Network considerations

- Large transfers over ExpressRoute or VPN use private endpoints; check gateway throughput limits.
- **Ingress to Azure is free**; **egress** out of Azure regions and between regions is charged. Plan for egress if data moves out again.
- Throttle transfers (`--cap-mbps`) during business hours on shared links.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Online or offline | Compute the transfer time; offline when it exceeds about two weeks or bandwidth is shared and limited |
| AzCopy or Storage Mover | AzCopy for engineers and scripts; Storage Mover for managed migration of many file shares with reporting |
| Cutover | Bulk copy, repeated delta syncs, final short freeze and last sync, then switch |
| Integrity | Use MD5 checks and compare counts and sizes before cutover |
| Permissions | Preserve SMB ACLs when moving to Azure Files; map identities for NFS |
| Cost | Ingress free; consider transaction costs for millions of small files; egress if moving out |

## Hands-on

```bash
# 1. Install AzCopy and log in
azcopy --version
azcopy login

# 2. Create 2,000 small files and upload them; note the throughput summary
mkdir -p data && for i in $(seq 1 2000); do head -c 10240 /dev/urandom > data/f$i.bin; done
azcopy copy "./data" "https://$SA.blob.core.windows.net/raw/" --recursive --put-md5

# 3. Change 10 files and sync; only changes move
for i in $(seq 1 10); do head -c 10240 /dev/urandom > data/f$i.bin; done
azcopy sync "./data" "https://$SA.blob.core.windows.net/raw/data" --recursive

# 4. Copy server-to-server into a second account
azcopy copy "https://$SA.blob.core.windows.net/raw/" "https://$SA2.blob.core.windows.net/raw/" --recursive
```

**Estimation exercise:** a customer has 180 TB of file data, a 500 Mbps internet link that can be used at 60% outside business hours (14 hours a day), and a six-week deadline. Compute the online transfer time and decide between online, Data Box, or a hybrid plan.

## Security, cost and reliability notes

- **Security:** use Entra auth (`azcopy login`) or short-lived user delegation SAS; never long-lived account SAS in scripts.
- **Security:** Data Box devices are encrypted and wiped, but the unlock password is a secret; handle chain of custody.
- **Cost:** many tiny files cost more in transactions than in capacity; consider packaging (tar or zip) for archives.
- **Cost:** copying into archive tier directly avoids hot-tier charges for cold data.
- **Reliability:** AzCopy jobs are resumable; log and verify. Keep the source read-only during final sync.

## Common mistakes

- Starting a 200 TB online copy over a 100 Mbps link.
- Downloading to a laptop and re-uploading instead of server-to-server copies.
- Forgetting the delta after a Data Box transfer.
- Not preserving file permissions when moving file servers.
- Overlooking egress costs when data has to come back out.
- Saturating a shared ExpressRoute circuit during business hours.

## Check yourself

1. How long does 10 TB take over 1 Gbps at 80% utilisation?
2. When should you choose Data Box over online transfer?
3. What does `azcopy sync` do differently from `azcopy copy`?
4. What is a server-to-server copy, and why is it faster?
5. What does Azure Storage Mover add over AzCopy?
6. Is data transfer into Azure charged?

<details>
<summary>Answers</summary>

1. About 1.2 days.
2. When the online transfer would take longer than about one to two weeks, or bandwidth is limited or shared.
3. It compares source and destination and transfers only new or changed files, optionally deleting extras.
4. Azure copies data directly between storage endpoints; the client only orchestrates, so data does not traverse the client's network.
5. Managed, centrally monitored migration projects with agents near the source, repeatable job definitions and reporting, suited to many shares.
6. No, ingress is free; egress out of Azure and between regions is charged.

</details>

## References

- [Choose an Azure solution for data transfer](https://learn.microsoft.com/azure/storage/common/storage-choose-data-transfer-solution)
- [Get started with AzCopy](https://learn.microsoft.com/azure/storage/common/storage-use-azcopy-v10)
- [Optimise AzCopy performance](https://learn.microsoft.com/azure/storage/common/storage-use-azcopy-optimize)
- [Azure Storage Explorer](https://learn.microsoft.com/azure/storage/storage-explorer/vs-azure-tools-storage-manage-with-storage-explorer)
- [Azure Storage Mover](https://learn.microsoft.com/azure/storage-mover/service-overview)
- [Azure Data Box documentation](https://learn.microsoft.com/azure/databox/)
- [Bandwidth pricing](https://azure.microsoft.com/pricing/details/bandwidth/)
