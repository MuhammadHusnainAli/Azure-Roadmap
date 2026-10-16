# High Performance Computing

> Topic: 06 · Compute · Tracks: E – A ○ C ○ · Status checked: 2026-10

## Why it matters

High performance computing is the simulation and modelling end of compute: crash tests, weather, drug discovery, seismic imaging, chip design, financial risk. It is a specialist field, and most readers of this roadmap will not build HPC clusters. It still earns a file for two reasons. Clients in engineering, energy, life sciences and finance will raise it. And large-scale AI training uses the same building blocks, so the vocabulary (InfiniBand, parallel file systems, schedulers) explains how big GPU clusters actually work.

## Key concepts

### What makes a workload HPC

| Type | Characteristics | Examples |
|---|---|---|
| **Embarrassingly parallel** (loosely coupled) | Many independent tasks; nodes do not talk to each other | Monte Carlo risk, rendering, parameter sweeps, genomics pipelines |
| **Tightly coupled** | One job spans many nodes that exchange data constantly through **MPI**; performance depends on network latency | Computational fluid dynamics, finite element analysis, weather and climate, molecular dynamics |
| **Data-intensive** | Bounded by storage throughput | Seismic processing, genomics, large-scale training data loading |
| **Memory-intensive** | Huge memory per core | Electronic design automation, some structural analysis |

Loosely coupled work runs well on ordinary VMs with Batch (file 08). **Tightly coupled** work is what needs specialised HPC infrastructure.

### HPC virtual machine families

| Family | Optimised for | Notes |
|---|---|---|
| **HB** | Memory-bandwidth-bound workloads: CFD, weather, explicit finite element analysis | AMD EPYC processors with large caches or high-bandwidth memory; InfiniBand |
| **HC** | Compute-bound workloads with high core frequency: implicit finite element analysis, molecular dynamics | Intel Xeon; InfiniBand |
| **HX** | Very high memory per core: electronic design automation, large structural models | AMD EPYC; InfiniBand |
| **ND** (GPU) | Multi-node GPU training and simulation | InfiniBand on RDMA-capable sizes |
| **NC, NV** (GPU) | GPU-accelerated solvers, visualisation | |
| **M, E** | Large shared-memory jobs | No InfiniBand |

Distinctive features of H-series:

- **InfiniBand** interconnect.
- **No hyperthreading** by default and predictable performance; many are effectively single-tenant per host.
- **Constrained-core variants** that expose fewer cores with the same memory and bandwidth, to reduce per-core software licence cost.

HPC sizes are available in a limited set of regions and need quota requests. Older generations are retired on a schedule, so target current ones.

### InfiniBand and RDMA

Ordinary Ethernet networking passes through the operating system's network stack, adding latency. **InfiniBand** with **remote direct memory access (RDMA)** lets one node read and write another's memory directly, bypassing the kernel, for latency of a few microseconds and very high bandwidth.

- Required for tightly coupled MPI jobs to scale beyond a handful of nodes.
- Available on sizes with `r` in the name, and on H-series.
- Nodes must be in the **same scale set or availability set** (the same InfiniBand partition) to communicate over it.
- Needs matching drivers and an MPI library (Intel MPI, Open MPI, MVAPICH, HPC-X). Marketplace HPC images include them.
- The same mechanism is what makes **multi-node GPU training** efficient.

### Schedulers and orchestration

HPC users submit jobs to a **scheduler** that queues them and allocates nodes.

| Option | Description |
|---|---|
| **Azure CycleCloud** | Microsoft's tool for deploying and autoscaling clusters with traditional schedulers: **Slurm**, PBS Professional, IBM Spectrum LSF, Grid Engine, HTCondor. Users keep their existing job scripts and workflows; the cluster grows and shrinks in Azure automatically |
| **Azure CycleCloud Workspace for Slurm** | A packaged, ready-made Slurm environment from the Marketplace, with a head node, autoscaling partitions (including GPU), storage integration and Open OnDemand portal |
| **Azure Batch** | Cloud-native scheduling without a traditional scheduler, including multi-instance MPI tasks |
| **AKS** | Container-based HPC and AI training with operators and queueing systems |
| **Azure Machine Learning** | Distributed training jobs with managed clusters |
| **Partner and domain platforms** | Vendor-managed simulation platforms and SaaS built on Azure |

