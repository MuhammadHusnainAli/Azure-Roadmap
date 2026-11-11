# Azure Machine Learning Overview

## Why it matters

Generative AI gets the headlines, but most business value from AI still comes from **predictive models**: churn, fraud, demand forecasting, credit risk, predictive maintenance, image classification. **Azure Machine Learning** (Azure ML) is the Azure platform for building, training, deploying and governing those models with MLOps discipline. It is also where you go when you need full control over training (custom PyTorch, distributed GPU jobs, open-model fine-tuning on your own compute). Architects need to know where Azure ML ends and Microsoft Foundry begins; engineers need its building blocks.

## Key concepts

### What Azure ML provides

| Area | Capabilities |
|---|---|
| **Develop** | Studio, notebooks on compute instances, VS Code integration, designer (drag-and-drop), data labelling |
| **Train** | Jobs on managed or serverless compute, distributed training, hyperparameter sweeps, Automated ML |
| **Orchestrate** | Pipelines built from reusable components, schedules |
| **Track and manage** | MLflow tracking, model registry, environments, lineage |
| **Deploy** | Managed online endpoints (real time), batch endpoints, Kubernetes endpoints |
| **Operate** | Model monitoring (data drift, prediction drift, data quality), CI/CD integration |
| **Govern** | Responsible AI dashboard, RBAC, network isolation, registries for cross-environment promotion |

Interfaces: **studio**, **Python SDK v2** (`azure-ai-ml`), **Azure CLI v2** (`az ml`), ARM/REST.

### Workspaces

A **workspace** is the top-level resource for ML work: jobs, experiments, pipelines, data assets, models, endpoints, compute and datastores, plus security settings. It keeps run history with logs, metrics, outputs, lineage and script snapshots.

Associated resources (created automatically if not supplied):

| Resource | Purpose |
|---|---|
| Azure Storage account | Default datastore, job logs, notebooks |
| Azure Container Registry | Custom environment images |
| Azure Key Vault | Secrets |
| Application Insights | Monitoring for deployed models |

Organising guidance from Microsoft: use Entra groups and roles (data scientist, ML engineer, admin), consider **one workspace per project** for cost and isolation, share associated resources, and pre-create secured resources for self-service.

**Hub workspaces** group project workspaces with shared security, connections and compute. Hubs are the same resource type as Foundry **hubs** (the classic Foundry model); current Foundry projects use the Foundry resource instead ([Topic 14 file 02](../14-microsoft-foundry-platform/02-foundry-resources-and-projects.md)).

### Registries

A **registry** decouples assets from workspaces, like a Git repository for ML assets: models, components, environments and data can be shared across workspaces, subscriptions and regions. Registries enable **promotion** from development to test to production workspaces with lineage back to training jobs (file 09).

### Azure ML and Microsoft Foundry

| Need | Use |
|---|---|
| Generative AI apps, agents, model catalogue, evaluations, managed fine-tuning of supported models | **Microsoft Foundry** ([Topic 14](../14-microsoft-foundry-platform/README.md)) |
| Classical ML (tabular, time series), AutoML, custom training code, distributed GPU training on your compute, MLOps for predictive models | **Azure ML** |
| Open models on dedicated compute (managed online endpoints) | Either: Foundry managed compute uses Azure ML infrastructure ([Topic 15 file 15](../15-foundry-models-and-azure-openai/15-managed-compute-and-self-hosted-deployment.md)) |
| Spark-based data engineering and analytics ML | Fabric Data Science or Azure Databricks ([Topic 12](../12-data-and-analytics/README.md)) |

Azure ML studio also exposes a model catalogue and **prompt flow**. **Prompt flow in Foundry and Azure ML retires on 20 April 2027**: it's not recommended for new development, its container images no longer receive updates, and Microsoft directs migration to **Microsoft Agent Framework** ([Topic 19](../19-agent-frameworks-and-protocols/README.md)).

## How it works on Azure

