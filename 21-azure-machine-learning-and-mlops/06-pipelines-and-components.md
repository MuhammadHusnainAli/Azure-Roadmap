# Pipelines and Components

## Why it matters

A training notebook that someone runs by hand is a liability: steps get skipped, data versions drift, and nobody can retrain the model six months later. **Azure ML pipelines** turn the ML workflow (prepare data, train, evaluate, register) into a reproducible, schedulable graph of **components**, each owned and versioned independently. Pipelines reuse outputs from unchanged steps, run each step on the right compute, and record full lineage. They're the backbone of retraining and MLOps (file 09).

## Key concepts

### Components

A **component** is a self-contained, versioned step, like a function:

| Part | Content |
|---|---|
| **Metadata** | Name, display name, version, type, description |
| **Interface** | Typed inputs and outputs (data, numbers, strings, models) with defaults |
| **Command, code and environment** | What runs, where the code lives, which environment |

Benefits: well-defined interfaces, sharing and reuse across pipelines, workspaces and subscriptions (via **registries**), version control, and unit testability. Components can be written in any language runnable from a shell command; there are also built-in components (for example AutoML, feature retrieval, model evaluation from the catalogue).

### Pipelines

A **pipeline job** connects component outputs to inputs. Azure ML manages execution order from data dependencies, runs independent steps in parallel, and **reuses** cached results from steps whose inputs, code and environment haven't changed.

| Feature | Use |
|---|---|
| Pipeline inputs/parameters | Change data version, hyperparameters or thresholds per run |
| Per-step compute | GPU only for training; CPU for data prep |
| Sub-pipelines | Compose pipelines from pipelines |
| Control flow | Conditional steps (`if_else`), parallel (`parallel_for`), loops (`do_while`) for retraining patterns |
| **Parallel jobs** | Split large inputs into mini-batches across nodes (batch scoring, preprocessing) |
| Outputs | Registered data assets, models, metrics |
| Lineage | Data → steps → model graph in studio |

### Which pipeline technology?

| Scenario | Azure offering |
|---|---|
| Model orchestration (ML steps, retraining) | **Azure ML pipelines** |
| Data orchestration (ETL/ELT) | Azure Data Factory / Fabric Data Factory ([Topic 12](../12-data-and-analytics/README.md)) |
| Code and app orchestration (CI/CD) | Azure Pipelines or GitHub Actions ([Topic 27](../Roadmap.md)) |

They complement each other: CI/CD triggers an Azure ML pipeline, which consumes data prepared by Data Factory.

### Scheduling

**Schedules** run pipeline jobs on a **recurrence** pattern (frequency, interval, days, hours) or a **cron** expression, with start/end times and time zone. Azure ML v2 schedules **don't support event-based triggers**; use Event Grid, Logic Apps or CI/CD to trigger on data arrival. Schedules can also drive model monitoring (file 09).

## How it works on Azure

Component (YAML):

```yaml
# components/train.yml
$schema: https://azuremlschemas.azureedge.net/latest/commandComponent.schema.json
name: train_churn
version: 3
display_name: Train churn model
type: command
inputs:
  training_data: {type: uri_folder}
  max_depth: {type: integer, default: 6}
outputs:
  model_output: {type: mlflow_model}
code: ../src/train
environment: azureml://registries/azureml/environments/sklearn-1.5/labels/latest
command: >-
  python train.py --data ${{inputs.training_data}} --max_depth ${{inputs.max_depth}}
  --model_output ${{outputs.model_output}}
```

Pipeline in Python:

