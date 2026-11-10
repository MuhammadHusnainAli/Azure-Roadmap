# Regulatory Compliance

## Why it matters

Regulated clients do not ask "is Azure secure?" They ask "can we show our auditor that this workload meets our obligations?" Compliance questions decide regions, deployment types, encryption, logging, and sometimes whether an AI use case can go ahead at all. A consultant has to answer them precisely and without overclaiming; an architect has to turn obligations into controls.

This file explains the mechanisms. It is not legal advice, and regulations change: confirm obligations with the client's compliance and legal teams.

## Key concepts

### Platform compliance vs workload compliance

The most important distinction in the topic:

| | Platform compliance | Workload compliance |
|---|---|---|
| Who | Microsoft | You |
| What | Azure's data centres, infrastructure and services are audited against standards | Your application, data handling, configuration and processes meet your obligations |
| Evidence | Third-party audit reports and certificates | Your own controls, configuration evidence, policies and records |

Azure being certified for a standard means the platform will not be the reason you fail. It does not make your workload compliant. This follows directly from the shared responsibility model (topic 01).

### Compliance offerings

Microsoft maintains a large portfolio of attestations. Categories and examples:

| Category | Examples |
|---|---|
| **Global** | ISO/IEC 27001, 27017, 27018, 27701; ISO 22301; SOC 1, 2 and 3; CSA STAR; ISO/IEC 42001 (AI management systems) for in-scope AI services |
| **US government** | FedRAMP High, DoD Impact Levels, CJIS, ITAR, IRS 1075, NIST SP 800-171 |
| **Financial services** | PCI DSS, SOX, GLBA, and regional regulators' frameworks such as the EU's DORA |
| **Healthcare and life sciences** | HIPAA and HITECH (with a Business Associate Agreement), HITRUST, GxP |
| **Regional** | GDPR (EU), UK G-Cloud and Cyber Essentials Plus, Germany C5, France HDS, Australia IRAP, Japan ISMAP, Singapore MTCS, India MeitY, and many more |

Scope matters. An attestation covers **specific services in specific regions and clouds**. Before telling a client "Azure is certified for X", check that every service in the design, including newer AI services, is in scope for that audit.

### Where the evidence lives

| Source | Contains |
|---|---|
| **Service Trust Portal** (`servicetrust.microsoft.com`) | Audit reports, certificates, penetration test summaries, white papers. Some documents need a signed-in customer account |
| **Azure compliance documentation** | A page per offering with scope and guidance |
| **Microsoft Trust Center** | Plain-language commitments on privacy, security and compliance |
| **Product Terms and Data Protection Addendum (DPA)** | The contractual commitments, including GDPR processor terms |
| **Azure Policy regulatory compliance initiatives** | Technical control mappings you can assign |
| **Transparency notes and responsible AI documentation** | Intended use, limitations and data handling for AI services |

### Data protection concepts

| Concept | Meaning in Azure |
|---|---|
| **Controller and processor** | Under GDPR the customer is usually the controller and Microsoft the processor for customer data in Azure services |
| **Data residency** | Where data is stored at rest. For regional services, in the chosen region's geography |
| **Data sovereignty** | Which jurisdiction's laws apply to the data, and who can be compelled to disclose it. Broader than residency (file 10) |
| **Data in transit and processing** | May differ from residency. Some services and deployment types process outside the region |
| **EU Data Boundary** | Microsoft's commitment to store and process customer data, pseudonymised personal data and professional services data within the EU and EFTA for in-scope services, with documented exceptions |
| **Customer Lockbox** | Requires your explicit approval before Microsoft engineers access your data during support |
| **Customer-managed keys** | You hold the encryption key in Key Vault or Managed HSM; revoking it renders data unreadable |
| **Confidential computing** | Protects data in use with hardware-based trusted execution environments |

### Compliance for AI workloads

AI adds obligations on top of ordinary data protection.

**What Microsoft commits for Foundry Models sold directly by Azure** (verify the current data, privacy and security documentation):

