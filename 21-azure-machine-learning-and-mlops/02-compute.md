# Compute

## Why it matters

Compute is where Azure ML spends your money and where most operational headaches start: GPU quota shortages, clusters that never scale down, notebooks left running over a weekend, jobs queued behind someone else's experiment. Azure ML offers several compute types, from fully managed **serverless compute** to your own **Kubernetes** clusters. Choosing the right one per stage (explore, train, infer) controls cost, isolation and time to results.

## Key concepts

### Compute types

| Type | What it is | Typical use |
|---|---|---|
| **Compute instance** | A managed, single-user development VM (Jupyter, VS Code, terminal) | Notebooks, debugging, small experiments |
| **Compute cluster** | Managed single- or multi-node cluster that **autoscales** per job | Repeatable training, pipelines, shared team capacity |
| **Serverless compute** | Azure ML provisions compute per job; no cluster to create or manage; managed network isolation | Most training jobs, AutoML, pipelines, model catalogue fine-tuning |
| **Attached Kubernetes** (AKS or Arc-enabled) | Your Kubernetes cluster with the Azure ML extension | Training and inference on existing clusters, on-premises/edge, custom networking |
| **Attached compute** | Remote VMs, Spark pools (Preview), Databricks (SDK local mode), HDInsight, Batch | Integration with existing estates |
| **Managed online/batch endpoints** | Managed inference compute | Model serving (file 08) |

### Capability matrix (training targets)

| Target | AutoML | Pipelines | Designer |
|---|---|---|---|
| Compute cluster | Yes | Yes | Yes |
| Serverless compute | Yes | Yes | Yes |
| Compute instance | Yes (SDK) | Yes | Yes |
| Kubernetes | — | Yes | Yes |
| Remote VM | Yes | Yes | — |

### Serverless compute

- Omit `compute` in a job (or set `compute: serverless`) and Azure ML provisions what the job needs.
- Choose size with `resources.instance_type` and nodes with `resources.instance_count`.
- Supports all job types, pipelines and fine-tuning from the model catalogue.
- Admins still control spend through **core quotas** (subscription and workspace) and **Azure Policy**.
- Requires at least the AzureML Data Scientist (or Contributor) role on the workspace.

### Cost controls

| Control | How |
|---|---|
| Cluster `min_instances: 0` | Scale to zero when idle |
| Idle scale-down time | Release nodes quickly after jobs |
| Compute instance **idle shutdown** and schedules | Stop forgotten notebooks (disk, IP and load balancer still bill when stopped) |
| **Low-priority** (spot) VMs on clusters | Cheaper, pre-emptible, for fault-tolerant jobs |
| Quotas per workspace | Prevent one team consuming all GPUs |
| Right-sizing | Microsoft's guidance for inference: start with about 150% of the model's RAM, profile, scale up before scaling out |

Retired GPU series (NC, NCv2, ND) must be replaced with current sizes.

## How it works on Azure

Python SDK v2, serverless job with a chosen GPU size:

```python
from azure.ai.ml import MLClient, command
from azure.ai.ml.entities import JobResourceConfiguration
from azure.identity import DefaultAzureCredential

ml = MLClient(DefaultAzureCredential(), "<sub>", "rg-ml", "mlw-churn-dev")
job = command(
    code="./src", command="python train.py --epochs 10",
    environment="azureml://registries/azureml/environments/sklearn-1.5/labels/latest",
    display_name="churn-train-serverless")
job.resources = JobResourceConfiguration(instance_type="Standard_E4s_v3", instance_count=1)
ml.jobs.create_or_update(job)              # no compute specified → serverless
```

Compute cluster with scale-to-zero (CLI YAML):

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/amlCompute.schema.json
name: cpu-cluster
type: amlcompute
size: Standard_DS3_v2
min_instances: 0
max_instances: 4
idle_time_before_scale_down: 300
tier: low_priority
```

`az ml compute create -f cluster.yml`

Compute instance with idle shutdown:

```bash
az ml compute create -n ci-alex --type computeinstance --size Standard_DS3_v2 \
  --idle-time-before-shutdown-minutes 60
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Serverless vs cluster** | Serverless by default (no lifecycle management, per-job sizing). Clusters when you need a pre-warmed shared pool, specific networking, or reserved capacity behaviour. |
| **GPU strategy** | Request quota early; use spot for interruptible training with checkpoints; dedicated for deadlines. |
| **Kubernetes** | Attach when you already run AKS/Arc clusters, need on-prem/edge, or want to share GPUs across platforms; adds operational responsibility. |
| **Dev environment** | Compute instances for data scientists with idle shutdown; or local VS Code connected to the workspace. |
| **Isolation** | Managed VNet for serverless and managed compute; private endpoints; no public IP options. |

## Hands-on

1. Submit the same training job three ways: compute instance, `cpu-cluster` (min 0), and serverless. Compare queue time, run time and cost.
2. Create a low-priority cluster and run a job with checkpointing; simulate pre-emption by cancelling and resuming.
3. Set a workspace-level core quota and observe a job that exceeds it.
4. Enable idle shutdown on your compute instance and a weekday start/stop schedule.

## Security, cost and reliability notes

- **Identity**: assign managed identities to compute for data access; avoid storing credentials in notebooks.
- **No public IP** options and managed VNet for regulated workloads.
- **Cost leaks**: running instances, clusters with `min_instances > 0`, and endpoints left deployed. Use budgets and alerts ([Topic 29](../Roadmap.md)).
- **Quota and capacity**: GPU availability varies by region; plan alternatives.
- **Pre-emption**: low-priority nodes can be evicted; checkpoint training.

## Common mistakes

| Mistake | Fix |
|---|---|
| Clusters with a non-zero minimum | `min_instances: 0` or serverless |
| Training large jobs on compute instances | Use clusters or serverless |
| Spot GPUs without checkpointing | Save checkpoints regularly |
| One giant GPU cluster shared without quotas | Workspace quotas and separate clusters |
| Still using retired NC/ND series | Recreate with current sizes |

## Check yourself

1. Compare compute instances, compute clusters and serverless compute.
2. How do you make a job use serverless compute?
3. How do admins control serverless spend?
4. What keeps billing when a compute instance is stopped?
5. When would you attach Kubernetes compute?
6. What is Microsoft's sizing tip for inference compute?

<details>
<summary>Answers</summary>

1. Instance: single-user dev VM; cluster: autoscaling shared training cluster; serverless: per-job managed compute with no lifecycle management.
2. Omit `compute` (or set it to serverless) and optionally set `instance_type` and `instance_count`.
3. Core quotas at subscription and workspace level plus Azure Policy.
4. Disk, public IP and standard load balancer.
5. To reuse existing AKS or Arc-enabled clusters, run on-premises/edge, or meet custom networking needs.
6. Start with a machine that has about 150% of the RAM the model needs, profile, scale up before scaling out.

</details>

## References

- [What are compute targets?](https://learn.microsoft.com/azure/machine-learning/concept-compute-target)
- [Train with serverless compute](https://learn.microsoft.com/azure/machine-learning/how-to-use-serverless-compute)
- [Create a compute cluster](https://learn.microsoft.com/azure/machine-learning/how-to-create-attach-compute-cluster)
- [Introduction to Kubernetes compute](https://learn.microsoft.com/azure/machine-learning/how-to-attach-kubernetes-anywhere)
