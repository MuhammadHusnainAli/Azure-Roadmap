# Retired and Retiring Services

## Why it matters

Inherited Azure estates are full of services that are retired, retiring or deprecated. They show up in assessments, migrations and support calls. A missed retirement can mean an outage on a known date, a security gap once patches stop, or a rushed and expensive migration. Architects and consultants add a lot of value by spotting these early, explaining the migration target and planning the work. This file is a reference of key retirements with dates and migration targets, grouped by area.

**Dates are as published on Microsoft Learn and Azure Updates at the time of writing. Always confirm on the retirement notice and in the Advisor retirement workbook before planning.**

## Key concepts

### Application platform and compute

| Service or feature | Status and date | Migration target |
|---|---|---|
| **Azure Spring Apps** (Basic, Standard, Enterprise) | Retirement period from 17 March 2025; **retires 31 March 2028**; no new customers | Azure Container Apps (recommended), AKS, App Service |
| **Cloud Services (classic)** | **Retired 31 August 2024** | Cloud Services (extended support) was the bridge; prefer modern services |
| **Cloud Services (extended support)** | Deprecated 31 March 2025; **retires 31 March 2027** | VM scale sets, App Service, Container Apps, AKS, Service Fabric |
| **Classic (ASM) VMs and classic resources** | Retired (classic VMs 2023, classic storage accounts 2024) | Azure Resource Manager equivalents |
| **App Service, Functions and Logic Apps on Arc-enabled Kubernetes** | **Retired 31 March 2026** | Container Apps on Arc or other hybrid options ([Topic 31](../31-hybrid-multicloud-and-edge/README.md)) |
| **Azure Functions in-process model (.NET)** | End of support **10 November 2026** | Isolated worker model |
| **Functions Consumption plan** | Legacy; retirement announced (September 2028) | Flex Consumption |
| **Visual Studio App Center** | Retired 31 March 2025 | GitHub Actions or Azure Pipelines plus alternatives for distribution |

### Data and caching

| Service | Status and date | Migration target |
|---|---|---|
| **Azure Database for MariaDB** | Retired (September 2025) | Azure Database for MySQL flexible server |
| **Azure Database for MySQL / PostgreSQL Single Server** | Retired (2024 and 2025 respectively) | Flexible server |
| **Azure Cache for Redis** | Enterprise and Enterprise Flash retire first (**March 2027** announced); Basic, Standard and Premium later (**September 2028** announced) | **Azure Managed Redis** |
| **Azure Data Lake Analytics** and **Data Lake Storage Gen1** | **Retired 29 February 2024** | Synapse or Fabric for analytics; Data Lake Storage (Gen2) |
| **Azure Time Series Insights** | **Retired 7 July 2024** | Azure Data Explorer, Fabric Real-Time Intelligence |
| **HDInsight 5.0 and older versions** | HDInsight 5.0 retired 31 March 2025 | HDInsight 5.1, or Fabric or Databricks ([file 05](05-legacy-data-and-compute.md)) |
| **General-purpose v1 storage accounts** | Retiring (October 2026) | Upgrade to general-purpose v2 |

### Media, IoT and specialised

| Service | Status and date | Migration target |
|---|---|---|
| **Azure Media Services** | **Retired 30 June 2024** | Partner solutions; Video Indexer for analysis |
| **Azure API for FHIR** | **Deprecated 30 September 2026** | FHIR service in Azure Health Data Services |
| **IoT Edge 1.5** | Support ends **10 November 2026** | IoT Edge 1.6 LTS |
| **Azure Stack Edge Pro FPGA** | End of life February 2024 | Pro 2 or Pro GPU |

### AI services (Foundry Tools and Azure OpenAI)

| Service or feature | Status and date | Migration target |
|---|---|---|
| **Anomaly Detector** | No new resources since 20 September 2023; **retires 1 October 2026** | Fabric (integrates the open-source anomaly-detector) or the open-source package |
| **Personalizer, Metrics Advisor** | Retiring 1 October 2026 (check notice) | Custom ML, Fabric, or partner solutions |
| **LUIS** | Retired | Conversational language understanding (CLU) or LLM-based intent detection |
| **QnA Maker** | Retired (March 2025) | Custom question answering, or RAG |
| **Content Moderator** | Retiring (check notice) | Azure AI Content Safety and Foundry guardrails |
| **Bing Search APIs** | Retired August 2025 | Grounding with Bing Search in Foundry Agent Service, or other search APIs |
| **Assistants API** (Azure OpenAI) | **Retired** | Foundry Agent Service and the Responses API ([file 04](04-legacy-ai-stack.md)) |
| **Prompt flow** (Foundry and Azure Machine Learning) | **Retires 20 April 2027** | Microsoft Agent Framework, code-first orchestration |
| **Foundry visual workflows** (Preview) | **Retires 1 December 2026** | Agent Framework workflows, Logic Apps |
| **Stored completions / distillation in the classic portal** | Retired 15 October 2026 | Responses API and agent traces |
| **Content Safety groundedness detection** and some other features | **Retire 30 April 2027** | Foundry evaluators or custom grounding checks |
| **Image Analysis 4.0 and Custom Vision** | **Retire 25 September 2028** | Multimodal models, Content Understanding, AutoML for images |
| **Specific model versions** | Fixed retirement dates per version (for example gpt-4o 2024-05-13 on 14 January 2027) | Newer versions; evaluate before auto-upgrade |

### Networking