- Prompts and completions are **not used to train** foundation models.
- Customer data is **not shared** with other customers or with the model provider (for example OpenAI).
- Data is processed within the boundary implied by the **deployment type** (Regional, Data Zone or Global) and stored at rest in the resource's geography.
- **Abuse monitoring** may store prompts and completions for a limited period for review; eligible customers can apply to modify this for sensitive workloads.

**Partner and community models** in the catalogue can have different terms, particularly when delivered through Marketplace or hosted by the provider. Read the model's licence and data handling terms before approving it. This is one job of a model allow-list.

**Questions you must be able to answer for any AI design**

| Question | Where the answer comes from |
|---|---|
| Where are prompts and responses processed? | Deployment type |
| Where is data stored at rest? | Region of each resource |
| Is customer data used for training? | Service terms; no for Azure-direct models |
| Who can see prompts? | RBAC, logging design, abuse monitoring settings |
| How long are conversations kept? | Your thread storage and log retention settings |
| Can a user's data be deleted on request? | Your application design across threads, memory, indexes, logs |
| Does grounding respect access rights? | Security trimming and identity design |
| Are outputs logged and reviewable? | Tracing and audit configuration |
| Which model produced a given output, and when? | Deployment and version logging |

**Regulation specific to AI**

| Framework | Nature | What it asks of you |
|---|---|---|
| **EU AI Act** | Law, phasing in through 2027 | Classify each use case by risk. Prohibited practices are banned; **high-risk** systems (for example in employment, credit, essential services, education, law enforcement) carry obligations for risk management, data governance, documentation, logging, human oversight, accuracy and robustness. **Transparency** duties apply to chatbots and generated content. General-purpose model providers have their own duties. Deployers, not just providers, have obligations |
| **NIST AI Risk Management Framework** | Voluntary framework | Govern, Map, Measure, Manage |
| **ISO/IEC 42001** | Certifiable standard | An AI management system: policy, roles, risk assessment, lifecycle controls |
| **ISO/IEC 23894** | Guidance | AI risk management |
| **Sector and national rules** | Varies | Financial model risk management, medical device regulation, employment and consumer protection law, and emerging national AI laws |
| **GDPR** | Law | Lawful basis, purpose limitation, data minimisation, data subject rights, and safeguards around solely automated decisions with legal or similarly significant effects; data protection impact assessments for high-risk processing |

Timelines and details of AI regulation are still moving. Check current status before advising.

The practical output of all this is an **AI use-case register**: every AI system, its purpose, risk classification, data categories, owner, model, and the controls applied. Topic 23 covers the governance operating model.

## How it works on Azure

### Turning obligations into controls

```text
Regulation / standard
   └─► Control objectives         ("Access to cardholder data is restricted by business need")
        └─► Technical controls    (RBAC at resource scope, PIM, private endpoints, no local auth)
             └─► Enforcement      (Azure Policy, Conditional Access, IaC modules)
                  └─► Evidence    (Policy compliance state, Defender for Cloud, logs, access reviews)
```

### Tools

| Tool | Role |
|---|---|
| **Azure Policy regulatory compliance initiatives** | Built-in initiatives mapped to standards: Microsoft cloud security benchmark, ISO 27001, NIST SP 800-53, PCI DSS, CIS benchmarks, HIPAA/HITRUST, SOC 2, and regional standards. Each maps policy definitions to the standard's controls |
| **Microsoft Defender for Cloud: regulatory compliance dashboard** | Shows compliance per standard and control, marks which controls are Microsoft's responsibility, links failing resources, and exports reports. Extends to AWS and GCP |
| **Microsoft cloud security benchmark (MCSB)** | Microsoft's own baseline, mapped to common frameworks. The default initiative behind Secure Score |
| **Microsoft Purview Compliance Manager** | Assessment templates for hundreds of regulations, improvement actions, evidence tracking and a compliance score, covering Microsoft 365, Azure and multicloud. Includes AI regulation templates |
| **Microsoft Purview** (data security and governance) | Classification, sensitivity labels, data loss prevention, retention, audit, eDiscovery, and data security posture management for AI |
| **Azure Monitor and Microsoft Sentinel** | Log retention, audit trails, detection |
| **Customer Lockbox, Key Vault and Managed HSM, confidential computing** | Technical safeguards for data access and encryption |

