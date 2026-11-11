# Deployment

## Why it matters

A model creates value only when applications can call it reliably: low latency for a checkout fraud check, high throughput for nightly scoring of millions of customers. Azure ML **endpoints** give you a stable URL with authentication, behind which you run one or more **deployments**, so you can roll out new model versions with traffic splitting and mirroring instead of a risky cut-over. Choosing between online, batch and standard (serverless) deployments drives cost and architecture.

## Key concepts

### Endpoints and deployments

| Concept | Meaning |
|---|---|
| **Endpoint** | Stable, durable URL with authentication and authorisation (for example `https://<name>.<region>.inference.ml.azure.com/score`) |
| **Deployment** | The compute, model, environment and scoring code behind an endpoint; online and batch endpoints can hold several |
| **Traffic rules** | Percentage split of live traffic across deployments |
| **Mirror traffic** | Copy a percentage of live requests to a deployment for testing (responses not returned to callers) |

### Endpoint types

| Type | Use | Billing |
|---|---|---|
| **Standard deployment** (serverless API) | Supported foundation models and fine-tunes, off-the-shelf, no custom inference stack | Pay per token/call; no compute quota ([Topic 15](../15-foundry-models-and-azure-openai/README.md)) |
| **Managed online endpoint** (recommended for online) | Low-latency, synchronous inference for your models; Azure ML manages serving, scaling, security, monitoring | VM hours per instance |
| **Kubernetes online endpoint** | Same model on your AKS or Arc-enabled cluster | Your cluster |
| **Batch endpoint** | Long-running asynchronous inference over large data in files or data assets; can also run pipeline components | Compute while jobs run |

Microsoft summary: standard deployments for supported foundation models, online endpoints for low-latency custom models, batch endpoints for large asynchronous workloads. Managed online endpoints replace legacy ACI and AKS v1 web service deployments.

### Authentication

| Mode | Notes |
|---|---|
| `key` | Static keys; rotate regularly |
| `aml_token` | Expiring Azure ML tokens |
| `aad_token` | **Microsoft Entra ID** tokens with RBAC (recommended for keyless access) |

### Scoring

- **MLflow models**: no-code deployment (scoring script and environment generated).
- **Custom models**: scoring script with `init()` (load model) and `run(data)` (predict), plus environment.
- Instance type and count, request settings (timeouts, concurrency), liveness/readiness probes, and **autoscale** with Azure Monitor rules.

### Safe rollout (blue/green)

