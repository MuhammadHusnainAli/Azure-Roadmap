# Microsoft Sentinel

## Why it matters

Individual security tools see slices of an attack: Entra sees a risky sign-in, Defender for Endpoint sees a suspicious process, Defender for Cloud sees an unusual Key Vault access, a firewall sees outbound traffic to a rare domain. A **SIEM** brings those signals together, correlates them into incidents, supports hunting across months of data, and automates response. **Microsoft Sentinel** is Microsoft's cloud-native SIEM (with SOAR capabilities), now part of the unified security operations platform in the **Microsoft Defender portal**, with a **Sentinel data lake** for cost-effective long-term retention. Architects design its data collection, workspace topology and cost; consultants are asked to compare it with other SIEMs and plan migrations.

## Key concepts

### Capabilities

| Area | Capability |
|---|---|
| **Collect** | Content hub solutions with data connectors (Microsoft, Azure, third-party, syslog/CEF, APIs); custom connectors; **ASIM** normalisation at query or ingestion time |
| **Detect** | Analytics rules (scheduled KQL, near-real-time, Microsoft security, anomaly, threat intelligence, Fusion-style correlation in the unified platform) grouping alerts into incidents; MITRE ATT&CK mapping |
| **Investigate** | Incidents with entities, timeline and investigation graph; UEBA (user and entity behaviour analytics) |
| **Hunt** | KQL hunting queries, bookmarks, livestream, notebooks |
| **Respond** | Automation rules and playbooks (Logic Apps) for enrichment, containment, ticketing |
| **Threat intelligence** | Microsoft threat intelligence, TAXII and API feeds, indicator matching |
| **Content** | Content hub solutions (connectors, rules, workbooks, playbooks) packaged per product or domain |

### Unified security operations (Defender portal)

Sentinel is integrated with Defender XDR in the **Microsoft Defender portal**: one incident queue, advanced hunting across Sentinel and XDR tables, unified entity pages and Security Copilot. **After 31 March 2027, Sentinel will no longer be supported in the Azure portal** and will be available only in the Defender portal; plan the transition now. Onboarding changes some behaviours (for example, Microsoft security product alerts route through the Defender XDR connector; `SecurityIncident` no longer includes a Description field, which can break automation rules relying on it).

### Data tiers and the Sentinel data lake

| Tier | Purpose | Notes |
|---|---|---|
| **Analytics tier** | High-performance data for detection, alerting, hunting and incidents | Higher cost per GB; interactive retention configurable |
| **Data lake tier** | Long-term, low-cost storage for querying and Python/ML analytics | Up to **12 years**; single copy in open Parquet; separation of storage and compute; analytics-tier data mirrored to the lake |

Choose the tier per table: high-value detection sources (Entra sign-ins, Defender alerts, firewall threats) in analytics; high-volume, low-detection-value logs (verbose flow logs, raw proxy logs) in the lake, promoted on demand.

### Cost model

Pay for ingestion and retention per tier (commitment tiers reduce analytics cost); some Microsoft sources (for example Azure Activity, Microsoft 365 audit and Defender alerts) are free to ingest. Data collection rules (DCRs) filter and transform data at ingestion to reduce volume.

## How it works on Azure

Workspace design:

| Decision | Guidance |
|---|---|
| **Number of workspaces** | As few as possible (often one per tenant/region of data residency); separate only for sovereignty, billing or strict access isolation |
| **Access** | Sentinel roles (Reader, Responder, Contributor) and table-level/resource-context RBAC |
| **Multi-tenant (MSSP)** | Azure Lighthouse and Defender multitenant management |
| **Retention** | Analytics tier for detection windows (for example 90 days); lake tier for compliance and forensics |

Example scheduled analytics rule logic (KQL): Key Vault secret reads from a new IP by an identity that normally doesn't read secrets.

```kusto
let lookback = 14d;
let baseline = AzureDiagnostics
  | where TimeGenerated between (ago(lookback) .. ago(1d))
  | where ResourceProvider == "MICROSOFT.KEYVAULT" and OperationName == "SecretGet"
  | summarize by identity_claim_oid_g, CallerIPAddress;
AzureDiagnostics
| where TimeGenerated > ago(1h)
| where ResourceProvider == "MICROSOFT.KEYVAULT" and OperationName == "SecretGet"
| join kind=leftanti baseline on identity_claim_oid_g, CallerIPAddress
| summarize reads = count(), vaults = make_set(Resource) by identity_claim_oid_g, CallerIPAddress
| where reads > 5
```

