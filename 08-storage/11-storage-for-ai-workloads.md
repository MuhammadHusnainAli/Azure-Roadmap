# Storage for AI Workloads

> Topic: 08 · Storage · Tracks: E ● A ● C ○ · Status checked: 2026-10

## Why it matters

AI systems are data systems. A RAG application is only as good, and only as secure, as the documents it indexes. A fine-tuning job needs curated, versioned training data. Training runs need fast access to large datasets and safe storage for checkpoints. Foundry and Agent Service projects store files, threads and outputs. Designing that storage well determines retrieval quality, permission enforcement, reproducibility and cost. This file brings the rest of the topic together for AI.

## Key concepts

### What AI workloads store

| Data | Typical store | Notes |
|---|---|---|
| RAG source documents | Blob or Data Lake, SharePoint, OneLake | The system of record; indexed into Azure AI Search or Foundry IQ |
| Extracted text, chunks, enrichment cache | Blob (indexer knowledge store or your own pipeline) | Makes re-indexing cheaper |
| Embeddings and indexes | Azure AI Search, or a vector-capable database (topic 09) | Derived data, can be rebuilt |
| Agent files (uploads to agents, code interpreter outputs) | Foundry-managed storage, or your own storage account in a standard (bring-your-own) agent setup | Customer data; subject to retention and access rules |
| Conversation threads and memory | Cosmos DB (Foundry standard setup or your own) | Topic 09 file 10 |
| Fine-tuning and evaluation datasets | Blob or Data Lake (JSONL), registered as Foundry or Azure ML data assets | Versioned |
| Training data at scale | Data Lake (Parquet, WebDataset shards), Managed Lustre for throughput | |
| Checkpoints and model artefacts | Blob (premium for fast writes), ACR for OCI-packaged models, Azure ML registries | |
| Prompts, evaluation results, traces | Blob, Application Insights, Foundry | Keep for audit and regression testing |

### A RAG corpus layout

```text
stragcorpus (ZRS, HNS on, private endpoints, shared key off, Defender malware scanning)
 ├─ landing/                        uploads from users and connectors (scanned, short retention)
 ├─ corpus/                         approved source documents (versioning, soft delete)
 │    ├─ hr/          policies/…    ← ACLs or ABAC: hr-readers group
 │    ├─ finance/     reports/…     ← finance-readers group
 │    └─ public/      handbook/…    ← all-employees group
 │       metadata + index tags: dept, classification, owner, effective_date, source_url
 ├─ derived/                        extracted text, chunks, images (rebuildable)
 └─ eval/                           golden question sets and expected answers
```

Design principles:

| Principle | Why |
|---|---|
| **Separate landing from corpus** | Scan and approve before content can influence answers |
| **Mirror access boundaries in the layout** | Folder or container per permission group makes security trimming and ACL inheritance simple |
| **Carry metadata into the index** | Department, classification, date and source URL enable filters, citations and freshness |
| **Keep derived data rebuildable** | Re-chunk and re-embed when models or strategies change |
| **Version the corpus** | Reproduce what the system knew at a point in time; roll back bad uploads |
| **Event-driven ingestion** | Blob events trigger incremental indexing; deletes must propagate to the index |

### Permission-aware retrieval

Indexing everything into one index and letting the model "decide" what a user may see is a data leak. Options:

| Approach | How |
|---|---|
| Security trimming with filters | Store allowed group IDs per chunk in the index; filter queries by the user's groups |
| **Document-level access control** in Azure AI Search | Index ACL and RBAC metadata from Data Lake (and other sources), and enforce at query time using the user's token (check current status and supported sources) |
| Separate indexes per audience | Simple, but duplicates data and cost |
| Foundry IQ knowledge sources with permission inheritance | Knowledge bases that respect source permissions (topic 17) |

Data Lake ACLs on the corpus are what such features read, which is a strong reason to enable hierarchical namespace for RAG corpora.

### Foundry and Agent Service storage

