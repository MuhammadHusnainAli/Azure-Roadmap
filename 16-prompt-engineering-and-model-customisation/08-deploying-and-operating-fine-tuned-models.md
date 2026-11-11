# Deploying and Operating Fine-Tuned Models

## Why it matters

Training a model is a one-off event; running it is a commitment. A fine-tuned model must be deployed somewhere, paid for by the hour on many deployment types, evaluated before it replaces anything, monitored for drift, and retrained when data changes or the base model retires. Most fine-tuning projects that disappoint do so here: a deployment left running unused for months, a model promoted on the strength of training loss, or a retirement notice nobody planned for. This file is the operational half of files 06 and 07.

## Key concepts

### Deployment options

Foundry offers three ways to serve a fine-tuned model:

| Option | What runs | Billing | Status |
|---|---|---|---|
| **Serverless** | Foundry hosts and manages serving | Per token, plus hourly hosting for some deployment types | GA for supported models |
| **Managed compute** | Dedicated GPUs run a matching base model with one or more compatible **LoRA adapters** | Per VM hour | Preview |
| **Fireworks on Foundry** | Fireworks runtime serves one adapter merged into a full copy of the base model | Provider pricing; provisioned | Preview; a subscription owner must enable it |

### Serverless deployment types for fine-tuned models

| Deployment type | Inference location | Billing | Notes |
|---|---|---|---|
| **Standard** | The deployment's region | Per token **plus hourly hosting** | Supported for a subset of models and regions (for example gpt-4.1 family in North Central US and Sweden Central) |
| **Data Zone Standard** | Within the US data zone | Per token | |
| **Global Standard** | Global capacity | Per token | Custom weights may be temporarily stored outside your geography |
| **Developer** | Global | Per token, **no hourly hosting fee** | For candidate evaluation only; lasts 24 hours; no availability SLA; no residency guarantee |
| **Provisioned Throughput** | Regional | PTUs | Predictable throughput; limited models and regions |
| **Global Provisioned** | Global | PTUs | For imported custom models |

Check the fine-tuning overview's support table: availability is per model, and some models only support one type.

### Storage vs hosting

A trained fine-tuned model can be **stored in Foundry at no cost** until you deploy it. Cost starts with deployment. Microsoft's classic deployment documentation also states that a fine-tuned deployment that receives no Chat Completions or Responses calls for a continuous **15 days** is deleted automatically; the underlying model is kept and can be redeployed. Do not rely on this as a cost control: hosting is charged throughout those 15 days.

### Promotion gates

Never promote on training metrics alone. A fine-tuned model should pass:

| Gate | What it checks |
|---|---|
| **Task quality** | Beats or matches the current production model on a held-out test set |
| **Regression** | Doesn't break cases the current model handles |
| **Safety** | Risk and safety evaluators; Foundry has already run its own pre-deployment safety evaluation |
| **Format** | Valid structured output rate |
| **Latency and cost** | Meets p95 latency and cost per request targets under load |
| **Statistical significance** | Differences are real, not noise (the Foundry evaluation GitHub Action reports confidence intervals) |

### Retraining triggers

| Trigger | Signal |
|---|---|
| **Base model retirement** | Retirement notice for the base version ([Topic 15 file 14](../15-foundry-models-and-azure-openai/14-model-lifecycle.md)) |
| **Data drift** | New product lines, policies, terminology, or input formats |
| **Quality drift** | Online evaluation scores or user feedback trending down |
| **New failure categories** | Repeated errors in production traces |
| **A better base model** | A newer base model, tuned, beats the current one in evaluation |
| **Cost change** | A cheaper base or deployment type becomes available |

## How it works on Azure

### Deploying

- **Portal:** open the completed job, select **Deploy**, choose Serverless, Managed compute or Fireworks, then the deployment type and capacity.
- **Code:** create a `Microsoft.CognitiveServices/accounts/deployments` resource whose model name is the fine-tuned model ID (for example `gpt-4.1-mini-2025-04-14.ft-<id>`) and format `OpenAI`. Cross-region deployment (training in one region, deploying in another) is done through the SDK or REST API, not the portal.
- **Roles:** Foundry Owner, or a role with `Microsoft.CognitiveServices/accounts/deployments/write`.

### Calling

A fine-tuned deployment is called like any other deployment, by deployment name, through the v1 endpoint. Use the **same system prompt** that was used in training.

### Monitoring

| Signal | Where |
|---|---|
| Requests, tokens, latency, throttling | Azure Monitor metrics for the Foundry resource |
| Per-request traces | Foundry tracing with Application Insights |
| Quality in production | Continuous (online) evaluation on sampled traffic ([Topic 22](../Roadmap.md)) |
| Spend | Cost Management, filtered by the hosting meter and token meters |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Developer vs Standard for testing** | Use Developer deployments to evaluate candidate checkpoints cheaply; deploy only the winner to a production type. |
| **Standard vs Global Standard** | Standard for regional residency (with hosting fee); Global Standard for per-token billing without residency. |
| **Serverless vs managed compute** | Serverless for OpenAI models and low operations; managed compute when you want several LoRA adapters on one base deployment or open-model control. |
| **Blue/green or canary** | Deploy the new model alongside the old one; shift a percentage of traffic via an AI gateway (APIM) or application flag; compare live metrics; then retire the old deployment. |
| **Low-traffic workloads** | Hourly hosting may exceed token savings; compare with a prompted base model or use a per-token deployment type. |
| **Many tenants or tasks** | Prefer one base deployment with multiple adapters (managed compute) or one well-prompted model over many separately hosted fine-tunes. |

