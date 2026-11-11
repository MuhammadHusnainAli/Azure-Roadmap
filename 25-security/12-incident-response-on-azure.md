# Incident Response on Azure

## Why it matters

Every organisation will have security incidents; what differs is how quickly they detect, contain and recover, and whether the investigation can establish what happened. Cloud incidents behave differently from datacentre ones: evidence comes from APIs and logs rather than seized hardware, attackers abuse identities and control-plane operations, the shared responsibility model decides who can investigate what, and containment can be automated in seconds or done wrong in seconds. Microsoft's guidance aligns with **NIST SP 800-61** (preparation, detection and analysis, containment and recovery, post-incident activity) and adds Azure-specific practices for logging, forensics and collaboration with Microsoft. This file pulls the topic together into an operational response capability.

## Key concepts

### NIST SP 800-61 phases on Azure

| Phase | Azure focus |
|---|---|
| **Preparation** | Azure-specific IR plan, security contacts in Defender for Cloud, logging, playbooks, access for responders, tabletop exercises |
| **Detection and analysis** | Defender XDR and Defender for Cloud alerts, Sentinel analytics, investigation graph, UEBA, asset criticality tags |
| **Containment, eradication and recovery** | Isolate VMs (NSG or Defender for Endpoint isolation), revoke sessions and credentials, disable identities, rotate keys, block IPs/domains, restore from immutable backups, redeploy from IaC |
| **Post-incident activity** | Root cause analysis, lessons learned, control improvements, detection tuning, regulatory notifications |

### Shared responsibility in incidents

| Model | You investigate | Microsoft investigates |
|---|---|---|
| **IaaS** | Guest OS, applications, data, identities, network configuration | Physical hosts, hypervisor, datacentre |
| **PaaS** | Configuration, identities, data, application code | Platform runtime and infrastructure |
| **SaaS** | Identities, data, configuration | Application and infrastructure |

Engage **Microsoft Support** and, for vulnerabilities in Microsoft services, the **Microsoft Security Response Center (MSRC)**. Configure **security contacts** in Defender for Cloud (24/7 primary, backups, role-based) so Microsoft can reach you.

### Essential logs

| Source | Why |
|---|---|
| Entra sign-in and audit logs, risky users and service principals | Identity compromise, consent abuse, privilege changes |
| Azure Activity Log | Control-plane operations (role assignments, key listing, resource changes) |
| VNet/NSG flow logs, firewall and WAF logs | Lateral movement, exfiltration, command and control |
| Resource diagnostics (Key Vault, Storage, SQL, App Gateway, Firewall) | Data access and attack evidence |
| Defender for Endpoint telemetry | Process, file and network activity on servers |
| Microsoft 365 audit (Purview) | Mail, file and collaboration activity |

Send to Log Analytics/Sentinel with retention long enough for investigations (attackers often dwell for weeks); use the Sentinel data lake for long-term retention (file 06).

### Common cloud incident types and first actions

| Incident | First containment actions |
|---|---|
| **Compromised user or admin** | Revoke sessions, reset credentials, require MFA re-registration, review and remove role assignments and app consents, check mailbox rules |
| **Compromised service principal / managed identity misuse** | Remove credentials (secrets/certs), disable the app, review role assignments and recent Activity Log operations |
| **Leaked storage key or SAS** | Rotate keys, revoke user delegation keys, disable shared key auth, review Storage logs |
| **Compromised VM** | Isolate (Defender for Endpoint or restrictive NSG), snapshot disks for forensics before changes, investigate, rebuild from known-good image |
| **Ransomware** | Isolate, protect backups (immutable vaults, multi-user authorisation), restore clean, rotate credentials broadly |
| **Cryptomining / resource abuse** | Identify identity used, stop resources, remove persistence (automation, functions), check quotas and costs |
| **AI app abuse** (jailbreak, data leakage) | Block user/IP, tighten guardrails, review traces and Defender AI alerts ([Topic 23](../23-responsible-ai-security-and-governance/README.md)) |

### Forensics on Azure

- **Disk snapshots** of managed disks (export to a locked forensic storage account with immutability) before rebuilding.
- **Memory capture** via Defender for Endpoint live response or VM tools.
- **Packet capture** with Network Watcher.
- **Chain of custody**: record who collected what, when; hash evidence; restrict access.
- **Isolated forensic subscription** with no trust to production.

### Metrics

Mean time to acknowledge (MTTA) and mean time to remediate (MTTR) are the key SecOps metrics; track per severity and incident type.

## How it works on Azure

Automated containment playbook (Sentinel automation rule → Logic App with managed identity):

```text
Trigger: Sentinel incident, severity High, entity type Account, tactic Credential Access, confidence high
Steps:
 1. Enrich: Entra user risk, recent sign-ins, Defender XDR related alerts
 2. Approval: post an adaptive card in the SOC Teams channel; never auto-disable break-glass or excluded accounts
 3. On approval: revoke sign-in sessions (Microsoft Graph), set account disabled, add incident comment
 4. Notify user's manager and service desk; create ITSM ticket
```