| Setup | Storage behaviour |
|---|---|
| **Basic** (Microsoft-managed) agent setup | Files and threads stored in Microsoft-managed storage, logically isolated per project |
| **Standard** (bring your own resources) setup | Agent files go to **your storage account**, threads to **your Cosmos DB**, vector stores to **your Azure AI Search**, all under your network and key controls |
| Foundry project | Connected storage for datasets, evaluation outputs, and tracing exports |

Regulated customers usually require the standard setup with private networking so that all agent data stays in resources they control. Check the current Foundry documentation for exact resource requirements.

### Training data and checkpoints

| Need | Storage choice |
|---|---|
| Datasets of millions of small files | Package into shards (WebDataset tar, Parquet, TFRecord) in Blob or Data Lake; small files are slow and costly |
| Fast data loading on single nodes | Premium block blob, BlobFuse2 with local caching, or local NVMe staging |
| Many GPU nodes reading in parallel | Azure Managed Lustre linked to Blob (file 08) |
| Checkpoints | Write frequently to local NVMe or Lustre, copy to Blob asynchronously; versioning on; lifecycle to cool after the run |
| Final models | Blob with immutability for released versions, Azure ML or Foundry registries, or ACR as OCI artefacts |
| Lineage | Register data assets with versions; record dataset version in each run |

## How it works on Azure

### Event-driven, permission-aware ingestion

```text
User or connector uploads to landing/
  → Defender malware scan (index tag: clean / malicious)
  → Event Grid (BlobCreated, tag filter clean) → Service Bus
  → Container Apps job / Function:
       move to corpus/<dept>/ with metadata and index tags
       (ACL inherited from folder default ACL)
  → Azure AI Search indexer (Data Lake source, ACL ingestion) or push API
       chunk, embed (Foundry embeddings model, keyless), index with metadata
  → BlobDeleted events → delete documents from the index
```

### Keyless access from Azure AI Search to the corpus

```bash
# Give the search service a managed identity and data read access to the corpus
az search service update -g $RG -n $SEARCH --identity-type SystemAssigned
SEARCH_PRINCIPAL=$(az search service show -g $RG -n $SEARCH --query identity.principalId -o tsv)
az role assignment create --assignee-object-id $SEARCH_PRINCIPAL --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Reader" --scope $(az storage account show -g $RG -n $SA --query id -o tsv)

# Let the search service through the storage firewall precisely (resource instance rule)
az storage account network-rule add -g $RG --account-name $SA \
  --resource-id $(az search service show -g $RG -n $SEARCH --query id -o tsv) --tenant-id $(az account show --query tenantId -o tsv)
```

The indexer data source then uses a connection string of the form `ResourceId=<storage-account-resource-id>;` with no key.

### Recording metadata at upload

```python
container.upload_blob(
    name="corpus/hr/policies/leave-policy-2026.pdf", data=f, overwrite=True,
    metadata={"title": "Leave policy 2026", "owner": "hr-policy-team",
              "effective_date": "2026-01-01", "source_url": "https://intranet/hr/leave"},
    tags={"dept": "hr", "classification": "internal", "status": "approved"},
)
```

Index the metadata fields as filterable and retrievable so the application can filter by department or date and cite sources.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| HNS for RAG corpora | Yes, for ACL-based security trimming and atomic reorganisation |
| Corpus in Blob vs SharePoint | Use the source where it lives when connectors support permission-aware indexing; copy to Blob when you need control, transformation or approval workflows |
| Foundry agent setup | Standard (bring your own storage, Cosmos DB, AI Search) for regulated or enterprise use; basic for prototypes |
| Data residency | Keep corpus, index and model deployment in approved regions; note where each processes data |
| Retention | Define retention for landing, derived data, agent files and conversation history; align with privacy obligations and deletion requests |
| Redundancy | ZRS for corpora; GZRS where the AI system has DR requirements. Derived data can be LRS if rebuildable |
| Performance | Premium block blob for high-throughput small-object workloads; Lustre for distributed training |

## Hands-on

