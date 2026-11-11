# Product Rename Map

## Why it matters

Microsoft renames products often, and Azure AI has been renamed several times in three years. Clients, job adverts, old documentation, Terraform modules and exam material all use different generations of names. Confusing an old name with a different product leads to wrong designs and lost credibility. This file maps **old names to current names** for the services used across this roadmap, so you can read legacy material and speak current language.

Names change often. Confirm on Microsoft Learn when a name matters, for example in a contract or SoW.

## Key concepts

### AI and Foundry

| Old name(s) | Current name |
|---|---|
| Azure AI Studio → Azure AI Foundry | **Microsoft Foundry** |
| Cognitive Services → Azure AI services | **Foundry Tools** (the AI service capabilities such as Speech, Language, Vision, Translator, Document Intelligence and Content Safety) |
| Azure OpenAI Service | **Azure OpenAI in Foundry Models** (Azure OpenAI models within Foundry Models) |
| Azure AI model catalog / model-as-a-service | **Foundry Models** |
| Azure AI Agent Service | **Foundry Agent Service** |
| Azure AI User / Azure AI Project Manager roles | **Foundry User** / **Foundry Project Manager** (role IDs unchanged) |
| Form Recognizer | **Azure Document Intelligence** |
| Azure Cognitive Search | **Azure AI Search** |
| Video Indexer | **Azure AI Video Indexer** |
| Azure Health Bot / Health Bot | **Microsoft healthcare agent service** |
| Edge RAG | **Agents and Tools with Foundry Local – Agentic Retrieval** (Preview) |
| Semantic Kernel and AutoGen (for new agent work) | **Microsoft Agent Framework** (successor, see [file 04](04-legacy-ai-stack.md)) |
| Power Virtual Agents | **Microsoft Copilot Studio** |
| Bing Chat Enterprise | **Microsoft 365 Copilot Chat** |
| Teams Toolkit | **Microsoft 365 Agents Toolkit** |
| Teams App Test Tool | **Microsoft 365 Agents Playground** |
| Teams AI Library | **Teams SDK** |
| GitHub Copilot coding agent | **Copilot cloud agent** |
| Copilot in Azure | **Azure Copilot** |

### Identity and security

| Old name | Current name |
|---|---|
| Azure Active Directory (Azure AD) | **Microsoft Entra ID** |
| Azure AD External Identities, Azure AD B2C (for new projects) | **Microsoft Entra External ID** (B2C is legacy for new customers) |
| Azure AD Identity Protection, PIM, etc. | **Microsoft Entra** ID Protection, Privileged Identity Management |
| Azure Security Center, Azure Defender | **Microsoft Defender for Cloud** |
| Azure Sentinel | **Microsoft Sentinel** |
| Azure Purview, Microsoft 365 compliance | **Microsoft Purview** |
| Trusted Signing | **Artifact Signing** |
| Guest configuration (Azure Policy) | **Machine configuration** |

### Platform, data and infrastructure

| Old name | Current name |
|---|---|
| Azure Stack HCI | **Azure Local** |
| CBL-Mariner | **Azure Linux** |
| Azure Cosmos DB for MongoDB vCore | **Azure DocumentDB** |
| Azure SQL Data Warehouse (SQL DW) | **Dedicated SQL pools in Azure Synapse Analytics** (Fabric Warehouse for new work) |
| Azure Data Lake Storage Gen2 (naming) | **Data Lake Storage** (hierarchical namespace on Blob Storage) |
| Azure Marketplace and AppSource | **Microsoft Marketplace** |
| Azure Native ISV Services | **Azure Native Integrations** |
| Azure Cache for Redis (successor) | **Azure Managed Redis** (a separate service, not a rename; migrate before Azure Cache for Redis retirements) |

### Programmes and commercial

| Old name | Current name |
|---|---|
| Microsoft Partner Network | **Microsoft AI Cloud Partner Program** |
| Azure Migration and Modernization Program, Azure Innovate → Azure Accelerate | **Frontier Accelerate for Azure** (programme branding at time of writing; check with the account team) |

### Certifications (2026)

| Old exam | Current exam |
|---|---|
| AI-900 | AI-901 |
| AI-102 | AI-103 |
| AZ-204 | AI-200 |
| DP-100 | AI-300 |
| AZ-500 | SC-500 |
| PL-200 | AB-410 |

See [Topic 36 file 01](../36-certifications-and-career/01-certification-landscape.md).

## How it works on Azure

### Renames rarely change resource providers

Marketing names change faster than APIs. For example:
- Foundry resources still use the `Microsoft.CognitiveServices/accounts` resource type (kind `AIServices`).
- Entra ID APIs are still Microsoft Graph.
- Many CLI commands keep old names (`az cognitiveservices`).

When you modernise IaC or scripts, map the old name to the resource type and API version, not just the label.

```kusto
// Find Foundry / AI services resources regardless of marketing name
resources
| where type =~ 'microsoft.cognitiveservices/accounts'
| project name, kind, location, resourceGroup, sku = sku.name
```

### Rename vs replacement

| Situation | Example | Action |
|---|---|---|
| **Pure rename** | Azure AD → Entra ID | Update documents and terminology; nothing to migrate |
| **Rename plus new capabilities** | Azure AI Foundry → Microsoft Foundry (new project model) | Learn the new model; migrate where the old model is legacy |
| **Successor product** | Azure Cache for Redis → Azure Managed Redis; Semantic Kernel → Agent Framework | Plan a migration |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Which names in client documents** | Current official names, with "formerly …" on first use for readers who know the old one |
| **Code identifiers** | Don't rename variables and resources just for marketing changes; document the mapping |
| **Searching documentation** | Search both names; older Q&A and blogs use legacy names |

## Hands-on

1. Review a past design document and update product names, adding "formerly" notes.
2. Run the Resource Graph query above to find AI resources in a subscription.
3. Classify five renames you encounter as pure rename, rename plus capabilities, or successor product.
4. Build a team glossary page that links to this map ([file 07](07-glossary.md)).

## Security, cost and reliability notes

- A "rename" can hide a **successor** with a retirement date. Always check whether migration is needed.
- RBAC role display names can change while IDs stay the same. Use role IDs in automation.

## Common mistakes

| Mistake | Fix |
|---|---|
| Using old names in proposals | Current names with "formerly" notes |
| Assuming a rename means no action | Check for successor products and retirements |
| Renaming resources unnecessarily | Rename only labels and documents |
| Confusing Azure Managed Redis with a renamed Azure Cache for Redis | It's a separate service; migration is required |

## Check yourself

1. What were Microsoft Foundry's previous names?
2. What are Cognitive Services called now?
3. What's the current name for Azure AD?
4. Is Azure Managed Redis a rename of Azure Cache for Redis?
5. Why should automation use role IDs rather than role names?

<details>
<summary>Answers</summary>

1. Azure AI Studio, then Azure AI Foundry.
2. Foundry Tools (they were Azure AI services in between).
3. Microsoft Entra ID.
4. No. It's a separate successor service; you must migrate.
5. Display names change in renames, but role IDs and permissions stay the same.

</details>

## References

- [New name for Azure AD](https://learn.microsoft.com/entra/fundamentals/new-name)
- [What is Microsoft Foundry?](https://learn.microsoft.com/azure/foundry/what-is-foundry)
- [Azure Updates](https://azure.microsoft.com/updates/)
- [Azure Local overview](https://learn.microsoft.com/azure/azure-local/overview)
