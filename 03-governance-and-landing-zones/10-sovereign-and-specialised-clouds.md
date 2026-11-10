# Sovereign and Specialised Clouds

## Why it matters

"Sovereignty" has moved from a niche public-sector concern to a board-level question, particularly in Europe, and AI has sharpened it: organisations want to know who can access the data they send to a model and under which country's law. Architects and consultants need to know the options, from controls inside the public cloud to fully separate clouds, and what each costs in capability. Microsoft's offerings here are evolving quickly, so treat the specifics as a snapshot and verify before advising.

## Key concepts

### Residency, sovereignty, and the rest

| Term | Question it answers |
|---|---|
| **Data residency** | Where is the data stored? |
| **Data sovereignty** | Which jurisdiction's laws govern the data, and who can lawfully access or demand it? |
| **Operational sovereignty** | Who operates the cloud, and can the customer see and control operator access? |
| **Technical / software sovereignty** | Can the workload run without dependence on a single provider, including disconnected? |
| **Digital sovereignty** | The overall ability of an organisation or state to control its digital assets and dependencies |

Residency is necessary and rarely sufficient. A client asking for "sovereignty" may mean any of these. The first task is to find out which, and what specific risk they are trying to address: foreign government access, provider personnel access, service continuity under geopolitical disruption, or regulatory compliance.

### The spectrum of options

```text
More capability, lower cost                              More control, more constraint
─────────────────────────────────────────────────────────────────────────────────────►
Public Azure     Public Azure +        Sovereign public     National partner    Sovereign private /
(regional        sovereign controls    cloud offering       clouds              disconnected
 controls)       (CMK, confidential,   (regional controls   (operated by a      (Azure Local,
                 policy, EU boundary)   and commitments)     local entity)       air-gapped)
```

Most organisations' needs are met in the left half. Moving right reduces service breadth, slows access to new features (AI especially) and increases cost.

### Sovereign controls in public Azure

Before considering a separate cloud, exhaust what the public cloud provides:

| Control | Addresses |
|---|---|
| **Region selection and allowed-locations policy** | Residency |
| **EU Data Boundary** | Storage and processing of customer data within the EU and EFTA for in-scope services |
| **Regional and Data Zone model deployments** | Where AI inference happens |
| **Customer-managed keys** in Key Vault or **Managed HSM**; **external key management** options | Control over encryption keys, including holding them outside Microsoft's control where supported |
| **Azure confidential computing** (confidential VMs, containers, confidential inferencing where available) | Protecting data in use from the operator through hardware-based trusted execution and attestation |
| **Customer Lockbox** | Approval of Microsoft support access |
| **Data Guardian**-style operational commitments in the European offering | Operator access by European-resident personnel, logged in tamper-evident form |
| **Azure Policy sovereignty baselines** | Enforcing the above |
| **Transparency logs** | Visibility of operator actions |
| **Private networking and no public endpoints** | Reducing exposure |

### Microsoft Sovereign Cloud

Microsoft groups its sovereignty offerings under the **Microsoft Sovereign Cloud** umbrella, with three delivery models:

| Model | What it is | Typical fit |
|---|---|---|
| **Sovereign Public Cloud** | Public Azure regions (initially all European regions) with added sovereign controls and commitments applied through policy and configuration. No migration to a different cloud | Organisations wanting full cloud capability with stronger data and operational guarantees |
| **Sovereign Private Cloud** | Azure services on customer or partner infrastructure, built on **Azure Local** with **Microsoft 365 Local**, supporting connected, hybrid and disconnected operation | Workloads that must run in a facility the customer controls, including air-gapped |
| **National Partner Clouds** | Clouds operated by an independent local entity using Microsoft technology, such as **Bleu** in France and **Delos Cloud** in Germany | Public-sector and critical-infrastructure customers bound by national requirements for local operation and control |

This replaced and absorbed the earlier **Microsoft Cloud for Sovereignty** branding. Supporting material includes the **sovereign landing zone** (the Azure landing zone with sovereignty-oriented policy, confidential computing and key management defaults) and the Cloud Adoption Framework **Sovereignty** scenario.

