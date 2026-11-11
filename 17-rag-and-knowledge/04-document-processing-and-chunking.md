# Document Processing and Chunking

## Why it matters

Retrieval quality is capped by chunk quality. If a table is flattened into a stream of numbers, if a heading is separated from its paragraph, or if a chunk mixes three unrelated topics, no embedding model or reranker can recover the meaning. Most "the AI can't find it" complaints in RAG trace back to parsing and chunking. This file covers how to turn real documents (PDFs, Office files, scans, HTML) into chunks that retrieve well and carry the metadata the rest of the system needs.

## Key concepts

### Why chunk at all?

1. **Model input limits.** For example, text-embedding-3-small accepts up to 8,191 tokens.
2. **Precision.** A single vector for a long, multi-topic page is a blurred average; smaller, coherent chunks match specific questions better.
3. **Token budget at generation time.** You want to pass the relevant paragraph, not the whole manual.

### Parsing comes first

| Content | Parsing need | Azure option |
|---|---|---|
| Clean text, Markdown, HTML | Basic extraction | Indexer document cracking; Markdown parsing mode |
| Digital PDFs and Office files with structure | Headings, sections, reading order, tables | **Document Layout skill** (Document Intelligence layout model) or **Azure Content Understanding skill** |
| Scanned documents | OCR | OCR skill, Document Layout, Content Understanding |
| Tables | Preserve rows and columns | Content Understanding outputs tables as Markdown (and handles cross-page tables); Layout outputs them as text |
| Images, charts, diagrams | Describe or embed | Content Understanding figure descriptions (Preview), GenAI Prompt skill for verbalisation, multimodal embeddings (file 11) |
| JSON, CSV | One record per document | JSON and delimited-text parsing modes (one-to-many) |

Microsoft's comparison notes that the Content Understanding skill outputs tables and figures as Markdown (easier for LLMs), handles tables spanning pages, lets chunks span pages via semantic units, and is cheaper than the Document Layout skill.

### Chunking techniques

| Approach | Description | Built-in support |
|---|---|---|
| **Fixed size** | Equal-length chunks with overlap | Text Split skill, `pages` mode |
| **Sentence or structure-based** | Split on sentences, paragraphs or headings | Text Split `sentences` mode; Document Layout by section; Markdown parsing |
| **Semantic** | Coherent units that may span pages | Content Understanding skill |
| **Custom combinations** | For example, section-based chunks with the document title prepended | Custom skill |
| **Document parsing** | One-to-many indexing of JSON, CSV, Markdown | Indexer parsing modes |

### Starting parameters (Microsoft guidance)

| Parameter | Suggested start |
|---|---|
| Chunk size | **512 tokens** (about 2,000 characters) |
| Overlap | **25%** (about 128 tokens) |
| Text Split skill defaults when using characters | `textSplitMode: pages`, `maximumPageLength: 2000`, `pageOverlapLength: 500` |

`pageOverlapLength` must be less than half of `maximumPageLength`. Token-based splitting is available in preview API versions. Structured data needs less overlap; narrative text benefits from more. Treat these as starting points and tune with evaluation (file 12).

### Chunk enrichment and metadata

Every chunk should carry enough context to stand alone and to be filtered:

| Field | Why |
|---|---|
| `parent_id`, `chunk_id`, `chunk_order` | Reassemble neighbours, deduplicate, cite |
| `title`, `section_heading` (header path) | Context for embedding and semantic ranker; display |
| `source_url`, `page_number` | Citations users can click |
| `last_modified`, `version` | Freshness filters and boosting |
| `category`, `product`, `region`, `language` | Filters |
| `group_ids` / ACL fields | Security trimming (file 10) |
| `content_type` (text, table, image) | Different handling at query time |

A common technique is to **prepend the document title and section heading** to the chunk text before embedding, so chunks from the middle of a document keep their context.

### Parent–child (small-to-big) retrieval

Retrieve on small, precise chunks; send the model a larger window around them (the parent section, or neighbouring chunks by `chunk_order`). Index projections (file 03) already store the parent key needed for this.

## How it works on Azure

### Layout-aware chunking with the Document Layout skill

The Document Layout skill calls Document Intelligence layout (v4.0) and can output **Markdown by section** (`outputMode: oneToMany`, `markdownHeaderDepth`), so each chunk is a heading-scoped section with its header path. It times out for documents needing more than five minutes of layout processing; charges still apply.

### Content Understanding skill

Bound to a billable Foundry resource (no free allowance). Produces Markdown, cross-page tables, semantic chunks and, in preview API versions, AI descriptions of embedded figures. It replaces the need for a separate Text Split skill.

### Outside the indexer

