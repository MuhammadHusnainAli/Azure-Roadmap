# Azure AI Search Fundamentals

## Why it matters

Azure AI Search is the retrieval engine behind most RAG solutions on Azure, behind Foundry IQ, and behind the Azure AI Search tool in Foundry Agent Service. Getting the fundamentals wrong (wrong tier, wrong field attributes, no replicas, a single index for everything) creates problems that are expensive to undo, because many index changes require a rebuild. Engineers need to design indexes; architects need to size and secure the service; consultants need to price it.

## Key concepts

### What the service is

Azure AI Search (formerly Azure Cognitive Search) is a managed search service that stores **indexes** and answers **queries** over them: full-text (BM25), vector, hybrid, with optional **semantic ranking** and **agentic retrieval**. It can pull data in with **indexers** and enrich it with **skillsets** (file 03), or you can **push** documents through the API.

### Core objects

| Object | What it is |
|---|---|
| **Index** | Schema plus data: a collection of documents with typed fields |
| **Document** | One searchable item (often one chunk in RAG) |
| **Field** | Typed attribute: `Edm.String`, `Collection(Edm.String)`, `Edm.Int32`, `Edm.DateTimeOffset`, `Collection(Edm.Single)` for vectors, complex types |
| **Data source** | Connection to Blob, ADLS Gen2, SQL, Cosmos DB, SharePoint, OneLake and others |
| **Indexer** | Crawler that reads a data source on a schedule and writes to an index |
| **Skillset** | Enrichment pipeline (chunking, OCR, embedding, entity extraction) run by an indexer |
| **Vectoriser** | Configuration that turns query text into a vector at query time |
| **Semantic configuration** | Fields the semantic ranker should read |
| **Scoring profile** | Rules that boost relevance (field weights, freshness, tags) |
| **Synonym map** | Equivalent terms for keyword search |
| **Analyser** | Text processing for a field (language analysers, custom analysers) |
| **Knowledge source / knowledge base** | Objects for agentic retrieval (file 07) |
| **Alias** | A stable name that points to an index, used for zero-downtime swaps |

### Field attributes

Attributes decide what you can do with a field and how much storage it uses. Changing most of them requires rebuilding the index.

| Attribute | Enables | Cost |
|---|---|---|
| `searchable` | Full-text search (tokenised, inverted index) | Storage |
| `filterable` | `$filter` (exact match, ranges, security filters) | Storage |
| `sortable` | `$orderby` | Storage |
| `facetable` | Facet counts | Storage |
| `retrievable` | Returned in results | Small |
| `stored` (vectors) | Whether raw vectors are kept for retrieval | Can set `false` to save space |

For RAG, typical fields are: `id` (key), `content` (searchable), `content_vector` (vector), `title`, `source_url`, `last_modified` (filterable, sortable), `category` (filterable, facetable), and security fields such as `group_ids` (filterable).

### Analysers

Full-text search tokenises text with an analyser. The default is Standard Lucene; Azure AI Search offers more than 50 language analysers (Lucene and Microsoft) plus custom analysers. Choose language-specific analysers for non-English content; they handle stemming and word breaking better.

## How it works on Azure

### Pricing models and tiers

| Pricing model | Billing | Best for | Status |
|---|---|---|---|
| **Dedicated** | Search units (SU = replicas × partitions) at an hourly tier rate | Steady, predictable workloads | GA |
| **Serverless (Developer tier)** | Compute units per hour plus indexed GB per month | Bursty, infrequent or variable workloads | **Preview**; billing started 13 September 2026; no SLA; limited regions; no index aliases, debug sessions or private networking for indexers |

Dedicated tiers:

