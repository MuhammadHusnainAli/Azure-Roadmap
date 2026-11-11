# RAG Evaluation and Tuning

## Why it matters

A RAG system has many knobs (chunk size, overlap, embedding model, hybrid weights, semantic ranker, top-k, reasoning effort, prompt) and every one of them interacts. Tuning by reading a handful of answers produces systems that demo well and fail in production. The only reliable path is to **measure retrieval and generation separately** on a fixed **golden dataset**, change one thing at a time, and keep the dataset growing from real failures. Microsoft Foundry provides built-in RAG evaluators for this; this file shows how to use them in a disciplined loop.

## Key concepts

### Evaluate the two halves separately

| Half | Question | If it fails… |
|---|---|---|
| **Retrieval** (process evaluation) | Did the right chunks come back, ranked high? | No prompt can fix it: tune chunking, search mode, ranking, filters |
| **Generation** (system evaluation) | Given those chunks, is the answer grounded, relevant and complete? | Tune the prompt, model, number of chunks, citations |

A good diagnostic habit: for every wrong answer, first ask "was the correct chunk in the top-k?".

### Retrieval metrics

| Metric | Meaning | Needs labels? |
|---|---|---|
| **Recall@k** | Fraction of questions where a relevant chunk appears in the top k | Yes |
| **Precision@k** | Fraction of top-k results that are relevant | Yes |
| **MRR** | Average of 1/rank of the first relevant result | Yes |
| **NDCG / XDCG** | Ranking quality with graded relevance | Yes |
| **Holes** | Retrieved documents with no relevance label (label coverage gaps) | Yes |
| **LLM-judged retrieval relevance** | An LLM rates how relevant retrieved chunks are | No |

### Foundry RAG evaluators

| Evaluator | Type | Inputs | Output |
|---|---|---|---|
| **Document Retrieval** | Process | `retrieval_ground_truth`, `retrieved_documents` | Fidelity, NDCG, XDCG, Max Relevance, Holes |
| **Retrieval** | Process (LLM judge) | `query`, `context` | 1–5 with pass/fail threshold |
| **Groundedness** | System (LLM judge) | `response`, `context` (+ optional `query`, tool definitions) | 1–5 with pass/fail |
| **Groundedness Pro** (Preview) | System (Azure AI Content Safety) | `query`, `response`, `context` | True/False |
| **Relevance** | System (LLM judge) | `query`, `response` | 1–5 with pass/fail |
| **Response Completeness** (Preview) | System (LLM judge) | `ground_truth`, `response` | 1–5 with pass/fail |

**Groundedness** is the precision side (nothing beyond the context); **completeness** is the recall side (nothing important missing). You need both. For agents, evaluators can extract context from tool-call results, so `context` is optional when tool calls are present.

### The golden dataset

| Element | Guidance |
|---|---|
| **Questions** | Real user questions (from logs or SMEs), not only ones written by the people who built the index |
| **Coverage** | Each content area, question type (lookup, comparison, multi-part, numeric, "not in corpus"), language, user role |
| **Labels** | Relevant chunk or document IDs (for retrieval metrics) and a reference answer (for completeness) |
| **Negative cases** | Questions whose answer is *not* in the corpus; correct behaviour is "I don't know" |
| **Permission cases** | Questions a given role must *not* be able to answer (file 10) |
| **Size** | Start with 50–100; grow to several hundred; version it in Git |

Foundry can also **generate synthetic queries** from your data to bootstrap coverage; review them before trusting them.

### The tuning loop

```text
1. Baseline: run golden set → retrieval + generation metrics
2. Diagnose failures: retrieval miss? ranking? generation?
3. Change ONE variable (chunk size, hybrid weight, top-k, ranker, prompt, model)
4. Re-run; compare with statistical caution
5. Keep or revert; add new failure cases to the golden set
6. Gate releases on the agreed thresholds (Topic 16 file 04)
```

### What to tune, in order of typical impact

