# Multimodal and Structured-Data RAG

## Why it matters

Text-chunk RAG works well for prose and badly for everything else. In real organisations the authoritative answer often sits inside a **diagram** in a PDF, a **table** that spans three pages, a **database** of orders, or a web of **relationships** between customers, contracts and incidents. Treating those as plain text loses meaning. This file covers three extensions of RAG: **multimodal RAG** (images and diagrams), **structured-data RAG** (querying tables with text-to-SQL or semantic models), and **graph-based retrieval** (relationships and dataset-wide questions). Architects need to recognise which pattern a question needs; engineers need the Azure building blocks.

## Key concepts

### Question types and the right pattern

| Question | Pattern |
|---|---|
| "What is the approval process for HR forms?" (answer lives in a flowchart) | **Multimodal RAG** |
| "Show me products that look like this photo." | **Multimodal embeddings** (image-to-image) |
| "What were total sales in Q3 for the north region?" | **Structured-data RAG**: text-to-SQL or semantic model |
| "Which suppliers are linked to the delayed shipments and their contracts?" | **Graph retrieval** |
| "What are the main themes across all customer complaints this year?" | **Graph or summary-based global retrieval** |
| "What does clause 7 of the Contoso contract say?" | Classic text RAG |

### Multimodal RAG in Azure AI Search

A multimodal pipeline:

1. **Extract** page text, inline images and structural metadata.
2. **Chunk** the text.
3. **Describe images** in natural language (verbalisation) and/or **embed** them directly.
4. **Embed** text and images.
5. **Store** extracted images (knowledge store) so the app can show them as citations.

Extraction options:

| | Document Extraction skill | Azure Content Understanding skill |
|---|---|---|
| Text location (pages, polygons) | No | Yes |
| Image location | Yes | Yes |
| Tables | No | Yes, including cross-page |
| Semantic chunking | No (use Text Split) | Yes |
| AI image descriptions | No | Yes (2026-05-01-preview and later) |
| File types for location metadata | PDF only | PDF, DOCX, XLSX, PPTX and more |
| Recommended for | Prototyping, simple pipelines | Advanced document analysis |

Microsoft recommends the Content Understanding skill for new skillsets; Document Layout remains supported for existing pipelines.

Two ways to make images searchable:

| Approach | How | Strengths | Weaknesses |
|---|---|---|---|
| **Verbalisation** | GenAI Prompt skill asks an LLM to describe each image; the description is embedded as text | Captures relationships in diagrams; gives the LLM citable text | An LLM call per image; slower, costlier indexing; no image-as-query |
| **Direct multimodal embeddings** | Azure Vision multimodal embeddings skill (Preview) or AML skill with a catalogue model; matching vectoriser | No LLM at indexing; supports image queries and visual similarity | Doesn't explain *why* items are related; less context for answers |

Many solutions combine both: verbalise explanatory visuals (diagrams, charts), embed photos and screenshots directly. Only multimodal embedding vectorisers support **image-to-vector queries**.

### Structured-data RAG

Embedding table rows as text and searching them is fine for lookups ("find the product called X") but wrong for aggregation ("total sales by region"). For questions that need calculation, generate a **query** instead:

| Approach | Description | Azure option |
|---|---|---|
| **Text-to-SQL** | LLM converts the question to SQL using the schema, runs it, then explains the result | Custom tool/function in an agent over Azure SQL, PostgreSQL, Fabric Warehouse ([Topic 18 file 06](../18-agents-with-foundry-agent-service/06-custom-tools.md)) |
| **Semantic-model grounding** | Ask over curated measures and relationships instead of raw tables | **Fabric data agents** over Power BI semantic models, lakehouses, warehouses and ontologies; Fabric Data Agent knowledge source (Preview) |
| **Indexed structured data** | Index rows (Azure SQL knowledge source, Preview) for search-style lookups | Agentic retrieval |
| **Hybrid** | Search to find relevant entities, then SQL for numbers | Agent with both tools |

