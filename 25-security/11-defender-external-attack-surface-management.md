# Defender External Attack Surface Management

## Why it matters

You can't protect what you don't know you own. Every organisation has internet-facing assets it has forgotten: a marketing microsite on an old domain, a test API with a public IP, an expired certificate on a partner portal, a subsidiary's web server running an unpatched CMS, a storage endpoint someone exposed "temporarily". Attackers scan the whole internet continuously and find these first. **Microsoft Defender External Attack Surface Management (Defender EASM)** looks at your organisation from the outside, the way an attacker does: it discovers domains, IP ranges, hosts, web applications and certificates related to your known assets, inventories them, and flags vulnerabilities and hygiene issues. It complements inside-out tools like Defender for Cloud (file 05), which only see resources in connected subscriptions.

## Key concepts

### Outside-in vs inside-out

| | Defender for Cloud (inside-out) | Defender EASM (outside-in) |
|---|---|---|
| **Sees** | Resources in connected Azure, AWS, GCP accounts | Anything internet-facing linked to your organisation, wherever hosted |
| **Finds** | Misconfigurations, vulnerabilities, attack paths in known estates | Unknown, unmanaged and third-party-hosted assets; exposed services |
| **Data source** | Cloud APIs, agents, agentless scanning | Microsoft's internet-wide crawling and discovery |

### Discovery

Defender EASM starts from **seeds** (known domains, IP blocks, ASNs, Whois organisations, email contacts, hosts) and recursively follows observed connections (DNS records, certificates, Whois, hosting relationships, web content) to infer related infrastructure. Discovery groups let you run and schedule discoveries per business unit or brand. Asset types include:

- domains and hosts;
- IP address blocks and IP addresses;
- autonomous system numbers (ASNs);
- Whois organisations and email contacts;
- pages (web applications), SSL certificates.

### Inventory states

| State | Meaning |
|---|---|
| **Approved Inventory** | Confirmed as yours; always monitored and analysed |
| **Candidate** | Related to your seeds but not confirmed; review manually |
| **Requires Investigation** | Similar to Candidate, with lower confidence; manual validation |
| **Dependency** | Third-party infrastructure your assets rely on (for example a CDN or SaaS provider) |
| **Monitor Only** | Relevant but not controlled (franchisees, related companies) |

Assets are also recent (active) or historic. Labels help track ownership (business unit, team).

### Insights

Dashboards highlight **attack surface priorities** (high-severity observations such as known exploited CVEs on exposed software), security hygiene (expired or expiring certificates, weak TLS, open admin ports, outdated software), compliance (for example GDPR-related issues such as cookies and forms on non-HTTPS pages) and OWASP-related web issues.

### Access and data

- Azure resource with Owner/Contributor (manage) and Reader (view) roles; **no cross-tenant access** (including Azure Lighthouse).
- Underlying internet data is global Microsoft data; your labels and configuration are customer data stored in the region you choose.
- Data connections export assets and insights to **Log Analytics** and **Azure Data Explorer** for Sentinel correlation and reporting.
- Exposure insights feed **Microsoft Security Exposure Management** for a unified exposure view with internal posture.

## How it works on Azure

Deployment and operating model:

1. **Create the Defender EASM resource** in a security subscription and region matching data residency needs.
2. **Seed discovery**: use the pre-built attack surface for your organisation if available, or add seeds (primary domains, IP ranges, ASNs).
3. **Triage**: review Candidate and Requires Investigation assets with business units; promote to Approved, mark Dependency or Monitor Only, or remove.
4. **Label** assets with owner and business unit.
5. **Export** to Log Analytics; join with Sentinel incidents and Defender for Cloud data.
6. **Remediate** high-priority observations through owners (patch, decommission, add WAF, renew certificates, close ports).
7. **Repeat** discovery on a schedule; review new assets weekly.

Example Sentinel/Log Analytics query after enabling the data connection (table names depend on the connection configuration):

```kusto
EasmAsset_CL
| where AssetState_s == "Confirmed" and Kind_s == "host"
| summarize lastSeen = max(TimeGenerated) by AssetName_s, Labels_s
| join kind=leftouter (EasmRisk_CL | summarize risks = make_set(RiskName_s) by AssetName_s) on AssetName_s
| where isnotempty(risks)
```

Validate actual table and column names in your workspace before building rules.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Scope** | Include subsidiaries, brands and acquired companies; use discovery groups per entity. |
| **Ownership** | Every Approved asset needs an owner label; unowned assets are decommission candidates. |
| **Third parties** | Mark dependencies; raise issues with suppliers through vendor risk management. |
| **Integration** | Export to Sentinel and Exposure Management rather than treating EASM as a separate console. |
| **Cost** | Priced by billable assets (approved hosts, domains, IPs); keep inventory accurate to avoid paying for noise. |
| **Remediation tracking** | Track in ITSM with SLAs by severity. |

## Hands-on

1. Deploy a Defender EASM resource; seed it with a domain you own.
2. Review discovered Candidate assets; classify them into states.
3. Identify the top three attack surface priorities and assign owners.
4. Find certificates expiring in the next 30 days.
5. Configure a data connection to Log Analytics and query exported assets.

## Security, cost and reliability notes

- **Only seed assets you're authorised to assess**: EASM is passive discovery, but triage and any follow-up testing must respect ownership.
- **False associations** happen (shared hosting, similar names); human review prevents chasing others' assets.
- **Cost** grows with inventory; prune decommissioned and unrelated assets.
- **Tenant-bound**: plan access for MSSPs without Lighthouse support.

## Common mistakes

| Mistake | Fix |
|---|---|
| Relying only on Defender for Cloud inventory | Add outside-in discovery |
| Never reviewing Candidate assets | Weekly triage |
| No owners for discovered assets | Mandatory labels |
| Keeping findings inside EASM only | Export to Sentinel and ITSM |
| Ignoring expired certificates and old domains | Hygiene dashboards and renewal processes |

## Check yourself

1. How does Defender EASM differ from Defender for Cloud?
2. What are seeds, and how does discovery work?
3. Name the five inventory states.
4. What kinds of insights does EASM provide?
5. How do you integrate EASM with your SOC?
6. What access limitation affects MSSPs?

<details>
<summary>Answers</summary>

1. EASM discovers internet-facing assets from the outside regardless of hosting; Defender for Cloud assesses resources inside connected cloud accounts.
2. Known assets (domains, IP blocks, ASNs, Whois organisations, contacts, hosts); EASM recursively follows observed connections to infer related infrastructure.
3. Approved Inventory, Candidate, Requires Investigation, Dependency, Monitor Only.
4. Attack surface priorities (critical vulnerabilities), security hygiene (certificates, ports, outdated software), compliance and web application issues.
5. Data connections to Log Analytics/Azure Data Explorer for Sentinel, plus Microsoft Security Exposure Management.
6. No cross-tenant access, including through Azure Lighthouse.

</details>

## References

- [Defender EASM overview](https://learn.microsoft.com/azure/external-attack-surface-management/overview)
- [What is discovery?](https://learn.microsoft.com/azure/external-attack-surface-management/what-is-discovery)
- [Understanding inventory assets](https://learn.microsoft.com/azure/external-attack-surface-management/understanding-inventory-assets)
- [Data connections](https://learn.microsoft.com/azure/external-attack-surface-management/data-connections)
- [Microsoft Security Exposure Management](https://learn.microsoft.com/security-exposure-management/microsoft-security-exposure-management)
