# Defender for Cloud AI Protection

## Why it matters

Guardrails block bad content in the request path, but the security operations centre (SOC) also needs to know: which AI resources exist, which are exposed to the internet or misconfigured, who is attacking them, whether an attack succeeded, and how it relates to the rest of an incident. **Microsoft Defender for Cloud** provides both halves: **AI security posture management** in the Defender CSPM plan (inventory, AI bill of materials, recommendations, attack paths) and **AI threat protection** through the Defender for AI Services plan (real-time alerts for jailbreaks, data leakage, malicious URLs, wallet attacks and suspicious access), integrated with Defender XDR and Microsoft Sentinel. It's how AI workloads join the existing security operating model rather than becoming a blind spot.

## Key concepts

### Two capabilities

| | AI security posture management | AI threat protection |
|---|---|---|
| **Plan** | Defender CSPM | Defender for AI Services (shown in Foundry as the Defender for Foundry Tools plan) |
| **Question** | Is my AI estate configured securely? | Is someone attacking my AI apps right now? |
| **Outputs** | AI inventory and AI BOM, recommendations, attack paths, internet exposure | Security alerts with prompt evidence, incidents in Defender XDR |
| **Status** | GA (agent discovery Preview) | GA |

### Posture management

- **Discovery** of AI workloads across Azure OpenAI, Microsoft Foundry, Azure Machine Learning, Amazon Bedrock and Google Vertex AI.
- **AI BOM**: application components, data and AI artefacts from code to cloud, including vulnerable AI library dependencies (TensorFlow, PyTorch, LangChain) found in code and container images.
- **Recommendations** on identity (local auth, managed identity), data security and network exposure.
- **Attack path analysis**: for example, an internet-exposed VM with a credential that can reach an AI resource grounded on sensitive data.
- **IaC scanning** via DevOps security for misconfigurations before deployment.
- **AI agent discovery** (Preview): from 1 July 2026, agent discovery and posture for Foundry and third-party cloud agents requires a **Microsoft Agent 365** licence; the inventory is in the Defender portal.

### Threat protection

Works with Azure AI Content Safety Prompt Shields and Microsoft threat intelligence, scanning text tokens (not images or audio) for supported Azure OpenAI and Azure AI Model Inference models.

| Alert family | Examples |
|---|---|
| **Prompt attacks** | Jailbreak attempt blocked or detected (direct injection); ASCII smuggling (invisible instructions, often indirect injection) |
| **Data leakage** | Credential theft attempt (credentials in model responses) |
| **Malicious content** | Malicious URL in model response, in user prompt, or from unknown source |
| **Suspicious access** | Access from Tor, suspicious IPs, suspicious user agents, access anomalies, anomalous high-risk operations |
| **Wallet attacks** | Duplicate request floods and volume anomalies (denial of wallet) |
| **Agentic and recon** | Anomalous tool invocation; LLM reconnaissance (attempts to extract system instructions) |

Alerts are mapped to MITRE tactics and flow to **Defender XDR**, where they correlate with identity, endpoint and email signals into incidents; Sentinel ingests them for SIEM workflows (Topic 25). Foundry shows alerts and recommendations in **Risks + alerts**.

**Suspicious prompt evidence** (optional) includes suspicious prompt and response segments in alerts, with sensitive data automatically redacted; when disabled, detection still runs but content is masked. It helps triage but exposes content to SOC analysts; decide deliberately. The plan settings can also enable **data security for AI interactions** (Purview analysis of prompts and responses), which needs a Purview licence.

### End-user and application context

Most AI calls come from an app or gateway, so the alert would otherwise show only the app's identity. Add **user security context** to Azure OpenAI calls (REST API 2025-01-01 and later; Python SDK 1.61.1+ via `extra_body`) so alerts show which end user, source IP and application were involved. Use the Entra object ID, not personal data.

### Pricing

Defender for AI Services is billed by tokens scanned, with a 30-day free trial capped at 75 billion tokens. Commercial clouds only (not Azure Government or 21Vianet at the time of writing).

## How it works on Azure

Enable the plans:

```bash
# Defender CSPM (posture, includes AI-SPM)
az security pricing create --name CloudPosture --tier Standard

# Defender for AI Services (threat protection); confirm plan names with: az security pricing list -o table
az security pricing create --name AI --tier Standard
```

Or use Azure Policy / the Defender for Cloud environment settings at management group scale. Enabling at subscription level needs Owner (or equivalent data actions).

Pass user context from your app:

```python
from openai import AzureOpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

client = AzureOpenAI(
    azure_endpoint="https://aif-contoso.openai.azure.com",
    azure_ad_token_provider=get_bearer_token_provider(DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default"),
    api_version="2025-04-01-preview",
)
resp = client.chat.completions.create(
    model="gpt-5-mini",
    messages=[{"role": "user", "content": user_text}],
    extra_body={"user_security_context": {
        "application_name": "contoso-support-portal",
        "end_user_id": user_object_id,          # Entra object ID, no names or emails
        "end_user_tenant_id": tenant_id,
        "source_ip": client_ip,
    }},
)
```

(User security context isn't supported for models called through the Azure AI Model Inference API.)

Operate it:

| Step | Where |
|---|---|
| Review AI inventory and recommendations | Defender for Cloud → Cloud security explorer / Inventory; Foundry **Risks + alerts** |
| Fix attack paths | Attack path analysis |
| Triage alerts | Defender XDR incidents; Sentinel analytics and playbooks |
| Automate response | Logic Apps playbooks: disable a key, block an IP in APIM, notify owner |
| Enable Purview data security | From Defender for Cloud (file 05) |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Scope** | Enable both plans on every subscription hosting AI resources, via policy at management group level. |
| **Prompt evidence** | Enable where SOC needs it for triage; restrict analyst access and document the privacy decision. |
| **User context** | Mandatory in your app standards for production AI apps. |
| **Agent posture** | Plan Agent 365 licensing if you need Defender agent inventory and posture. |
| **Integration** | Defender XDR as primary; Sentinel for cross-source correlation and long retention. |
| **Response automation** | Automate containment for high-confidence alerts (wallet attacks, malicious IPs); human review for jailbreak alerts. |

## Hands-on

1. Enable Defender CSPM and Defender for AI Services on a test subscription.
2. Review AI inventory and fix one recommendation (for example, disable local auth on a Foundry resource).
3. Add `user_security_context` to a test app; send a known jailbreak prompt and find the alert with user context in Defender XDR.
4. Explore an attack path involving an AI resource.
5. Create a Logic Apps playbook that notifies the AI app owner on a jailbreak alert.

## Security, cost and reliability notes

- **Coverage gaps**: text tokens only; specific model APIs; commercial clouds; verify for your workloads.
- **Cost**: billed by tokens scanned; high-volume apps should model cost during the trial.
- **Privacy**: prompt evidence and user IDs in alerts are personal data; apply SOC access controls and retention.
- **Alert fatigue**: jailbreak attempts on public apps are frequent; tune automation and severity handling.

## Common mistakes

| Mistake | Fix |
|---|---|
| Relying on guardrails with no SOC visibility | Enable AI threat protection and XDR integration |
| Alerts show only the app's identity | Pass user security context |
| Enabling per subscription by hand | Policy at management group scope |
| Ignoring posture recommendations | Treat AI resources like any production workload |
| Putting emails in `end_user_id` | Use Entra object IDs |

## Check yourself

1. What's the difference between AI security posture management and AI threat protection?
2. Which platforms does Defender discover AI workloads on?
3. Name five AI alert types.
4. Why add user security context to API calls?
5. What changed for AI agent discovery on 1 July 2026?
6. Where do AI alerts surface for the SOC?

<details>
<summary>Answers</summary>

1. Posture (Defender CSPM): inventory, AI BOM, recommendations, attack paths. Threat protection (Defender for AI Services): real-time alerts on attacks and misuse.
2. Azure OpenAI, Microsoft Foundry, Azure Machine Learning, Amazon Bedrock, Google Vertex AI.
3. Any five: jailbreak blocked/detected, ASCII smuggling, credential theft, malicious URL, access from Tor or suspicious IP, suspicious user agent, wallet attacks, anomalous tool invocation, LLM reconnaissance.
4. So alerts identify the end user, source IP and application behind app-mediated calls, enabling triage and response.
5. Agent discovery and posture for Foundry and third-party cloud agents moved to requiring a Microsoft Agent 365 licence.
6. Defender for Cloud, Defender XDR incidents, Microsoft Sentinel, and Foundry's Risks + alerts.

</details>

## References

- [AI threat protection](https://learn.microsoft.com/azure/defender-for-cloud/ai-threat-protection)
- [AI security posture management](https://learn.microsoft.com/azure/defender-for-cloud/ai-security-posture)
- [Enable threat protection for AI services](https://learn.microsoft.com/azure/defender-for-cloud/ai-onboarding)
- [Alerts for AI services](https://learn.microsoft.com/azure/defender-for-cloud/alerts-ai-workloads)
- [Gain end-user context for AI alerts](https://learn.microsoft.com/azure/defender-for-cloud/gain-end-user-context-ai)