| Order | Knob | File |
|---|---|---|
| 1 | Parsing and chunking (layout-aware, size, overlap, headings) | 04 |
| 2 | Hybrid search + semantic ranker | 06 |
| 3 | Metadata filters and freshness boosting | 06, 13 |
| 4 | Query rewriting / agentic retrieval planning | 06, 07 |
| 5 | Number of chunks passed to the model and reranker threshold | 06 |
| 6 | Embedding model and dimensions | 05 |
| 7 | Prompt (citations, refusal rule) and generator model | [Topic 16](../16-prompt-engineering-and-model-customisation/README.md) |

## How it works on Azure

| Capability | Where |
|---|---|
| Built-in evaluators (RAG, agent, safety, quality) | Foundry evaluator catalogue: portal **Evaluations**, SDK (`azure-ai-projects` cloud evaluation via the OpenAI-compatible evals API) |
| Evaluate stored traces of a deployed agent | Cloud evaluation over traces |
| Continuous evaluation | Sample production traffic and score it automatically ([Topic 22](../Roadmap.md)) |
| CI gating | AI Agent Evaluation GitHub Action, azd evaluation extension ([Topic 16 file 04](../16-prompt-engineering-and-model-customisation/04-prompt-management.md)) |
| Retrieval debugging | Agentic retrieval activity log; search explorer; debug sessions for skillsets |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Labelled metrics vs LLM judges** | Labelled retrieval metrics (recall, NDCG) are objective and cheap to rerun; LLM judges scale to unlabelled data but add cost and variance. Use both. |
| **Judge model** | Use a capable model as judge, different from or at least pinned separately from the generator; keep it fixed between comparisons. |
| **Thresholds** | Set per metric from the baseline and business risk (for example groundedness pass rate ≥ 95% for policy answers). |
| **Sample size** | Small differences on 50 questions are often noise; use more samples or statistical comparison before declaring a win. |
| **Offline vs online** | Offline golden sets gate changes; online evaluation catches drift and new question types. |

## Hands-on

**Step 1: compute retrieval recall@5 yourself** (labels in `golden.jsonl` with `question` and `relevant_ids`):

```python
import json
from azure.identity import DefaultAzureCredential
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizableTextQuery

client = SearchClient("https://<search>.search.windows.net", "policies", DefaultAzureCredential())
rows = [json.loads(l) for l in open("golden.jsonl")]
hits, results = 0, []
for r in rows:
    docs = list(client.search(search_text=r["question"],
        vector_queries=[VectorizableTextQuery(text=r["question"], k_nearest_neighbors=50, fields="content_vector")],
        query_type="semantic", semantic_configuration_name="default", top=5, select=["id", "content"]))
    ids = [d["id"] for d in docs]
    hits += any(i in ids for i in r["relevant_ids"])
    results.append({**r, "context": "\n\n".join(d["content"] for d in docs)})
print("recall@5:", hits / len(rows))
```

**Step 2: generate answers** for each row with your RAG prompt and store `response`.

**Step 3: score with Foundry evaluators** (cloud evaluation):

```python
from azure.ai.projects import AIProjectClient
from openai.types.eval_create_params import DataSourceConfigCustom
from openai.types.evals.create_eval_jsonl_run_data_source_param import (
    CreateEvalJSONLRunDataSourceParam, SourceFileContent, SourceFileContentContent)

project = AIProjectClient(endpoint="<project-endpoint>", credential=DefaultAzureCredential())
oai = project.get_openai_client()
judge = "gpt-5-mini"

ev = oai.evals.create(
    name="policies-rag",
    data_source_config=DataSourceConfigCustom(type="custom", item_schema={
        "type": "object",
        "properties": {k: {"type": "string"} for k in ["query", "context", "response", "ground_truth"]},
        "required": ["query", "context", "response"]}),
    testing_criteria=[
        {"type": "azure_ai_evaluator", "name": "groundedness", "evaluator_name": "builtin.groundedness",
         "initialization_parameters": {"deployment_name": judge},
         "data_mapping": {"query": "{{item.query}}", "context": "{{item.context}}", "response": "{{item.response}}"}},
        {"type": "azure_ai_evaluator", "name": "relevance", "evaluator_name": "builtin.relevance",
         "initialization_parameters": {"deployment_name": judge},
         "data_mapping": {"query": "{{item.query}}", "response": "{{item.response}}"}},
        {"type": "azure_ai_evaluator", "name": "retrieval", "evaluator_name": "builtin.retrieval",
         "initialization_parameters": {"deployment_name": judge},
         "data_mapping": {"query": "{{item.query}}", "context": "{{item.context}}"}},
    ])

run = oai.evals.runs.create(eval_id=ev.id, name="baseline",
    data_source=CreateEvalJSONLRunDataSourceParam(type="jsonl", source=SourceFileContent(
        type="file_content",
        content=[SourceFileContentContent(item={"query": r["question"], "context": r["context"],
                                                "response": r["response"], "ground_truth": r.get("answer", "")})
                 for r in results])))
print(run.id, run.status)   # poll with oai.evals.runs.retrieve(...) and view results in the portal
```

