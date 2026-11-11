# Training

## Why it matters

Training on a laptop doesn't scale, isn't reproducible and can't be audited. In Azure ML every training run is a **job**: code snapshot, environment, inputs, compute, metrics, outputs and lineage are recorded automatically. From there you can scale out to distributed GPU training or launch hyperparameter **sweeps** that run hundreds of trials with early termination. This is the core skill behind AI-300 (MLOps) and any serious predictive ML platform.

## Key concepts

### Ways to train

| Method | Use |
|---|---|
| **Command job** (`command()` / `type: command`) | Your script, environment and compute; the standard approach |
| **Sweep job** | Hyperparameter tuning over a command job |
| **Automated ML** | Algorithm and hyperparameter search for tabular, vision and NLP tasks (file 05) |
| **Pipeline job** | Multi-step workflows that include training (file 06) |
| **Designer** | Drag-and-drop pipelines for proofs of concept |

### Anatomy of a command job

| Part | Description |
|---|---|
| `code` | Local folder uploaded as a snapshot |
| `command` | What to run, with `${{inputs.x}}` and `${{outputs.y}}` bindings |
| `environment` | Curated (for example `azureml://registries/azureml/environments/sklearn-1.5/labels/latest`) or custom (Docker image or conda file), versioned |
| `inputs` / `outputs` | Data assets, URIs, literal values |
| `compute` | Cluster, serverless (omit), instance, Kubernetes |
| `resources` / `distribution` | Instance type and count; PyTorch/TensorFlow/MPI distribution settings |
| `experiment_name`, `tags` | Grouping and search |

What happens on submit: snapshot upload → environment image build or cache hit (ACR) → compute provisioning → data mount/download → run → MLflow metrics and outputs stored → lineage recorded.

### Environments

Curated environments cover common frameworks; custom environments are built from a Dockerfile or conda specification and cached in the workspace's ACR. Version them and reuse via registries so dev and prod use identical dependencies.

### Distributed training

| Approach | How | When |
|---|---|---|
| **Data parallelism** | Each node holds a model copy and trains on a data partition; gradients synchronised | Most cases; simplest |
| **Model parallelism** | Model split across nodes; each part on the same data | Models too large for one GPU's memory |

Azure ML integrates with **PyTorch** (`distribution: type: pytorch`, launches processes per node and GPU), **TensorFlow** and **MPI**. Use InfiniBand-enabled GPU SKUs for multi-node efficiency; checkpoint to outputs so you can resume.

### Hyperparameter sweeps

| Element | Options |
|---|---|
| **Search space** | `Choice`, `Uniform`, `LogUniform`, `Normal`, quantised variants (`QUniform`…) |
| **Sampling** | **Random** (incl. Sobol quasi-random; good default), **Grid** (only `Choice`; exhaustive), **Bayesian** (uses prior results; budget ≥ 20× number of hyperparameters; no early termination) |
| **Objective** | `primary_metric` (must exactly match the logged metric name) and `goal` (`maximize`/`minimize`) |
| **Early termination** | **Bandit** (slack vs best), **Median stopping** (below running median), **Truncation selection** (cancel lowest X%), or none |
| **Limits** | `max_total_trials` (1–1000), `max_concurrent_trials`, `timeout`, `trial_timeout` |

Microsoft's guidance: Median stopping with `evaluation_interval=1`, `delay_evaluation=5` is conservative (roughly 25–35% savings without losing promising runs); Bandit with small slack or Truncation with a larger percentage is more aggressive.

## How it works on Azure

Command job plus sweep (Python SDK v2):

```python
from azure.ai.ml import MLClient, command, Input
from azure.ai.ml.sweep import Choice, Uniform, MedianStoppingPolicy
from azure.identity import DefaultAzureCredential

ml = MLClient(DefaultAzureCredential(), "<sub>", "rg-ml", "mlw-churn-dev")

base = command(
    code="./src",
    command="python train.py --data ${{inputs.data}} --lr ${{inputs.lr}} --max_depth ${{inputs.max_depth}}",
    inputs={"data": Input(type="uri_folder", path="azureml:churn-training:3"),
            "lr": 0.1, "max_depth": 6},
    environment="azureml://registries/azureml/environments/sklearn-1.5/labels/latest",
    experiment_name="churn")

sweep = base(lr=Uniform(min_value=0.01, max_value=0.3),
             max_depth=Choice(values=[3, 5, 7, 9])).sweep(
    compute="cpu-cluster", sampling_algorithm="random",
    primary_metric="val_auc", goal="maximize")
sweep.early_termination = MedianStoppingPolicy(delay_evaluation=5, evaluation_interval=1)
sweep.set_limits(max_total_trials=40, max_concurrent_trials=4, timeout=7200)

returned = ml.jobs.create_or_update(sweep)
print(returned.studio_url)
```