```text
Workspace ── compute (instances, clusters, serverless, attached Kubernetes)
   │      ── datastores ─► data assets ─► jobs / pipelines (components, environments)
   │                                          │ MLflow metrics, artefacts
   │                                          ▼
   │                                    registered models ─► registry (cross-workspace)
   │                                          ▼
   └────────────────────────────── online / batch endpoints ─► monitoring
```

Security integrations: managed virtual network or your own VNet, private endpoints, managed identity, Key Vault, customer-managed keys, Azure Policy.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Azure ML vs Foundry** | Foundry for LLM apps and agents; Azure ML for predictive ML and custom training. Many organisations use both. |
| **Azure ML vs Databricks vs Fabric** | Azure ML for ML lifecycle and managed endpoints; Databricks/Fabric when ML lives inside a lakehouse and Spark-centric team. They integrate (MLflow is common). |
| **Workspace topology** | Dev/test/prod workspaces (often separate subscriptions) plus a registry for promotion. |
| **Networking** | Managed virtual network (simpler) vs bring-your-own VNet (more control). |
| **Prompt flow** | Don't start new work on it; plan migration before 20 April 2027. |

## Hands-on

Create a workspace and submit a first job with the CLI v2.

```bash
az extension add -n ml
az ml workspace create -n mlw-churn-dev -g rg-ml -l westeurope
cat > job.yml <<'EOF'
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: python -c "import sklearn, sys; print('sklearn', sklearn.__version__)"
environment: azureml://registries/azureml/environments/sklearn-1.5/labels/latest
display_name: hello-azureml
EOF
az ml job create -f job.yml -g rg-ml -w mlw-churn-dev --stream
```

Without a `compute:` entry the job runs on **serverless compute** (file 02). Open the job in studio and explore logs, metrics and the snapshot.

## Security, cost and reliability notes

- **Least privilege**: built-in roles (AzureML Data Scientist, AzureML Compute Operator) or custom roles; Entra groups.
- **Network isolation**: managed VNet with private endpoints for regulated data.
- **Cost**: compute is the main cost (instances left running, idle clusters, endpoints); storage and ACR are smaller; workspaces themselves are free.
- **Reliability**: keep definitions (jobs, pipelines, endpoints) as YAML in Git; workspaces are regional.

## Common mistakes

| Mistake | Fix |
|---|---|
| Building LLM apps on prompt flow today | Use Foundry and Agent Framework |
| One shared workspace for everything | Separate per project/environment; registry for sharing |
| Leaving compute instances running | Idle shutdown and schedules |
| Click-ops only | YAML + CLI/SDK in source control |

## Check yourself

1. What does an Azure ML workspace contain?
2. Which resources are associated with a workspace?
3. What problem do registries solve?
4. When do you choose Azure ML instead of Foundry?
5. What happens to prompt flow, and when?
6. Which interfaces can you use with Azure ML?

<details>
<summary>Answers</summary>

1. Jobs, experiments, pipelines, data assets, models, endpoints, compute targets, datastores and security settings, plus run history and lineage.
2. Storage account, container registry, key vault and Application Insights.
3. Sharing and promoting models, components, environments and data across workspaces, subscriptions and regions with lineage.
4. For classical/predictive ML, AutoML, custom training code and MLOps on your own compute.
5. It retires on 20 April 2027 in Foundry and Azure ML; migrate to Microsoft Agent Framework.
6. Studio, Python SDK v2, CLI v2 and ARM/REST APIs.

</details>

## References

- [What is Azure Machine Learning?](https://learn.microsoft.com/azure/machine-learning/overview-what-is-azure-machine-learning)
- [What is a workspace?](https://learn.microsoft.com/azure/machine-learning/concept-workspace)
- [Registries for MLOps](https://learn.microsoft.com/azure/machine-learning/concept-machine-learning-registries-mlops)
- [What is Microsoft Foundry?](https://learn.microsoft.com/azure/foundry/what-is-foundry)
