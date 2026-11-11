# Microsoft Security Copilot

## Why it matters

Security teams are short of people and long on alerts. Analysts spend time writing KQL, reverse-engineering scripts, summarising incidents for managers and hunting for context across Defender, Sentinel, Entra, Intune and Purview. **Microsoft Security Copilot** applies generative AI to those tasks, embedded in the security tools analysts already use and in a standalone portal, with **agents** that take on repeatable work such as phishing triage, alert triage and vulnerability remediation. For architects and consultants it matters in three ways: it's a "buy" option for security operations (file 05), it's now included with Microsoft 365 E5/E7 in a capacity allocation that changes the business case, and it's a high-privilege AI system that needs governance like any other.

## Key concepts

### Primary use cases

| Use case | Example |
|---|---|
| **Investigate and remediate threats** | Summarise an incident with step-by-step response guidance |
| **Build KQL and analyse scripts** | Natural language to KQL; explain an obfuscated PowerShell script |
| **Understand risk and posture** | Prioritised risks across the environment |
| **Troubleshoot IT issues** | Intune device and sign-in problems |
| **Define and manage policies** | Draft a Conditional Access policy and check conflicts |
| **Secure lifecycle workflows** | Group and access configuration guidance |
| **Report for stakeholders** | Executive summary tuned to the audience |
| **Build and add agents** | Extend the Security Copilot ecosystem |

### How it works

```text
Prompt (standalone portal or embedded in Defender, Sentinel, Entra, Intune, Purview)
  └─► Grounding: plugins add organisational context (incidents, alerts, logs, policies) and threat intelligence
        └─► Language model generates a response
              └─► Post-processing with plugins for contextual information
                    └─► Response for analyst review (sessions, promptbooks, pinned results)
```

| Element | Description |
|---|---|
| **Experiences** | Standalone portal and embedded experiences in Microsoft security products |
| **Plugins** | Microsoft (Defender XDR, Sentinel, Intune, Entra, Purview, Defender Threat Intelligence) and third-party security products; custom plugins (KQL, API, GPT) |
| **Promptbooks** | Reusable sequences of prompts for common investigations |
| **Agents** | Autonomous or semi-autonomous agents for tasks such as phishing triage, alert triage, conditional access optimisation, vulnerability remediation and threat intelligence briefings, from Microsoft and partners, and custom agents you build |
| **Threat intelligence** | Defender Threat Intelligence articles and profiles, threat analytics, vulnerability disclosures |

### Capacity and licensing

| Model | Detail |
|---|---|
| **Security Compute Units (SCUs)** | Capacity unit for Security Copilot usage; provisioned capacity plus overage options |
| **Microsoft 365 E5/E7 inclusion** | Per Microsoft's inclusion page: **400 SCUs per month for every 1,000 paid user licences, up to 10,000 SCUs per month**, at no additional cost; rollout timing varied by tenant (check Message Center) |
| **Additional capacity** | Purchase standalone SCUs for heavier use |

Commercial clouds only: not designed for US government clouds (GCC, GCC High, DoD, Azure Government).

### Role-based access

Access uses Security Copilot roles (owner, contributor) plus the user's permissions in each underlying product: a plugin only returns data the user could see in that product. Agents run with configured identities and permissions, so they need explicit scoping.

## How it works on Azure

Security Copilot is SaaS, but it draws heavily on Azure-hosted signals and can be extended with Azure-hosted components:

