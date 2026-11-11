# MLOps

## Why it matters

Models decay. Customer behaviour shifts, upstream data pipelines change, seasonality arrives, and a model that scored 0.91 AUC at launch quietly drifts into costing money. **MLOps** applies DevOps discipline to ML: versioned code, data and models; automated training and evaluation; gated, safe deployment; production monitoring; and retraining when signals say so. Azure ML provides the building blocks; GitHub Actions or Azure Pipelines tie them together. This is the heart of the AI-300 (MLOps Engineer) certification and of any credible predictive ML platform.

## Key concepts

### The MLOps loop

```text
code + data change ─► CI: lint, unit tests, component validation
         │
         ▼
  training pipeline (dev workspace) ─► evaluate vs current champion ─► register candidate (+ tags)
         │                                                                   │
         ▼                                                                   ▼
  promote to registry ─► CD: deploy to test endpoint ─► integration tests ─► approval
         │                                                                   │
         ▼                                                                   ▼
  production endpoint (blue/green, mirroring, canary) ─► monitoring ─► alert / schedule ─► retrain
```

### Environments and promotion

| Element | Practice |
|---|---|
| **Workspaces** | Separate dev, test and prod workspaces (often separate subscriptions) |
| **Registries** | Promote models, components and environments across workspaces with lineage (file 07) |
| **Infrastructure as code** | Workspaces, compute, endpoints in Bicep/Terraform ([Topic 04](../04-tooling-and-infrastructure-as-code/README.md)) |
| **Definitions as code** | Jobs, pipelines, components, endpoints and monitors as YAML in Git |
| **Identity** | CI/CD uses OIDC federated credentials (no secrets) with least-privilege roles per environment |

### CI/CD

| Stage | Typical checks |
|---|---|
| **CI** | Code quality, unit tests for components, schema validation of YAML, small smoke training run |
| **Continuous training** | Pipeline run on new data or code; evaluation step compares candidate to champion on a fixed test set and fairness metrics |
| **Gates** | Metric thresholds, Responsible AI review, human approval for production |
| **CD** | Deploy to test endpoint, integration and load tests, then safe rollout to production (file 08) |

Tools: **GitHub Actions** (`azure/login` with OIDC, `az ml` commands) or **Azure Pipelines** (Azure ML service connections). Microsoft publishes MLOps solution accelerators (v2) with templates.

### Model monitoring

Azure ML model monitoring computes **signals** by comparing production inference data with reference data (training data or a past production window) on a schedule, alerting through Azure ML or **Event Grid**.

| Signal | What it detects | Example metrics |
|---|---|---|
| **Data drift** | Input feature distributions shift | Jensen-Shannon distance, Population Stability Index, Normalised Wasserstein, Kolmogorov-Smirnov, Chi-squared |
| **Prediction drift** | Output distribution shifts | Same family of distance tests |
| **Data quality** | Nulls, type errors, out-of-range values | Null/data-type/out-of-bounds rates |
| **Feature attribution drift** | Changes in feature importance | NDCG of attributions |
| **Model performance** | Accuracy against ground truth (actuals) when available | Accuracy, precision, recall, RMSE |
| **Custom signals** | Your own metric via a custom component | — |

Key settings: monitoring **frequency**, **lookback window** size and offset (ISO 8601, for example `P7D`; keep reference and production windows non-overlapping), **top N features**, thresholds per metric.

Data collection: online endpoints can collect inputs and outputs automatically (**model data collector**); for batch endpoints or models outside Azure ML, you collect the data yourself.

Microsoft's best practices: start monitoring immediately after deployment, involve the model's data scientists in choosing signals and thresholds, combine signals (drift + attribution drift), use performance monitoring when ground truth exists, and match frequency to data volume.

### Retraining triggers

| Trigger | Mechanism |
|---|---|
| Schedule | Azure ML schedule on the training pipeline |
| Drift or quality alert | Monitor → Event Grid → Function/Logic App/GitHub workflow dispatch → pipeline |
| New labelled data | Data arrival event → pipeline |
| Performance drop | Ground-truth job computes metrics → alert |

Retraining doesn't mean auto-deploying: the new model goes through the same evaluation and approval gates.

## How it works on Azure

GitHub Actions job that trains and registers on merge (excerpt):

