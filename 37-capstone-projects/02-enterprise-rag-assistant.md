# Enterprise RAG Assistant

## Why it matters

Retrieval-augmented generation over company documents is the most common enterprise generative AI workload. It's also where demos and production differ most: permissions, mixed document formats, freshness, evaluation and cost. In this capstone you build a **private, permission-aware RAG assistant** over mixed documents with measured quality. It's the core portfolio piece for the **Azure AI Engineer** track and the demo consultants reach for most.

**Brief:** Fabrikam's HR and legal teams want an assistant that answers employee questions from policies (PDF, Word, scanned forms and SharePoint pages). Employees must only see answers from documents they're allowed to read, every answer must cite sources, and quality must be measured before launch.

## Key concepts

### Requirements

| # | Requirement |
|---|---|
| R1 | Ingest PDF, DOCX, scanned images and web pages; extract tables and layout |
| R2 | Hybrid search (keyword plus vector) with semantic ranking |
| R3 | Document-level security trimming based on the user's Entra group membership |
| R4 | Answers cite sources; the assistant says it doesn't know when retrieval finds nothing relevant |
| R5 | Private networking for all data and AI services; keyless authentication |
| R6 | Evaluation: groundedness, relevance, retrieval quality and safety on a test set of at least 100 questions |
| R7 | Tracing and dashboards for latency, tokens, cost and user feedback |
| R8 | Incremental indexing when documents change |

### Skills practised

Topics [17](../17-rag-and-knowledge/README.md), [14](../14-microsoft-foundry-platform/README.md), [15](../15-foundry-models-and-azure-openai/README.md), [20](../20-foundry-tools/README.md), [22](../22-evaluation-observability-and-genaiops/README.md), [23](../23-responsible-ai-security-and-governance/README.md) and [34 file 03](../34-ai-solution-architecture/03-enterprise-rag-architecture.md).

## How it works on Azure

### Architecture

```text
Sources (Blob / SharePoint) ─▶ Indexer + skillset (Document Intelligence layout or Content Understanding,
                                chunking, embeddings) ─▶ Azure AI Search index (text, vectors, group_ids)
User ─▶ Entra sign-in ─▶ App (Container Apps / App Service, managed identity)
         ├─ get user's groups (token claims / Graph)
         ├─ query Search with filter group_ids/any(g: search.in(g, '<user groups>'))  — or native ACL trimming where supported
         ├─ call model in Foundry with retrieved chunks and citation instructions
         └─ log traces to Application Insights; store feedback
All PaaS behind private endpoints in the AI spoke from capstone 01.
```

A quicker start is the `azure-search-openai-demo` sample, which you then harden against R1–R8. You can also use Foundry IQ knowledge bases for agentic retrieval (check the feature's GA or Preview status). Document what you changed from the sample and why.

### Milestones

| Milestone | Deliverable |
|---|---|
| M1 | Document inventory, test question set with expected answers and source documents |
| M2 | Ingestion pipeline and index schema (fields, vector profile, semantic configuration, security field) |
| M3 | Retrieval baseline: measure recall at k for keyword, vector and hybrid; tune chunk size and overlap |
| M4 | Generation with citations and refusal behaviour; prompt in source control |
| M5 | Security trimming tests: same question, two users, different answers |
| M6 | Evaluation run (groundedness, relevance, retrieval, safety); fix the top three failure categories |
| M7 | Private networking, keyless auth, tracing, dashboard, cost per question |

### Evaluation snippet (shape)

```python
from azure.ai.evaluation import GroundednessEvaluator, RelevanceEvaluator, evaluate
model_config = {"azure_endpoint": "<endpoint>", "azure_deployment": "<judge-deployment>"}
result = evaluate(
    data="eval/questions_with_answers.jsonl",   # query, context, response, ground_truth
    evaluators={"groundedness": GroundednessEvaluator(model_config),
                "relevance": RelevanceEvaluator(model_config)},
    output_path="eval/results.json",
)
print(result["metrics"])
```

Check the current Foundry evaluation SDK documentation for project-based evaluation and agent evaluators.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Extraction | Document Intelligence layout for structure and tables; Content Understanding for multimodal and field extraction |
| Chunking | Structure-aware chunks (by heading) beat fixed-size chunks for policies; measure, don't guess |
| Security trimming | Filter on group IDs at query time; never rely on the prompt to hide content |
| Model | Smaller model for rewriting queries, stronger model for answers; compare cost and quality |
| Classic RAG vs agentic retrieval | Agentic retrieval for complex multi-part questions; classic for low latency and cost |

## Hands-on

1. Complete milestones M1–M7.
2. Publish a results table: baseline vs tuned retrieval and groundedness scores.
3. Record a two-minute demo showing citations, a refusal and security trimming.
4. Write ADRs for chunking, security trimming and model choice.
5. Estimate monthly cost for 2,000 employees asking 5 questions a day.

## Security, cost and reliability notes

- Security trimming must be enforced in code at query time and tested automatically.
- Use prompt shields and content filters; treat retrieved documents as untrusted input (indirect prompt injection).
- AI Search replicas and partitions drive cost; size from measured query load.
- Handle 429s with retries and back-off; consider a gateway (capstone 06) for multiple back ends.

## Common mistakes

| Mistake | Fix |
|---|---|
| No test set | Build it before tuning |
| Security by prompt | Index-level filters |
| Ignoring scanned documents | OCR or layout extraction |
| Judging quality by a few demos | Evaluate on 100+ questions |
| Stale index | Incremental indexing with change detection |

## Check yourself

1. Why must security trimming happen in the search query, not the prompt?
2. Which metrics prove retrieval quality separately from answer quality?
3. When would you choose agentic retrieval?
4. How do you defend against indirect prompt injection in documents?
5. What evidence goes in your portfolio README?

<details>
<summary>Answers</summary>

1. The model can't be trusted to withhold content it has been given. If the chunks are never retrieved, they can't leak.
2. Retrieval metrics such as recall at k and document retrieval scores, separate from groundedness and relevance.
3. For complex, multi-part or conversational questions where query planning across sources improves answers.
4. Prompt shields, isolating retrieved content in the prompt, limiting tools and actions, and output checks.
5. Architecture, ADRs, evaluation results before and after tuning, security trimming test, cost estimate, demo video.

</details>

## References

- [RAG in Azure AI Search](https://learn.microsoft.com/azure/search/retrieval-augmented-generation-overview)
- [Security filters for trimming results](https://learn.microsoft.com/azure/search/search-security-trimming-for-azure-search)
- [azure-search-openai-demo sample](https://github.com/Azure-Samples/azure-search-openai-demo)
- [Baseline Microsoft Foundry chat reference architecture](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-chat)
- [Document Intelligence overview](https://learn.microsoft.com/azure/ai-services/document-intelligence/overview)