| Element | Azure role |
|---|---|
| **Microsoft Sentinel** | SIEM data in Log Analytics; Security Copilot generates KQL and summarises incidents |
| **Defender for Cloud** | Cloud posture and AI threat alerts surface in Defender XDR incidents ([Topic 23 file 06](../23-responsible-ai-security-and-governance/06-defender-for-cloud-ai-protection.md)) |
| **Custom plugins** | KQL plugins over your Log Analytics/ADX data; API plugins calling services on Azure Functions or APIM |
| **Logic Apps** | Automate Security Copilot prompts in playbooks (for example, enrich and summarise every high-severity incident) |
| **Capacity resource** | Provisioned SCU capacity is an Azure resource in a subscription and region (where capacity isn't included) |

Example KQL generated from a prompt ("sign-ins for alice from new countries in the last 7 days"):

```kusto
SigninLogs
| where TimeGenerated > ago(7d)
| where UserPrincipalName =~ "alice@contoso.com"
| summarize firstSeen = min(TimeGenerated) by Location
| join kind=leftanti (
    SigninLogs | where TimeGenerated between (ago(90d) .. ago(7d))
    | where UserPrincipalName =~ "alice@contoso.com" | distinct Location
  ) on Location
```

Analysts should review generated queries before acting on results.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Embedded vs standalone** | Embedded for in-flow assistance (incident summaries in Defender); standalone for cross-product investigations and promptbooks. |
| **Agents** | Start agents in advisory mode (recommendations, triage verdicts) and measure accuracy before allowing automated actions. |
| **Capacity planning** | Monitor SCU consumption by workload and agent; included E5/E7 capacity may cover pilots; buy more for heavy agent use. |
| **Plugins** | Enable only plugins whose data is needed; review third-party plugin data flows. |
| **Custom agents** | Build where you have repeatable, high-volume tasks unique to your environment. |
| **Measuring value** | Mean time to triage/respond, analyst hours saved, false-positive handling, quality of reports. |

## Hands-on

1. In the Security Copilot portal, investigate a Defender XDR incident; save a session.
2. Ask for KQL to hunt for a technique in Sentinel; review and run it.
3. Run a promptbook for incident summarisation; customise it for your executive report format.
4. Enable the phishing triage agent in a test tenant; review its verdicts against analyst decisions for a week.
5. Check SCU usage and the included E5/E7 allocation in the usage dashboard.

## Security, cost and reliability notes

- **High privilege**: Security Copilot sees sensitive security data; restrict access and use role assignments.
- **Hallucination risk**: verify outputs before containment or legal actions.
- **Prompt injection**: malicious content in emails, scripts or logs can try to manipulate analysis; agents should not act on untrusted instructions.
- **Cost**: SCU consumption from agents can be continuous; set alerts on usage.
- **Data handling**: review Security Copilot's data sharing and residency settings during onboarding.

## Common mistakes

| Mistake | Fix |
|---|---|
| Treating answers as verdicts | Analyst review |
| Turning on every agent at once | Pilot agents individually and measure |
| Ignoring capacity limits | Monitor SCUs |
| Giving all users access | Role-based access for security staff |
| No custom promptbooks | Encode your playbooks for consistency |

## Check yourself

1. Name five Security Copilot use cases.
2. Describe how Security Copilot processes a prompt.
3. What are plugins, promptbooks and agents?
4. What is an SCU, and what's included with Microsoft 365 E5/E7?
5. How does Security Copilot respect permissions?
6. Which Azure services commonly integrate with it?

<details>
<summary>Answers</summary>

1. Any five: investigate/remediate threats, KQL and script analysis, risk and posture, IT troubleshooting, policy management, secure lifecycle workflows, stakeholder reports, building agents.
2. Prompt → grounding with plugins (organisational and threat intelligence context) → language model → post-processing with plugins → response for review.
3. Plugins: data and capability integrations; promptbooks: reusable prompt sequences; agents: autonomous or semi-autonomous task performers.
4. Security Compute Unit, the capacity measure; E5/E7 includes 400 SCUs per month per 1,000 paid licences, up to 10,000 per month.
5. Plugins return only data the user can access in the underlying product; Security Copilot roles control access to the service.
6. Microsoft Sentinel (Log Analytics), Defender for Cloud, Logic Apps, Azure Functions/APIM for custom plugins, ADX.

</details>

## References

- [What is Microsoft Security Copilot?](https://learn.microsoft.com/copilot/security/microsoft-security-copilot)
- [Security Copilot inclusion in Microsoft 365 E5](https://learn.microsoft.com/copilot/security/security-copilot-inclusion)
- [Microsoft Sentinel documentation](https://learn.microsoft.com/azure/sentinel/)
