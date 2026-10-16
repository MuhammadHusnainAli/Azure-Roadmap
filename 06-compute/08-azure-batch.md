# Azure Batch

> Topic: 06 · Compute · Tracks: E ○ A ○ C – · Status checked: 2026-10

## Why it matters

Some work is not a service that answers requests. It is a large pile of independent jobs: render ten thousand frames, score a million documents, run a Monte Carlo simulation, transcode a video archive, embed an entire corpus. Azure Batch runs that kind of work across hundreds or thousands of VMs without you building a scheduler. It is the VM-level answer to large-scale batch compute, and a useful comparison point for the container-based alternatives most AI teams reach for first.

## Key concepts

### What it is

A managed job-scheduling service. You describe pools of compute and the tasks to run; Batch provisions the VMs, distributes tasks, retries failures, scales the pool and reports status.

**There is no charge for Batch itself.** You pay for the VMs, storage and networking it uses.

### Core objects

```text
Batch account
├── Pool          a set of identical VMs (nodes)
├── Job           a container for tasks, bound to a pool
│   └── Task      one unit of work: a command line run on a node
├── Job schedule  runs jobs on a recurrence
└── Applications / certificates / identities
```

| Object | Detail |
|---|---|
| **Batch account** | The management endpoint for a region. Linked to a storage account ("auto-storage") for application packages and task output |
| **Pool** | VM size, image, node count or autoscale formula, dedicated and Spot node targets, start task, networking, identity, container settings |
| **Node** | One VM in a pool. Runs one or more tasks concurrently (**task slots per node**) |
| **Job** | Groups tasks, sets priority, constraints (maximum wall-clock time, retry count) and what happens when tasks complete or fail |
| **Task** | A command line, with resource files to download first, environment variables, output files to upload, exit-code handling, retry limits and constraints |

### Pool allocation modes

| Mode | VMs are created in | Use when |
|---|---|---|
| **Batch service** (default) | Microsoft-managed subscriptions, with core quota on the Batch account | Simplicity |
| **User subscription** | **Your** subscription, visible as ordinary resources and using your VM quota | You need your reservations and savings plans to apply, custom networking and policy control, or specific compliance requirements |

User subscription mode is the usual choice in governed enterprise environments.

### Node types

| Type | Notes |
|---|---|
| **Dedicated** | Standard VMs, reserved for you |
| **Spot** | Deeply discounted, can be pre-empted. Batch re-queues interrupted tasks automatically. Well suited to Batch workloads |

A pool can mix both: a dedicated base plus Spot for bulk throughput.

### Task types

| Type | Purpose |
|---|---|
| **Standard task** | An independent command. The common case ("embarrassingly parallel") |
| **Start task** | Runs on each node as it joins the pool: install software, mount storage, warm caches. Keep it short; use a custom image for heavy setup |
| **Job preparation / release tasks** | Run once per node before the job's first task and after its last |
| **Job manager task** | A controller that creates and monitors the job's other tasks |
| **Multi-instance task** | Runs across several nodes at once for **MPI** workloads |
| **Task dependencies** | A task runs only after others succeed, giving simple DAGs |
| **Container tasks** | Tasks run in Docker-compatible containers, with images prefetched on the pool |

### Autoscaling

Pools resize according to an **autoscale formula** evaluated on an interval:

```text
// Scale to the number of pending tasks, capped at 100 nodes, all Spot; remove nodes only when their task finishes
$tasks = max(0, $PendingTasks.GetSample(TimeInterval_Minute * 5));
$TargetLowPriorityNodes = min(100, $tasks);
$TargetDedicatedNodes   = 0;
$NodeDeallocationOption = taskcompletion;
```

Variables include pending, active and running task counts, and CPU and memory samples. `taskcompletion` prevents killing running work on scale-in. A pool scaled to zero costs nothing.

### Getting software and data to nodes

| Mechanism | For |
|---|---|
| **Custom image** from an Azure Compute Gallery | Large or slow-to-install dependencies, GPU drivers |
| **Container images** from a registry | Portable, versioned application environments. The preferred modern approach |
| **Application packages** | Versioned zip files deployed to nodes |
| **Resource files** | Per-task or per-pool files downloaded from Blob Storage |
| **Start task** | Light configuration |
| **Mounted storage** | Azure Files, Blob (BlobFuse), NFS, or a high-performance file system mounted on every node |
| **Output files** | Task outputs uploaded to Blob Storage on completion or failure |

### Networking and identity

- Pools can be placed in **your virtual network subnet**, without public IPs, using the **simplified node communication** model and an outbound path to the Batch service (or private endpoints for the Batch account).
- The Batch account supports **private endpoints** and disabling public access.
- Pools take **user-assigned managed identities**, so tasks reach Storage, Key Vault and registries without keys.
- Authenticate to the Batch service with **Microsoft Entra ID**; shared key authentication can be disabled.

### Tooling