Service availability, the exact controls and the certification status of partner clouds differ by offering and change over time. Confirm each for the specific country and workload.

### Azure Government

A physically and logically separate instance of Azure for US government agencies and their partners.

| Aspect | Detail |
|---|---|
| **Eligibility** | US federal, state, local and tribal government entities and partners handling their data, subject to validation |
| **Regions** | US Gov Virginia, US Gov Texas, US Gov Arizona, plus **DoD** regions; **Azure Government Secret** and **Top Secret** are separate air-gapped clouds for classified data |
| **Operations** | Screened US persons |
| **Compliance** | FedRAMP High, DoD Impact Levels 2, 4 and 5 (IL6 in Secret), CJIS, ITAR, IRS 1075 |
| **Identity** | Separate Entra ID environment (`login.microsoftonline.us`) |
| **Endpoints** | Different from public Azure: portal `portal.azure.us`, ARM `management.usgovcloudapi.net`, storage suffix `core.usgovcloudapi.net`, and so on |
| **Feature parity** | Lags public Azure. New services and AI models arrive later and in a narrower selection |

AI in Azure Government: Azure OpenAI and Foundry capabilities are available with a smaller model catalogue, fewer deployment options and later arrival of new features. Check the current availability tables before promising a specific model.

### Azure operated by 21Vianet (China)

A physically separate instance of Azure in mainland China, **operated and transacted by 21Vianet**, a Chinese company, to meet Chinese regulation.

| Aspect | Detail |
|---|---|
| **Contract** | With 21Vianet, not Microsoft |
| **Regions** | China North, China North 2, China North 3, China East, China East 2, China East 3 |
| **Identity and endpoints** | Separate (`portal.azure.cn`, `login.chinacloudapi.cn`, `management.chinacloudapi.cn`) |
| **Connectivity** | Not connected to Microsoft's global backbone; cross-border links are regulated |
| **Requirements** | ICP filing or licence for public-facing sites; data localisation and cross-border transfer rules |
| **Feature parity** | Narrower service list; AI offerings differ substantially from the global cloud |

A global company with a China presence typically runs a **separate deployment** there, with its own tenant, subscriptions, pipelines and operational processes.

### What "separate cloud" means in practice

A sovereign or national cloud is not a region of public Azure. Expect:

| Area | Impact |
|---|---|
| **Identity** | A separate Entra tenant in that cloud. No shared users, groups or Conditional Access with your public tenant; B2B across clouds is limited and must be explicitly configured |
| **Endpoints** | Different URLs for every service. SDKs, the CLI and IaC need the cloud specified |
| **Services and SKUs** | A subset. Newer services and features arrive later or not at all |
| **Tooling and Marketplace** | Fewer third-party offerings; some management tools unavailable |
| **Networking** | No VNet peering to public Azure. Connectivity is over the internet, VPN or separate private circuits, where permitted |
| **Support and operations** | Different support processes and personnel |
| **Cost** | Usually higher |

Working with another cloud from the CLI:

```bash
az cloud list --query "[].{Name:name, Active:isActive}" -o table
az cloud set --name AzureUSGovernment        # or AzureChinaCloud
az login
az cloud show --query "endpoints.{ARM:resourceManager, Login:activeDirectory, Portal:portal}"
az cloud set --name AzureCloud               # back to public
```

In code, set the authority host and the resource manager endpoint. For example the Azure Identity library's `AzureAuthorityHosts.AZURE_GOVERNMENT`, and the Bicep/ARM `environment()` function, which returns the correct suffixes so templates stay portable:

```bicep
output blobSuffix string = environment().suffixes.storage
output armEndpoint string = environment().resourceManager
```

Hard-coded public endpoints are the commonest reason a template fails in another cloud.

### Sovereignty and AI

AI raises the stakes in three ways:

1. **Where inference happens.** Use Regional or Data Zone deployments and enforce the choice with policy. Global deployments process wherever there is capacity.
2. **Who can see prompts.** Consider abuse-monitoring settings, logging design, Customer Lockbox, and, where the requirement is protection from the operator, confidential inferencing options as they become available.
3. **Model availability vs constraint.** The strictest environments have the fewest and oldest models. For disconnected or highly restricted settings, the realistic choices are open-weight and small language models run on infrastructure you control: Foundry Local, Azure Local with GPU, or AKS with KAITO.

A practical framing for clients:

| Requirement | Likely answer |
|---|---|
| Data stored and processed in the EU | Public Azure, EU region, Data Zone or Regional deployment, EU Data Boundary |
| Plus control of keys and protection from operator access | Add customer-managed or external keys, Customer Lockbox, confidential computing, sovereign policy baseline |
| Plus European operational commitments | Sovereign Public Cloud |
| Must be operated by a national entity | National partner cloud |
| Must run in our facility or disconnected | Sovereign Private Cloud on Azure Local, with locally hosted models |
| US government regulated data | Azure Government, at the right impact level |
| Operations in mainland China | Azure operated by 21Vianet, as a separate deployment |

## How it works on Azure

### The sovereign landing zone

The sovereign landing zone is the standard Azure landing zone with additions:

- Extra **management groups** for confidential workloads (for example *Confidential Corp* and *Confidential Online*).
- A **sovereignty policy baseline**: allowed regions, customer-managed keys, confidential computing SKUs, restricted resource types.
- **Key management** with Managed HSM.
- **Transparency and logging** defaults.
- Deployed with the same accelerators and operating model as any landing zone.

So the skills from files 02 to 05 carry over directly. Sovereignty is implemented largely as policy on the same architecture.

### Assessing a sovereignty requirement

1. **Identify the driver.** Regulation, contract, internal risk policy, or perception.
2. **Classify workloads and data.** Sovereignty requirements usually apply to a subset.
3. **Name the threat.** Residency breach, foreign legal access, operator access, supply disruption.
4. **Map to controls**, starting from the public cloud and moving right only as far as needed.
5. **Check service and model availability** in the candidate environment.
6. **Quantify the trade-off:** capability lost, cost added, operational complexity.
7. **Document** the decision and residual risk, with legal input.

## Design decisions and trade-offs

| Decision | Trade-off |
|---|---|
| Public cloud with controls vs separate cloud | Capability, cost and pace vs stronger jurisdictional separation |
| One environment vs split by classification | Simplicity vs putting only the truly sensitive workloads under the heaviest constraints |
| Customer-managed or external keys | Control vs operational risk and service limitations |
| Confidential computing | Protection in use vs SKU availability, performance and complexity |
| Hosted frontier models vs self-hosted open models | Quality and simplicity vs control and offline operation |
| Disconnected operation | Maximum control vs losing most managed services and all automatic updates |

**Advise proportionately.** A common failure is defaulting to the most restrictive option for the whole organisation, which removes access to current AI capabilities for workloads that never needed that level. Segment.

## Hands-on

1. List available clouds and inspect their endpoints:

   ```bash
   az cloud list -o table
   az cloud show --name AzureUSGovernment --query "{ARM:endpoints.resourceManager, Login:endpoints.activeDirectory, Storage:suffixes.storageEndpoint}"
   az cloud show --name AzureChinaCloud   --query "{ARM:endpoints.resourceManager, Login:endpoints.activeDirectory, Storage:suffixes.storageEndpoint}"
   ```

2. Find and rewrite any hard-coded `.windows.net` or `management.azure.com` strings in a template you own, using `environment()`.
3. Assign an **allowed locations** policy restricting a sandbox subscription to EU regions, then try to deploy elsewhere.
4. Using the policy from file 05, restrict model deployment types to Regional and Data Zone in the same sandbox.
5. Read the Cloud Adoption Framework *Sovereignty* scenario and the Microsoft Sovereign Cloud overview. List the controls it offers and mark which your sandbox already has.
6. **Paper exercise:** a European public-sector body wants an internal document assistant and says "it must be sovereign". Write the five questions you would ask, then propose an option from the table above for two different sets of answers.

