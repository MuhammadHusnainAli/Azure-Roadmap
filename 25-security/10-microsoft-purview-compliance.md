# Microsoft Purview Compliance

## Why it matters

Security keeps attackers out; compliance proves you handle data the way laws, regulators and contracts require, and helps you find, preserve and produce data when asked. Organisations must know where sensitive data lives, label and protect it, stop it leaking, retain records for the right time (and delete them after), detect risky insider behaviour, keep an audit trail and respond to legal discovery. **Microsoft Purview** unifies these capabilities across Microsoft 365, Azure, on-premises, other clouds and AI apps. For Azure architects, Purview is where data classification and labels come from, where DLP policies are defined, where audit and eDiscovery happen, and where regulations are mapped to controls in Compliance Manager.

## Key concepts

### Purview solution families

| Family | Solutions |
|---|---|
| **Data security** | Information Protection (sensitivity labels, classification), Data Loss Prevention, Insider Risk Management, Information Barriers, Privileged Access Management, Data Security Investigations, **Data Security Posture Management** (DSPM, Preview; DSPM for AI covered in [Topic 23 file 05](../23-responsible-ai-security-and-governance/05-data-protection-for-ai.md)) |
| **Data governance** | Data Map (scan and classify data sources including Azure SQL, Storage, Databricks, multicloud), Unified Catalog (data products, glossary, quality, access) |
| **Data compliance** | Audit, Communication Compliance, Compliance Manager, Data Lifecycle Management, eDiscovery, Records Management |
| **Shared capabilities** | Classifiers (sensitive information types, trainable classifiers, exact data match), connectors, data and activity explorer, sensitivity labels |

### Information protection

| Capability | Detail |
|---|---|
| **Sensitive information types (SITs)** | Pattern-based detection (credit cards, national IDs, custom regex and keywords) |
| **Trainable classifiers** | ML-based classification (contracts, CVs, source code) |
| **Exact data match** | Match against your actual sensitive records |
| **Sensitivity labels** | Classify and protect: encryption, content marking, access restrictions; apply to files, emails, sites, Teams, Power BI, Fabric and schematised Azure data assets |
| **Auto-labelling** | Client-side and service-side policies based on classifiers |

### Data loss prevention

DLP policies detect sensitive content and act (audit, warn, block, block with override) across Exchange, SharePoint, OneDrive, Teams, endpoints (Windows/macOS), on-premises repositories, Power BI/Fabric, browsers and network (third-party AI sites), and Copilot/AI interactions. Policies use SITs, labels and conditions (sharing externally, uploading to unsanctioned cloud apps).

### Insider risk and communication compliance

| Solution | Purpose |
|---|---|
| **Insider Risk Management** | Detect risky activity patterns (data theft by departing users, data leaks, risky AI usage) with privacy by default (pseudonymised users) |
| **Communication Compliance** | Detect policy violations in communications (harassment, regulatory conduct, sensitive info sharing), including AI prompts and responses |

### Audit, retention and eDiscovery

| Solution | Capability |
|---|---|
| **Audit** (Standard/Premium) | Unified audit log across Microsoft 365 and related services; Premium adds longer retention and high-value events |
| **Data Lifecycle Management** | Retention policies and labels to retain and/or delete content |
| **Records Management** | Declare records, disposition reviews, regulatory records |
| **eDiscovery** | Cases, holds, search, review and export for legal matters (unified eDiscovery experience) |

### Compliance Manager

Assessments mapping regulations (ISO 27001, NIST, GDPR, EU AI Act, ISO/IEC 42001 and hundreds more) to **improvement actions**, split into Microsoft-managed and customer-managed controls, with a **compliance score**. Some actions are automatically tested via integrations (including Azure signals through Defender for Cloud).

## How it works on Azure

| Azure scenario | Purview capability |
|---|---|
| Discover sensitive data in Azure SQL, Storage, ADLS, Cosmos DB, Databricks | **Data Map** scans with SITs; classifications and labels on assets |
| Govern analytics data products | **Unified Catalog** with ownership, glossary and access workflows |
| Protect Fabric/Power BI | Sensitivity labels with inheritance and DLP policies |
| AI apps on Foundry | Purview for Foundry: audit, DLP, retention, DSPM for AI (Topic 23 file 05) |
| Regulatory evidence for Azure | Compliance Manager with Defender for Cloud regulatory compliance (file 02) |
| Investigations | Audit log + eDiscovery + Sentinel for security incidents (file 12) |