**What built-in initiatives do and do not prove.** A compliance initiative covers the controls that can be checked from resource configuration. Many controls are procedural (training, incident response, vendor management) and cannot be assessed by policy. A green dashboard is evidence for part of an audit, never the whole of it. Controls marked *manual* need attestation.

### Common technical requirements and how to meet them

| Requirement | Azure implementation |
|---|---|
| Data stays in a jurisdiction | Allowed-locations policy; Regional or Data Zone model deployments; check non-regional services; disable geo-replication outside the boundary |
| Encryption at rest with customer control | Customer-managed keys in Key Vault or Managed HSM |
| Encryption in transit | TLS 1.2 or later enforced by policy |
| Restricted access | Entra ID with MFA, RBAC least privilege, PIM, no local authentication |
| Network isolation | Private endpoints, no public network access, controlled egress |
| Audit trail | Activity log, resource diagnostic logs and Entra logs to a central workspace with defined retention and immutability where required |
| Separation of duties | Distinct roles, PIM approval, pipeline-based change |
| Provider access control | Customer Lockbox |
| Data subject rights | Application-level design for export and deletion across all stores |
| Resilience and recoverability | Zone redundancy, backup, tested disaster recovery |
| Vendor assurance | Service Trust Portal reports, DPA, sub-processor list |

### A compliance conversation, structured

1. **Which obligations apply?** Regulations, contracts, internal policy, by data category and geography.
2. **Classify the data and the use case.** Personal, special category, financial, health; AI risk class.
3. **Check service scope.** Is every service and model in scope for the needed attestations, in the needed region?
4. **Decide residency and processing boundaries.** Regions and deployment types.
5. **Map controls.** Choose the policy initiatives and Purview templates; identify procedural controls.
6. **Design evidence.** What will be shown to the auditor, and where it is kept.
7. **Record residual risk and exceptions**, with owners.

## Design decisions and trade-offs

| Decision | Trade-off |
|---|---|
| Regional vs Global model deployments | Tighter processing boundary vs model choice, quota and price |
| Customer-managed vs platform-managed keys | Control and revocability vs operational burden and the risk of losing access to your own data |
| Logging prompts and responses | Auditability and investigation vs storing sensitive content and creating a new privacy obligation |
| Strict deny policies vs audit | Assurance vs delivery friction |
| One compliance baseline for all vs per-workload | Simplicity vs over-constraining low-risk workloads |
| Approving partner models | Capability vs additional terms and due diligence |

**Proportionality.** Apply the strict baseline where the data or use case warrants it (a dedicated management group for regulated workloads is a legitimate pattern) and avoid imposing the heaviest controls on a public-information chatbot.

## Hands-on

1. **Service Trust Portal:** sign in and download the Azure SOC 2 Type II report and the ISO 27001 certificate. Find the list of in-scope services and check whether Azure AI Search and Microsoft Foundry (or Azure OpenAI) appear.
2. **Compliance offering page:** open the documentation for one standard relevant to you and note the in-scope clouds and services.
3. **Regulatory compliance dashboard:** in Defender for Cloud, open *Regulatory compliance*. Review the Microsoft cloud security benchmark. Open one failing control, see the resources and the remediation. Note which controls are marked as Microsoft-managed.
4. **Add a standard:** in *Environment settings → Security policies*, add a standard such as ISO 27001 or NIST SP 800-53 to a test subscription and view it in the dashboard after evaluation.
5. **Inspect an initiative:**

   ```bash
   az policy set-definition list \
     --query "[?policyType=='BuiltIn' && contains(metadata.category,'Regulatory Compliance')].displayName" -o tsv | sort

   az policy set-definition list \
     --query "[?displayName=='Microsoft cloud security benchmark'].{Policies:length(policyDefinitions), Groups:length(policyDefinitionGroups)}"
   ```

