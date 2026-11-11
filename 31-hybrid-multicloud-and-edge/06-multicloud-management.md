# Multicloud Management

## Why it matters

Most large organisations use more than one cloud, by acquisition, by team preference, or deliberately for specific services. Multicloud multiplies operational complexity: separate identity systems, separate security tools, inconsistent policies, fragmented cost data and network paths that nobody fully understands. The goal isn't to make clouds identical but to give the organisation **one view of inventory, security posture, threats, identity and cost**, with clear connectivity and governance patterns. Azure provides much of this through **Defender for Cloud** (AWS and GCP connectors), **Azure Arc** and the **multicloud connector**, **Microsoft Sentinel**, **Microsoft Entra** as the identity plane, and **FOCUS**-based cost data.

## Key concepts

### Capabilities across clouds

| Need | Microsoft capability |
|---|---|
| **Inventory** | **Multicloud connector enabled by Azure Arc**: inventory of AWS (and GCP, Preview) resources in Azure Resource Graph |
| **Server management** | Multicloud connector auto-onboards EC2 instances and GCP VMs to Azure Arc; then Policy, Update Manager, AMA, Defender (files 01–02) |
| **Kubernetes** | Arc onboarding for Amazon EKS (Preview via connector), Arc-enabled Kubernetes for GKE and others; GitOps and policy |
| **Security posture (CSPM)** | **Defender for Cloud** AWS and GCP connectors (agentless): recommendations against MCSB and other standards, attack paths across clouds |
| **Workload protection** | Defender for Servers, Containers, Databases on AWS/GCP |
| **SIEM** | **Microsoft Sentinel** connectors for AWS (CloudTrail, GuardDuty, S3) and GCP logs |
| **Identity** | **Microsoft Entra ID** as IdP: SSO and provisioning to AWS IAM Identity Center and Google Cloud; workload identity federation between clouds; Conditional Access; PIM |
| **Data** | Purview scanning multicloud data sources; Storage Mover for S3 to Azure Storage; Fabric OneLake shortcuts to S3/GCS |
| **Cost** | FOCUS exports from each cloud into FinOps hubs/Power BI (Topic 29) |
| **AI** | Foundry Models across providers; MCP/A2A for agent interoperability; Defender AI posture across Azure, Bedrock and Vertex AI |

### Connectivity patterns

| Pattern | Notes |
|---|---|
| **Internet with TLS** | Simplest; for public APIs |
| **Site-to-site VPN** | Azure VPN Gateway to AWS/GCP VPN; quick, modest bandwidth |
| **ExpressRoute + partner interconnect** | Via colocation/cloud exchange providers to AWS Direct Connect/GCP Interconnect; private, predictable |
| **Private endpoints and private DNS** | Resolve cross-cloud private names carefully; centralised DNS forwarding |
| **API gateways** | APIM or equivalent for cross-cloud service contracts |

Keep cross-cloud traffic to defined integration points; egress costs and latency make chatty cross-cloud architectures expensive and fragile.

### Governance model

| Element | Approach |
|---|---|
| **Landing zones per cloud** | Each cloud has its own landing zone, aligned to common principles |
| **Common policies** | Express security baselines in MCSB/CIS terms; enforce natively in each cloud; measure centrally in Defender for Cloud |
| **Tagging** | Common tag taxonomy across clouds for ownership and cost |
| **Single identity** | Entra as IdP; no local cloud user accounts |
| **Central SOC** | Sentinel + Defender XDR covering all clouds |

## How it works on Azure

Connect an AWS organisation to Defender for Cloud: in Defender for Cloud **Environment settings → Add environment → Amazon Web Services**, select organisation or single account, choose plans (Defender CSPM, Servers, Containers, Databases), and deploy the generated CloudFormation template (creates IAM roles with OIDC trust to Microsoft). Repeat for GCP with a Cloud Shell script.

Multicloud connector for inventory and Arc onboarding: **Azure Arc → Multicloud connectors → Create** → AWS → choose Inventory and Arc onboarding solutions → deploy the CloudFormation template → EC2 instances with SSM get the Connected Machine agent automatically.