| Feature | Status and date | Migration target |
|---|---|---|
| **Basic SKU public IP addresses** | **Retired 30 September 2025** | Standard SKU |
| **Basic Load Balancer** | Retired 30 September 2025 | Standard Load Balancer |
| **Application Gateway v1** | **Retired 28 April 2026** | Application Gateway v2 |
| **Default outbound access** | Retired for new VNets from 31 March 2026 | NAT Gateway, Firewall or Load Balancer outbound rules |
| **NSG flow logs** | **Retire 30 September 2027** | VNet flow logs |
| **Azure Front Door (classic)**, **Azure CDN Standard from Microsoft (classic)** | Retire 31 March 2027 and 30 September 2027 | Azure Front Door Standard or Premium |
| **kubenet in AKS** | Retiring (March 2028) | Azure CNI Overlay |

### Governance, security, operations and DevOps

| Service or feature | Status and date | Migration target |
|---|---|---|
| **Azure Blueprints** | **Retires 31 January 2027** (phased wind-down from July 2026) | Template specs or Bicep plus Azure Policy; Deployment Stacks |
| **Log Analytics agent (MMA/OMS)** | Retired (August 2024) | Azure Monitor Agent with data collection rules |
| **Automation Run As accounts** | Retired | Managed identities |
| **Azure Disk Encryption** | **Retires 15 September 2028** | Encryption at host |
| **Docker Content Trust in ACR** | Removed **31 March 2028** | Notary Project signing |
| **Sentinel in the Azure portal** | Moves to the Defender portal; Azure portal experience ends after 31 March 2027 | Microsoft Defender portal |
| **Managed Grafana Essential tier** | Deprecated by 31 March 2027 | Standard tier |
| **Microsoft Dev Box** | **Retires 18 September 2028** | Check retirement notice for alternatives |
| **Azure Deployment Environments** | **Retires 22 February 2027** | Platform engineering with IaC and pipelines |
| **Azure Data Studio** | Retired in 2026 | VS Code with the MSSQL extension |
| **Integration Service Environment** (Logic Apps) | Retired August 2024 | Logic Apps Standard |

## How it works on Azure

### Finding affected resources

1. Open **Azure Advisor → Workbooks → Service Retirement** for each tenant or subscription scope.
2. Use **Service Health → Health advisories** filtered to retirement notices.
3. Run Resource Graph queries for known resource types and SKUs.

```kusto
// Example: Basic SKU public IPs and Application Gateway v1 remaining
resources
| where (type =~ 'microsoft.network/publicipaddresses' and sku.name =~ 'Basic')
     or (type =~ 'microsoft.network/applicationgateways' and properties.sku.tier in~ ('Standard','WAF'))
| project name, type, resourceGroup, subscriptionId, location
```

### Migration planning template

| Field | Example |
|---|---|
| Service and resources affected | 12 Azure Cache for Redis Premium caches |
| Retirement date | September 2028 (announced) |
| Target | Azure Managed Redis |
| Effort and risk | Medium: client library settings, networking, data migration |
| Deadline | Three months before retirement at the latest |
| Owner | Platform team |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Like-for-like vs modernise** | Use retirements as a trigger to modernise when the effort is similar (for example Spring Apps → Container Apps) |
| **Timing** | Migrate early for low-risk items; schedule complex migrations with buffer |
| **New designs** | Never propose deprecated or retiring services for new work |

## Hands-on

1. Run the Advisor retirement workbook on a subscription and list the top five retirements.
2. Write a migration plan for one retiring service using the template.
3. Check a past proposal or design for any service in these tables.
4. Create a Resource Graph query for one retiring SKU in your estate.

## Security, cost and reliability notes

- After retirement, services may stop working or stop receiving security updates: both are risks.
- Migrations often reduce cost (for example newer SKUs and serverless) but can add networking or licensing changes. Estimate both.
- Test migrations in non-production first, and keep rollback options until cut-over is proven.

## Common mistakes

| Mistake | Fix |
|---|---|
| Relying on memory of dates | Check official notices; dates sometimes move |
| Treating deprecated as still fine | Plan migration when deprecation is announced |
| Ignoring feature-level retirements | Track features (for example groundedness detection) as well as services |
| Forgetting model version retirements | Monthly model lifecycle review |

## Check yourself

1. What's the migration target for Azure Spring Apps, and when does it retire?
2. Which service replaces Azure Cache for Redis?
3. What replaced the Assistants API?
4. When do NSG flow logs retire, and what replaces them?
5. How do you find retirements affecting your resources?

<details>
<summary>Answers</summary>

1. Azure Container Apps (recommended); retirement is on 31 March 2028.
2. Azure Managed Redis.
3. Foundry Agent Service, using the Responses API.
4. 30 September 2027; VNet flow logs.
5. Azure Advisor service retirement workbook, Service Health advisories and Resource Graph queries.

</details>

## References

- [Service retirement workbook in Azure Advisor](https://learn.microsoft.com/azure/advisor/advisor-workbook-service-retirement)
- [Azure Spring Apps retirement announcement](https://learn.microsoft.com/azure/spring-apps/basic-standard/retirement-announcement)
- [Cloud Services (extended support) overview](https://learn.microsoft.com/azure/cloud-services-extended-support/overview)
- [Upgrade Basic public IP addresses](https://learn.microsoft.com/azure/virtual-network/ip-services/public-ip-basic-upgrade-guidance)
- [Model deprecations and retirements](https://learn.microsoft.com/azure/foundry/concepts/model-lifecycle-retirement)
- [Azure Updates](https://azure.microsoft.com/updates/)
