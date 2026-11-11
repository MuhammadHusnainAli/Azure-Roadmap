# Azure Document Intelligence

## Why it matters

Invoices, receipts, contracts, ID documents, tax forms, bank statements and scanned letters still drive core business processes. **Azure Document Intelligence in Foundry Tools** (formerly Form Recognizer) turns them into structured, typed data with high-accuracy OCR, layout understanding and specialised extraction models. It is also the recommended OCR successor to Vision's Read API, the engine behind Azure AI Search's Document Layout skill, and part of Microsoft's Content Understanding family. For structured and semi-structured documents at volume, it's usually more accurate, consistent and cheaper than asking an LLM to read pages.

## Key concepts

### API versions

| Version | Status | End of support |
|---|---|---|
| **v4.0 (2024-11-30)** | GA, current | None announced |
| v3.1 (2023-07-31) | GA, previous | None announced |
| v3.0 (2022-08-31) | Retiring | 30 March 2029 |
| v2.1 | Retiring | 15 September 2027 |

Migrate to v4.0 before retirement dates.

### Model families

| Family | Models | Use |
|---|---|---|
| **Document analysis** | **Read** (printed and handwritten text), **Layout** (text, tables, selection marks, paragraphs, sections, figures; Markdown output) | OCR, RAG ingestion, any document |
| **Prebuilt** | Invoice, receipt, contract, bank statement, cheque, credit card, pay stub, ID document, health insurance card, marriage certificate, business card, unified US tax and W-2/1098/1099/1040, US mortgage forms (1003, 1004, 1005, 1008, disclosure) | Common business documents without training |
| **Custom extraction** | **Custom neural** (mixed and variable layouts), **custom template** (fixed layouts), **custom composed** (collection of models) | Your own forms and fields with labelled samples |
| **Custom classification** | **Custom classifier** | Identify document type before routing to an extraction model; split multi-document files |

### Outputs and add-ons

| Feature | Notes |
|---|---|
| **Typed fields** | Values returned as string, number, date, currency, address, phone number and more, with confidence scores and bounding regions |
| **Markdown output** (Layout) | Ideal for chunking and RAG ([Topic 17 file 04](../17-rag-and-knowledge/04-document-processing-and-chunking.md)) |
| **Add-on capabilities** | High-resolution OCR, formulas, font styles, barcodes, languages, **key-value pairs**, **query fields** (ask for extra fields without training) |
| **Searchable PDF** | Read model can output searchable PDFs |
| **Batch analysis** | Analyse many documents asynchronously from Blob Storage |

### Document Intelligence vs Content Understanding vs LLMs

Microsoft's guidance (Content Understanding documentation):

| | Document Intelligence | Content Understanding | Build your own with Foundry models |
|---|---|---|---|
| Best for | Structured/semi-structured documents with common templates; consistency, low latency, proven accuracy | Unstructured, highly variable or multimodal content; inferred fields; reasoning | Niche workflows needing full control of models and prompts |
| OCR | Industry-leading | Industry-leading (uses DI underneath) | Needs preprocessing |
| Confidence and grounding | Yes | Yes | Custom implementation |
| Effort | Prebuilt or labelled training | Zero-shot schema-based extraction, no labelling | Prompting, orchestration, scaling |

Existing Document Intelligence workloads need **no migration**; Content Understanding is for new and adjacent use cases.

## How it works on Azure

Analyse an invoice with keyless authentication (Python, `azure-ai-documentintelligence`):

```python
# pip install azure-ai-documentintelligence azure-identity
from azure.identity import DefaultAzureCredential
from azure.ai.documentintelligence import DocumentIntelligenceClient
from azure.ai.documentintelligence.models import AnalyzeDocumentRequest

client = DocumentIntelligenceClient("https://contoso-foundry.cognitiveservices.azure.com/",
                                    DefaultAzureCredential())
poller = client.begin_analyze_document(
    "prebuilt-invoice", AnalyzeDocumentRequest(url_source="https://<storage>/invoices/inv-001.pdf"))
result = poller.result()

for doc in result.documents:
    f = doc.fields
    print("Vendor:", f["VendorName"].value_string if "VendorName" in f else None,
          "Total:", f["InvoiceTotal"].value_currency.amount if "InvoiceTotal" in f else None,
          "Confidence:", doc.confidence)
```

Layout to Markdown for RAG:

```python
poller = client.begin_analyze_document("prebuilt-layout",
    AnalyzeDocumentRequest(url_source=url), output_content_format="markdown")
markdown = poller.result().content
```