1. Create a Data Lake account with the corpus layout above and two Entra groups, `rag-hr-readers` and `rag-finance-readers`. Set default ACLs on `corpus/hr` and `corpus/finance`.
2. Upload five documents to each folder with metadata and index tags.
3. Create an Azure AI Search service with a managed identity, grant it read access, and create a Data Lake data source and indexer that maps `metadata_storage_path`, your metadata fields and the tags into the index. Use the keyless `ResourceId=` connection string.
4. Query with a filter on `dept eq 'hr'` and confirm only HR documents return.
5. Delete one blob and confirm your deletion detection policy removes it from the index on the next run.
6. Extension: enable the document-level access control feature for Data Lake in Azure AI Search (if available in your region) and test with two users in different groups.

## Security, cost and reliability notes

- **Security:** the AI system can expose anything it can read. Scope the indexer's and agent's identities to the corpus they need, and enforce user permissions at query time.
- **Security:** scan and approve uploads before indexing; prompt injection in documents is a real attack (topic 23).
- **Security:** disable shared key, use private endpoints for `blob` and `dfs`, and use resource instance rules for Azure AI Search and Foundry.
- **Cost:** re-indexing and re-embedding large corpora costs tokens. Cache extracted text and only re-embed changed content.
- **Cost:** apply lifecycle rules to landing, derived data, old checkpoints and agent file uploads.
- **Reliability:** the index is derived; the corpus is the source of truth. Protect the corpus with versioning and soft delete; keep the ability to rebuild indexes from it.

## Common mistakes

- One flat container with every document and no permission metadata.
- Letting users upload directly into the indexed corpus without scanning.
- Not propagating deletions to the index, so removed documents keep appearing in answers.
- Key-based connections from AI Search or Foundry to storage.
- Training directly on millions of tiny files.
- Keeping all checkpoints forever in the hot tier.
- Choosing the basic agent setup for a regulated customer whose data must stay in their own resources.

## Check yourself

1. Why separate a landing area from the indexed corpus?
2. How does folder structure help permission-aware retrieval?
3. What does the Foundry Agent Service standard setup change about where data is stored?
4. Why should derived data (chunks, embeddings) be treated as rebuildable?
5. How should Azure AI Search authenticate to a locked-down storage account?
6. What storage pattern suits checkpoints during a distributed training run?
7. What must happen in the index when a source document is deleted?

<details>
<summary>Answers</summary>

1. To scan, validate and approve content before it can influence AI answers, preventing malware and poisoned or unapproved documents from entering the corpus.
2. Folders per permission group let ACLs (and default ACLs) map cleanly to who may read content, and those ACLs can be carried into the index for security trimming.
3. Agent files, threads and vector stores live in the customer's own storage account, Cosmos DB and AI Search, under their network and key controls.
4. Chunking strategies and embedding models change; being able to regenerate them from the corpus avoids lock-in and makes improvements possible.
5. With its managed identity granted a Blob data reader role, a resource instance rule (or private endpoint via shared private link) through the firewall, and a keyless `ResourceId=` connection string.
6. Frequent writes to fast local NVMe or Managed Lustre, asynchronous copy to Blob, versioning on, and lifecycle rules to move old checkpoints to cheaper tiers.
7. The corresponding documents and chunks must be removed, through deletion detection or explicit delete calls.

</details>

## References

- [RAG solution design and evaluation guide](https://learn.microsoft.com/azure/architecture/ai-ml/guide/rag/rag-solution-design-and-evaluation-guide)
- [Index data from Azure Data Lake Storage (AI Search)](https://learn.microsoft.com/azure/search/search-howto-index-azure-data-lake-storage)
- [Document-level access control in Azure AI Search](https://learn.microsoft.com/azure/search/search-document-level-access-overview)
- [Connect to storage with a managed identity (AI Search)](https://learn.microsoft.com/azure/search/search-howto-managed-identities-storage)
- [Foundry Agent Service: standard setup with your own resources](https://learn.microsoft.com/azure/foundry/agents/concepts/standard-agent-setup)
- [Data assets in Azure Machine Learning](https://learn.microsoft.com/azure/machine-learning/concept-data)
- [Azure Managed Lustre for AI training](https://learn.microsoft.com/azure/azure-managed-lustre/amlfs-overview)
