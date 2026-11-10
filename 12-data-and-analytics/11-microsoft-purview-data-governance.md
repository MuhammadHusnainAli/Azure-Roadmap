# Microsoft Purview Data Governance

## Why it matters

AI makes data governance urgent. An agent with access to a lakehouse, a SharePoint library and a CRM can surface any of it in a single answer. Executives ask "what data do we have, who owns it, is it any good, and who's allowed to use it for AI?" **Microsoft Purview** is Microsoft's family of data security, governance and compliance solutions. Its **data governance** capabilities, mainly the **Unified Catalog** and **Data Map**, answer those questions across Azure, Fabric, on-premises and multicloud sources. Consultants run data governance assessments; architects design how Purview fits the platform.

## Key concepts

### The Purview family (context)

| Area | Examples | Covered in |
|---|---|---|
| **Data governance** | Data Map, Unified Catalog, data quality, lineage, data products | This file |
| **Data security** | Information Protection (sensitivity labels), Data Loss Prevention, Insider Risk Management, DSPM for AI | Topics 23, 25 |
| **Risk and compliance** | Audit, eDiscovery, Data Lifecycle Management, Compliance Manager, Communication Compliance | Topics 23, 25 |

All are managed from the **Microsoft Purview portal**. Sensitivity labels, classifications and policies are shared across these areas, so governance and security reinforce each other.

### Data Map

The **Data Map** is the metadata foundation:

| Concept | Meaning |
|---|---|
| **Data sources and scans** | Registered sources (Azure SQL, ADLS, Synapse, Databricks Unity Catalog, Fabric, Power BI, SQL Server, Oracle, SAP, Snowflake, Amazon S3 and many more) scanned on a schedule for metadata |
| **Collections and domains** | Organise assets and delegate permissions |
| **Classifications** | Automatic detection of data types (credit card numbers, national IDs, emails...) using built-in and custom classifiers |
| **Sensitivity labels** | Microsoft Information Protection labels applied to data assets (and flowing into Fabric and Power BI) |
| **Lineage** | Automatic lineage from Azure Data Factory, Synapse pipelines, Fabric, Power BI, Databricks and others, plus custom lineage via API (Apache Atlas-compatible) |
| **Integration runtimes** | Azure, managed VNet, or self-hosted IR for scanning private or on-premises sources |

### Unified Catalog

The **Unified Catalog** is the business-facing governance experience built on the Data Map:

| Concept | Meaning |
|---|---|
| **Governance domains** | Business areas (Finance, Sales, HR) with owners and stewards; the unit of federated governance |
| **Data products** | Curated, business-ready groupings of data assets (tables, files, reports, models) with description, owner, use cases, terms of use and access policies, which consumers **discover and request access to** |
| **Glossary terms** | Business definitions linked to data products and assets |
| **Critical data elements (CDEs)** | Business-critical data concepts (for example Customer ID) mapped to columns across assets for consistent quality tracking |
| **OKRs** | Objectives and key results tying data products to business outcomes |
| **Data quality** | Profiling, rules (built-in and custom), scoring by dimension (completeness, uniqueness, validity, freshness...), alerts and scheduled scans |
| **Health management** | **Controls** and **actions** that measure governance health (for example "data products have owners", "assets are classified") with recommendations |
| **Access requests and policies** | Consumers request access to data products; approvers grant it with workflows (check supported source types for automatic provisioning) |

### Governance roles

| Role | Typical responsibility |
|---|---|
| **Governance domain owner** | Accountable for a domain's data products and policies |
| **Data product owner** | Defines and maintains a data product |
| **Data steward** | Curates metadata, quality rules and glossary |
| **Data quality steward** | Configures and monitors quality rules |
| **Data consumer / reader** | Discovers products and requests access |

Unified Catalog roles are assigned within the catalog (and at domain level), separately from Azure RBAC on resources.

## How it works on Azure

### Typical rollout

```text
1. Set up the Purview account/tenant experience and Data Map permissions
2. Register and scan sources (Fabric tenant, Azure SQL, ADLS, Databricks UC, Power BI)
   → classifications + sensitivity labels + lineage appear automatically
3. Create governance domains aligned to the business (e.g. Sales, Finance)
4. Define glossary terms and critical data elements
5. Publish data products (e.g. "Customer 360", "Monthly Revenue") with owners and access policies
6. Configure data quality rules on CDEs; schedule scans; alert on score drops
7. Track health controls; report governance KPIs to leadership
```

### Governing data for AI