1. Deploy **blue** (v1) with 100% traffic.
2. Deploy **green** (v2) with 0% traffic.
3. Test green directly using the `azureml-model-deployment: green` header.
4. **Mirror** a percentage of live traffic to green (maximum 50%, to protect the endpoint's bandwidth quota).
5. Shift a small percentage of live traffic (canary), then 100%.
6. Delete blue once stable.

### Networking

Managed online endpoints support private endpoints for inbound access and a **workspace managed virtual network** for outbound isolation (egress to storage, ACR, Key Vault over private links).

## How it works on Azure

Endpoint and deployment (CLI YAML):

```yaml
# endpoint.yml
$schema: https://azuremlschemas.azureedge.net/latest/managedOnlineEndpoint.schema.json
name: churn-ep
auth_mode: aad_token
```

```yaml
# green-deployment.yml
$schema: https://azuremlschemas.azureedge.net/latest/managedOnlineDeployment.schema.json
name: green
endpoint_name: churn-ep
model: azureml:churn-model:5          # MLflow model → no scoring script needed
instance_type: Standard_DS3_v2
instance_count: 2
```

```bash
az ml online-endpoint create -f endpoint.yml
az ml online-deployment create -f green-deployment.yml            # 0% traffic by default for new deployments
az ml online-endpoint invoke -n churn-ep --deployment-name green -r sample.json
az ml online-endpoint update -n churn-ep --mirror-traffic "green=10"
az ml online-endpoint update -n churn-ep --mirror-traffic "green=0" --traffic "blue=90 green=10"
az ml online-endpoint update -n churn-ep --traffic "blue=0 green=100"
az ml online-deployment delete -n blue -e churn-ep --yes
```

Batch endpoint scoring a folder:

```bash
az ml batch-endpoint create -n churn-batch
az ml batch-deployment create -f batch-deployment.yml --set-default
az ml batch-endpoint invoke -n churn-batch --input azureml:customers-monthly:12
```

Call the online endpoint with Entra ID:

```python
import requests
from azure.identity import DefaultAzureCredential
token = DefaultAzureCredential().get_token("https://ml.azure.com/.default").token
r = requests.post(scoring_uri, json=payload, headers={"Authorization": f"Bearer {token}"})
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Online vs batch** | Online for interactive, per-request latency; batch for large periodic scoring—far cheaper for bulk. |
| **Managed vs Kubernetes** | Managed for simplicity and built-in scaling/monitoring; Kubernetes when standardising on AKS/Arc or running on-premises. |
| **Auth** | `aad_token` for keyless; keys only for clients that can't use Entra. |
| **Rollout** | Always blue/green with mirroring and canary; automate in CI/CD (file 09). |
| **Sizing and autoscale** | Load test; set min instances for availability (≥ 2 for production) and autoscale on CPU/latency. |
| **Foundation models** | Prefer standard (serverless) deployments when available; managed compute when not ([Topic 15 file 15](../15-foundry-models-and-azure-openai/15-managed-compute-and-self-hosted-deployment.md)). |

## Hands-on

1. Deploy `churn-model` v4 as **blue** to a managed online endpoint with `aad_token` auth; invoke it from Python.
2. Deploy v5 as **green** with 0% traffic; test with the deployment header.
3. Mirror 10% of traffic, review green's metrics and logs, then canary 10% → 100%.
4. Create a batch endpoint and score a monthly customer file; review the output in storage.
5. Configure autoscale on the online deployment (instances 2–5 on CPU).

## Security, cost and reliability notes

- **Keyless and private**: Entra auth, private endpoints, managed VNet egress.
- **Cost**: online deployments bill per instance hour even when idle; delete unused deployments; batch compute scales to zero between jobs.
- **Reliability**: at least two instances; health probes; autoscale; safe rollout; Application Insights logging.
- **Data**: request and response logging can capture personal data; configure data collection deliberately (needed for monitoring, file 09).
- **Quota**: dedicated instances consume subscription VM quota; plan GPU capacity.

## Common mistakes

| Mistake | Fix |
|---|---|
| Real-time endpoints for nightly bulk scoring | Batch endpoints |
| Direct replacement of the production deployment | Blue/green with mirroring |
| Single instance in production | Minimum two plus autoscale |
| Key auth everywhere | Entra tokens |
| Leaving test deployments running | Delete after rollout |

## Check yourself

1. What's the difference between an endpoint and a deployment?
2. Compare standard deployments, online endpoints and batch endpoints.
3. What authentication modes do online endpoints support?
4. What's needed to deploy a custom (non-MLflow) model?
5. List the blue/green rollout steps and the mirroring limit.
6. How do you test a specific deployment directly?

<details>
<summary>Answers</summary>

1. The endpoint is the stable URL with auth; deployments are the compute/model/code behind it, with traffic split between them.
2. Standard: serverless pay-per-use for supported foundation models; online: low-latency custom inference on dedicated instances; batch: asynchronous large-scale inference jobs.
3. `key`, `aml_token`, `aad_token` (Entra ID).
4. A scoring script with `init()` and `run()` plus an environment.
5. Deploy blue (100%), deploy green (0%), test green, mirror traffic (max 50%), canary a small percentage, move to 100%, delete blue.
6. Send the `azureml-model-deployment: <name>` header.

</details>

## References

- [Endpoints for inference](https://learn.microsoft.com/azure/machine-learning/concept-endpoints)
- [Online endpoints](https://learn.microsoft.com/azure/machine-learning/concept-endpoints-online)
- [Safe rollout for online endpoints](https://learn.microsoft.com/azure/machine-learning/how-to-safely-rollout-online-endpoints)
- [Batch endpoints](https://learn.microsoft.com/azure/machine-learning/concept-endpoints-batch)
- [Authenticate clients for online endpoints](https://learn.microsoft.com/azure/machine-learning/how-to-authenticate-online-endpoint)