Snapshot a VM disk for forensics with the CLI:

```bash
DISK_ID=$(az vm show -g rg-app -n vm-app01 --query "storageProfile.osDisk.managedDisk.id" -o tsv)
az snapshot create -g rg-forensics -n snap-vm-app01-$(date +%Y%m%d%H%M) --source "$DISK_ID" \
  --network-access-policy DenyAll --tags case=IR-2026-042 collectedBy=analyst1
```

KQL to find role assignments created in the last 24 hours (a common persistence step):

```kusto
AzureActivity
| where TimeGenerated > ago(1d)
| where OperationNameValue =~ "MICROSOFT.AUTHORIZATION/ROLEASSIGNMENTS/WRITE" and ActivityStatusValue == "Success"
| project TimeGenerated, Caller, CallerIpAddress, ResourceGroup, Properties
```

Use Microsoft's incident response playbooks (phishing, password spray, app consent grant, compromised applications) as templates for your own runbooks.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Automation vs approval** | Automate enrichment always; automate containment for high-confidence detections; require approval where business impact is high. |
| **Log retention** | At least 90 days hot for investigation; one year or more in lower-cost tiers for forensics and compliance. |
| **Responder access** | Pre-provisioned, PIM-eligible investigator roles and break-glass procedures; don't create access during an incident. |
| **Backups** | Immutable, separate subscription/tenant protection, regularly tested restores. |
| **Rebuild vs clean** | Prefer redeploying from IaC and golden images over cleaning compromised systems. |
| **Communication** | Pre-agreed legal, PR and regulator notification paths (GDPR 72-hour breach notification, NIS2, sector rules). |

## Hands-on

1. Write an Azure-specific IR plan section: responsibilities by service model, contacts, evidence procedures.
2. Configure security contacts in Defender for Cloud.
3. Build the account-compromise playbook with an approval step; test with a test user.
4. Practise a disk snapshot and evidence handling procedure on a test VM.
5. Run a tabletop exercise for a leaked storage account key; measure time to rotate and assess impact.
6. Write the post-incident review template: timeline, root cause, impact, what worked, actions with owners.

## Security, cost and reliability notes

- **Containment can cause outages**: understand dependencies before disabling identities or isolating hosts.
- **Preserve evidence before remediation** where possible; remediation can destroy it.
- **Attackers watch the SOC**: assume compromised accounts may see incident tickets; use out-of-band channels for sensitive coordination.
- **Cost**: retention and forensic storage cost money; budget for it rather than discovering gaps mid-incident.

## Common mistakes

| Mistake | Fix |
|---|---|
| No logs for the period of compromise | Enable and retain logs before incidents |
| Resetting passwords without revoking sessions and tokens | Revoke sessions; review app consents and credentials |
| Restoring from backups the attacker could delete | Immutable vaults and separate admin domains |
| No practice | Quarterly tabletop exercises |
| No post-incident improvement | Track actions to closure |

## Check yourself

1. List the NIST SP 800-61 phases and an Azure activity for each.
2. How does shared responsibility affect investigation in IaaS vs PaaS?
3. Which logs are essential for Azure incident investigation?
4. What are the first containment actions for a compromised service principal?
5. How do you preserve VM evidence on Azure?
6. Which metrics measure incident response effectiveness?

<details>
<summary>Answers</summary>

1. Preparation (IR plan, contacts, logging), detection and analysis (Defender/Sentinel alerts and investigation), containment/eradication/recovery (revoke, isolate, restore, redeploy), post-incident (root cause, lessons learned, improvements).
2. In IaaS you investigate guest OS and apps; in PaaS you investigate configuration, identities, data and code while Microsoft owns the runtime and infrastructure.
3. Entra sign-in and audit logs, Azure Activity Log, flow/firewall/WAF logs, resource diagnostic logs, endpoint telemetry, Microsoft 365 audit.
4. Remove its secrets and certificates, disable it, review role assignments and recent control-plane operations.
5. Snapshot managed disks to a locked forensic location with tags and hashes, capture memory and packets where needed, and maintain chain of custody.
6. Mean time to acknowledge (MTTA) and mean time to remediate (MTTR).

</details>

## References

- [Incident response overview for Azure](https://learn.microsoft.com/azure/security/fundamentals/incident-response-overview)
- [Plan incident response (Security Adoption Framework)](https://learn.microsoft.com/security/operations/incident-response-overview)
- [Incident response playbooks](https://learn.microsoft.com/security/operations/incident-response-playbooks)
- [Shared responsibility in the cloud](https://learn.microsoft.com/azure/security/fundamentals/shared-responsibility)
- [Well-Architected: incident response](https://learn.microsoft.com/azure/well-architected/security/incident-response)
- [Immutable vaults for Azure Backup](https://learn.microsoft.com/azure/backup/backup-azure-immutable-vault-concept)