`train.py` logs `val_auc` with MLflow (`mlflow.log_metric("val_auc", auc)`; file 07).

Distributed PyTorch (YAML excerpt):

```yaml
resources:
  instance_type: Standard_NC24ads_A100_v4
  instance_count: 2
distribution:
  type: pytorch
  process_count_per_instance: 1
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Curated vs custom environment** | Curated for speed; custom when you need pinned dependencies or system packages; register and version. |
| **Sampling algorithm** | Random first to find regions; Bayesian to refine with enough budget; Grid only for small discrete spaces. |
| **Early termination** | Use it for iterative training (epochs) with intermediate metrics; not applicable to Bayesian sampling. |
| **Distributed or not** | Distribute only when a single GPU/node is too slow or too small; communication overhead is real. |
| **Concurrency** | Higher concurrency finishes sooner but makes Bayesian sampling less effective. |
| **Reproducibility** | Pin environment and data versions; set seeds; record git commit (automatic with code snapshots). |

## Hands-on

1. Write `train.py` (scikit-learn gradient boosting on a churn dataset) that logs `val_auc` with MLflow and saves the model to `./outputs`.
2. Submit it as a command job on serverless compute.
3. Launch the random sweep above with median stopping; inspect the parallel coordinates chart in studio.
4. Re-run with Bayesian sampling (no early termination) and compare best metric and total compute time.
5. Register the best trial's model (file 07).

## Security, cost and reliability notes

- **Identity**: compute managed identity for data access; no secrets in code or environment variables.
- **Cost**: sweeps multiply compute; set `max_total_trials`, timeouts and early termination; use low-priority nodes with checkpointing.
- **Data**: training snapshots and logs may contain sensitive data; control workspace access.
- **Reliability**: checkpoint long jobs; retries for transient failures; environment image caching speeds reruns.

## Common mistakes

| Mistake | Fix |
|---|---|
| `primary_metric` name doesn't match the logged metric | Use the exact name |
| Bayesian sweep with tiny budget | ≥ 20× number of hyperparameters or use random |
| Unbounded sweeps | Set trial and time limits |
| Unpinned dependencies | Versioned environments |
| Distributed training on a small model | Single node is cheaper |

## Check yourself

1. What does a command job consist of?
2. What happens when you submit a job?
3. Compare data and model parallelism.
4. Compare random, grid and Bayesian sampling.
5. Name three early termination policies and which is conservative.
6. Which sweep limits can you set?

<details>
<summary>Answers</summary>

1. Code, command, environment, inputs/outputs, compute, resources/distribution and metadata (experiment, tags).
2. Snapshot upload, environment build or cache, compute provisioning, data access, run, metrics/outputs storage and lineage.
3. Data parallelism copies the model and splits data (simplest, most common); model parallelism splits the model across nodes (for very large models).
4. Random: broad, supports continuous values and early termination; grid: exhaustive over discrete choices; Bayesian: uses previous results, needs larger budget, no early termination.
5. Bandit, median stopping, truncation selection; median stopping with delay 5 and interval 1 is conservative.
6. `max_total_trials`, `max_concurrent_trials`, `timeout`, `trial_timeout`.

</details>

## References

- [Train models with Azure ML](https://learn.microsoft.com/azure/machine-learning/concept-train-machine-learning-model)
- [Hyperparameter tuning (sweep jobs)](https://learn.microsoft.com/azure/machine-learning/how-to-tune-hyperparameters)
- [Distributed training](https://learn.microsoft.com/azure/machine-learning/concept-distributed-training)
- [Manage environments](https://learn.microsoft.com/azure/machine-learning/how-to-manage-environments-v2)
