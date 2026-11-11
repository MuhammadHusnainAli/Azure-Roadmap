# 25 · Security

> Tracks: E ○ A ● C ● · Part E · Secure, Operate, Migrate and Extend

Security on Azure is a set of layered, continuously operated capabilities rather than a single product. This topic covers Zero Trust as the organising strategy; the Microsoft cloud security benchmark as the measurable baseline (including the new AI domain in v2); Azure Key Vault, Managed HSM and Cloud HSM; encryption at rest, in transit and in use, including customer-managed keys, double encryption, confidential computing and the Azure Disk Encryption retirement; Microsoft Defender for Cloud as a CNAPP with attack path analysis; Microsoft Sentinel, its data lake and the move to the Defender portal; network security design; application and API security mapped to the OWASP API Top 10; DevSecOps and software supply chain security; Microsoft Purview compliance; Defender External Attack Surface Management; and incident response on Azure.

## What you will be able to do

- Apply Zero Trust principles and pillars to an Azure estate.
- Use the Microsoft cloud security benchmark and Secure Score as a measurable baseline.
- Design secret, key and certificate management with Key Vault and HSM options.
- Choose encryption and key management models with clear trade-offs.
- Roll out Defender for Cloud plans by risk and act on attack paths.
- Design Sentinel data collection, tiers, detections and automation.
- Design segmented, private-by-default networks with controlled ingress and egress.
- Secure applications and APIs with Entra, APIM, WAF and Defender for APIs.
- Build secure pipelines with scanning, signing, SBOMs and provenance.
- Apply Purview for classification, DLP, retention, audit and eDiscovery.
- Discover and reduce the external attack surface.
- Prepare for and run incident response on Azure.

## Prerequisites

[Topic 02](../02-identity-and-access/README.md) (identity), [Topic 03](../03-governance-and-landing-zones/README.md) (policy and landing zones) and [Topic 05](../05-networking/README.md) (networking). [Topic 23](../23-responsible-ai-security-and-governance/README.md) covers AI-specific security.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Zero Trust Architecture](01-zero-trust-architecture.md) | 35 min | Pillar assessment, Conditional Access, PIM |
| 02 | [Microsoft Cloud Security Benchmark](02-microsoft-cloud-security-benchmark.md) | 35 min | Compliance review, remediation |
| 03 | [Azure Key Vault](03-azure-key-vault.md) | 45 min | Hardened vault, references, rotation |
| 04 | [Encryption](04-encryption.md) | 40 min | CMK storage, encryption at host |
| 05 | [Microsoft Defender for Cloud](05-microsoft-defender-for-cloud.md) | 45 min | Plans, attack paths, governance rules |
| 06 | [Microsoft Sentinel](06-microsoft-sentinel.md) | 50 min | Connectors, analytics rule, playbook |
| 07 | [Network Security Design](07-network-security-design.md) | 45 min | Firewall egress, private endpoints, AVNM |
| 08 | [Application and API Security](08-application-and-api-security.md) | 45 min | Entra-protected API behind APIM |
| 09 | [DevSecOps and Supply Chain](09-devsecops-and-supply-chain.md) | 50 min | Scanning, attestation, signing |
| 10 | [Microsoft Purview Compliance](10-microsoft-purview-compliance.md) | 40 min | Labels, DLP, Data Map scan |
| 11 | [Defender External Attack Surface Management](11-defender-external-attack-surface-management.md) | 30 min | Discovery and triage |
| 12 | [Incident Response on Azure](12-incident-response-on-azure.md) | 45 min | Playbook, forensics, tabletop |

**Architects:** all of it. **Consultants:** 01, 02, 04, 05, 06, 10 and 12 for strategy and assessments. **AI engineers:** 03, 08 and 09, plus 01 for context.

## Completion checklist

- [ ] I can explain Zero Trust principles and map controls to each pillar.
- [ ] I have reviewed MCSB compliance and remediated Must-have controls.
- [ ] I have deployed a hardened Key Vault with RBAC, purge protection, private access and rotation.
- [ ] I can choose between platform keys, CMK, double encryption and confidential computing, and I know the ADE retirement date.
- [ ] I have enabled Defender plans by risk and fixed an attack path.
- [ ] I have built a Sentinel analytics rule and playbook and planned the Defender portal transition.
- [ ] I have designed segmented networking with inspected egress and private endpoints.
- [ ] I have protected an API with Entra, APIM and WAF and tested object-level authorisation.
- [ ] I have a pipeline with secret scanning, code scanning, provenance and signed images verified at deploy.
- [ ] I have configured labels, DLP and a Data Map scan in Purview.
- [ ] I have run EASM discovery and triaged assets.
- [ ] I have an Azure incident response plan and have practised it.

## Maps to

- **AZ-305**: design identity, security, data protection and monitoring solutions.
- **SC-100**: cybersecurity architect (Zero Trust, MCSB, security operations strategy).
- **SC-500 / SC-200** (adjacent): cloud security engineering and security operations.
- **AZ-400** (adjacent): DevSecOps practices.

## Next

26 · [Monitoring and Observability](../26-monitoring-and-observability/README.md)