```python
from azure.ai.ml import MLClient, Input, load_component
from azure.ai.ml.dsl import pipeline
from azure.identity import DefaultAzureCredential

ml = MLClient(DefaultAzureCredential(), "<sub>", "rg-ml", "mlw-churn-dev")
prep = load_component(source="components/prep.yml")
train = load_component(source="components/train.yml")
evaluate = load_component(source="components/evaluate.yml")

@pipeline(default_compute="serverless", description="Churn training pipeline")
def churn_pipeline(raw_data, max_depth: int = 6):
    p = prep(raw_data=raw_data)
    t = train(training_data=p.outputs.clean_data, max_depth=max_depth)
    e = evaluate(model=t.outputs.model_output, test_data=p.outputs.test_data)
    return {"model": t.outputs.model_output, "report": e.outputs.report}

job = churn_pipeline(raw_data=Input(type="uri_folder", path="azureml:churn-raw:5"))
ml.jobs.create_or_update(job, experiment_name="churn-pipeline")
```

Weekly schedule (YAML):

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/schedule.schema.json
name: churn-weekly-retrain
trigger:
  type: recurrence
  frequency: week
  interval: 1
  schedule: {hours: [2], minutes: [0], week_days: [monday]}
  time_zone: "GMT Standard Time"
create_job: ./pipeline.yml
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Granularity** | One component per logical step with a clear owner; avoid tiny steps that add overhead. |
| **Reuse vs freshness** | Reuse speeds reruns; force rerun when upstream data changed outside Azure ML. |
| **Registries** | Publish stable components to a registry so dev, test and prod pipelines use identical versions. |
| **Triggers** | Time-based schedules in Azure ML; event-based via Event Grid/Functions/Logic Apps or CI/CD. |
| **SDK vs YAML vs designer** | YAML/SDK in Git for production; designer for prototypes. |
| **Parallel jobs** | For embarrassingly parallel scoring or preprocessing over many files. |

## Hands-on

1. Split a churn notebook into `prep`, `train` and `evaluate` components with YAML definitions.
2. Build the pipeline in Python and run it; inspect the graph, step logs and lineage.
3. Change only `max_depth` and rerun; confirm `prep` is reused.
4. Register the components in a registry and run the same pipeline in a second workspace.
5. Create the weekly schedule; disable it after testing.

## Security, cost and reliability notes

- **Identity**: steps run with compute managed identity; grant least-privilege data access.
- **Secrets**: use workspace connections/Key Vault, not pipeline parameters.
- **Cost**: per-step compute sizing and reuse cut costs; schedules can silently run up bills—monitor.
- **Reliability**: retries for flaky steps; idempotent outputs; versioned components prevent "works on my machine".

## Common mistakes

| Mistake | Fix |
|---|---|
| One giant step doing everything | Split into components |
| Unversioned components | Version and register |
| Expecting event triggers from Azure ML schedules | Use Event Grid or CI/CD |
| GPU for every step | Per-step compute |
| No evaluation gate before registration | Add an evaluate step with thresholds |

## Check yourself

1. What are the three parts of a component?
2. How do pipelines save cost?
3. Which control-flow constructs are available?
4. Which Azure technology fits data orchestration versus model orchestration versus CI/CD?
5. What trigger types do Azure ML schedules support?
6. Why publish components to a registry?

<details>
<summary>Answers</summary>

1. Metadata; interface (inputs/outputs); command, code and environment.
2. Reusing outputs of unchanged steps and running each step on appropriately sized compute.
3. Conditional (`if_else`), parallel-for and do-while loops, plus parallel jobs for mini-batch processing.
4. Data: Data Factory/Fabric; model: Azure ML pipelines; CI/CD: Azure Pipelines or GitHub Actions.
5. Time-based recurrence and cron; not event-based.
6. To share identical, versioned building blocks across workspaces and environments for reproducible promotion.

</details>

## References

- [What are Azure ML pipelines?](https://learn.microsoft.com/azure/machine-learning/concept-ml-pipelines)
- [What is a component?](https://learn.microsoft.com/azure/machine-learning/concept-component)
- [Create pipelines with the SDK v2](https://learn.microsoft.com/azure/machine-learning/how-to-create-component-pipeline-python)
- [Schedule pipeline jobs](https://learn.microsoft.com/azure/machine-learning/how-to-schedule-pipeline-job)