## Hands-on

Deploy a fine-tuned model with Bicep, then compare it with the base model.

```bicep
param accountName string
param fineTunedModel string   // e.g. 'gpt-4.1-mini-2025-04-14.ft-b044a9d3cf9c4228b5d393567f693b83'

resource account 'Microsoft.CognitiveServices/accounts@2025-06-01' existing = {
  name: accountName
}

resource ftDeployment 'Microsoft.CognitiveServices/accounts/deployments@2025-06-01' = {
  parent: account
  name: 'returns-ft-v1'
  sku: {
    name: 'GlobalStandard'
    capacity: 10
  }
  properties: {
    model: {
      format: 'OpenAI'
      name: fineTunedModel
      version: '1'
    }
  }
}
```

```bash
az deployment group create -g rg-ai -f ft-deploy.bicep \
  -p accountName=contoso-foundry fineTunedModel=<ft-model-id>
```

Then run an evaluation that compares `returns-ft-v1` with the current production deployment on your held-out set (Foundry portal **Evaluations**, the SDK, or the GitHub Action from file 04). Promote only if it wins on your gates. Finally, delete whichever deployment loses:

```bash
az cognitiveservices account deployment delete \
  -g rg-ai -n contoso-foundry --deployment-name returns-ft-v1
```

## Security, cost and reliability notes

- **Idle hosting is the classic waste.** Tag fine-tuned deployments with owner and expiry, and alert on deployments with low traffic.
- **Cancelling a training job doesn't delete deployments.** Clean up deployments separately.
- **Residency.** Global Standard may store custom weights temporarily outside your geography; Developer has no residency guarantee.
- **Reliability.** Fine-tuned deployments have their own quota and may be available in fewer regions than base models. Plan a fallback (for example the prompted base model) behind your gateway.
- **Access.** Fine-tuned models can encode sensitive business behaviour; restrict who can deploy and call them with Foundry RBAC and keyless auth.
- **Content filtering** still applies to fine-tuned deployments; keep your content filter configuration aligned with production.

## Common mistakes

| Mistake | Fix |
|---|---|
| Promoting because training loss looked good | Gate on held-out task, regression and safety evaluations |
| Leaving candidate deployments running | Use Developer deployments for candidates; delete losers |
| Different system prompt in production than in training | Keep training and inference prompts identical |
| No plan for base-model retirement | Track retirement dates; keep the data and training pipeline in code |
| Assuming the 15-day rule controls cost | Hosting accrues until deletion; delete deliberately |
| Hard cut-over to the new model | Canary through a gateway and compare live metrics |

## Check yourself

1. Name the three deployment options for fine-tuned models in Foundry.
2. Which serverless deployment type has no hourly hosting fee, and what are its limits?
3. What does it cost to store an undeployed fine-tuned model?
4. What happens to a fine-tuned deployment with no calls for 15 continuous days?
5. List four promotion gates.
6. List four retraining triggers.
7. How would you roll out a new fine-tuned version safely?
8. When might a fine-tuned model cost more than a prompted base model?

<details>
<summary>Answers</summary>

1. Serverless, managed compute (Preview) and Fireworks on Foundry (Preview).
2. Developer: for evaluation only, lasts 24 hours, no availability SLA and no data residency guarantee.
3. Nothing; charges start when it's deployed.
4. The deployment is deleted (per the classic deployment documentation); the model itself remains and can be redeployed.
5. Any four of task quality, regression, safety, format validity, latency and cost, statistical significance.
6. Any four of base-model retirement, data drift, quality drift, new failure categories, a better base model, cost changes.
7. Deploy alongside the current model, route a small percentage of traffic through a gateway or flag, compare live quality and latency, then shift traffic and delete the old deployment.
8. At low request volume, when the fixed hourly hosting charge outweighs the token savings from shorter prompts.

</details>

## References

- [Deploy fine-tuned models](https://learn.microsoft.com/azure/foundry/fine-tuning/deploy-fine-tuned-models)
- [Fine-tuning in Microsoft Foundry (overview, deployment types and support)](https://learn.microsoft.com/azure/foundry/fine-tuning/overview)
- [Deploy a fine-tuned model (classic, inactivity and hosting notes)](https://learn.microsoft.com/azure/foundry-classic/openai/how-to/fine-tuning-deploy)
- [Plan and manage costs for Microsoft Foundry](https://learn.microsoft.com/azure/foundry/concepts/manage-costs)
- [Run an evaluation in GitHub Actions](https://learn.microsoft.com/azure/foundry/how-to/evaluation-github-action)
- [Model deprecations and retirements](https://learn.microsoft.com/azure/foundry/openai/concepts/model-retirements)
