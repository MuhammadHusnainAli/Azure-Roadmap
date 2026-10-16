# Azure Landing Zones

> Topic: 03 · Governance and Landing Zones · Tracks: E ○ A ● C ● · Status checked: 2026-10

## Why it matters

An Azure landing zone is Microsoft's reference architecture for a governed, secure, scalable multi-subscription environment. It is the recommended starting point for any organisation running more than a handful of workloads, and it is what "being ready for AI at scale" means in infrastructure terms. Architects design and tailor it; consultants assess clients against it; engineers deploy workloads into it and need to understand the guardrails they meet.

## Key concepts

### Definition

An Azure landing zone has two components:

| Component | What it is | Owned by |
|---|---|---|
| **Platform landing zone** | The centralised foundation: management group hierarchy, policies, and shared services such as connectivity, identity, management and security | Platform team |
| **Workload landing zones** | The environments where workload teams deploy and operate, inside the platform's guardrails. Each workload has one, containing its environments (dev, test, production), each of one or more subscriptions | Workload teams |

Most organisations have **one platform landing zone per Entra tenant** and many workload landing zones.

Terminology note: older material calls workload landing zones *application landing zones*, and the whole architecture *enterprise-scale*. You will still see both.

### Design principles

| Principle | Meaning |
|---|---|
| **Subscription democratisation** | Subscriptions are units of management and scale, given freely to workload teams |
| **Policy-driven governance** | Guardrails are enforced by Azure Policy, giving teams autonomy within safe bounds |
| **Single control and management plane** | One consistent way to manage everything, through Azure Resource Manager |
| **Application-centric service model** | Organise around workloads, not around infrastructure silos |
| **Align with Azure-native design and roadmaps** | Prefer native platform capabilities over custom tooling |

When a design question has no obvious answer, test the options against these.

### The design areas

Eight areas, each a set of decisions to make before building.

**Environment design areas**

| Area | Key decisions |
|---|---|
| **Azure billing and Microsoft Entra tenant** | Agreement type, billing structure, single or multiple tenants |
| **Identity and access management** | Hybrid identity, RBAC model, privileged access, workload identities |
| **Resource organisation** | Management group hierarchy, subscription strategy, naming and tagging |
| **Network topology and connectivity** | Hub-and-spoke or Virtual WAN, hybrid connectivity, DNS, private endpoints, internet ingress and egress |

**Compliance design areas**

| Area | Key decisions |
|---|---|
| **Security** | Security baseline, Defender for Cloud, Sentinel, key management, encryption |
| **Management** | Monitoring, logging, patching, backup, business continuity |
| **Governance** | Policy set, cost management, compliance reporting |
| **Platform automation and DevOps** | IaC toolchain, repositories, pipelines, how the platform itself is deployed and changed |

### Platform landing zone contents

| Subscription | Typical contents |
|---|---|
| **Connectivity** | Hub VNet or Virtual WAN hub, Azure Firewall, VPN and ExpressRoute gateways, DDoS Protection plan, private DNS zones, DNS Private Resolver |
| **Identity** | Domain controllers or Entra Domain Services, identity-related services |
| **Management** | Central Log Analytics workspace, Automation, Update Manager configuration, monitoring baselines |
| **Security** | Microsoft Sentinel workspace and security tooling (often a separate subscription and workspace from operations) |

What you centralise depends on your needs. The rule: centralise a capability only where there is a clear governance, operational or economic benefit across many workloads.

### Network topologies

| | **Hub-and-spoke** | **Virtual WAN** |
|---|---|---|
| Hub | A VNet you build and operate | A Microsoft-managed hub |
| Routing | You manage route tables | Managed, with routing intent |
| Control | Maximum | Less, in exchange for simplicity |
| Fits | Custom requirements, few regions, need for specific network virtual appliances | Many branches and regions, large-scale any-to-any connectivity |

Either way, each workload landing zone gets a **spoke** VNet peered to the hub, with traffic routed through central inspection. Topic 05 covers the detail.

### The policy baseline

The accelerators ship with a large set of policy assignments (well over a hundred) organised by management group. Themes:

- **Guardrails:** deny public IPs where inappropriate, deny public network access on PaaS in *Corp*, allowed regions, enforce TLS versions.
- **Hygiene by automation:** deploy diagnostic settings, deploy Defender for Cloud plans, deploy monitoring agents, configure backup.
- **Networking:** private DNS zone integration for private endpoints, deny IP forwarding, enforce NSGs on subnets.
- **Workload-specific guardrail sets:** optional initiatives per service, including AI services.

The baseline is a starting point. Review every assignment: decide what to keep, parameterise or drop, and document why.

### Workload landing zone types