Text-to-SQL success factors: good table and column descriptions, few-shot example queries, a restricted read-only view or schema, validation of generated SQL, row limits, and showing the user the SQL or the data used.

### Graph-based retrieval

| Technique | Idea |
|---|---|
| **Knowledge graph lookup** | Entities and relationships stored in a graph; agent traverses ("Order → Shipment → Sensor → Breach") |
| **GraphRAG** (Microsoft Research, open source) | An LLM extracts entities and relationships from text, builds communities and summaries. **Local search** answers entity-specific questions; **global search** answers dataset-wide questions via community summaries (map-reduce); **DRIFT search** combines both. |
| **Ontology-grounded agents** | Fabric IQ ontology (Preview) and graph bind business entities to live data (file 09) |

Azure options for graphs: **Graph in Microsoft Fabric** (Preview), **Fabric ontology**, **Apache AGE** on Azure Database for PostgreSQL, Cosmos DB for Apache Gremlin. GraphRAG is a library and pattern, not a managed Azure service; indexing is LLM-intensive and costly, so use it where global or relational questions justify it.

## How it works on Azure

| Need | Build with |
|---|---|
| Multimodal index quickly | Azure portal **Import data (new)** wizard with the multimodal option; multimodal RAG sample app |
| Image descriptions | GenAI Prompt skill or Content Understanding figure descriptions |
| Image vectors | Azure Vision multimodal embeddings skill (Preview) + Azure Vision vectoriser; AML skill + Foundry catalogue vectoriser |
| Answers over data | Fabric data agent published to Foundry; custom function tool with Azure SQL (managed identity, read-only role) |
| Graph | Fabric Graph or ontology; PostgreSQL with Apache AGE; GraphRAG library writing outputs to storage and search |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Verbalise or embed images?** | Verbalise diagrams and charts used to answer "how/why"; embed photos for "looks like". |
| **Which images to process** | Skip logos and decorative images (size and position filters) to control cost. |
| **Text-to-SQL vs semantic models** | Semantic models where trusted metrics exist; text-to-SQL for operational databases without a semantic layer, with strict guardrails. |
| **Graph or not?** | Use graphs when questions depend on multi-hop relationships or whole-corpus themes. Otherwise hybrid search plus agentic retrieval is cheaper. |
| **One agent, many tools** | An agent with a knowledge tool (Foundry IQ), a data tool (Fabric data agent or SQL) and possibly a graph tool, with instructions on when to use each. |

## Hands-on

**Part A: multimodal index.** In the Azure portal, run **Import data (new)** → multimodal option on a container of PDFs that contain diagrams. Choose Content Understanding extraction, an Azure OpenAI chat model for image verbalisation, and an embedding model. Ask a question only answerable from a diagram; inspect the returned chunk (content type image, description text, image path).

**Part B: safe text-to-SQL tool.**

```python
import pyodbc, re, struct
from azure.identity import DefaultAzureCredential

ALLOWED = re.compile(r"^\s*SELECT\b", re.IGNORECASE)
BLOCKED = re.compile(r"\b(INSERT|UPDATE|DELETE|MERGE|DROP|ALTER|EXEC|GRANT)\b", re.IGNORECASE)

def run_readonly_sql(sql: str) -> list[dict]:
    """Tool exposed to the agent: run a read-only query against the sales views."""
    if not ALLOWED.match(sql) or BLOCKED.search(sql):
        raise ValueError("Only SELECT statements are allowed.")
    token = DefaultAzureCredential().get_token("https://database.windows.net/.default").token.encode("utf-16-le")
    conn = pyodbc.connect(
        "Driver={ODBC Driver 18 for SQL Server};Server=tcp:<server>.database.windows.net;Database=sales;",
        attrs_before={1256: struct.pack(f"<I{len(token)}s", len(token), token)})  # SQL_COPT_SS_ACCESS_TOKEN
    cur = conn.cursor()
    cur.execute(f"SELECT TOP 200 * FROM ({sql}) AS q")
    cols = [c[0] for c in cur.description]
    return [dict(zip(cols, row)) for row in cur.fetchall()]
```

