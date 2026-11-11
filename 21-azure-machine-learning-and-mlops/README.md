# 21 · Azure Machine Learning and MLOps

> Tracks: E ○ A ○ C – · Part D · AI on Azure

Not every AI problem is a generative one. Churn, fraud, forecasting, risk scoring, quality inspection and many other high-value use cases run on predictive models, and Azure Machine Learning is the Azure platform for building, deploying and operating them with MLOps discipline. This topic covers workspaces and registries and how Azure ML relates to Microsoft Foundry (including the April 2027 retirement of prompt flow); compute from instances to serverless and Kubernetes; datastores, versioned data assets and the managed feature store; training jobs, environments, distributed training and hyperparameter sweeps; Automated ML; pipelines, components and schedules; MLflow tracking and the model registry; managed online and batch endpoints with safe rollout; CI/CD, model monitoring and retraining; and the Responsible AI dashboard.

## What you will be able to do

- Decide when to use Azure ML versus Foundry, Databricks or Fabric.
- Choose and govern compute for development, training and inference at the right cost.
- Manage data with identity-based datastores, versioned assets and, where useful, a feature store.
- Run reproducible training jobs, distributed training and efficient hyperparameter sweeps.
- Build strong baselines quickly with Automated ML, including vision and forecasting.
- Build reusable pipelines from versioned components and schedule them.
- Track experiments with MLflow and manage models across environments with registries.
- Deploy models to online and batch endpoints with Entra auth and blue/green rollout.
- Implement CI/CD, model monitoring (drift, quality, performance) and controlled retraining.
- Assess error, fairness and explainability with the Responsible AI dashboard.

## Prerequisites

[Topic 13](../13-ai-and-generative-ai-foundations/README.md) file 01 (ML essentials), [Topic 04](../04-tooling-and-infrastructure-as-code/README.md) (CLI, IaC), and basic Python and scikit-learn. [Topic 08](../08-storage/README.md) helps for file 03.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Azure Machine Learning Overview](01-azure-machine-learning-overview.md) | 35 min | Workspace, first job |
| 02 | [Compute](02-compute.md) | 35 min | Serverless vs cluster |
| 03 | [Data and Feature Management](03-data-and-feature-management.md) | 45 min | Datastores, data assets |
| 04 | [Training](04-training.md) | 50 min | Command job, sweep |
| 05 | [Automated ML](05-automated-ml.md) | 40 min | AutoML classification |
| 06 | [Pipelines and Components](06-pipelines-and-components.md) | 50 min | Three-step pipeline, schedule |
| 07 | [Model Management and MLflow](07-model-management-and-mlflow.md) | 40 min | Tracking, registry |
| 08 | [Deployment](08-deployment.md) | 50 min | Blue/green online endpoint, batch |
| 09 | [MLOps](09-mlops.md) | 60 min | GitHub Actions, monitoring |
| 10 | [Responsible AI Dashboard](10-responsible-ai-dashboard.md) | 40 min | RAI dashboard and scorecard |

**AI engineers** working on predictive ML: all of it, with 04, 06, 08 and 09 in depth. **Architects:** 01, 02, 08 and 09. **Consultants:** 01 and the decision tables in 05 and 10.

## Completion checklist

- [ ] I can explain the difference between Azure ML and Foundry and the prompt flow retirement.
- [ ] I have run the same job on serverless compute and a scale-to-zero cluster and compared cost.
- [ ] I have registered versioned data assets over an identity-based datastore.
- [ ] I have run a hyperparameter sweep with early termination and limits.
- [ ] I have built an AutoML baseline and checked it for leakage.
- [ ] I have built a component-based pipeline, observed step reuse and scheduled it.
- [ ] I have tracked runs with MLflow and promoted a model through a registry.
- [ ] I have deployed with blue/green, mirroring and canary on a managed online endpoint using Entra auth.
- [ ] I have built a CI/CD workflow with OIDC and set up drift monitoring.
- [ ] I have produced a Responsible AI dashboard and scorecard for a model.

## Maps to

- **AI-300**: MLOps engineering on Azure ML: compute, data, training, pipelines, registries, deployment, monitoring.
- **AZ-305**: design ML infrastructure and operations.
- **DP-700/Fabric** (adjacent): data preparation feeding ML.

## Next

22 · [Evaluation, Observability and GenAIOps](../22-evaluation-observability-and-genaiops/README.md)