Automation rule + playbook pattern:

```text
Incident created (severity High, tactic Credential Access)
  → automation rule assigns owner, adds tags, runs playbook
      → Logic App: enrich IP with threat intel, get user risk from Entra,
        if confirmed: disable user / revoke sessions, post to Teams, create ServiceNow ticket
```

Deploy content as code: analytics rules, workbooks and playbooks in Git, deployed via repositories integration or Bicep/ARM in CI/CD.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Data sources first** | Identity (Entra sign-in/audit), Defender XDR, Azure Activity, firewall/WAF, Key Vault, critical app logs; add others by detection value. |
| **Tiering** | Analytics tier for detection; lake for volume and retention. |
| **Normalisation** | Use ASIM parsers so rules work across vendors. |
| **Automation level** | Automate enrichment broadly; containment only for high-confidence detections with approvals. |
| **Portal** | Plan the Defender portal transition before 31 March 2027. |
| **Detection engineering** | Treat rules as code with tests, tuning and MITRE coverage tracking. |

## Hands-on

1. Create a Log Analytics workspace, enable Sentinel and onboard it to the Defender portal.
2. Install content hub solutions for Entra ID, Azure Activity and Azure Key Vault; connect data.
3. Create the Key Vault analytics rule above; generate test activity and view the incident.
4. Build a playbook that posts high-severity incidents to Teams with enrichment.
5. Configure a high-volume table to the data lake tier and query it.

## Security, cost and reliability notes

- **Cost control**: DCR filtering, commitment tiers, lake tier for verbose logs, free sources; monitor ingestion daily.
- **Access**: restrict who can read sensitive tables; use table-level RBAC.
- **Integrity**: Sentinel inherits Azure Monitor's append-only, tamper-resistant design.
- **Playbook identities**: managed identities with least privilege for containment actions.
- **CMK nuance**: after Defender portal onboarding, alerts and incidents are no longer CMK-encrypted; check requirements.

## Common mistakes

| Mistake | Fix |
|---|---|
| Ingesting everything into the analytics tier | Tier by value |
| Many workspaces without need | Consolidate |
| Rules enabled without tuning | Tune and measure false positives |
| Ignoring the Azure portal retirement | Plan the Defender portal move |
| Manual-only response | Automation rules and playbooks |

## Check yourself

1. What are Sentinel's main capability areas?
2. What changes on 31 March 2027?
3. Compare the analytics tier and data lake tier.
4. What is ASIM, and why use it?
5. How do automation rules and playbooks differ?
6. List three ways to control Sentinel cost.

<details>
<summary>Answers</summary>

1. Collect, detect, investigate, hunt, respond, threat intelligence and content.
2. Sentinel is no longer supported in the Azure portal; it's available only in the Microsoft Defender portal.
3. Analytics: high-performance detection, hunting and incidents at higher cost. Data lake: low-cost long-term storage (up to 12 years), Parquet, separate compute, for queries and Python/ML analysis.
4. The Advanced Security Information Model, normalising data from different sources so detections and hunting work across vendors.
5. Automation rules are lightweight incident-handling rules (assign, tag, run playbooks); playbooks are Logic Apps workflows for complex enrichment and response.
6. Any three: DCR filtering and transformation, commitment tiers, data lake tier for verbose logs, free data sources, retention tuning.

</details>

## References

- [What is Microsoft Sentinel?](https://learn.microsoft.com/azure/sentinel/overview)
- [Microsoft Sentinel data lake](https://learn.microsoft.com/azure/sentinel/datalake/sentinel-lake-overview)
- [Transition Sentinel to the Defender portal](https://learn.microsoft.com/azure/sentinel/move-to-defender)
- [Advanced Security Information Model (ASIM)](https://learn.microsoft.com/azure/sentinel/normalization)
- [Automation in Microsoft Sentinel](https://learn.microsoft.com/azure/sentinel/automation/automation)
