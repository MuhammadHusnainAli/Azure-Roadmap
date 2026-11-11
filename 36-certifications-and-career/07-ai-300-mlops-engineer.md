# AI-300 MLOps Engineer

## Why it matters

AI-300 (Operationalizing Machine Learning and Generative AI Solutions) leads to **Microsoft Certified: Machine Learning Operations Engineer Associate**. It replaced DP-100 (Azure Data Scientist Associate), which retired on 1 June 2026. DP-100 focused on training models. AI-300 focuses on operating them. It covers **MLOps** for traditional machine learning with Azure Machine Learning, and **GenAIOps** for generative AI with Microsoft Foundry. Microsoft calls the two together AI operations (AIOps). That makes AI-300 the natural advanced certification for AI engineers who build platforms and pipelines.

## Key concepts

### Skills measured (study guide)

| Domain | Weight | Sub-areas |
|---|---|---|
| **Design and implement an MLOps infrastructure** | 15–20% | Resources and assets in a Machine Learning workspace; IaC for Machine Learning |
| **Implement machine learning model lifecycle and operations** | 25–30% | Orchestrate training; model registration and versioning; deploy models to production; monitor and maintain models |
| **Design and implement a GenAIOps infrastructure** | 20–25% | Foundry environments and platform configuration; deploy and manage foundation models; prompt versioning with source control |
| **Implement generative AI quality assurance and observability** | 10–15% | Evaluation and validation for apps and agents; observability |
| **Optimise generative AI systems and model performance** | 10–15% | RAG performance and accuracy; advanced fine-tuning and model customisation |

Expected tooling: Azure Machine Learning, Foundry, GitHub Actions, Bicep and the Azure CLI. Candidates should have a data science background with Python and an entry-level understanding of DevOps.

### Roadmap mapping

| Exam area | Roadmap topics |
|---|---|
| Machine Learning workspace, training, registry, endpoints, monitoring | [21 · Azure Machine Learning and MLOps](../21-azure-machine-learning-and-mlops/README.md) |
| Evaluation, tracing, GenAIOps pipelines | [22 · Evaluation, Observability and GenAIOps](../22-evaluation-observability-and-genaiops/README.md) |
| Foundry environments, deployments | [14](../14-microsoft-foundry-platform/README.md), [15](../15-foundry-models-and-azure-openai/README.md) |
| Fine-tuning, prompt management | [16 · Prompt Engineering and Model Customisation](../16-prompt-engineering-and-model-customisation/README.md) |
| RAG optimisation | [17 · RAG and Knowledge](../17-rag-and-knowledge/README.md) |
| IaC and CI/CD | [04](../04-tooling-and-infrastructure-as-code/README.md), [27 · DevOps and Platform Engineering](../27-devops-and-platform-engineering/README.md) |

## How it works on Azure

### Eight-week study plan

| Week | Focus | Build |
|---|---|---|
| 1 | Workspace and IaC | Deploy a Machine Learning workspace with Bicep, including compute clusters and data assets |
| 2 | Training pipelines | Command jobs and a pipeline; MLflow tracking; hyperparameter sweep |
| 3 | Registry and deployment | Register models in a shared registry; deploy to a managed online endpoint with blue/green traffic |
| 4 | Monitoring | Model monitoring for data drift; batch endpoint for scoring |
| 5 | Foundry environments | Dev, test and prod Foundry projects; model deployments with quotas; prompts in Git |
| 6 | Evaluation | Evaluation dataset; quality and safety evaluators; GitHub Actions gate |
| 7 | Observability and optimisation | Tracing, continuous evaluation, RAG tuning (chunking, hybrid search, reranking) |
| 8 | Fine-tuning and revision | Fine-tune a small model and compare it with prompting; practice assessment |

### Reference pipeline

```text
PR ─▶ unit tests ─▶ train / build ─▶ offline evaluation gate ─▶ register model or agent version
   ─▶ deploy to staging ─▶ safety and red-team checks ─▶ approval ─▶ blue/green to production
   ─▶ online monitoring (drift, quality, safety, cost) ─▶ alerts and retraining triggers
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **AI-300 vs AI-103 order** | AI-103 first if you build apps; AI-300 first if you come from data science. |
| **Machine Learning vs Foundry** | The exam expects both: Machine Learning for classic ML, Foundry for generative AI. |
| **Fine-tuning vs RAG** | Know when each is appropriate and how to evaluate the result. |

## Hands-on

1. Deploy a Machine Learning workspace and pipeline entirely from Bicep and GitHub Actions.
2. Run blue/green deployment on a managed online endpoint.
3. Build a GenAIOps pipeline with an evaluation gate (see [capstone 08](../37-capstone-projects/08-genaiops-pipeline.md)).
4. Version prompts in Git and promote them across environments.
5. Compare a fine-tuned small model with a prompted large model on cost and quality.

## Security, cost and reliability notes

- Use managed identities for jobs, endpoints and pipelines; use OIDC federation for GitHub Actions.
- Use low-priority or serverless compute for training labs; scale compute clusters to zero.
- Set alerts for drift and evaluation regressions, not just for infrastructure.

## Common mistakes

| Mistake | Fix |
|---|---|
| Studying DP-100 data science content only | Focus on operations: IaC, pipelines, deployment, monitoring |
| Ignoring generative AI | GenAIOps domains are about 40–55% of the exam |
| Manual deployments | Practise CI/CD with gates |
| No evaluation datasets | Build and version them |

## Check yourself

1. Which certification does AI-300 lead to, and what did it replace?
2. What does Microsoft mean by AIOps in this exam?
3. Which tools does the exam expect?
4. Name the two generative AI quality domains.
5. Which roadmap topics are core for AI-300?

<details>
<summary>Answers</summary>

1. Machine Learning Operations Engineer Associate; it replaced the Azure Data Scientist Associate (DP-100), which retired on 1 June 2026.
2. MLOps and GenAIOps together.
3. Azure Machine Learning, Foundry, GitHub Actions, and IaC with Bicep and the Azure CLI.
4. Generative AI quality assurance and observability; optimising generative AI systems and model performance.
5. Topics 21 and 22, with 14–17, 04 and 27 in support.

</details>

## References

- [Exam AI-300](https://learn.microsoft.com/credentials/certifications/exams/ai-300/)
- [AI-300 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-300)
- [Machine Learning Operations Engineer Associate](https://learn.microsoft.com/credentials/certifications/operationalizing-machine-learning-and-generative-ai-solutions/)
- [MLOps with Azure Machine Learning](https://learn.microsoft.com/azure/machine-learning/concept-model-management-and-deployment)