A workload landing zone is a subscription (or several) prepared for a kind of workload. Microsoft publishes accelerators for common ones.

| Accelerator | For |
|---|---|
| AKS landing zone | Kubernetes workloads |
| App Service landing zone | Web applications |
| Container Apps landing zone | Serverless containers |
| API Management landing zone | API platforms |
| **Foundry / AI landing zone** | AI applications and agents (file 04) |
| Data landing zones (cloud-scale analytics, Fabric) | Data platforms |
| Azure Virtual Desktop, SAP, Azure VMware Solution, Oracle | Specialised platforms |
| Azure Local | Workloads at the edge or in sovereign environments |

A workload team does not need an accelerator; they can deploy anything that satisfies the guardrails. Accelerators just encode good practice for a given technology.

### Implementation options

| Option | Description | Fits |
|---|---|---|
| **Portal accelerator** | A guided deployment in the Azure portal | Evaluation, small estates, teams without IaC maturity |
| **Bicep accelerator** | IaC built on Azure Verified Modules, with bootstrap for repositories and pipelines | Teams standardised on Bicep |
| **Terraform accelerator** | IaC built on Azure Verified Modules, with the same bootstrap | Teams standardised on Terraform |
| **Custom build** | Your own implementation of the same architecture | Organisations with specific needs and strong platform engineering |
| **Partner solutions** | Partner-delivered landing zones | Organisations wanting managed delivery |

The **IaC accelerator** bootstraps the delivery tooling too: it creates the repository, pipelines, identities and state storage (for Terraform), then deploys the platform through those pipelines. That means the landing zone is managed as code from day one.

The older `ALZ-Bicep` and `caf-enterprise-scale` Terraform module have been succeeded by the AVM-based accelerators. Existing deployments on the older code should plan migration.

For most organisations an accelerator is the quickest route to an environment aligned with Microsoft's recommendations.

### Variants

- **Sovereign landing zone:** the reference architecture with added controls for data sovereignty, confidential computing and stricter policy (file 10).
- **Azure landing zones for regulated industries:** additional policy initiatives for financial services, healthcare and others.
- **Start small and expand:** a reduced starting point for smaller organisations that follows the same principles and can grow.

## How it works on Azure

### Platform and workload responsibilities

| Concern | Platform team | Workload team |
|---|---|---|
| Management groups and policy | Designs and operates | Complies; requests exemptions |
| Hub networking, firewall, DNS, hybrid connectivity | Owns | Consumes |
| Spoke VNet | Provisions through vending (or delegates) | Uses; manages subnets and NSGs within limits |
| Subscription | Creates and hands over | Owns contents and cost |
| Identity platform | Owns Entra configuration, PIM, Conditional Access | Manages its own workload identities and role assignments within its scope |
| Central logging and security monitoring | Owns workspaces and Sentinel | Sends logs; owns application telemetry |
| Workload resources, code, data | – | Owns end to end |
| Workload reliability, cost optimisation | Provides guidance and tooling | Accountable |

The split is the enterprise operating model from CAF in practice. The platform team's product is the landing zone; its customers are workload teams.

### Technology adoption

A landing zone does not change shape for new technology. When something new arrives, AI included:

1. Workload resources go into ordinary workload landing zones.
2. The platform team adds or updates **policies** for the new resource types.
3. Shared services are added only if they benefit many workloads.

Current reference diagrams reflect this: Microsoft Foundry and Foundry IQ appear as optional capabilities attached to workload landing zones, with Entra, Defender, Purview and Agent 365 providing unified identity, security and governance across the top.

### Reviewing an existing environment

The **Azure landing zone review** assessment compares an environment to the reference across all design areas and produces prioritised recommendations. For a brownfield client:

1. Run the review and Resource Graph inventory.
2. Agree the target hierarchy and policy set.
3. Build the platform landing zone alongside the existing estate.
4. Assign policies in **audit** (or `DoNotEnforce`) mode and measure non-compliance.
5. Move subscriptions in waves; remediate.
6. Switch policies to **deny** once the estate is clean.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Accelerator vs custom | Accelerator unless you can name a requirement it cannot meet. You still own and must understand what it deploys |
| Bicep vs Terraform | Whichever your platform team already runs well. Do not introduce a second IaC language for the landing zone alone |
| Hub-and-spoke vs Virtual WAN | Hub-and-spoke for control and simpler estates; Virtual WAN for global scale and many branches |
| Single vs multi-region hubs | One hub per region where workloads live, with defined cross-region routing |
| Central vs separate Sentinel workspace | Commonly a dedicated security workspace, with operational logs in the management workspace |
| How much of the policy baseline | Start with defaults in audit where you are unsure, tighten deliberately |
| Workload autonomy | Give teams Owner-like control inside their subscription with guardrails by policy, rather than Contributor plus tickets for every role assignment |
| Networking in workload subscriptions | Platform-provided spoke by default; workload-managed VNets for isolated or specialised cases |

