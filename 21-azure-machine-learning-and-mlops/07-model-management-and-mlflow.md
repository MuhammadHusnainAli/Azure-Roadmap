# Model Management and MLflow

## Why it matters

"Which model is in production, what data trained it, and how did it score?" If you can't answer in a minute, you don't have model management. Azure ML uses **MLflow**, the open-source standard, for experiment tracking and model packaging, and adds a governed **model registry** with versioning, tags, lineage and cross-workspace **registries**. Because the workspace speaks MLflow, the same tracking code works locally, in Azure ML, in Databricks or in another cloud.

## Key concepts

### MLflow in Azure ML

| Capability | Behaviour |
|---|---|
| **Tracking server** | The workspace is MLflow-compatible: point any MLflow client at it (Azure ML doesn't host a separate MLflow server) |
| **Logging** | `mlflow.log_param`, `log_metric`, `log_artifact`, autologging (`mlflow.autolog()`); the SDK v2 itself has no logging API, keeping code portable |
| **Automatic capture** | Jobs record code, environment, inputs and outputs; you add parameters, metrics, models and plots |
| **Querying** | Search and compare runs with the MLflow client |
| **Where it works** | Azure ML jobs, local machines, Databricks, Synapse; Python fully supported; R and Java with limitations |

### Models

| Concept | Description |
|---|---|
| **Registered model** | A named, versioned logical container for one or more files; registering the same name increments the version |
| **Model types** | `mlflow_model` (MLflow format with signature and environment), `custom_model` (any files), `triton_model` |
| **MLflow model format** | `MLmodel` file with flavour (sklearn, pytorch, transformers…), **signature** (input/output schema), and dependencies |
| **Tags and properties** | Metadata for search and governance (owner, data version, metric values, approval state) |
| **Lineage** | Model → job → data assets and code |
| **Archiving** | Hide old versions; models in active deployments can't be deleted |
| **Registries** | Share and promote models across workspaces, subscriptions and regions |

### Why MLflow format matters

MLflow models enable **no-code deployment**: Azure ML generates the scoring script and environment from the model's flavour and signature for online and batch endpoints (file 08). Custom models need a scoring script (`init()` and `run()`) and environment.

### Packaging

Azure ML can **package** a model with its environment and dependencies ahead of deployment (model packages), which helps with reproducibility and private network deployments where installing packages at deploy time isn't possible.

## How it works on Azure

Track and log a model inside a job (works locally too if `MLFLOW_TRACKING_URI` points to the workspace):

```python
# train.py
import mlflow, mlflow.sklearn
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.metrics import roc_auc_score

mlflow.autolog(log_models=False)
with mlflow.start_run():
    model = GradientBoostingClassifier(max_depth=args.max_depth).fit(X_train, y_train)
    auc = roc_auc_score(y_val, model.predict_proba(X_val)[:, 1])
    mlflow.log_metric("val_auc", auc)
    mlflow.sklearn.log_model(model, artifact_path="model",
                             input_example=X_val.head(3))          # captures signature
```

Register from a job output (CLI):

```bash
az ml model create --name churn-model --type mlflow_model \
  --path azureml://jobs/<job-id>/outputs/artifacts/paths/model/ \
  --tags data_version=3 val_auc=0.91 stage=candidate
```

Or with the MLflow client:

```python
import mlflow
from azure.ai.ml import MLClient
from azure.identity import DefaultAzureCredential

ml = MLClient(DefaultAzureCredential(), "<sub>", "rg-ml", "mlw-churn-dev")
mlflow.set_tracking_uri(ml.workspaces.get("mlw-churn-dev").mlflow_tracking_uri)
mlflow.register_model(f"runs:/{run_id}/model", "churn-model")
```

Promote to a registry for production:

```bash
az ml model share --name churn-model --version 4 --share-with-name churn-model \
  --share-with-version 4 --registry-name contoso-ml-registry
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **MLflow vs custom format** | MLflow for standard frameworks (no-code deployment, signatures); custom for unusual runtimes. |
| **Autologging** | Great for quick tracking; add explicit metrics that match your business KPI. |
| **Naming and tags** | Consistent model names per use case; tags for data version, metrics, approval status, owner. |
| **Stages** | Use tags or separate workspaces/registries for candidate → staging → production rather than ad hoc naming. |
| **Central registry** | One organisational registry for promoted models; workspace registries for experiments. |
| **External training** | Register models trained elsewhere (Databricks, local) to bring them under governance. |

## Hands-on

1. Add MLflow logging to `train.py`; run locally with the workspace as tracking URI, then as a job.
2. Compare three runs in studio and via `mlflow.search_runs`.
3. Register the best run's model with tags; register a second version and view lineage.
4. Create a registry and share the model to it; reference it from another workspace.
5. Archive an old version.

## Security, cost and reliability notes

- **Access**: control who can register and promote models (custom roles); registries have their own RBAC.
- **Integrity**: record data and code versions; treat model files as supply-chain artefacts (scan dependencies, sign images where possible).
- **Sensitive artefacts**: models can memorise training data; protect registries like data stores.
- **Cost**: minimal (storage), but artefact bloat grows; clean up unregistered run artefacts per retention policy.

## Common mistakes

| Mistake | Fix |
|---|---|
| Using Azure ML SDK v1 logging APIs | Use MLflow |
| Models without signatures | Log with `input_example` or explicit signature |
| "model_final_v2_really_final" naming | Versioned registry with tags |
| Promoting by copying files manually | Use registries |
| No link between model and data version | Tag and use lineage |

## Check yourself

1. How does Azure ML relate to MLflow?
2. What does a job capture automatically, and what must you log?
3. What are the three model types?
4. Why does the MLflow model format matter for deployment?
5. What happens when you register a model with an existing name?
6. How do you promote a model across environments?

<details>
<summary>Answers</summary>

1. Workspaces are MLflow-compatible tracking and model servers; you use MLflow APIs directly and the SDK v2 has no separate logging.
2. Code, environment, inputs and outputs are automatic; parameters, metrics, models and plots are logged with MLflow.
3. `mlflow_model`, `custom_model`, `triton_model`.
4. It enables no-code deployment because Azure ML can generate the scoring script and environment from the flavour and signature.
5. The version number increments.
6. Share or register it into an Azure ML registry and deploy from there in test and production workspaces.

</details>

## References

- [MLflow and Azure Machine Learning](https://learn.microsoft.com/azure/machine-learning/concept-mlflow)
- [Model management and deployment concepts](https://learn.microsoft.com/azure/machine-learning/concept-model-management-and-deployment)
- [Work with models](https://learn.microsoft.com/azure/machine-learning/how-to-manage-models)
- [Share models across workspaces with registries](https://learn.microsoft.com/azure/machine-learning/how-to-share-models-pipelines-across-workspaces-with-registries)