**Choose CycleCloud** when users know and want Slurm or PBS, when migrating an on-premises cluster, or for hybrid bursting. **Choose Batch** for cloud-native pipelines without a scheduler dependency.

### HPC storage

Parallel jobs need storage that many nodes can read and write at once, fast.

| Service | Type | Use |
|---|---|---|
| **Azure Managed Lustre** | Managed parallel file system | High-throughput shared scratch for simulation and AI training. Integrates with Blob Storage: import data from a container, export results back, and delete the file system between campaigns to save cost |
| **Azure NetApp Files** | Enterprise NFS and SMB | Home directories, application binaries, mixed workloads, low latency |
| **Azure Files** (premium, NFS) | Managed file shares | Smaller clusters, shared configuration |
| **Blob Storage / Data Lake** | Object storage | Durable, cheap long-term data; staging for Lustre; BlobFuse for direct mounts |
| **Local NVMe on nodes** | Ephemeral scratch | Fastest per-node temporary space |
| **Self-managed parallel file systems** (BeeGFS, and others) | Software on VMs | Specific requirements |

Azure HPC Cache has been retired; Managed Lustre and NetApp Files cover its use cases.

A typical tiering: durable data in Blob, hydrated into Managed Lustre for the run, results written back to Blob, file system removed.

### Placement and performance

- **Proximity placement groups** and single-zone placement keep nodes physically close for low latency. HPC clusters usually trade zone redundancy for performance; resilience comes from checkpointing and re-running.
- **Accelerated networking** for the Ethernet side.
- **Process pinning and NUMA awareness** matter on large-core-count sizes.
- **Images:** use the marketplace HPC images (AlmaLinux or Ubuntu based) with drivers and MPI preinstalled, or build your own from them.

### Hybrid and burst

Many organisations keep an on-premises cluster and **burst** to Azure for peaks or for hardware they do not own. Requirements: ExpressRoute for data and licence-server connectivity, identity integration, a data movement plan (large datasets take time), and a scheduler configuration (CycleCloud supports this) that routes overflow jobs to cloud partitions.

### Licensing

Commercial solvers (Ansys, Siemens, Dassault, Altair, Synopsys, Cadence and others) are frequently licensed per core, per job or through tokens, and often cost more than the compute. Consequences:

- Choose sizes for **performance per licensed core**, not lowest price per core. Constrained-core sizes exist for this.
- Plan **licence server** placement and connectivity.
- Check the vendor's cloud licensing terms and any pay-per-use options.

## How it works on Azure

### Reference HPC architecture

```text
Users ── SSH / web portal ──► Login node
                                   │ job submission
                         ┌─────────▼──────────┐
                         │ Scheduler (Slurm)  │◄── Azure CycleCloud (autoscale)
                         └─────────┬──────────┘
          ┌────────────────────────┼─────────────────────────┐
   HB / HC partition          GPU (ND) partition         High-throughput partition
   InfiniBand, PPG            InfiniBand                 D/F-series, Spot
          └────────────────────────┼─────────────────────────┘
                 Azure Managed Lustre (scratch)  ·  Azure NetApp Files (home, apps)
                              Blob Storage (durable data, archive)
        Private VNet · ExpressRoute to on-premises (data, licence servers) · Entra ID / AD
```

Nodes exist only while jobs need them. The scheduler's partitions map to scale sets that scale to zero.

### HPC and AI convergence

| HPC concept | AI training equivalent |
|---|---|
| MPI over InfiniBand | Distributed training frameworks communicating over InfiniBand between GPU nodes |
| Parallel file system | High-throughput dataset and checkpoint storage |
| Scheduler and queue | The same (Slurm is widely used for training), or Kubernetes with a queueing system |
| Checkpoint and restart | Checkpointing for long training runs and Spot |
| Proximity placement | Single-zone, tightly placed GPU clusters |
| Performance per licensed core | Tokens or samples per GPU-hour |