## Security, cost and reliability notes

- **Security:** sovereignty controls address a particular set of risks (jurisdiction and operator access). They do not replace ordinary security hygiene, which remains the likelier cause of an incident.
- **Security:** customer-held keys create a self-inflicted availability risk. Losing the key means losing the data. Design key backup and recovery carefully.
- **Cost:** separate clouds, Managed HSM, confidential SKUs and on-premises hardware all cost more. Local GPU capacity for self-hosted models is a significant capital and operational commitment.
- **Reliability:** sovereign and national clouds have fewer regions, which narrows disaster recovery options. Disconnected environments require you to provide all resilience yourself.

## Common mistakes

- Treating residency as equivalent to sovereignty.
- Recommending a separate cloud without identifying the actual risk.
- Assuming feature and model parity with public Azure.
- Hard-coded public endpoints in code and templates.
- Expecting one Entra tenant to span clouds.
- Using Global model deployments in a design sold as "data stays in region".
- Applying the strictest regime to every workload.
- Quoting details of fast-changing sovereign offerings from memory.

## Check yourself

1. Define data residency, data sovereignty and operational sovereignty.
2. Name four controls in public Azure that address sovereignty concerns.
3. What are the three delivery models of Microsoft Sovereign Cloud?
4. Who operates Azure in China, and what follows from that?
5. Give three practical differences when deploying to Azure Government instead of public Azure.
6. Which Foundry deployment types suit a requirement for in-region processing?
7. What are the AI options for a disconnected environment?
8. What is the sovereign landing zone?

<details>
<summary>Answers</summary>

1. Residency: where data is stored. Sovereignty: which jurisdiction's laws govern the data and who can lawfully access it. Operational sovereignty: who operates the cloud and what visibility and control the customer has over operator access.
2. Any four of: region restriction by policy, EU Data Boundary, Regional or Data Zone deployments, customer-managed or external keys, confidential computing, Customer Lockbox, private networking.
3. Sovereign Public Cloud, Sovereign Private Cloud, National Partner Clouds.
4. 21Vianet. The contract is with them, the cloud is physically separate with its own endpoints and identity, service availability differs, and Chinese regulatory requirements apply.
5. Any three of: separate Entra tenant and sign-in endpoint, different service endpoints, fewer services and later feature availability, eligibility requirements, no network peering to public Azure, different support and operations.
6. Regional, and Data Zone where the zone matches the requirement. Not Global.
7. Open-weight and small language models hosted on infrastructure you control, such as Foundry Local, Azure Local with GPUs, or AKS with KAITO.
8. The Azure landing zone architecture with additional management groups, a sovereignty policy baseline, confidential computing and key management defaults.

</details>

## References

- [Sovereignty (Cloud Adoption Framework scenario)](https://learn.microsoft.com/azure/cloud-adoption-framework/sovereignty/executive-strategy)
- [Microsoft Sovereign Cloud documentation](https://learn.microsoft.com/industry/sovereign-cloud/)
- [Sovereign landing zone](https://learn.microsoft.com/industry/sovereign-cloud/sovereign-public-cloud/sovereign-landing-zone/overview-slz)
- [What is Azure Government?](https://learn.microsoft.com/azure/azure-government/documentation-government-welcome)
- [Compare Azure Government and global Azure](https://learn.microsoft.com/azure/azure-government/compare-azure-government-global-azure)
- [Azure in China documentation](https://learn.microsoft.com/azure/china/)
- [EU Data Boundary](https://learn.microsoft.com/privacy/eudb/eu-data-boundary-learn)
- [Azure confidential computing](https://learn.microsoft.com/azure/confidential-computing/overview)
- [Azure Local overview](https://learn.microsoft.com/azure/azure-local/overview)
- [Deployment types for Foundry Models](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/deployment-types)