Data Map scanning uses the Purview account's **managed identity** with read access to sources (for example Storage Blob Data Reader, SQL `db_datareader`), and managed private endpoints or self-hosted integration runtime for private sources.

Example DLP intent (implemented in the Purview portal):

```text
Policy: Protect customer card data
Locations: Exchange, SharePoint, OneDrive, Teams, Devices, Microsoft 365 Copilot
Condition: content contains Credit Card Number (high confidence, count ≥ 1) AND shared outside organisation
Action: block; allow override with business justification for Finance group; notify user; alert compliance team (medium severity)
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Label taxonomy** | 4–5 top-level labels (Public, General, Confidential, Highly Confidential) with few sublabels; publish before enforcing. |
| **Rollout** | Discover (audit-only DLP, content explorer) → label (default and auto) → protect (encryption, blocking). |
| **Encryption with labels** | Strong protection but affects search, AI and third-party apps; grant EXTRACT/VIEW rights appropriately. |
| **Retention** | Align with records schedule; avoid "keep everything forever" (privacy and cost risk). |
| **Data governance scope** | Scan high-value Azure data estates first; integrate with data platform teams. |
| **Licensing** | Many capabilities need Microsoft 365 E5 Compliance or add-ons; some Azure scanning is pay-as-you-go. |

## Hands-on

1. Create a sensitivity label taxonomy and publish it to a pilot group; apply a default label to a SharePoint library.
2. Create an audit-only DLP policy for credit card numbers across Exchange and SharePoint; review matches in activity explorer.
3. Register an Azure Storage account in Data Map, scan it with the Purview managed identity and view classifications.
4. Run an audit search for file access by a test user.
5. Open Compliance Manager, add the ISO 27001 assessment and assign two improvement actions.

## Security, cost and reliability notes

- **Over-blocking** disrupts business; start in audit mode and tune.
- **Privacy**: insider risk and communication compliance monitor employees; involve HR, legal and works councils.
- **Access to eDiscovery and audit** is highly privileged; restrict roles and audit their use.
- **Cost**: licensing tiers and pay-as-you-go meters (Data Map scanning, some AI and Azure capabilities).

## Common mistakes

| Mistake | Fix |
|---|---|
| Too many labels | Simple taxonomy |
| Enforcing DLP before understanding data | Audit first |
| Retention conflicts unexamined | Understand retention principles |
| Purview treated as Microsoft 365-only | Use Data Map for Azure data |
| Compliance score treated as proof | Verify controls and evidence |

## Check yourself

1. Name the three Purview solution families with two solutions each.
2. What are SITs, trainable classifiers and exact data match?
3. What can sensitivity labels do?
4. Where can DLP policies apply?
5. What does Compliance Manager provide?
6. How does Purview discover sensitive data in Azure data stores?

<details>
<summary>Answers</summary>

1. Data security (Information Protection, DLP, Insider Risk Management, DSPM); data governance (Data Map, Unified Catalog); data compliance (Audit, eDiscovery, Data Lifecycle Management, Records Management, Compliance Manager, Communication Compliance).
2. SITs: pattern-based detectors; trainable classifiers: ML models for document types; exact data match: matching against your real sensitive records.
3. Classify and protect content with encryption, markings and access restrictions across files, email, sites, Teams, Power BI/Fabric and data assets.
4. Exchange, SharePoint, OneDrive, Teams, endpoints, on-premises repositories, Power BI/Fabric, browsers and network, and Copilot/AI interactions.
5. Regulatory assessments mapped to Microsoft- and customer-managed controls, improvement actions and a compliance score.
6. Data Map scans using the Purview managed identity (and private endpoints or integration runtimes for private sources), applying classifiers to assets.

</details>

## References

- [Learn about Microsoft Purview](https://learn.microsoft.com/purview/purview)
- [Information Protection](https://learn.microsoft.com/purview/information-protection)
- [Data loss prevention](https://learn.microsoft.com/purview/dlp-learn-about-dlp)
- [Insider Risk Management](https://learn.microsoft.com/purview/insider-risk-management)
- [Audit solutions](https://learn.microsoft.com/purview/audit-solutions-overview)
- [eDiscovery](https://learn.microsoft.com/purview/ediscovery)
- [Compliance Manager](https://learn.microsoft.com/purview/compliance-manager)