**Step 4:** change one variable (for example chunk size 512 → 1,024 tokens), re-run steps 1–3, and record the deltas in a table.

## Security, cost and reliability notes

- **Golden sets contain sensitive data** (real questions, answers, document IDs); store them with the same controls as the corpus.
- **Evaluation costs tokens.** LLM judges cost money per row per evaluator; run smoke sets on pull requests and full sets nightly.
- **Judge drift.** Pin the judge deployment and version; a judge upgrade can shift scores without any system change.
- **Safety evaluation too.** Add risk and safety evaluators and indirect-injection test cases (documents containing malicious instructions) to the RAG suite.

## Common mistakes

| Mistake | Fix |
|---|---|
| Evaluating only final answers | Measure retrieval separately |
| Golden set written by the index builders only | Use real user questions and SMEs |
| No "not in corpus" questions | Add them; test refusal behaviour |
| Changing several knobs at once | One variable per experiment |
| Declaring victory on a 2-point change over 40 questions | Larger sets, statistical comparison |
| Never updating the golden set | Add every production failure |

## Check yourself

1. Why evaluate retrieval and generation separately?
2. Define recall@k and MRR.
3. Which Foundry evaluator needs labelled relevant documents, and what does it output?
4. What is the difference between groundedness and response completeness?
5. Which inputs does the Retrieval evaluator need?
6. List five kinds of questions a golden dataset should contain.
7. What is usually the highest-impact knob to tune first?
8. Why pin the judge model?

<details>
<summary>Answers</summary>

1. Different fixes apply: retrieval failures need search and chunking changes; generation failures need prompt or model changes. Without separation you can't tell which.
2. Recall@k: fraction of questions with a relevant result in the top k. MRR: mean of 1/rank of the first relevant result.
3. Document Retrieval; it outputs Fidelity, NDCG, XDCG, Max Relevance and Holes.
4. Groundedness checks the answer contains nothing beyond the context (precision); completeness checks it doesn't miss information from the ground truth (recall).
5. `query` and `context` (plus a judge deployment).
6. Any five: lookups, comparisons, multi-part, numeric, not-in-corpus, permission-restricted, different languages, different roles, adversarial/injection.
7. Parsing and chunking.
8. A different judge can change scores even when the system is unchanged, invalidating comparisons.

</details>

## References

- [RAG evaluators in Microsoft Foundry](https://learn.microsoft.com/azure/foundry/concepts/evaluation-evaluators/rag-evaluators)
- [Run cloud evaluations with the Foundry SDK](https://learn.microsoft.com/azure/foundry/observability/how-to/cloud-evaluation)
- [RAG solution design and evaluation guide (Azure Architecture Center)](https://learn.microsoft.com/azure/architecture/ai-ml/guide/rag/rag-solution-design-and-evaluation-guide)
- [Relevance in Azure AI Search](https://learn.microsoft.com/azure/search/search-relevance-overview)
- [Run an evaluation in GitHub Actions](https://learn.microsoft.com/azure/foundry/how-to/evaluation-github-action)