6. **Purview Compliance Manager:** browse the assessment templates and find those for the EU AI Act, ISO/IEC 42001 and NIST AI RMF.
7. **Paper exercise:** for the HR assistant from earlier files, answer the nine AI design questions in the table above.

## Security, cost and reliability notes

- **Security:** compliance is a floor, not a ceiling. A compliant system can still be breached.
- **Cost:** compliance drives cost through premium SKUs (customer-managed keys, Managed HSM, private networking), log retention and the people who maintain evidence. Estimate it.
- **Cost:** Defender for Cloud's compliance dashboard for standards beyond the default benchmark requires a paid Defender plan.
- **Reliability:** resilience is itself a regulatory requirement in several sectors (for example operational resilience rules in financial services). Disaster recovery tests produce compliance evidence.

## Common mistakes

- "Azure is certified, so we are compliant."
- Not checking whether a new AI service or model is in scope for the required attestation.
- Treating data residency at rest as the whole answer and overlooking where processing happens.
- Assuming all models in the catalogue have the same data-handling terms.
- A green policy dashboard presented as full compliance, ignoring procedural controls.
- Logging full prompts and responses indefinitely with broad access.
- No way to delete a user's data from conversation history, memory and indexes.
- No register of AI use cases, so nobody can say which are high-risk.
- Giving legal conclusions instead of involving the client's compliance and legal teams.

## Check yourself

1. Distinguish platform compliance from workload compliance.
2. Where do you obtain Azure's audit reports?
3. What is the difference between data residency and data sovereignty?
4. What does Microsoft commit regarding training on customer prompts for Azure-direct models?
5. Why must partner models be reviewed separately?
6. What can and cannot a regulatory compliance initiative in Azure Policy demonstrate?
7. Name three obligations the EU AI Act places on high-risk AI systems.
8. What is Customer Lockbox?

<details>
<summary>Answers</summary>

1. Platform compliance is Microsoft's audited assurance for Azure itself. Workload compliance is the customer's responsibility for their own application, data and configuration.
2. The Service Trust Portal.
3. Residency is where data is stored. Sovereignty concerns which laws and authorities have jurisdiction over it and who can access or compel disclosure.
4. Prompts and completions are not used to train foundation models and are not shared with other customers or the model provider.
5. They may be offered under different licence and data-handling terms, sometimes hosted or billed through a third party.
6. It demonstrates the state of controls that can be evaluated from resource configuration. It cannot demonstrate procedural controls, which need manual attestation and other evidence.
7. Any three of: risk management, data governance, technical documentation, logging and record-keeping, transparency to users, human oversight, accuracy, robustness and cybersecurity.
8. A feature that requires the customer's explicit approval before Microsoft support engineers can access customer data.

</details>

## References

- [Azure compliance documentation](https://learn.microsoft.com/azure/compliance/)
- [Service Trust Portal](https://servicetrust.microsoft.com/)
- [Microsoft Trust Center](https://www.microsoft.com/trust-center)
- [Regulatory compliance in Microsoft Defender for Cloud](https://learn.microsoft.com/azure/defender-for-cloud/regulatory-compliance-dashboard)
- [Azure Policy regulatory compliance built-in initiatives](https://learn.microsoft.com/azure/governance/policy/samples/#regulatory-compliance)
- [Microsoft cloud security benchmark](https://learn.microsoft.com/security/benchmark/azure/overview)
- [Microsoft Purview Compliance Manager](https://learn.microsoft.com/purview/compliance-manager)
- [Data, privacy, and security for Azure OpenAI in Foundry Models](https://learn.microsoft.com/azure/foundry/responsible-ai/openai/data-privacy)
- [EU Data Boundary](https://learn.microsoft.com/privacy/eudb/eu-data-boundary-learn)
- [Customer Lockbox for Microsoft Azure](https://learn.microsoft.com/azure/security/fundamentals/customer-lockbox-overview)
- [Microsoft Products and Services Data Protection Addendum](https://www.microsoft.com/licensing/docs/view/Microsoft-Products-and-Services-Data-Protection-Addendum-DPA)
