# Azure Managed Lustre

## Why it matters

Training a large model or running a big HPC simulation means hundreds of GPUs or CPU cores reading the same data at once and writing large checkpoints. Object storage is too slow per client for that, and ordinary file shares do not scale to that concurrency. **Azure Managed Lustre** provides a managed parallel file system that delivers very high aggregate throughput and low latency, so expensive compute is not left idle waiting for I/O. It is a niche service, but when it fits, it saves far more in GPU time than it costs.

## Key concepts

### What a parallel file system does

```text
Clients (hundreds of GPU/CPU nodes, Lustre client installed)
   │  each client talks to many storage servers in parallel
   ▼
Metadata servers (MDS)  ── file names, directories, permissions
Object storage servers (OSS) ── file data striped across many targets
```

Files are **striped** across many storage targets, so one large file can be read by many clients, and many files in parallel, with aggregate throughput far above a single NAS head.

### Azure Managed Lustre (AMLFS)

| Property | Detail |
|---|---|
| Model | Fully managed Lustre file system deployed into your VNet |
| Performance tiers | Throughput per provisioned TiB (SKUs such as 40, 125, 250 and 500 MB/s per TiB); higher tiers cost more per TiB |
| Capacity | From several TiB to multiple PiB, in tier-specific increments |
| Blob integration | **Hierarchical storage management (HSM)**: import from and export to a linked Blob container; files appear in Lustre and are loaded on access or in bulk |
| Clients | Linux VMs with the Lustre client, **AKS** through the Managed Lustre CSI driver, Azure CycleCloud and Slurm clusters, Azure Machine Learning compute |
| Durability | Data is durable within the file system; for long-term protection, export to Blob |
| Networking | Private only, in a dedicated subnet sized for the file system |
| Encryption | At rest with platform or customer-managed keys |

### Typical workflow for AI training

```text
1. Data prepared in Blob / Data Lake (system of record)
2. Create AMLFS sized for throughput; link the Blob container (HSM)
3. Import: file metadata appears immediately; data loads on first read or by bulk import
4. Training job on ND GPU cluster reads shards in parallel; writes checkpoints to Lustre
5. Export checkpoints and final model artefacts back to Blob
6. Delete or scale down the file system when the run ends
```

Lustre is often treated as a **high-speed scratch and staging layer**, with Blob as the durable source and destination.

### When to use it

| Good fit | Poor fit |
|---|---|
| Distributed training across many GPU nodes | Single-node fine-tuning (use local NVMe or Blob) |
| Frequent large checkpoint writes | Small team file shares (Azure Files) |
| HPC simulation, genomics, seismic, EDA, rendering | Low-throughput workloads |
| Datasets read repeatedly at high concurrency | Workloads needing SMB or Windows clients |
| Lift of on-premises Lustre or GPFS workloads | Long-lived archival storage |

## How it works on Azure

### Sizing

Required throughput drives size:

```text
Required capacity (TiB) = max( dataset + checkpoints + headroom ,  target throughput (MB/s) ÷ tier throughput per TiB )
```

Example: training needs 20 GB/s aggregate read. At a 250 MB/s per TiB tier, you need at least 80 TiB provisioned, even if the dataset is 30 TiB. A 500 MB/s per TiB tier needs 40 TiB. Compare the cost of both options.

### Creating and connecting (outline)

```bash
az amlfs create -g $RG -n amlfs-train -l $LOC \
  --sku AMLFS-Durable-Premium-250 --storage-capacity 16 --zones "[1]" \
  --filesystem-subnet $SUBNET_ID --maintenance-window "{dayOfWeek:saturday,timeOfDayUtc:'22:00'}"
# Optional: --hsm-settings to link a Blob container (with import and export paths)

# On a Linux client with the Lustre client package installed:
sudo mount -t lustre -o noatime,flock <mgs-ip>@tcp:/lustrefs /mnt/lustre
```

On AKS, install the Managed Lustre CSI driver and create a PersistentVolume pointing at the file system.

Check CLI extension names, SKUs and minimum capacities in the current documentation before using these commands; they have changed between releases.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Lustre or Blob directly | Blob (with BlobFuse2 caching or streaming data loaders) for modest scale; Lustre when GPUs are I/O-starved or checkpoints are slow |
| Lustre or local NVMe | Local NVMe for per-node caching and single-node jobs; Lustre for shared data across nodes |
| Tier choice | Higher throughput per TiB if the dataset is small relative to throughput needs; lower tier if capacity dominates |
| Lifetime | Create per project or training campaign and delete afterwards; keep data durable in Blob |
| Zone | Place in the same zone as the GPU cluster to minimise latency |

## Hands-on

Managed Lustre has large minimum sizes. Do the analysis unless you have budget.

1. Pick a training scenario: 64 GPUs, each consuming 400 MB/s of training data, and a 2 TiB checkpoint every hour that must be written in under five minutes.
2. Compute the aggregate read and write throughput needed.
3. Size Managed Lustre at two tiers and price both with the pricing calculator.
4. Compare with the cost of the GPUs idling for the extra time a slower storage option would cause. This is the real business case.
5. Read the AKS CSI driver tutorial and sketch the PersistentVolume and pod spec you would use.

## Security, cost and reliability notes

- **Security:** private networking only; restrict the subnet with NSGs to client subnets. Use root squash settings to limit client root privileges.
- **Cost:** you pay for provisioned capacity per hour at the tier price. Delete file systems between campaigns.
- **Cost:** the saving is in compute time. Measure GPU utilisation before and after.
- **Reliability:** treat Lustre as a working tier. Export results and checkpoints to Blob regularly; a lost file system should cost time, not data.

## Common mistakes

- Using Lustre for small workloads that Blob or Azure Files would serve.
- Sizing by capacity and missing throughput targets.
- Leaving the file system running after the training run ends.
- Keeping the only copy of checkpoints on Lustre.
- Placing the file system in a different zone or region from the compute.

## Check yourself

1. What makes a parallel file system faster than a NAS for many clients?
2. What does the HSM integration with Blob Storage provide?
3. How do you size Managed Lustre when throughput, not capacity, is the constraint?
4. How do AKS workloads use Managed Lustre?
5. What is the main economic argument for Lustre in AI training?

<details>
<summary>Answers</summary>

1. Files are striped across many storage servers, so clients read and write in parallel across them, giving very high aggregate throughput.
2. A linked Blob container from which data is imported (lazily or in bulk) and to which results are exported, keeping Blob as the durable system of record.
3. Divide the target aggregate throughput by the tier's throughput per TiB to get the minimum capacity, and take the larger of that and the data size.
4. Through the Azure Managed Lustre CSI driver, mounting the file system as a PersistentVolume.
5. It keeps expensive GPUs busy instead of waiting on I/O; the GPU time saved usually far exceeds the storage cost.

</details>

## References

- [Azure Managed Lustre overview](https://learn.microsoft.com/azure/azure-managed-lustre/amlfs-overview)
- [Managed Lustre prerequisites and sizing](https://learn.microsoft.com/azure/azure-managed-lustre/amlfs-prerequisites)
- [Blob integration (HSM)](https://learn.microsoft.com/azure/azure-managed-lustre/blob-integration)
- [Use the Managed Lustre CSI driver with AKS](https://learn.microsoft.com/azure/azure-managed-lustre/use-csi-driver-kubernetes)
- [Storage for HPC on Azure](https://learn.microsoft.com/azure/architecture/topics/high-performance-computing)