Organisations doing serious model training typically run either a Slurm cluster through CycleCloud, AKS with a training operator and queueing, or Azure Machine Learning. Simulation and AI are also combining directly, with machine-learned surrogate models accelerating traditional solvers.

For the great majority of AI application work, none of this is needed. It matters when a client says they want to train or heavily fine-tune large models themselves.

### Cost levers

| Lever | Notes |
|---|---|
| **Scale to zero** | Clusters should have no compute nodes when the queue is empty |
| **Spot** | For loosely coupled and checkpointed work. Risky for long tightly coupled jobs without checkpointing |
| **Reservations / savings plans** | For a steady baseline of always-busy nodes |
| **Right hardware** | The fastest suitable size often finishes sooner and costs less overall, especially when licences are per core-hour |
| **Storage lifecycle** | Delete Lustre file systems between campaigns; tier Blob data to cool and archive |
| **Job efficiency** | Scaling studies to find where adding nodes stops helping |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Batch or CycleCloud | CycleCloud with Slurm for scheduler-based workflows and migrations. Batch for cloud-native parallel pipelines |
| H-series or general-purpose | H-series with InfiniBand for tightly coupled MPI. General-purpose or compute-optimised for loosely coupled |
| Zone redundancy | Usually not for the compute tier. Protect data and make jobs restartable |
| Storage | Managed Lustre for throughput scratch, NetApp Files for shared home and applications, Blob for durable data |
| Build or buy | Consider vendor-managed simulation platforms before building a cluster for a single application |
| Hybrid burst or all-in cloud | Burst to start and for peaks. Full migration when the on-premises refresh cycle comes up |
| Region | Where the required HPC sizes and capacity exist, with data gravity in mind |

## Hands-on

Exploration only; HPC sizes are quota-restricted and costly.

```bash
LOC=swedencentral     # try also: westeurope, eastus, southcentralus

# 1. Which HPC and InfiniBand-capable sizes are in the region?
az vm list-skus -l $LOC --resource-type virtualMachines \
  --query "[?starts_with(name,'Standard_H')].{Size:name, vCPU:capabilities[?name=='vCPUs'].value|[0], MemGB:capabilities[?name=='MemoryGB'].value|[0], RDMA:capabilities[?name=='RdmaEnabled'].value|[0], Restricted:restrictions[0].reasonCode}" -o table

az vm list-skus -l $LOC --resource-type virtualMachines \
  --query "[?capabilities[?name=='RdmaEnabled' && value=='True']].name" -o tsv | sort

# 2. Your quota for those families
az vm list-usage -l $LOC --query "[?contains(name.value,'HB') || contains(name.value,'HC') || contains(name.value,'HX')].{Family:name.localizedValue, Used:currentValue, Limit:limit}" -o table

# 3. HPC marketplace images
az vm image list --publisher microsoft-dsvm --offer ubuntu-hpc --all -l $LOC --query "[-3:].urn" -o tsv
az vm image list --publisher almalinux --offer almalinux-hpc --all -l $LOC --query "[-3:].urn" -o tsv
```

In the portal, open the Marketplace entry for **Azure CycleCloud Workspace for Slurm** and read through the deployment wizard: partitions, node sizes, storage choices and networking. Cancel before creating.

Design exercise: an engineering firm runs CFD jobs of 500 to 2,000 cores each, about 30 a week, on a five-year-old on-premises cluster, with a solver licensed per core. Propose an Azure design: VM family, scheduler, storage, connectivity, licence handling, and three cost controls. State what you would measure in a pilot before committing.

<details>
<summary>Notes</summary>

HB-series with InfiniBand in a single zone with proximity placement; CycleCloud with Slurm so existing job scripts carry over; Managed Lustre for scratch (created per campaign), NetApp Files for home and application binaries, Blob for results; ExpressRoute for data and the licence server, or a cloud-hosted licence server. Cost controls: scale to zero, a reservation for the baseline node count if the queue is steadily busy, constrained-core sizes to reduce licence spend, and storage lifecycle. In the pilot: time to solution and scaling efficiency per job size, cost per job including licences, data transfer times, and queue wait.

</details>

## Security, cost and reliability notes

