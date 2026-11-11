# Azure Content Understanding

## Why it matters

Most enterprise content isn't a tidy form: it's contracts with variable clauses, scanned correspondence, call recordings, product photos, training videos, slide decks with charts. Extracting structured, trustworthy data from that mix used to require stitching together OCR, speech, vision and custom LLM prompts, then building confidence scoring and review. **Azure Content Understanding in Foundry Tools** is Microsoft's managed, generative-AI-based service for this: define a schema in an **analyser**, send documents, images, audio or video, and get structured JSON or Markdown back with **confidence scores** and **grounding**. It underpins multimodal RAG ingestion in Azure AI Search and is a suggested successor for several retiring Vision capabilities.

## Key concepts

### What it does

| Capability | Description |
|---|---|
| **Multimodal input** | Documents, images, audio and video in one service |
| **Content extraction** | OCR (via Document Intelligence technology), layout, selection marks, barcodes, formulas, transcripts, keyframes |
| **Segmentation** | Split documents (by type) or videos (into chapters) for targeted processing (`enableSegment`) |
| **Field extraction** | Schema-defined fields produced by three methods: **Extract** (values as they appear), **Classify** (choose from categories), **Generate** (inferred or summarised values) |
| **Confidence scores** | 0–1 per field (`estimateFieldSourceAndConfidence`) for straight-through processing decisions |
| **Grounding** | Source regions for each value so reviewers can verify quickly |
| **Classification** | Categorise content and route to the right analyser, in the unified Analyze API |
| **Output** | Markdown (for search and RAG) or JSON matching your schema (for automation) |

### Analysers

An **analyser** is the reusable configuration: content extraction settings, field schema, segmentation and the **model deployments** it uses. You **bring your own** Foundry model deployments (generative and embedding models); Content Understanding orchestrates them with its contextualisation layer.

| Type | Examples |
|---|---|
| **Prebuilt analysers** | `prebuilt-invoice`, `prebuilt-contract`, `prebuilt-idDocument`, `prebuilt-read`, `prebuilt-layout`, `prebuilt-imageSearch`, `prebuilt-audioSearch`, `prebuilt-videoSearch` and industry analysers (tax, procurement, call centre, media) |
| **Custom analysers** | Your schema, optionally improved with labelled training examples |
| **Classifiers** | Categorise documents before extraction |

### Versions

| API version | Status | Notes |
|---|---|---|
| **2025-11-01** | **GA** | Recommended for production: document, image, audio, video extraction; classification; labelled training |
| **2026-06-01-preview** | Preview (Build 2026 wave) | **Agentic mode** for complex reasoning over document evidence, richer classification and in-page segmentation, signature detection, document metadata, improved training, wider model support |

### Content Understanding vs Document Intelligence vs your own LLM

| Need | Choose |
|---|---|
| Templated, structured documents; consistency; low latency; proven accuracy | **Document Intelligence** ([file 06](06-azure-document-intelligence.md)) |
| Unstructured, variable, multimodal content; inferred fields; reasoning; no labelling | **Content Understanding** |
| Full control over models, prompts and pipeline for niche workflows | **Build your own** on Foundry models |

Existing Document Intelligence workloads don't need to migrate.

## How it works on Azure

Content Understanding is part of the **Microsoft Foundry resource**. Use the Foundry portal (Content Understanding tool), **Content Understanding Studio** for advanced analyser design, REST, or SDKs (Python `azure-ai-contentunderstanding`, prerelease).

REST call with a prebuilt analyser:

```bash
curl -X POST "https://contoso-foundry.services.ai.azure.com/contentunderstanding/analyzers/prebuilt-invoice:analyze?api-version=2025-11-01" \
  -H "Authorization: Bearer $(az account get-access-token --resource https://cognitiveservices.azure.com --query accessToken -o tsv)" \
  -H "Content-Type: application/json" \
  -d '{"inputs":[{"url":"https://<storage>/invoices/inv-001.pdf"}]}'
# returns an operation location; GET it until the result is ready
```

Python SDK shape:

```python
from azure.identity import DefaultAzureCredential
from azure.ai.contentunderstanding import ContentUnderstandingClient
from azure.ai.contentunderstanding.models import AnalysisInput

client = ContentUnderstandingClient(endpoint="https://contoso-foundry.services.ai.azure.com/",
                                    credential=DefaultAzureCredential())
result = client.begin_analyze(analyzer_id="prebuilt-videoSearch",
                              inputs=[AnalysisInput(url=video_url)]).result()
print(result.contents[0])          # transcript, keyframes, chapter segments, fields
```

Integrations:

- **Azure AI Search** Content Understanding skill: extraction, semantic chunking, Markdown tables, cross-page tables, figure descriptions for multimodal RAG ([Topic 17 files 03, 04, 11](../17-rag-and-knowledge/11-multimodal-and-structured-data-rag.md)).
- **Agentic retrieval knowledge sources** and **Agent Framework** context providers.
- **Foundry agents**: use extraction as a step or tool.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Prebuilt vs custom analyser** | Start prebuilt; customise the schema for your fields; add labelled examples only if accuracy needs it. |
| **Extract vs generate fields** | Prefer Extract for values that appear verbatim (auditable); Generate for summaries or inferred values, with confidence and review. |
| **Confidence thresholds** | Straight-through above a threshold; human review below; tune per field. |
| **Which models to deploy** | Content Understanding uses your Foundry deployments; balance quality and cost; keep deployments available in the same resource/region. |
| **GA vs preview** | Production on 2025-11-01; trial agentic mode and new features on preview behind flags. |
| **Content Understanding vs raw LLM** | Content Understanding adds OCR, grounding, confidence, segmentation and managed scaling; a raw LLM needs all of that built. |

## Hands-on

1. In the Foundry portal, open Content Understanding and run **prebuilt-invoice** on a sample; inspect fields, confidence and grounding regions.
2. Create a **custom analyser** for supplier contracts with fields: `parties` (extract), `governing_law` (extract), `contract_type` (classify: services, licence, NDA), `termination_risk_summary` (generate).
3. Run it on five contracts; set a confidence threshold of 0.8 and list fields that would go to review.
4. Run **prebuilt-audioSearch** on a call recording and **prebuilt-videoSearch** on a short video; compare outputs with Speech batch transcription and Video Indexer (file 08).
5. In Azure AI Search, configure the Content Understanding skill on a container of PDFs with charts and check figure descriptions in the index.

## Security, cost and reliability notes

- **Data flow**: content is processed with your Foundry model deployments; review data handling, region and deployment type (Global vs Data Zone vs Standard) for residency ([Topic 15 file 04](../15-foundry-models-and-azure-openai/04-deployment-types.md)).
- **Keyless** access with Entra; private endpoints on the Foundry resource.
- **Generated fields can be wrong**: rely on confidence, grounding and review for consequential decisions.
- **Cost**: billed per content unit (pages, minutes, images) plus the generative model tokens consumed by your deployments; segmentation and generated fields add tokens. Test on representative samples.
- **Quotas**: model deployment TPM limits affect throughput; size deployments for peak loads.
- **Preview features** have no SLA.

## Common mistakes

| Mistake | Fix |
|---|---|
| Using Content Understanding for high-volume fixed forms already handled well by DI | Keep DI |
| Treating generated fields as facts | Confidence thresholds and review |
| Forgetting to deploy the models analysers need | Deploy supported models in the resource |
| Production on preview API versions | Use 2025-11-01 GA |
| Ignoring data residency of model deployments | Choose deployment types deliberately |

## Check yourself

1. Which content types does Content Understanding process?
2. What is an analyser, and what does it configure?
3. Describe the three field extraction methods.
4. What do confidence scores and grounding provide?
5. Which API version is GA, and what does the preview add?
6. When would you choose Document Intelligence instead?
7. How does Content Understanding integrate with Azure AI Search?
8. What drives Content Understanding cost?

<details>
<summary>Answers</summary>

1. Documents, images, audio and video.
2. A reusable processing definition: content extraction settings, field schema, segmentation and model deployments.
3. Extract (values as they appear), Classify (choose from defined categories), Generate (inferred or summarised values).
4. Confidence: a 0–1 reliability estimate per field for automation decisions; grounding: the source regions so values can be verified.
5. 2025-11-01 is GA; 2026-06-01-preview adds agentic mode, richer classification and segmentation, signature detection, metadata extraction and improved training.
6. For structured or templated documents needing consistency, low latency and proven deterministic extraction.
7. Through the Content Understanding skill (extraction, semantic chunking, Markdown tables, figure descriptions) and knowledge sources for agentic retrieval.
8. Content units processed plus the generative model tokens consumed by your deployments.

</details>

## References

- [What is Azure Content Understanding?](https://learn.microsoft.com/azure/ai-services/content-understanding/overview)
- [Quickstart: Content Understanding REST and SDK](https://learn.microsoft.com/azure/ai-services/content-understanding/quickstart/use-rest-api)
- [Choose the right Foundry Tool for document processing](https://learn.microsoft.com/azure/ai-services/content-understanding/choosing-right-ai-tool)
- [Content Understanding video analysis](https://learn.microsoft.com/azure/ai-services/content-understanding/video/overview)
- [Azure Content Understanding skill in Azure AI Search](https://learn.microsoft.com/azure/search/cognitive-search-skill-content-understanding)