Batch is driven by the REST API, SDKs (.NET, Python, Java, JavaScript), the Azure CLI, and Batch Explorer. Orchestrators that submit work to it include Azure Data Factory (custom activities), Azure Machine Learning in some configurations, and workflow engines such as Nextflow and Cromwell for genomics.

## How it works on Azure

### The standard pattern

```text
1. Upload inputs to Blob Storage
2. Client (or job manager) creates a job and one task per unit of work
3. Pool autoscales up from zero on pending tasks
4. Each node: start task → pulls container image → runs tasks → uploads outputs
5. Failed or pre-empted tasks are retried automatically
6. Job completes → pool scales back to zero
```

Design rules that make it work:

- **Idempotent tasks.** Any task may run more than once.
- **Right-sized tasks.** Minutes to an hour or two each. Thousands of second-long tasks pay too much scheduling overhead; a single day-long task loses everything on pre-emption.
- **Externalise state.** Inputs and outputs in storage, never only on the node.
- **Checkpoint** long tasks.
- **Stay within limits.** Check quotas for cores, pools, and active jobs; very large jobs need batching of task submissions.

### Batch for AI workloads

| Use | Notes |
|---|---|
| **Bulk embedding or document processing** on your own models | GPU or CPU pools, one task per shard of the corpus |
| **Batch inference** with a self-hosted model | GPU Spot nodes, container tasks |
| **Large-scale evaluation runs** or synthetic data generation | Parallel tasks calling a model endpoint, bounded by the endpoint's rate limits, not by compute |
| **Data preparation** before indexing | OCR, parsing, chunking at scale |
| **Hyperparameter sweeps and many small training runs** | Azure Machine Learning usually fits better, with tracking built in |

When the heavy lifting is done by a **managed model API**, compute is rarely the bottleneck: the model's rate limit is. In that case use the provider's **batch API** (for example Foundry batch deployments, which process large asynchronous jobs at a discount) instead of fanning out thousands of nodes that will just be throttled.

### Alternatives

| Service | Choose it when |
|---|---|
| **Azure Batch** | Very large scale on VMs, HPC and MPI, Windows workloads, specialised VM sizes, rendering, mixed dedicated and Spot with fine scheduling control |
| **Container Apps jobs** | Event-driven or scheduled containerised jobs at modest scale, scale to zero, minimal operations, optional serverless GPUs |
| **AKS (Jobs, KEDA, queue systems such as Kueue)** | You already run Kubernetes and want batch alongside services |
| **Azure Functions / Durable Functions** | Many small, short tasks orchestrated in code; fan-out and fan-in |
| **Azure Machine Learning** | ML training, batch endpoints and pipelines with experiment tracking |
| **Fabric / Databricks / Synapse Spark** | Data-parallel processing over large datasets |
| **Data Factory / Fabric pipelines** | Orchestrating the above and moving data |
| **Azure CycleCloud** | Traditional HPC schedulers (Slurm, PBS) and tightly coupled workloads (file 09) |
| **A model provider's batch API** | The work is model calls, not compute |

For a typical AI team processing documents in containers, Container Apps jobs or AKS are usually simpler. Batch earns its place at larger scale, with VM-specific needs, or for classic HPC-style workloads.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Batch or a container job service | Start with the simpler service. Move to Batch for scale, VM control or MPI |
| Pool allocation mode | User subscription for enterprise governance and to use existing commitments |
| Dedicated or Spot | Spot for most batch work, with a small dedicated base if a deadline matters |
| Custom image or containers | Containers for the application; a custom image only for host-level needs such as drivers |
| One pool or several | Separate pools for different VM sizes or software. Share a pool across jobs with job priorities where the environment is the same |
| Fixed or autoscaled pool | Autoscale to zero unless tasks arrive continuously |
| Task granularity | Coarse enough to amortise start-up, fine enough to retry cheaply |

## Hands-on

A conceptual walk-through; run it if you want to see the mechanics. Batch service allocation mode keeps it simple.

```bash
RG=rg-batch-lab; LOC=swedencentral; ACC=batchlab$RANDOM; ST=stbatchlab$RANDOM
az group create -n $RG -l $LOC -o none
az storage account create -g $RG -n $ST -l $LOC --sku Standard_LRS -o none
az batch account create -g $RG -n $ACC -l $LOC --storage-account $ST -o none
az batch account login -g $RG -n $ACC --shared-key-auth      # lab convenience; use Entra auth in real use

# A small pool: two Spot nodes
az batch pool create --id pool-lab --vm-size Standard_D2as_v5 \
  --image canonical:ubuntu-24_04-lts:server --node-agent-sku-id "batch.node.ubuntu 24.04" \
  --target-dedicated-nodes 0 --target-low-priority-nodes 2
az batch pool show --pool-id pool-lab --query "{state:allocationState, spot:currentLowPriorityNodes}"

# A job with twenty independent tasks
az batch job create --id job-lab --pool-id pool-lab
for i in $(seq 1 20); do
  az batch task create --job-id job-lab --task-id task$i \
    --command-line "/bin/bash -c 'echo task $i on \$(hostname); sleep 20'" -o none
done

# Watch progress, then read one task's output
az batch job task-counts show --job-id job-lab
az batch task list --job-id job-lab --query "[].{id:id, state:state, node:nodeInfo.nodeId, exit:executionInfo.exitCode}" -o table
az batch task file download --job-id job-lab --task-id task1 --file-path stdout.txt --destination ./stdout.txt && cat stdout.txt

# Switch the pool to autoscale on pending tasks
az batch pool autoscale enable --pool-id pool-lab --auto-scale-evaluation-interval PT5M \
  --auto-scale-formula '$TargetLowPriorityNodes = min(4, max(0, $PendingTasks.GetSample(1))); $NodeDeallocationOption = taskcompletion;'

az group delete -n $RG --yes --no-wait
```