| Tier | Partition storage (new services) | Vector quota per partition | Use |
|---|---|---|---|
| **Free** | Shared | Small | Tutorials; one per subscription; may be deleted when inactive |
| **Basic** | 15 GB | 5 GB | Small production; up to 3 partitions and 3 replicas on new services |
| **S1** | 160 GB | 35 GB | Default for most production |
| **S2** | 512 GB | 150 GB | Larger corpora and query loads |
| **S3** | 1 TB | 300 GB | Large |
| **S3 HD** | 1 TB | 300 GB | Many small indexes (multi-tenant SaaS) |
| **L1 / L2** | 2 TB / 4 TB | 150 / 300 GB | Large, rarely updated indexes; higher latency |

Values are for services created after May 2024 in most regions; older services or some regions (for example Israel Central, Qatar Central, Spain Central, South India) have lower limits. Maximum vector dimensions per field is 4,096 on every tier. Check the limits page for index counts per tier (for example 15 on Basic, 50 on S1, 200 on S2 and S3).

### Replicas, partitions and SLA

| Unit | Gives you |
|---|---|
| **Replica** | A copy of all indexes: more query throughput and availability |
| **Partition** | A slice of storage: more capacity (documents, vectors) and indexing throughput |

SLA rules: **two or more replicas** for read (query) SLA; **three or more replicas** for read-write (query and indexing) SLA. Partitions don't affect SLA. Free and preview features have no SLA.

### Feature availability by tier

Most features exist on all tiers, but some matter for enterprise designs: private endpoints (not on Free; skillset-based indexers using private outbound access need S2 or higher), managed identities for outbound access (not on Free), customer-managed keys (not on Free), availability zones (not on Free).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Tier** | Start with the smallest tier that fits a realistic index, then load test. Basic and S1 can switch between each other and up to S3; other changes need a new service. |
| **Dedicated vs Serverless** | Dedicated for production with SLA needs; Serverless (Preview) for dev/test and bursty, low-utilisation workloads. |
| **Replicas** | Minimum two in production (three if indexing continuously). Add more for query throughput. |
| **Partitions** | Driven by storage and **vector quota**. Vectors often hit quota before text storage does. |
| **One index or many** | One index per content domain with consistent schema; separate indexes when schemas or security models differ, or for tenant isolation (file 13). |
| **Push or pull** | Indexers for supported sources and low code; push API when data comes from events or unsupported sources. |
| **Keyless access** | Disable API keys (`disableLocalAuth`) and use Entra ID roles: Search Service Contributor, Search Index Data Contributor, Search Index Data Reader. |

## Hands-on

Create a search service with Bicep, keyless and with a managed identity:

```bicep
param location string = resourceGroup().location
param name string

resource search 'Microsoft.Search/searchServices@2025-05-01' = {
  name: name
  location: location
  sku: { name: 'standard' }
  identity: { type: 'SystemAssigned' }
  properties: {
    replicaCount: 2
    partitionCount: 1
    hostingMode: 'default'
    semanticSearch: 'standard'
    disableLocalAuth: true
    publicNetworkAccess: 'Enabled'   // use 'Disabled' plus private endpoint in production
  }
}
```

```bash
az deployment group create -g rg-rag -f search.bicep -p name=contoso-search
# give yourself data-plane access
az role assignment create --assignee $(az ad signed-in-user show --query id -o tsv) \
  --role "Search Index Data Contributor" \
  --scope $(az search service show -g rg-rag -n contoso-search --query id -o tsv)
```

Create an index in Python:

```python
from azure.identity import DefaultAzureCredential
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import (
    SearchIndex, SimpleField, SearchableField, SearchField, SearchFieldDataType,
    VectorSearch, HnswAlgorithmConfiguration, VectorSearchProfile,
    SemanticConfiguration, SemanticPrioritizedFields, SemanticField, SemanticSearch)

client = SearchIndexClient("https://contoso-search.search.windows.net", DefaultAzureCredential())
index = SearchIndex(
    name="policies",
    fields=[
        SimpleField(name="id", type=SearchFieldDataType.String, key=True),
        SearchableField(name="title", type=SearchFieldDataType.String),
        SearchableField(name="content", type=SearchFieldDataType.String, analyzer_name="en.microsoft"),
        SimpleField(name="category", type=SearchFieldDataType.String, filterable=True, facetable=True),
        SimpleField(name="last_modified", type=SearchFieldDataType.DateTimeOffset, filterable=True, sortable=True),
        SearchField(name="content_vector", type=SearchFieldDataType.Collection(SearchFieldDataType.Single),
                    searchable=True, vector_search_dimensions=1536, vector_search_profile_name="hnsw"),
    ],
    vector_search=VectorSearch(
        algorithms=[HnswAlgorithmConfiguration(name="hnsw-algo")],
        profiles=[VectorSearchProfile(name="hnsw", algorithm_configuration_name="hnsw-algo")]),
    semantic_search=SemanticSearch(configurations=[SemanticConfiguration(
        name="default",
        prioritized_fields=SemanticPrioritizedFields(
            title_field=SemanticField(field_name="title"),
            content_fields=[SemanticField(field_name="content")]))]),
)
client.create_or_update_index(index)
```

Upload a few documents, then try `search_text="leave"` with and without `filter="category eq 'HR'"`.

## Security, cost and reliability notes

- **Keyless by default.** Set `disableLocalAuth: true` and use RBAC. Query keys are shared secrets that can't express per-user permissions.
- **Network.** Use private endpoints for inbound access and shared private links for indexer outbound access in regulated environments ([Topic 05 file 05](../05-networking/05-private-link-and-private-endpoints.md)).
- **Encryption.** Data is encrypted at rest with Microsoft-managed keys; customer-managed keys are available on billable tiers.
- **Cost.** Dedicated services bill every hour regardless of traffic; each replica or partition adds a full SU. Semantic ranker and agentic retrieval have their own usage charges.
- **Reliability.** Two replicas for read SLA, three for read-write; enable availability zones where supported; keep index definitions in source control so you can rebuild.

## Common mistakes

| Mistake | Fix |
|---|---|
| Making every field searchable, filterable, sortable and facetable | Set only the attributes you need; they cost storage |
| Production on one replica | At least two |
| Sizing on text only | Check the vector quota; it is often the binding limit |
| Using admin keys in apps | Entra ID and RBAC, keys disabled |
| Editing a live index schema in place | Build a new index and switch an alias |
| Default analyser for non-English text | Use language analysers |

## Check yourself

1. Name eight objects in an Azure AI Search service.
2. What do replicas and partitions each provide?
3. How many replicas are needed for read SLA and for read-write SLA?
4. What is the difference between Dedicated and Serverless pricing, and what is Serverless's status?
5. Which tier suits a SaaS app with thousands of small tenant indexes?
6. Why might a service run out of capacity while text storage is still half empty?
7. Which RBAC role does an application need to query an index?
8. Why should you avoid changing field attributes on a live index?

<details>
<summary>Answers</summary>

1. Any eight of index, document, field, data source, indexer, skillset, vectoriser, semantic configuration, scoring profile, synonym map, analyser, alias, knowledge source, knowledge base.
2. Replicas add query throughput and availability; partitions add storage, vector quota and indexing capacity.
3. Two for read; three for read-write.
4. Dedicated bills fixed search units per hour; Serverless bills compute units and storage on consumption. Serverless (Developer tier) is in Preview with no SLA.
5. S3 HD.
6. The vector index quota per partition is separate from total storage and is often reached first.
7. Search Index Data Reader.
8. Most attribute changes require dropping and rebuilding the index; use a new index plus an alias swap.

</details>

## References

- [What is Azure AI Search?](https://learn.microsoft.com/azure/search/search-what-is-azure-search)
- [Choose a pricing model and tier](https://learn.microsoft.com/azure/search/search-sku-tier)
- [Service limits](https://learn.microsoft.com/azure/search/search-limits-quotas-capacity)
- [Estimate and manage capacity](https://learn.microsoft.com/azure/search/search-capacity-planning)
- [Connect using role-based access control](https://learn.microsoft.com/azure/search/search-security-rbac)
- [Create an index](https://learn.microsoft.com/azure/search/search-how-to-create-search-index)
