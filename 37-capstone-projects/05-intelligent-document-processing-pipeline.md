# Intelligent Document Processing Pipeline

## Why it matters

Invoices, claims, contracts, onboarding forms and delivery notes still drive large parts of most businesses. Intelligent document processing (IDP) is one of the most dependable AI investments because value is measurable: fewer manual keying hours, faster cycle time, fewer errors. Production IDP is more than extraction, though. It needs classification, validation, confidence-based human review, auditing and integration with downstream systems. In this capstone you build that complete pipeline.

**Brief:** Woodgrove Insurance receives 10,000 claim documents a month by email and upload (claim forms, invoices, photos and medical letters). They want key fields extracted, validated against the policy system and posted to the claims platform, with humans reviewing only uncertain cases.

## Key concepts

### Requirements

| # | Requirement |
|---|---|
| R1 | Ingest from email attachments and a web upload into Blob Storage |
| R2 | Classify document type (claim form, invoice, medical letter, photo, other) |
| R3 | Extract fields per type (policy number, dates, amounts, line items, provider) with confidence scores |
| R4 | Validate: policy exists and is active (API), totals add up, dates are plausible, no duplicate submission |
| R5 | Route low-confidence or failed-validation documents to human review; corrections feed improvement |
| R6 | Post approved results to the claims system (API or queue) idempotently |
| R7 | Audit trail: original document, extracted values, confidence, validator results, reviewer actions |
| R8 | Throughput: handle a 3× spike; measure straight-through processing rate and field accuracy |
| R9 | Personal data protection: private networking, encryption, retention rules |

### Skills practised

Topics [20 files 06–07](../20-foundry-tools/06-azure-document-intelligence.md), [11](../11-integration-messaging-and-apis/README.md), [10](../10-application-platform/README.md), [08](../08-storage/README.md), [22](../22-evaluation-observability-and-genaiops/README.md) and [34 file 10](../34-ai-solution-architecture/10-document-and-voice-solution-architectures.md).

## How it works on Azure

### Architecture

```text
Email (Logic Apps / Graph) ┐
Web upload (App)           ┴─▶ Blob (raw) ─▶ Event Grid ─▶ Service Bus queue
     ─▶ Processor (Functions or Container Apps jobs)
          ├─ classify (Content Understanding classifier or Document Intelligence custom classifier)
          ├─ extract (Content Understanding analyzer / Document Intelligence prebuilt invoice + custom model)
          ├─ validate (rules engine + policy API via API Management + duplicate hash check)
          ├─ confidence ≥ threshold and valid ─▶ claims system (Service Bus topic / API)
          └─ else ─▶ review queue ─▶ review app (Power Apps or web) ─▶ corrections stored as labelled data
     ─▶ Cosmos DB / SQL: document record, fields, confidence, status, audit events
Monitoring: STP rate, accuracy, queue depth, latency, cost per document
```

### Extraction options

| Option | Best for |
|---|---|
| **Document Intelligence prebuilt models** (invoice, receipt, ID, layout) | Standard documents with known schemas |
| **Document Intelligence custom models** | Organisation-specific forms with stable layouts |
| **Content Understanding analyzers** | Multimodal inputs, generative field extraction, confidence and grounding, classification |
| **LLM with structured outputs** | Long-tail or free-text documents; pair with validation and evaluation |

### Milestones

| Milestone | Deliverable |
|---|---|
| M1 | Sample set of 200 documents with ground-truth labels; field definitions |
| M2 | Ingestion, queueing and the document state model |
| M3 | Classification and extraction for two document types; field-level accuracy report |
| M4 | Validation rules and duplicate detection |
| M5 | Human review app and the corrections loop |
| M6 | Downstream integration with idempotency keys |
| M7 | Load test at 3× volume; dashboard (STP rate, accuracy, cost per document) |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Confidence threshold | Set per field from measured accuracy; tune to balance review workload against error cost |
| Prebuilt vs custom vs generative | Prebuilt first; custom for stable proprietary forms; generative for variable documents |
| Functions vs Container Apps jobs | Functions for event-driven small work; Container Apps jobs for heavier or longer processing |
| Review UI | Power Apps for speed of delivery; custom web app for complex review |

## Hands-on

1. Complete milestones M1–M7.
2. Publish field-level precision and recall per document type.
3. Show the cost per document and the review rate at two thresholds.
4. Demonstrate that re-sending the same document doesn't create duplicate claims.
5. Write ADRs for the extraction approach and the confidence thresholds.

## Security, cost and reliability notes

- Classify documents as personal or health data; apply Purview labels and retention policies.
- Use private endpoints and managed identities throughout; don't send documents to the public internet.
- Use dead-letter queues and poison message handling; reprocess safely.
- Cost scales with pages: pre-filter irrelevant pages, and choose the cheapest model per document type.

## Common mistakes

| Mistake | Fix |
|---|---|
| Measuring "looks right" instead of accuracy | Labelled ground truth and field-level metrics |
| One global confidence threshold | Thresholds per field and document type |
| No idempotency | Hash documents and use idempotency keys downstream |
| Ignoring the review experience | Design the review UI with reviewers |

## Check yourself

1. Which metrics define IDP success?
2. When would you use Content Understanding rather than a Document Intelligence prebuilt model?
3. How do you prevent duplicate claims?
4. How do reviewer corrections improve the system?
5. Why route through a queue rather than processing synchronously on upload?

<details>
<summary>Answers</summary>

1. Straight-through processing rate, field-level accuracy, cycle time, review effort and cost per document.
2. For multimodal or variable documents, generative field extraction, or combined classification and extraction with confidence.
3. Hash documents for duplicate detection and use idempotency keys when posting downstream.
4. They become labelled data for retraining or tuning, and reveal validation and threshold issues.
5. Queues absorb spikes, enable retries and dead-lettering, and decouple ingestion from processing.

</details>

## References

- [Content Understanding overview](https://learn.microsoft.com/azure/ai-services/content-understanding/overview)
- [Document Intelligence overview](https://learn.microsoft.com/azure/ai-services/document-intelligence/overview)
- [Service Bus dead-letter queues](https://learn.microsoft.com/azure/service-bus-messaging/service-bus-dead-letter-queues)
- [Azure Logic Apps overview](https://learn.microsoft.com/azure/logic-apps/logic-apps-overview)