Image and node agent SKU names must match a current supported pair; list them with `az batch pool supported-images list` and adjust. Notice that tasks landed on different nodes and that failed tasks would be retried.

Design exercise: you must embed 8 million documents using a self-hosted embedding model on GPUs within 12 hours. Sketch the pool (size, dedicated and Spot mix), task granularity, data flow, checkpointing and autoscale formula. Then say how the design changes if you used a managed embedding API instead.

## Security, cost and reliability notes

- **Security:** use Entra authentication and managed identities; disable shared key access. Keep pools in a VNet without public IPs and the account behind a private endpoint for sensitive work.
- **Security:** tasks run arbitrary commands. Run as a non-administrator user unless elevation is required, and control who can submit jobs.
- **Cost:** you pay for nodes whether or not tasks are running. Autoscale to zero and set a maximum.
- **Cost:** Spot nodes cut cost dramatically for retry-tolerant work.
- **Reliability:** design for retries and pre-emption. Set task and job constraints so stuck work does not run for ever.
- **Reliability:** start-task failures leave nodes unusable. Keep start tasks minimal and test them.

## Common mistakes

- Using Batch for a handful of small scheduled jobs that a container job would handle.
- Fixed-size pools left running idle.
- Tasks that are not idempotent.
- Heavy installs in the start task on every node.
- Results stored only on node disks.
- Fanning out compute for work that is really limited by a model endpoint's rate limit.
- Batch service allocation mode in an environment that needs its own policies, networking and reservations to apply.
- Shared-key authentication left enabled.
- No retry limits or timeouts.

## Check yourself

1. What does Azure Batch provide, and what do you pay for?
2. Define pool, job and task.
3. What is the difference between Batch service and user subscription pool allocation modes?
4. Why are Spot nodes a good fit for Batch?
5. What does an autoscale formula with `taskcompletion` achieve?
6. When would you choose Container Apps jobs over Batch?
7. When is a model provider's batch API better than fanning out compute?

<details>
<summary>Answers</summary>

1. Managed scheduling of large-scale parallel and HPC jobs across pools of VMs. The service is free; you pay for the VMs, storage and networking used.
2. A pool is a set of identical VMs. A job is a container of tasks bound to a pool. A task is a single command run on a node.
3. In Batch service mode VMs are created in Microsoft-managed subscriptions against Batch account quota. In user subscription mode they are created in your subscription, so your quota, reservations, policies and networking apply.
4. Batch work is usually interruptible and retryable, and Batch re-queues pre-empted tasks automatically, so the large discount comes with little downside.
5. The pool grows and shrinks with workload, and nodes are removed only after their running tasks finish.
6. For containerised, event-driven or scheduled jobs at modest scale where simplicity and scale-to-zero matter more than VM-level control.
7. When the work consists of model calls limited by the endpoint's rate limits, not by compute; the batch API handles large asynchronous volumes at a discount.

</details>

## References

- [What is Azure Batch?](https://learn.microsoft.com/azure/batch/batch-technical-overview)
- [Batch service workflow and resources](https://learn.microsoft.com/azure/batch/batch-service-workflow-features)
- [Nodes and pools in Azure Batch](https://learn.microsoft.com/azure/batch/nodes-and-pools)
- [Jobs and tasks in Azure Batch](https://learn.microsoft.com/azure/batch/jobs-and-tasks)
- [Create a formula to automatically scale compute nodes](https://learn.microsoft.com/azure/batch/batch-automatic-scaling)
- [Use Spot VMs with Batch workloads](https://learn.microsoft.com/azure/batch/batch-spot-vms)
- [Run container workloads on Azure Batch](https://learn.microsoft.com/azure/batch/batch-docker-container-workloads)
- [Azure Batch best practices](https://learn.microsoft.com/azure/batch/best-practices)
- [Jobs in Azure Container Apps](https://learn.microsoft.com/azure/container-apps/jobs)
- [Choose an Azure compute option for batch processing](https://learn.microsoft.com/azure/architecture/data-guide/technology-choices/batch-processing)