**Cost of a platform landing zone.** The fixed monthly cost is dominated by Azure Firewall, gateways, DDoS Protection and log ingestion. For a small organisation this can exceed the workloads themselves at first. Size the hub for actual need (for example Firewall Basic or Standard, IP-level DDoS protection) and grow.

## Hands-on

Deploying a full landing zone needs a dedicated tenant and elevated rights. Do the review path unless you have a sandbox tenant.

**Review path**

1. Open the Azure landing zone reference architecture diagram on Microsoft Learn (download the Visio or PDF). Trace one workload's traffic from a user on the internet to a private database.
2. Read the list of default policy assignments (`aka.ms/alz/policies`). Pick ten and write down whether your organisation would keep, tune or drop each.
3. Run the **Azure landing zone review** assessment for a real or imagined environment.
4. For each of the eight design areas, write the single most important decision for that environment.

**Deploy path (sandbox tenant only)**

1. Read the accelerator documentation at `aka.ms/alz/accelerator`.
2. Install the prerequisites and run the bootstrap for your IaC of choice, selecting the simplest networking option.
3. Inspect what the bootstrap created: repository, pipelines, managed identities with federated credentials, state storage.
4. Run the pipeline, then explore the resulting management groups, policy assignments and platform subscriptions.
5. Vend one workload subscription and deploy a storage account that violates a policy. Read the denial message.
6. Destroy the environment through the same pipeline.

## Security, cost and reliability notes

- **Security:** the pipelines that deploy the landing zone hold tenant-wide power. Protect the repository with branch policies and approvals, use federated workload identities, and separate plan and apply identities.
- **Security:** the default policy set gives a strong baseline but is not a compliance certification. Map it to your own control framework.
- **Cost:** central log ingestion grows quickly. Decide what is collected, at which table plan, and for how long.
- **Reliability:** platform services are shared dependencies. A misconfigured firewall rule or DNS zone affects every workload. Treat platform changes like production releases, with testing in a non-production platform hierarchy.

## Common mistakes

- Treating the landing zone as a one-off project with no team to run it.
- Deploying an accelerator without understanding the policies it assigns, then being surprised by denials.
- Over-customising until upgrades to newer accelerator versions become impossible.
- Centralising so much that workload teams have no autonomy and every change needs a ticket.
- Building a single-subscription environment and calling it a landing zone.
- No non-production platform environment to test platform changes.
- Designing a separate architecture for AI instead of using workload landing zones with added policy.
- Under-sizing or forgetting DNS design for private endpoints, which blocks most PaaS and AI deployments.

## Check yourself

1. What are the two components of an Azure landing zone, and who owns each?
2. Name the five design principles.
3. List the eight design areas.
4. What typically lives in the connectivity subscription?
5. What does the IaC accelerator bootstrap besides the Azure resources?
6. How does a landing zone accommodate a new technology such as AI?
7. Describe a safe approach to applying landing zone policies to an existing estate.

<details>
<summary>Answers</summary>

1. The platform landing zone, owned by the platform team, and workload landing zones, owned by workload teams.
2. Subscription democratisation, policy-driven governance, single control and management plane, application-centric service model, alignment with Azure-native design and roadmaps.
3. Azure billing and Entra tenant, identity and access management, resource organisation, network topology and connectivity, security, management, governance, platform automation and DevOps.
4. The hub network or Virtual WAN hub, firewall, VPN and ExpressRoute gateways, DDoS protection, private DNS zones and DNS resolver.
5. The repository, pipelines, deployment identities with federated credentials, and state storage, so the platform is managed as code.
6. Workloads go into normal workload landing zones; the platform team adds policies for the new resource types and adds shared services only if broadly useful. The architecture itself does not change.
7. Build the target hierarchy alongside, assign policies in audit mode, measure, move subscriptions in waves and remediate, then switch to deny.

</details>

## References

- [What is an Azure landing zone?](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/)
- [Azure landing zone design principles](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/design-principles)
- [Azure landing zone design areas](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/design-areas)
- [Platform landing zone implementation options](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/implementation-options)
- [Azure landing zones documentation and accelerators](https://azure.github.io/Azure-Landing-Zones/)
- [Deploy Azure landing zones (Architecture Center)](https://learn.microsoft.com/azure/architecture/landing-zones/landing-zone-deploy)
- [Microsoft assessments (includes the Azure landing zone review)](https://learn.microsoft.com/assessments/)
- [Transition existing Azure environments to the landing zone architecture](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/enterprise-scale/transition)
