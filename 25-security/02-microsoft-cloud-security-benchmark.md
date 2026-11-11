# Microsoft Cloud Security Benchmark

## Why it matters

"Is our Azure environment secure?" needs a yardstick. Without one, security reviews become opinion, every team invents its own checklist, and auditors can't map your controls to their frameworks. The **Microsoft cloud security benchmark (MCSB)** is Microsoft's canonical set of cloud security controls, with Azure-specific implementation guidance, mappings to NIST SP 800-53, PCI DSS and CIS Controls, and more than 420 Azure Policy definitions to measure them. It's the **default policy initiative in Defender for Cloud**, drives **Secure Score**, and underpins per-service **security baselines**. Using it as your baseline gives you a defensible, measurable, continuously assessed security posture.

## Key concepts

### Security domains

| Domain | ID | Covers |
|---|---|---|
| Network security | NS | VNets, private connectivity, external attack mitigation, DNS |
| Identity management | IM | SSO, strong authentication, managed identities, conditional access |
| Privileged access | PA | Admin model, privileged accounts, privileged access workstations, JIT |
| Data protection | DP | Discovery and classification, encryption at rest and in transit, key and certificate management |
| Asset management | AM | Inventory, approved services, security team visibility |
| Logging and threat detection | LT | Audit logs, threat detection, high-quality alerts |
| Incident response | IR | Preparation, detection, containment, post-incident |
| Posture and vulnerability management | PV | Configuration baselines, vulnerability scanning, penetration testing |
| Endpoint security | ES | EDR, anti-malware |
| Backup and recovery | BR | Backups at each tier, validated and protected |
| DevOps security | DS | SAST, supply chain, IaC scanning, pipeline security |
| **Artificial intelligence security** (v2) | AI | AI platform security, AI application security, AI security monitoring (seven recommendations) |

### Versions

| Version | Status | Notes |
|---|---|---|
| **MCSB v1** | GA | Multicloud (Azure, AWS, GCP) control guidance; default Defender for Cloud standard |
| **MCSB v2** | **Preview** | Azure-focused, expanded domains (including AI), 420+ Azure Policy mappings, risk/threat-based guidance with MITRE ATT&CK and criticality levels |

### Control structure (v2)

Each control has: **ID** (for example NS-2, DP-3, AI-1), Azure Policy links, security principle, risk to mitigate, MITRE ATT&CK techniques, numbered implementation guidance (NS-1.1, NS-1.2…), an implementation example, **criticality** (Must have, Should have, Nice to have) and mappings to **NIST SP 800-53 Rev. 5, PCI DSS v4 and CIS Controls v8.1**.

### Related artefacts

| Artefact | Use |
|---|---|
| **Security baselines** | Per-service application of MCSB (for example, the Azure Storage security baseline) listing which controls apply and how |
| **Defender for Cloud regulatory compliance** | Dashboard showing MCSB (and other standards such as ISO 27001, PCI DSS, NIST) compliance per subscription |
| **Secure Score** | Aggregated measure of posture based on recommendations; higher is better; grouped by security controls |
| **Azure Policy initiative** | The MCSB initiative assigned by default when Defender for Cloud is enabled |

### Secure Score

Secure Score reflects the percentage of security recommendations you've remediated, weighted by control. Defender CSPM adds risk-based prioritisation (attack paths, exposure). Use it as a trend metric per subscription and management group, not as a target to game.

## How it works on Azure

Typical baseline implementation:

1. **Enable Defender for Cloud** (foundational CSPM is free) on all subscriptions; the MCSB initiative is assigned automatically.
2. **Review regulatory compliance** for MCSB; add other standards (ISO 27001, PCI DSS, NIST SP 800-53, CIS) as needed.
3. **Prioritise** by criticality and exposure: Must-have identity and network controls first.
4. **Remediate**: fix recommendations, use "Fix" and "Enforce" (deny policies) where available, assign exemptions with justification and expiry.
5. **Prevent regressions**: deny or deploy-if-not-exists policies in the landing zone, IaC scanning in pipelines (file 09).
6. **Report**: Secure Score trend and compliance percentages to leadership.