- **Security:** clusters are multi-user Linux systems. Integrate with Entra ID or AD, restrict login nodes, keep the cluster in a private network, and separate projects where data sensitivity requires it.
- **Security:** much HPC data is valuable intellectual property or export-controlled. Apply the same network isolation, encryption and access controls as any sensitive workload, and check regional and sovereignty constraints.
- **Cost:** HPC nodes are expensive and idle clusters are pure waste. Enforce scale-to-zero and budgets; give users visibility of job cost.
- **Cost:** software licences often dominate. Optimise for them.
- **Reliability:** tightly coupled jobs fail entirely if one node fails. Checkpoint, and resubmit automatically.
- **Reliability:** capacity for specific HPC sizes in a region is finite. Agree capacity for large planned campaigns in advance.

## Common mistakes

- Running tightly coupled MPI jobs on sizes without InfiniBand and seeing no scaling.
- Nodes in different scale sets expecting to communicate over InfiniBand.
- General-purpose storage as the scratch file system for a parallel job.
- Optimising for cheapest VM per hour when licences are charged per core.
- A cluster that never scales down.
- Spot for long, uncheckpointed, tightly coupled jobs.
- Building a general cluster for one application that a vendor offers as a managed service.
- Assuming HPC sizes and quota are available in the nearest region.
- Leaving a Managed Lustre file system provisioned between campaigns.
- Underestimating how long it takes to move terabytes of data.

## Check yourself

1. What distinguishes tightly coupled from embarrassingly parallel workloads, and why does it matter for infrastructure?
2. What do the HB, HC and HX families each target?
3. What is RDMA over InfiniBand, and what constraint applies to nodes using it?
4. When would you use Azure CycleCloud instead of Azure Batch?
5. Which storage services suit parallel scratch, shared home directories and durable data?
6. Why might the most expensive VM per hour be the cheapest choice for a licensed solver?
7. How does HPC infrastructure relate to large-scale AI training?

<details>
<summary>Answers</summary>

1. Tightly coupled jobs span nodes that communicate constantly through MPI, so they need a very low-latency interconnect. Embarrassingly parallel tasks are independent and run on ordinary VMs.
2. HB: memory-bandwidth-bound workloads such as CFD and weather. HC: compute-bound, high-frequency workloads. HX: very high memory per core, such as electronic design automation.
3. Direct memory-to-memory communication between nodes that bypasses the operating system network stack, giving microsecond latency. Nodes must be in the same scale set or availability set to share the InfiniBand partition.
4. When users need a traditional scheduler such as Slurm or PBS, when migrating an existing cluster, or for hybrid bursting.
5. Azure Managed Lustre for parallel scratch, Azure NetApp Files for home directories and applications, Blob Storage for durable data.
6. If it finishes the job faster per licensed core, total licence cost and time to solution fall by more than the extra compute cost.
7. Distributed training uses the same building blocks: InfiniBand between GPU nodes, parallel file systems, schedulers and checkpointing.

</details>

## References

- [High-performance computing on Azure](https://learn.microsoft.com/azure/architecture/topics/high-performance-computing)
- [HPC VM sizes](https://learn.microsoft.com/azure/virtual-machines/sizes/overview#high-performance-compute)
- [Enable InfiniBand](https://learn.microsoft.com/azure/virtual-machines/extensions/enable-infiniband)
- [What is Azure CycleCloud?](https://learn.microsoft.com/azure/cyclecloud/overview)
- [Azure CycleCloud Workspace for Slurm](https://learn.microsoft.com/azure/cyclecloud/overview-ccws)
- [What is Azure Managed Lustre?](https://learn.microsoft.com/azure/azure-managed-lustre/amlfs-overview)
- [What is Azure NetApp Files?](https://learn.microsoft.com/azure/azure-netapp-files/azure-netapp-files-introduction)
- [Set up Message Passing Interface for HPC](https://learn.microsoft.com/azure/virtual-machines/setup-mpi)
- [Azure HPC landing zone accelerator](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/azure-hpc/)
- [Run a Linux VM cluster with Azure Batch and MPI](https://learn.microsoft.com/azure/batch/batch-mpi)
