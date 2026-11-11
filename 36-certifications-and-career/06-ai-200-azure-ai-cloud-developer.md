# AI-200 Azure AI Cloud Developer

## Why it matters

AI-200 (Developing AI Cloud Solutions on Azure) leads to **Microsoft Certified: Azure AI Cloud Developer Associate**. It replaced AZ-204 (Azure Developer Associate), which retired on 31 July 2026. AI-103 covers the AI layer itself. AI-200 covers what surrounds it: containerised back ends, vector-capable databases, caching, messaging, serverless functions, security and monitoring. Most production AI systems are mostly this kind of code, which makes AI-200 valuable for AI engineers and for architects who want credible developer depth.

## Key concepts

### Skills measured (study guide)

| Domain | Weight | Sub-areas |
|---|---|---|
| **Develop containerised solutions on Azure** | 20–25% | Container application hosting; container-orchestrated solutions |
| **Develop AI solutions by using Azure data management services** | 25–30% | Azure Cosmos DB for NoSQL; Azure Database for PostgreSQL; Azure Managed Redis |
| **Connect to and consume Azure services** | 20–25% | Event- and message-based AI solutions; Azure Functions |
| **Secure, monitor and troubleshoot Azure solutions** | 20–25% | Secure solutions; monitoring and troubleshooting |

The certification page says candidates should know Azure SDKs, data management services, monitoring, messaging and eventing, **vector databases**, Python and containerised apps.

### AZ-204 vs AI-200

| Area | AZ-204 (retired) | AI-200 |
|---|---|---|
| Compute | App Service, Functions, containers | Containers first (Container Apps, AKS), Functions |
| Data | Cosmos DB, Blob | Cosmos DB, PostgreSQL and Redis, used **for AI** (vectors, semantic cache, memory) |
| Integration | Event Grid, Event Hubs, Service Bus, API Management | Event- and message-based AI solutions |
| Language | C# or Python | Python |

### Roadmap mapping

| Exam area | Roadmap topics |
|---|---|
| Containers and orchestration | [07 · Containers and Kubernetes](../07-containers-and-kubernetes/README.md) |
| Cosmos DB, PostgreSQL, Managed Redis, vectors | [09 · Databases](../09-databases/README.md), with vector use in [17 · RAG and Knowledge](../17-rag-and-knowledge/README.md) |
| Functions, App Service | [10 · Application Platform](../10-application-platform/README.md) |
| Events and messaging | [11 · Integration, Messaging and APIs](../11-integration-messaging-and-apis/README.md) |
| Security and monitoring | [25 · Security](../25-security/README.md) (Key Vault, managed identity), [26 · Monitoring](../26-monitoring-and-observability/README.md) |

## How it works on Azure

### Eight-week study plan

| Week | Focus | Build |
|---|---|---|
| 1 | Python SDKs, identity | `DefaultAzureCredential` with a user-assigned managed identity |
| 2 | Container Apps | Containerised FastAPI service; revisions, scaling rules, Dapr basics |
| 3 | AKS basics | Deploy the same service to AKS with workload identity |
| 4 | Cosmos DB for NoSQL | Vector index and vector search; partition key design; change feed |
| 5 | PostgreSQL and Managed Redis | pgvector similarity search; semantic cache and session memory in Redis |
| 6 | Events and messaging | Service Bus queue for long-running AI jobs; Event Grid trigger on blob upload |
| 7 | Functions | Durable or queue-triggered function that calls a model and writes results |
| 8 | Security, monitoring, revision | Key Vault references, private endpoints, OpenTelemetry to Application Insights; practice assessment |

### Pattern to know: asynchronous AI processing

```text
Blob upload ─▶ Event Grid ─▶ Function ─▶ Service Bus queue ─▶ Container App worker
                                                              ├─ calls Foundry model
                                                              ├─ stores vectors in Cosmos DB / PostgreSQL
                                                              └─ caches results in Managed Redis
Telemetry from every hop ─▶ Application Insights (distributed tracing)
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Cosmos DB vs PostgreSQL for vectors** | Cosmos DB for global scale, document data and serverless options; PostgreSQL for relational data with pgvector. AI Search for advanced hybrid retrieval. |
| **Container Apps vs AKS** | Container Apps for most app teams; AKS when you need full Kubernetes control. |
| **Queue vs event** | Service Bus for commands that must be processed; Event Grid for reactive notifications. |
| **AI-200 vs AI-103 first** | Back-end developers start with AI-200; AI specialists start with AI-103. |

## Hands-on

1. Build the asynchronous pipeline above end to end.
2. Implement vector search in both Cosmos DB and PostgreSQL and compare.
3. Add a semantic cache in Azure Managed Redis and measure the latency saving.
4. Remove every secret from the code by using managed identity and Key Vault.
5. Trace one request across all services in Application Insights.

## Security, cost and reliability notes

- Use managed identities for every service connection; disable local authentication where supported.
- Use dead-letter queues and retries with back-off for model calls.
- Scale-to-zero (Container Apps, Functions consumption plans) keeps lab costs low.

## Common mistakes

| Mistake | Fix |
|---|---|
| Studying AZ-204 material | Use the AI-200 study guide |
| Treating vectors as an AI Search-only topic | Practise vectors in Cosmos DB and PostgreSQL |
| Synchronous calls for long AI jobs | Queue-based processing |
| Ignoring observability | Distributed tracing from day one |

## Check yourself

1. Which certification does AI-200 lead to, and what did it replace?
2. Which three data services appear in the data domain?
3. When would you choose Service Bus over Event Grid?
4. Why use Azure Managed Redis in an AI solution?
5. Which roadmap topics map to AI-200?

<details>
<summary>Answers</summary>

1. Azure AI Cloud Developer Associate; it replaced the Azure Developer Associate (AZ-204), which retired on 31 July 2026.
2. Azure Cosmos DB for NoSQL, Azure Database for PostgreSQL and Azure Managed Redis.
3. When messages are commands that must be reliably processed, with ordering, sessions or dead-lettering.
4. For semantic caching, session or conversation memory and low-latency lookups.
5. Topics 07, 09, 10 and 11, with 17, 25 and 26 in support.

</details>

## References

- [Exam AI-200](https://learn.microsoft.com/credentials/certifications/exams/ai-200/)
- [AI-200 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-200)
- [Azure AI Cloud Developer Associate](https://learn.microsoft.com/credentials/certifications/azure-ai-cloud-developer-associate/)
- [Vector search in Azure Cosmos DB for NoSQL](https://learn.microsoft.com/azure/cosmos-db/vector-search)
