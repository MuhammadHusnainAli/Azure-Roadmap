# Microsoft Defender for Cloud

## Why it matters

Cloud environments change daily: new subscriptions, resources, containers, APIs and AI services, often deployed by teams outside central IT. Security teams need continuous answers to two questions: **what's misconfigured or vulnerable, and which of those issues actually matter?** and **what's being attacked right now?** **Microsoft Defender for Cloud** is Microsoft's cloud-native application protection platform (CNAPP), combining posture management, DevSecOps and workload protection across Azure, AWS, GCP and on-premises. It's the operational engine behind the Microsoft cloud security benchmark (file 02) and feeds Defender XDR and Sentinel (file 06). Knowing its plans, attack path analysis and how to roll it out economically is core to any Azure security architecture.

## Key concepts

### CNAPP components

| Component | Purpose |
|---|---|
| **CSPM** (cloud security posture management) | Assess and improve configuration: recommendations, Secure Score, regulatory compliance, attack paths |
| **DevSecOps** | Code-to-cloud security: connect GitHub, Azure DevOps and GitLab, IaC scanning, code-to-runtime mapping (file 09) |
| **CWPP** (cloud workload protection) | Threat detection and protection for servers, containers, storage, databases, App Service, Key Vault, APIs, AI services |

Defender for Cloud is expanding into the **Microsoft Defender portal** for a unified experience across cloud and code; some features now live there alongside the Azure portal.

### Plans

| Plan | Protects / provides |
|---|---|
| **Foundational CSPM** (free) | Recommendations, Secure Score, MCSB compliance, asset inventory |
| **Defender CSPM** | Agentless vulnerability scanning, data-aware posture, **cloud security graph**, **attack path analysis**, cloud security explorer, AI security posture, governance rules, code-to-cloud |
| **Defender for Servers** (P1/P2) | Defender for Endpoint integration, vulnerability management, JIT VM access, file integrity monitoring (P2), agentless scanning |
| **Defender for Containers** | Kubernetes hardening, image vulnerability assessment, runtime threat detection (AKS, EKS, GKE, Arc) |
| **Defender for Storage** | Malware scanning on upload, sensitive data threat detection, SAS token misuse |
| **Defender for Databases** | Azure SQL, SQL on machines, open-source relational databases, Cosmos DB |
| **Defender for App Service** | Attacks against web apps |
| **Defender for Key Vault** | Unusual access to vaults |
| **Defender for Resource Manager** | Suspicious management operations |
| **Defender for APIs** | API inventory, posture and threat detection for APIs in API Management (file 08) |
| **AI services** | AI threat protection for generative AI apps ([Topic 23 file 06](../23-responsible-ai-security-and-governance/06-defender-for-cloud-ai-protection.md)) |

### Attack path analysis and the security graph

Defender CSPM builds a **cloud security graph** of resources, identities, network exposure, vulnerabilities and data sensitivity, then computes **attack paths**: chains such as "internet-exposed VM with a critical vulnerability → managed identity with Storage Blob Data Contributor → storage account containing sensitive data". Fixing the **choke point** breaks many paths. The **cloud security explorer** lets you query the graph ("VMs with high-severity CVEs that are internet-exposed and have access to Key Vault").

### Multicloud and hybrid

Connect **AWS accounts** and **GCP projects** (agentless connectors), on-premises and other-cloud servers through **Azure Arc**, and DevOps platforms. Recommendations and plans apply across them.

### Workflow features

| Feature | Use |
|---|---|
| **Governance rules** | Assign owners and due dates to recommendations automatically |
| **Workflow automation** | Logic Apps triggered by recommendations or alerts |
| **Continuous export** | Recommendations and alerts to Log Analytics or Event Hubs |
| **Exemptions** | Waive recommendations with justification and expiry |
| **Security alerts** | Correlated into incidents in Defender XDR |

## How it works on Azure

Rollout pattern:

1. **Foundational CSPM everywhere** (free): enable on all subscriptions; review Secure Score and MCSB compliance.
2. **Defender CSPM** on production subscriptions for attack paths and agentless scanning.
3. **Workload plans by risk**: Servers P2 for internet-facing and critical VMs, Containers for AKS, Storage for data lakes, Databases for SQL estates, Key Vault, Resource Manager everywhere, APIs for APIM, AI services for AI workloads.
4. **Enable at scale** with Azure Policy (`DeployIfNotExists` initiatives for Defender plans) at management group scope.
5. **Connect XDR and Sentinel** for incident handling (file 06).
6. **Operate**: weekly attack-path review, governance rules for owners, Secure Score trends.

Enable plans with the CLI:

```bash
for plan in CloudPosture VirtualMachines Containers StorageAccounts KeyVaults Arm Api; do
  az security pricing create --name "$plan" --tier Standard
done
az security pricing create --name VirtualMachines --tier Standard --subplan P2
```

Cloud security explorer-style question as Resource Graph (simplified):

```kusto
securityresources
| where type == "microsoft.security/attackpaths"
| extend riskLevel = tostring(properties.riskLevel), title = tostring(properties.displayName)
| summarize paths = count() by title, riskLevel
| order by riskLevel desc
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Which plans** | Driven by asset criticality and exposure; avoid enabling every plan everywhere without budget. |
| **Servers P1 vs P2** | P2 for production and internet-facing; P1 where Defender for Endpoint plus basic features suffice. |
| **Agentless vs agent** | Agentless scanning for broad coverage; Defender for Endpoint sensor for runtime EDR. |
| **Prioritisation** | Attack paths and risk-based recommendations over raw counts. |
| **Ownership** | Governance rules mapping recommendations to resource owners via tags. |
| **Multicloud** | One CNAPP across clouds simplifies operations and reporting. |

## Hands-on

1. Enable Foundational CSPM and review the top ten recommendations by risk.
2. Enable Defender CSPM on a test subscription; deploy an intentionally exposed VM with a managed identity and storage access; find the resulting attack path.
3. Use the cloud security explorer to find internet-exposed resources with high-severity vulnerabilities.
4. Enable Defender for Storage malware scanning and upload the EICAR test file.
5. Create a governance rule assigning storage recommendations to owners by tag with a 14-day due date.

## Security, cost and reliability notes

- **Cost**: per resource/hour or per transaction depending on plan; use the cost calculator and enable by risk.
- **Coverage gaps**: check plan support per resource type and region; agentless scanning has scope limits.
- **Alert handling**: route to Defender XDR/Sentinel with clear ownership; avoid email-only alerts.
- **Permissions**: Security Reader for visibility, Security Admin for configuration; avoid Owner for security staff.

## Common mistakes

| Mistake | Fix |
|---|---|
| Only free CSPM, ignoring attack paths on critical estates | Defender CSPM on production |
| Treating recommendation counts as risk | Use attack paths and exposure |
| Enabling plans manually per subscription | Policy at management group scope |
| No owners for recommendations | Governance rules |
| Ignoring AWS/GCP | Connect multicloud environments |

## Check yourself

1. What are the three CNAPP components in Defender for Cloud?
2. What does Defender CSPM add over Foundational CSPM?
3. Name six workload protection plans.
4. Explain attack path analysis and choke points.
5. How do you enable Defender at scale?
6. Which features help operationalise remediation?

<details>
<summary>Answers</summary>

1. CSPM, DevSecOps, CWPP.
2. Agentless vulnerability scanning, data-aware posture, cloud security graph, attack path analysis, cloud security explorer, AI posture, governance and code-to-cloud capabilities.
3. Any six: Servers, Containers, Storage, Databases, App Service, Key Vault, Resource Manager, APIs, AI services.
4. The security graph links exposure, vulnerabilities, identities and data to compute exploitable chains; a choke point is a resource whose fix breaks many paths.
5. Azure Policy `DeployIfNotExists` initiatives at management group scope (or CLI/IaC per subscription).
6. Governance rules, workflow automation, continuous export, exemptions and XDR integration.

</details>

## References

- [What is Microsoft Defender for Cloud?](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-cloud-introduction)
- [Attack path analysis](https://learn.microsoft.com/azure/defender-for-cloud/concept-attack-path)
- [Secure score](https://learn.microsoft.com/azure/defender-for-cloud/secure-score-security-controls)
- [Defender for APIs](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-apis-introduction)