Unified inventory query:

```kusto
resources
| where type startswith "microsoft.awsconnector/" or type =~ "microsoft.hybridcompute/machines"
| summarize count() by type, location
```

Entra SSO to AWS: configure the AWS IAM Identity Center enterprise application with SAML and SCIM provisioning; map Entra groups to permission sets; require Conditional Access (MFA, compliant device) for cloud consoles.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Primary cloud** | Choose a primary control plane for security and governance (often Azure with Defender/Sentinel/Entra) without forcing all workloads into one cloud. |
| **Defender connectors vs multicloud connector** | Use Defender for security posture; multicloud connector for inventory and Arc onboarding; they work side by side. |
| **Data placement** | Keep data and compute in the same cloud; integrate via APIs and events. |
| **Portability** | Use Kubernetes, OpenTelemetry, Terraform and open protocols where portability matters; don't pay portability costs where it doesn't. |
| **Skills** | Platform teams per cloud with shared standards; avoid lowest-common-denominator designs. |
| **Cost visibility** | FOCUS-based reporting across clouds. |

## Hands-on

1. Connect an AWS account to Defender for Cloud and review cross-cloud recommendations and attack paths.
2. Create a multicloud connector with inventory and Arc onboarding; confirm EC2 instances appear as Arc servers.
3. Configure Entra SSO and provisioning to AWS IAM Identity Center with Conditional Access.
4. Connect AWS CloudTrail to Sentinel and write a detection for root account usage.
5. Build a cross-cloud cost report from FOCUS exports.

## Security, cost and reliability notes

- **IAM trust**: connectors create cross-cloud roles; scope them minimally and review regularly.
- **Egress**: cross-cloud data transfer is billed in both directions' providers; minimise.
- **Shared fate**: avoid designs where an outage in one cloud breaks workloads in another unexpectedly.
- **Consistent incident response**: runbooks cover all clouds (Topic 25 file 12).

## Common mistakes

| Mistake | Fix |
|---|---|
| Separate identity per cloud with local users | Entra as IdP, SSO, PIM |
| No central security posture | Defender for Cloud multicloud connectors |
| Chatty cross-cloud architectures | Keep data and compute together |
| Different tag and cost models | Common taxonomy and FOCUS |
| Lowest-common-denominator designs everywhere | Portability only where valuable |

## Check yourself

1. What does the multicloud connector provide?
2. How does Defender for Cloud cover AWS and GCP?
3. How should identity be handled across clouds?
4. Name three cross-cloud connectivity patterns.
5. How do you get unified cost visibility?
6. When is portability worth its cost?

<details>
<summary>Answers</summary>

1. Inventory of AWS (and GCP, Preview) resources in Azure, Arc onboarding of EC2/GCP VMs (and EKS in Preview), and S3 data management for Storage Mover.
2. Agentless connectors for CSPM (recommendations, attack paths) plus workload protection plans for servers, containers and databases.
3. Entra ID as the identity provider with SSO and provisioning, Conditional Access, PIM and workload identity federation.
4. Internet with TLS, site-to-site VPN, ExpressRoute with partner interconnects to Direct Connect/Interconnect (plus API gateways).
5. FOCUS exports from each cloud into a common FinOps platform.
6. When regulation, negotiation leverage or specific exit requirements justify it; otherwise use native services.

</details>

## References

- [Multicloud connector enabled by Azure Arc](https://learn.microsoft.com/azure/azure-arc/multicloud-connector/overview)
- [Connect AWS to Defender for Cloud](https://learn.microsoft.com/azure/defender-for-cloud/quickstart-onboard-aws)
- [Connect GCP to Defender for Cloud](https://learn.microsoft.com/azure/defender-for-cloud/quickstart-onboard-gcp)
- [Connect Sentinel to AWS](https://learn.microsoft.com/azure/sentinel/connect-aws)
- [Hybrid and multicloud scenarios (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/hybrid/)