```yaml
permissions: { id-token: write, contents: read }
jobs:
  train:
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - run: az extension add -n ml
      - run: |
          RUN=$(az ml job create -f pipelines/train.yml -g rg-ml-dev -w mlw-dev --query name -o tsv)
          az ml job stream -n $RUN -g rg-ml-dev -w mlw-dev
```

Model monitor (YAML excerpt):

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/schedule.schema.json
name: churn-monitor
trigger: {type: recurrence, frequency: day, interval: 1, schedule: {hours: 3, minutes: 0}}
create_monitor:
  compute: {instance_type: standard_e4s_v3, runtime_version: "3.4"}
  monitoring_target:
    ml_task: classification
    endpoint_deployment_id: azureml:churn-ep:green
  alert_notification:
    emails: [ml-oncall@contoso.com]
```

Default signals (data drift, prediction drift, data quality) apply when using an online endpoint with data collection; customise thresholds and add signals as needed.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Automation level** | Start with manual approval before production; automate gates as confidence grows. |
| **Reference data** | Training data detects drift from what the model learned; recent production data detects sudden changes. |
| **Thresholds** | Calibrate on history to avoid alert fatigue; alert on business-relevant features. |
| **Ground truth** | Plan how actuals are collected and joined (often delayed); without them, drift is only a proxy. |
| **Champion/challenger** | Always compare against the current production model on the same test set. |
| **Platform** | GitHub Actions or Azure Pipelines both work; follow your organisation's DevOps standard. |

## Hands-on

1. Structure a repo: `src/`, `components/`, `pipelines/`, `endpoints/`, `monitoring/`, `.github/workflows/`.
2. Configure OIDC federation for a GitHub repo to a dev and a prod identity with least-privilege roles.
3. Build a workflow: PR → unit tests; merge → training pipeline in dev with evaluation gate → register candidate.
4. Add a manual-approval environment that promotes the model to a registry and deploys green to production with 10% canary.
5. Enable data collection on the endpoint and create the monitor; simulate drift by sending shifted data; trigger retraining via Event Grid.

## Security, cost and reliability notes

- **Secretless CI/CD** with OIDC; separate identities and approvals per environment.
- **Audit**: lineage from data to deployment; record approvals in the pipeline.
- **Data collection** stores production inputs/outputs: minimise personal data, set retention and access controls.
- **Cost**: monitoring jobs and retraining consume compute; size schedules sensibly.
- **Reliability**: automated rollback by shifting traffic back to the previous deployment.

## Common mistakes

| Mistake | Fix |
|---|---|
| Automatic deployment of every retrained model | Evaluation and approval gates |
| Monitoring only infrastructure metrics | Add data and prediction drift signals |
| Overlapping reference and production windows | Non-overlapping lookback windows |
| Secrets in pipeline variables | OIDC federation |
| No champion comparison | Evaluate against the production model |

## Check yourself

1. Describe the MLOps loop from code change to retraining.
2. How do registries support promotion?
3. Name five model monitoring signals.
4. What's the difference between using training data and past production data as reference?
5. How can a drift alert trigger retraining?
6. Why shouldn't retraining deploy automatically?

<details>
<summary>Answers</summary>

1. CI tests → training pipeline → evaluation vs champion → register candidate → promote to registry → deploy to test → approval → safe production rollout → monitoring → alert/schedule → retrain.
2. They hold versioned models, components and environments that any workspace can consume, preserving lineage across environments.
3. Data drift, prediction drift, data quality, feature attribution drift, model performance (plus custom signals).
4. Training data shows drift from what the model learned; recent production data highlights sudden recent changes.
5. Monitor alert → Event Grid → Function/Logic App/CI workflow → start the training pipeline.
6. A retrained model can be worse or biased; it must pass the same evaluation, fairness and approval gates.

</details>

## References

- [Model monitoring in Azure ML](https://learn.microsoft.com/azure/machine-learning/concept-model-monitoring)
- [Model management, deployment and MLOps](https://learn.microsoft.com/azure/machine-learning/concept-model-management-and-deployment)
- [Use GitHub Actions with Azure ML](https://learn.microsoft.com/azure/machine-learning/how-to-github-actions-machine-learning)
- [Collect production data from online endpoints](https://learn.microsoft.com/azure/machine-learning/how-to-collect-production-data)
- [MLOps v2 solution accelerator](https://github.com/Azure/mlops-v2)