| Need | Purview capability |
|---|---|
| Know what sensitive data an AI system could reach | Classifications, sensitivity labels, **DSPM for AI** (data security posture for AI) |
| Restrict AI from processing labelled content | Sensitivity labels with encryption and usage rights; DLP policies for Copilot and agents |
| Curate approved sources for RAG and agents | Data products and endorsed assets in the Unified Catalog |
| Prove quality before using data for fine-tuning or grounding | Data quality scores and rules on the product's assets |
| Trace where an answer's data came from | Lineage from source to lakehouse to semantic model to report or agent |
| Audit agent interactions | Purview Audit and eDiscovery for Copilot and Fabric data agents |

### Lineage example

```text
SQL Server (on-prem) ──ADF copy──► ADLS bronze ──Fabric notebook──► lakehouse silver/gold
        └───────────────────── lineage captured ─────────────────────► semantic model ──► Power BI report
```

For custom pipelines (for example a Python ingestion job), push lineage through the Data Map REST API using Atlas entity and process types.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Where to start | Start with **high-value domains and data products** that feed BI and AI, not with "scan everything" |
| Domain model | Align governance domains to business ownership (matching Fabric domains and data mesh domains where used) |
| Central vs federated | Central team defines standards, classifications and quality dimensions; domains own products, stewardship and access approvals |
| Purview vs Unity Catalog / Fabric governance | Unity Catalog and Fabric enforce access inside their platforms; Purview provides the cross-platform catalogue, glossary, quality, lineage and labels. Use them together |
| Data quality location | Use Purview data quality for cross-platform scoring and business visibility; keep pipeline-level checks (expectations, tests) in engineering code as well |
| Cost management | Scope scans (frequency and sources) deliberately; check the current billing model for data governance (governed assets and processing meters) |

## Hands-on

1. In the Purview portal, register a Fabric tenant (or an ADLS account and an Azure SQL database) and run a scan. Review discovered assets, classifications and lineage.
2. Create a governance domain "Sales" with yourself as owner, and three glossary terms (Customer, Order, Revenue).
3. Create a data product "Monthly Revenue" that includes a gold lakehouse table and its Power BI semantic model; link the glossary terms; publish it.
4. Configure data quality rules (completeness and uniqueness on `customer_id`) and run a quality scan; review the score.
5. As a second user, discover the data product and submit an access request; approve it.

## Security, cost and reliability notes

- **Security:** scanning requires read access to sources. Use managed identities and least-privilege roles for the scanning identity, and private endpoints or managed VNet IRs for private sources.
- **Security:** catalogue metadata (column names, sample classifications, lineage) can itself be sensitive. Control catalogue roles carefully.
- **Cost:** scanning frequency, number of governed assets and data quality runs drive cost. Start narrow and expand.
- **Reliability:** governance is a programme, not a project. Without owners and stewards actively curating, the catalogue goes stale within months.

## Common mistakes

- Scanning the whole estate and calling it "governance".
- Data products with no owner, no description and no terms of use.
- Treating Purview as purely an IT tool with no business stewards.
- Ignoring data quality until an AI system gives a wrong answer in front of executives.
- Assuming a data product grants access automatically in every source system.

## Check yourself

1. What is the difference between the Data Map and the Unified Catalog?
2. What is a data product in Purview?
3. What are critical data elements used for?
4. How does Purview help govern data used by AI?
5. Why use both Unity Catalog (or Fabric security) and Purview?
6. Name three data quality dimensions.

<details>
<summary>Answers</summary>

1. The Data Map is the metadata foundation (scans, classifications, labels, lineage). The Unified Catalog is the business governance experience on top (domains, data products, glossary, quality, health, access requests).
2. A curated, business-ready set of data assets with owners, description, use cases, terms of use and access policies, which consumers can discover and request access to.
3. To define business-critical data concepts once and map them to columns across assets, so quality and definitions are tracked consistently.
4. By classifying and labelling sensitive data, assessing AI data security posture, curating approved data products for grounding, measuring data quality, providing lineage, and auditing AI interactions.
5. Platform catalogues enforce access inside their platforms, while Purview provides a cross-platform catalogue, glossary, quality, lineage and labels for the whole estate.
6. Any three of: completeness, uniqueness, validity, accuracy, consistency, timeliness/freshness, conformity.

</details>

## References

- [Microsoft Purview overview](https://learn.microsoft.com/purview/purview)
- [Data governance with Microsoft Purview](https://learn.microsoft.com/purview/data-governance-overview)
- [Unified Catalog overview](https://learn.microsoft.com/purview/unified-catalog)
- [Governance domains](https://learn.microsoft.com/purview/concept-governance-domain)
- [Data products](https://learn.microsoft.com/purview/concept-data-products)
- [Data quality overview](https://learn.microsoft.com/purview/unified-catalog-data-quality)
- [Data Map overview](https://learn.microsoft.com/purview/data-map)
- [Data lineage](https://learn.microsoft.com/purview/concept-data-lineage)
- [DSPM for AI](https://learn.microsoft.com/purview/ai-microsoft-purview)