If you push content, you can parse with Document Intelligence or Content Understanding directly ([Topic 20 files 06 and 07](../20-foundry-tools/06-azure-document-intelligence.md)) and chunk with libraries (LangChain text splitters, or your own Markdown-header splitter), then upload.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Fixed vs structure-aware** | Fixed-size is fine for homogeneous prose. Use structure-aware chunking for manuals, policies, contracts and anything with headings and tables. |
| **Chunk size** | Smaller chunks raise precision but lose context; larger chunks keep context but blur embeddings. Test 256, 512 and 1,024 tokens on your golden set. |
| **Overlap** | Prevents answers being split across boundaries. Too much inflates the index and returns near-duplicates. |
| **Tables** | Keep a table in one chunk with its caption and heading; for very large tables, split by row groups and repeat the header row. Consider structured-data RAG (file 11). |
| **Images** | Verbalise important figures to text, or use multimodal embeddings; skip decorative images to save cost. |
| **Layout vs Content Understanding** | Content Understanding for Markdown tables, cross-page tables and cost; Layout where it is already established or region constraints apply. |
| **Re-chunking** | Changing chunking means re-indexing everything; get it right on a sample before a full load. |

## Hands-on

Compare chunking strategies on the same 10 documents.

```python
# Fixed-size baseline with a header-aware variant (push model)
import re, tiktoken
enc = tiktoken.get_encoding("cl100k_base")

def fixed_chunks(text, size=512, overlap=128):
    toks = enc.encode(text)
    step = size - overlap
    return [enc.decode(toks[i:i + size]) for i in range(0, len(toks), step)]

def header_chunks(markdown, title, max_tokens=512):
    chunks, path = [], []
    for block in re.split(r"(?m)^(?=#{1,3} )", markdown):
        m = re.match(r"^(#{1,3}) (.+)", block)
        if m:
            level = len(m.group(1)); path = path[:level - 1] + [m.group(2).strip()]
        prefix = f"{title} > {' > '.join(path)}\n"
        for piece in fixed_chunks(block, max_tokens, 64):
            chunks.append({"content": prefix + piece, "section": " > ".join(path)})
    return chunks
```

1. Get Markdown for each PDF from Document Intelligence layout or Content Understanding.
2. Index three variants into three indexes: fixed 512/128 on plain text, fixed 2,000/500 characters (Text Split defaults), header-aware.
3. Run the same 30 questions against each and record whether the correct chunk appears in the top 5 (recall@5).
4. Note which question types fail with which strategy (tables, cross-section questions, definitions).

## Security, cost and reliability notes

- **Metadata propagation.** Permission and classification fields must be copied to every chunk; a chunk without ACLs is a leak.
- **PII in chunks.** Consider the PII detection skill to mask or flag sensitive content before indexing.
- **Cost.** Layout and Content Understanding are billed per page; figure descriptions call an LLM per image. Sample first and skip unnecessary pages and images.
- **Index size.** Overlap and small chunks multiply document count and vector storage; check the vector quota (file 02).
- **Timeouts.** Very large documents can exceed skill time limits; split files upstream.

## Common mistakes

| Mistake | Fix |
|---|---|
| Plain-text extraction of table-heavy PDFs | Use layout-aware parsing with Markdown tables |
| Chunks without titles or headings | Prepend title and header path |
| One chunk size for every content type | Tune per source type |
| No chunk IDs or page numbers | Store them for citations and neighbour expansion |
| Indexing decorative images | Filter images by size or position before verbalising |
| Choosing chunk size by intuition | Measure recall on a golden set |

## Check yourself

1. Give three reasons to chunk documents.
2. What starting chunk size and overlap does Microsoft suggest?
3. What constraint applies to `pageOverlapLength` in the Text Split skill?
4. Name three advantages of the Content Understanding skill over the Document Layout skill.
5. Why prepend the title and section heading to each chunk?
6. What is parent–child (small-to-big) retrieval?
7. Which metadata fields should every chunk carry?
8. Why should you validate chunking on a sample before a full index load?

<details>
<summary>Answers</summary>

1. Model input limits, retrieval precision (coherent vectors), and generation token budget.
2. 512 tokens (about 2,000 characters) with 25% overlap (about 128 tokens).
3. It must be less than half of `maximumPageLength`.
4. Any three: tables and figures as Markdown, cross-page tables as one unit, semantic chunks spanning pages, lower cost, AI figure descriptions.
5. So chunks from the middle of a document keep their context for embedding, ranking and display.
6. Retrieve with small precise chunks, then pass the model a larger surrounding window (parent section or neighbours).
7. IDs and order, title and heading path, source URL and page, timestamps or version, filter categories, ACL fields, content type.
8. Changing chunking later requires re-indexing (and re-embedding) the entire corpus, which is slow and costly.

</details>

## References

- [Chunk large documents for RAG and vector search](https://learn.microsoft.com/azure/search/vector-search-how-to-chunk-documents)
- [Document Layout skill](https://learn.microsoft.com/azure/search/cognitive-search-skill-document-intelligence-layout)
- [Azure Content Understanding skill](https://learn.microsoft.com/azure/search/cognitive-search-skill-content-understanding)
- [Define index projections](https://learn.microsoft.com/azure/search/search-how-to-define-index-projections)
- [Chunking strategies (Azure Architecture Center)](https://learn.microsoft.com/azure/architecture/ai-ml/guide/rag/rag-chunking-phase)
- [azure-search-vector-samples](https://github.com/Azure/azure-search-vector-samples)