Query recommendations at scale with Resource Graph:

```kusto
securityresources
| where type == "microsoft.security/assessments"
| extend status = tostring(properties.status.code), name = tostring(properties.displayName),
         severity = tostring(properties.metadata.severity)
| where status == "Unhealthy"
| summarize unhealthy = count() by name, severity
| order by severity asc, unhealthy desc
```

Assign MCSB at management group scope with the CLI (built-in initiative; look up its ID with `az policy set-definition list --query "[?displayName=='Microsoft cloud security benchmark'].name"`):

```bash
az policy assignment create --name mcsb --display-name "Microsoft cloud security benchmark" \
  --scope /providers/Microsoft.Management/managementGroups/mg-contoso \
  --policy-set-definition "<initiative name>"
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Baseline choice** | MCSB as the technical baseline; map to ISO/NIST/PCI through its mappings rather than separate programmes. |
| **v1 vs v2** | v1 for production compliance reporting; review v2 (Preview) for AI and enhanced guidance. |
| **Audit vs deny** | Audit first to measure; deny for well-understood, low-impact controls (public network access, TLS). |
| **Exemptions** | Time-bound with owner and compensating control. |
| **Scope** | Management group assignment for consistency; workload-specific additions where needed. |
| **Metrics** | Secure Score trend, Must-have control compliance, mean time to remediate high-severity recommendations. |

## Hands-on

1. Open Defender for Cloud regulatory compliance and review MCSB status for a subscription.
2. Pick three failing Must-have controls; read their implementation guidance and remediate.
3. Read the security baseline for Azure Key Vault and compare with your configuration.
4. Run the Resource Graph query above; build a workbook of unhealthy recommendations by severity.
5. Add an exemption with expiry for one justified exception.

## Security, cost and reliability notes

- **Compliance isn't security**: a green dashboard doesn't prove absence of risk; combine with threat modelling and testing.
- **Deny policies can break deployments**: communicate, test in non-production, and provide compliant templates.
- **Cost**: foundational CSPM is free; Defender CSPM and workload plans are paid.
- **Multi-cloud**: connect AWS and GCP to assess them against MCSB too.

## Common mistakes

| Mistake | Fix |
|---|---|
| Chasing Secure Score points without context | Prioritise by criticality and exposure |
| Permanent exemptions | Expiry dates and reviews |
| Separate control frameworks per auditor | Use MCSB mappings |
| Remediating manually without preventing recurrence | Policy and IaC |
| Ignoring the AI domain | Review MCSB v2 AI controls for AI workloads |

## Check yourself

1. What is the MCSB, and where is it used by default?
2. Name eight MCSB domains.
3. What's new in MCSB v2?
4. What does each control include?
5. What is Secure Score, and how should you use it?
6. Which frameworks does MCSB map to?

<details>
<summary>Answers</summary>

1. Microsoft's cloud security control set with Azure guidance and policy mappings; the default initiative/standard in Defender for Cloud.
2. Any eight: NS, IM, PA, DP, AM, LT, IR, PV, ES, BR, DS, AI.
3. Azure-focused guidance, a new AI security domain, 420+ Azure Policy mappings, risk/threat-based guidance with MITRE ATT&CK, implementation examples and criticality levels (Preview).
4. ID, Azure Policy links, principle, risk, MITRE ATT&CK, implementation guidance, example, criticality, framework mappings.
5. A posture measure based on remediated recommendations; track trends and prioritise by risk rather than gaming points.
6. NIST SP 800-53 Rev. 5, PCI DSS v4, CIS Controls v8.1.

</details>

## References

- [Microsoft cloud security benchmark v2 (Preview) overview](https://learn.microsoft.com/security/benchmark/azure/overview)
- [Microsoft cloud security benchmark introduction](https://learn.microsoft.com/security/benchmark/azure/introduction)
- [Secure score in Defender for Cloud](https://learn.microsoft.com/azure/defender-for-cloud/secure-score-security-controls)
- [Regulatory compliance in Defender for Cloud](https://learn.microsoft.com/azure/defender-for-cloud/regulatory-compliance-dashboard)