Give the agent the schema of a few **views** (not base tables) with column descriptions, connect with a managed identity that only has `db_datareader` on those views, and log every generated query. The regex checks are a second line of defence, not the primary control: database permissions are.

## Security, cost and reliability notes

- **Least privilege for SQL tools**: read-only database role on curated views; row-level security for per-user data; never use an admin connection.
- **Prompt injection into SQL**: user text can steer query generation; validate, limit and log, and rely on database permissions.
- **Images may contain sensitive data** (screenshots with personal data); verbalisation sends them to an LLM; apply the same data classification as text.
- **Cost**: image verbalisation (an LLM call per image), Content Understanding per page, GraphRAG indexing (many LLM calls). Sample before full runs.
- **Freshness**: graphs and verbalised descriptions must be rebuilt when sources change; design incremental updates.

## Common mistakes

| Mistake | Fix |
|---|---|
| Embedding table rows to answer aggregate questions | Use text-to-SQL or semantic models |
| Plain-text extraction of diagram-heavy manuals | Multimodal pipeline with verbalisation |
| Verbalising every image | Filter decorative images |
| Text-to-SQL with a write-capable connection | Read-only identity on views |
| Building GraphRAG for simple FAQ retrieval | Use hybrid search; reserve graphs for relational/global questions |
| Expecting image queries with a verbalisation-only index | Add multimodal embeddings and a vectoriser |

## Check yourself

1. Which retrieval pattern fits "total revenue by quarter", and why not vector search?
2. Name the five steps of a multimodal pipeline in Azure AI Search.
3. Compare verbalisation and direct multimodal embeddings.
4. Which extraction skill does Microsoft recommend for new multimodal skillsets, and why?
5. What enables image-as-query search?
6. List four guardrails for a text-to-SQL tool.
7. What are GraphRAG's local, global and DRIFT search modes for?
8. Name two Azure options for storing and querying graphs.

<details>
<summary>Answers</summary>

1. Structured-data RAG (text-to-SQL or a semantic model); vector search finds similar text but can't aggregate numbers correctly.
2. Extract text and images, chunk text, describe images, embed text and images, store images for citations.
3. Verbalisation uses an LLM to describe images as citable text (good for diagrams, costly); direct embeddings place images and text in one vector space (cheap, supports visual similarity and image queries, less explanatory).
4. Azure Content Understanding: it combines extraction and chunking, supports semantic chunking, cross-page tables and AI image descriptions.
5. A multimodal embedding skill (Azure Vision or AML) with a matching vectoriser.
6. Any four: read-only identity, curated views, schema descriptions and examples, SQL validation, row limits, logging, row-level security, showing users the query.
7. Local: entity-specific questions; global: dataset-wide themes via community summaries; DRIFT: a combination of both.
8. Any two: Graph in Microsoft Fabric, Fabric ontology, Apache AGE on PostgreSQL, Cosmos DB for Apache Gremlin.

</details>

## References

- [Multimodal search in Azure AI Search](https://learn.microsoft.com/azure/search/multimodal-search-overview)
- [Quickstart: multimodal search in the portal](https://learn.microsoft.com/azure/search/search-get-started-portal-image-search)
- [GenAI Prompt skill](https://learn.microsoft.com/azure/search/cognitive-search-skill-genai-prompt)
- [Azure Vision multimodal embeddings skill](https://learn.microsoft.com/azure/search/cognitive-search-skill-vision-vectorize)
- [Fabric data agent concepts](https://learn.microsoft.com/fabric/data-science/concept-data-agent)
- [Graph in Microsoft Fabric](https://learn.microsoft.com/fabric/graph/overview)
- [Apache AGE on Azure Database for PostgreSQL](https://learn.microsoft.com/azure/postgresql/azure-ai/generative-ai-age-overview)
- [GraphRAG (Microsoft Research)](https://microsoft.github.io/graphrag/)