Tools: **Document Intelligence Studio** (try models, label and train custom models), SDKs (.NET, Python, Java, JavaScript), REST, **containers** (Read, Layout, prebuilt and custom; connected and disconnected), Azure AI Search skills, Logic Apps and Power Automate connectors.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Prebuilt vs custom** | Prebuilt first; custom neural for your own varied forms (often a few labelled samples per variation); custom template for fixed forms. |
| **Classifier + extractors** | Use a custom classifier to route mixed batches (invoice, PO, delivery note) to the right extractor. |
| **DI vs Content Understanding** | DI for templated, high-volume, consistency-critical extraction; Content Understanding for unstructured, inferred or multimodal fields. |
| **DI vs LLM** | DI for OCR, tables and typed fields with confidence; LLMs on DI's Markdown output for reasoning ("is this invoice compliant with policy X?"). |
| **Confidence thresholds** | Route low-confidence fields to human review (human-in-the-loop). |
| **Sync vs batch** | Batch analysis for backlogs; per-document calls for interactive flows. |
| **Containers** | For on-premises or disconnected processing of sensitive documents. |

## Hands-on

1. In Document Intelligence Studio, run **prebuilt-invoice** on five sample invoices; inspect fields and confidence.
2. Run the Python sample above against your own invoice in Blob Storage (grant the resource's managed identity read access).
3. Use **Layout** with Markdown output on a 20-page manual; check tables and headings; chunk it by headings.
4. Add a **query field** (for example `PurchaseOrderNumber`) without training and test it.
5. Label 5–10 samples of a custom form and train a **custom neural** model; evaluate on held-out samples.
6. Build a simple routing rule: fields under 0.8 confidence go to a review queue.

## Security, cost and reliability notes

- **Sensitive documents**: IDs, bank statements and tax forms carry personal and financial data; restrict access, use private endpoints, and set retention for results.
- **Keyless** with managed identity for both the service and storage access.
- **Data handling**: analysed documents and results are processed transiently for prebuilt analysis; training data stays in your storage.
- **Cost**: billed per page by model type (Read cheapest, prebuilt and custom higher, add-ons extra); batch large volumes and avoid re-analysing unchanged documents.
- **Limits**: file size and page limits per request; use batch for large jobs; handle 429s.
- **Versions**: move off v2.1 and v3.0 before their end dates.

## Common mistakes

| Mistake | Fix |
|---|---|
| Using an LLM for bulk OCR and table extraction | Use Read/Layout |
| Custom models where a prebuilt exists | Try prebuilt first |
| Ignoring confidence scores | Threshold and human review |
| One model for mixed document types | Custom classifier, then extractors |
| Staying on v2.1/v3.0 | Migrate to v4.0 |
| Plain-text extraction for RAG | Layout with Markdown |

## Check yourself

1. Which Document Intelligence API version is current, and which are retiring when?
2. What's the difference between Read and Layout?
3. Name eight prebuilt models.
4. Compare custom neural, custom template and composed models.
5. What does a custom classifier do?
6. What are query fields?
7. When should you use Content Understanding instead of Document Intelligence?
8. How should you handle low-confidence extractions?

<details>
<summary>Answers</summary>

1. v4.0 (2024-11-30) is current; v3.0 retires 30 March 2029 and v2.1 on 15 September 2027.
2. Read extracts printed and handwritten text; Layout adds tables, selection marks, paragraphs, sections and structure, with Markdown output.
3. Any eight: invoice, receipt, contract, bank statement, cheque, credit card, pay stub, ID document, health insurance card, marriage certificate, business card, US tax forms, US mortgage forms.
4. Custom neural handles varied layouts; custom template handles fixed layouts; composed combines several custom models under one ID.
5. Identifies document types (and can split files) so documents can be routed to the right extraction model.
6. An add-on that extracts additional named fields without training a custom model.
7. For unstructured, highly variable or multimodal content, inferred fields and reasoning without labelling.
8. Apply confidence thresholds and route low-confidence fields to human review.

</details>

## References

- [What is Azure Document Intelligence?](https://learn.microsoft.com/azure/ai-services/document-intelligence/overview)
- [Choose a Document Intelligence model](https://learn.microsoft.com/azure/ai-services/document-intelligence/concept/choose-model-feature)
- [Choose the right Foundry Tool for document processing](https://learn.microsoft.com/azure/ai-services/content-understanding/choosing-right-ai-tool)
- [Document Intelligence layout model](https://learn.microsoft.com/azure/ai-services/document-intelligence/prebuilt/layout)
- [Install and run Document Intelligence containers](https://learn.microsoft.com/azure/ai-services/document-intelligence/containers/install-run)
