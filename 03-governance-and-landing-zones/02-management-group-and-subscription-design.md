# Management Group and Subscription Design

> Topic: 03 · Governance and Landing Zones · Tracks: E ○ A ● C ● · Status checked: 2026-10

## Why it matters

The management group tree and the way you carve subscriptions decide how policy and access are inherited for years. A good structure makes governance mostly automatic. A poor one produces hundreds of exceptions and is painful to change once workloads have landed. This is one of the first diagrams an architect draws for a client.

## Key concepts

### What each level is for

| Level | Use it to | Do not use it to |
|---|---|---|
| **Management group** | Apply policy and access to many subscriptions that share governance needs | Mirror the org chart; separate environments |
| **Subscription** | Create a unit of management, billing and scale with a clear owner | Act as a folder with no owner |
| **Resource group** | Group resources that share a lifecycle | Provide hard isolation between workloads |

### The reference hierarchy

The Azure landing zone reference structure:

```text
Tenant root group
└── Contoso                              (intermediate root)
    ├── Platform
    │   ├── Security                     Sentinel, security tooling
    │   ├── Management                   Log Analytics, automation, monitoring
    │   ├── Identity                     Domain controllers, identity services
    │   └── Connectivity                 Hubs, firewalls, gateways, DNS
    ├── Landing zones                    (workload landing zones)
    │   ├── Corp                         Internal workloads; private connectivity
    │   ├── Online                       Internet-facing workloads
    │   └── Local                        Workloads on Azure Local (optional)
    ├── Sandbox                          Experimentation; isolated from corporate network
    └── Decommissioned                   Subscriptions being retired
```

Why each exists:

| Group | Reason |
|---|---|
| **Intermediate root** | Never assign at the tenant root group. An intermediate root lets you test changes in a parallel hierarchy and keeps the true root clean |
| **Platform** | Shared services run by the platform team, with policies different from workloads |
| **Landing zones** | Where workload teams deploy. The parent holds policies common to all workloads |
| **Corp** | Workloads that need connectivity to the corporate network and must not be publicly exposed |
| **Online** | Workloads that are internet-facing and may not need corporate connectivity |
| **Sandbox** | Safe space to learn, with loose policy, no corporate network peering and tight budgets |
| **Decommissioned** | A holding area with deny policies while a subscription is cleaned up and cancelled |

Platform child groups are optional in smaller estates: a single platform subscription can hold management, identity and connectivity at first.

### Design principles for the tree

1. **Organise by governance need, not by organisation.** Ask "do these subscriptions need different policies or access?" If not, they do not need separate management groups.
2. **Keep it shallow.** Three to four levels. Azure allows six below the root, and you should not need them.
3. **No environment management groups.** Do not create *Prod*, *Test* and *Dev* groups. It multiplies the tree, encourages policy drift between environments so that production differs from what was tested, and complicates promotion. Separate environments with **subscriptions** inside the same management group, and vary only what must differ through policy parameters or a small number of targeted assignments.
4. **No business-unit tree by default.** Org charts change more often than governance needs do. If a business unit truly needs different policy or delegated administration, give it a group under *Landing zones*.
5. **Assign policy high, exempt low.** Apply at the highest group where the rule holds and handle exceptions with exemptions on specific scopes.
6. **Assign access to groups, with PIM.** Platform team roles at *Platform*; workload team roles at their subscriptions.

### Tenant root group settings

- **Default management group for new subscriptions:** change it from the root to a dedicated group (often *Sandbox* or a *Quarantine* group) so that a new subscription is never born ungoverned.
- **Require permission to create management groups:** enable, so not every user can create them.
- **Root-level access:** nobody has it by default. Global Administrators can elevate. Keep it that way.

### Subscriptions as the unit of scale

The landing zone principle of **subscription democratisation**: subscriptions are cheap, plentiful, and handed out to workload teams as their own bounded space. Scale by adding subscriptions, not by cramming more into one.

**Reasons to create a new subscription**

| Reason | Example |
|---|---|
| Separate **environments** | Production and non-production for a workload |
| Separate **workloads** with different owners | Each application team has its own |
| **Quota and limit** isolation | An AI workload needing its own model quota; avoiding the resource group or role assignment ceilings |
| **Blast radius** | Contain a compromise or a mistake |
| **Billing** separation | Different cost centre or invoice section |
| **Compliance boundary** | Regulated data handled under stricter policy |
| **Connectivity** differences | Different network topology or isolation requirements |

**Reasons not to**

- To separate teams that share one workload and lifecycle.
- As a substitute for RBAC at resource-group scope.
- Because a single project asked, when a resource group in a shared subscription would serve.

### Common subscription patterns

| Pattern | Layout | Fits |
|---|---|---|
| **One per workload per environment** | `app-a-prod`, `app-a-nonprod` | The default for significant workloads |
| **Shared landing zone subscription** | Several small workloads in one subscription, separated by resource groups | Small or low-risk apps where per-workload subscriptions would be overhead |
| **Platform subscriptions** | `connectivity`, `identity`, `management`, `security` | Every enterprise landing zone |
| **Sandbox per person or team** | Budget-capped, auto-expiring | Learning and experimentation |
| **Shared AI platform subscription** | Central model gateway, shared Foundry resources or capacity | Organisations offering AI as an internal platform (file 04) |

### Subscription vending

A standard, automated way to create and hand over subscriptions.

```text
Request (form / pull request / ITSM)
   └─► Approval and metadata (owner, cost centre, environment, data classification, network needs)
        └─► Automation
             ├─ create subscription (alias API) under the right billing scope
             ├─ place in the correct management group
             ├─ apply tags and budget
             ├─ create spoke VNet, peer to hub, configure DNS and routes (if needed)
             ├─ assign RBAC to the workload team's groups; create workload identity for pipelines
             ├─ register resource providers
             └─ hand over to the workload team
```

Microsoft publishes subscription vending modules for Bicep and Terraform as part of Azure Verified Modules. Vending is what makes "a subscription per workload" realistic: without it, each request is a ticket and a week.

The requester supplies data, not infrastructure decisions. The platform team encodes its standards in the module.

### Limits that drive design

| Limit | Value | Consequence |
|---|---|---|
| Management groups per tenant | 10,000 | Not a practical constraint |
| Management group depth | 6 below root | Keep shallow anyway |
| Role assignments per management group | 500 | Use groups |
| Resource groups per subscription | 980 | Rarely hit with per-workload subscriptions |
| Subscriptions per EA account / MCA billing account | 5,000 / 10,000 | Plan billing structure for very large estates |
| Quotas (compute cores, model TPM, and others) | Per subscription per region | A frequent reason to split AI workloads into their own subscriptions |

### Moving subscriptions

Moving a subscription between management groups changes the policies and role assignments it inherits. Before moving:

- Compare inherited policy at source and destination; resources may become non-compliant or deployments may start failing.
- Check inherited RBAC; people may lose or gain access.
- You need write permission on the subscription and on both management groups.

## How it works on Azure

### A worked example

A mid-sized company with a customer web platform, internal line-of-business apps, a data platform and a new AI programme:

```text
Contoso
├── Platform
│   ├── sub-connectivity
│   ├── sub-management
│   └── sub-identity
├── Landing zones
│   ├── Corp
│   │   ├── sub-erp-prod            sub-erp-nonprod
│   │   ├── sub-dataplatform-prod   sub-dataplatform-nonprod
│   │   ├── sub-hr-assistant-prod   sub-hr-assistant-nonprod     ← internal AI workload
│   │   └── sub-ai-platform-prod    sub-ai-platform-nonprod      ← shared AI gateway and capacity
│   └── Online
│       ├── sub-webshop-prod        sub-webshop-nonprod
│       └── sub-support-bot-prod    sub-support-bot-nonprod      ← customer-facing AI workload
├── Sandbox
│   └── sub-sandbox-ai-01 …
└── Decommissioned
```

Note that AI workloads are placed by **exposure and data sensitivity**, exactly like any other workload: the internal HR assistant under *Corp*, the public support bot under *Online*. There is no "AI" management group. File 04 expands on this.

### Policy placement in that example

| Scope | Example policies |
|---|---|
| Contoso (intermediate root) | Allowed regions; require tags; deploy diagnostic settings; Defender for Cloud plans; deny classic resources |
| Platform | Stricter change control; protect networking resources |
| Landing zones | Deny public IP on NICs where inappropriate; enforce TLS; require private DNS integration; AI service baseline (disable local auth) |
| Corp | Deny public network access on PaaS, including Foundry, Search and Storage; require private endpoints |
| Online | Require WAF on public endpoints; DDoS protection |
| Sandbox | Budget enforcement; deny peering to corporate network; restrict expensive SKUs |
| Decommissioned | Deny all deployments |

## Design decisions and trade-offs

| Decision | Options and guidance |
|---|---|
| Depth of the tree | Start with the reference. Add a level only when a group of subscriptions needs policy that its siblings must not have |
| Environments | Subscriptions, not management groups |
| Per-workload vs shared subscriptions | Per-workload for anything with its own team, data sensitivity or quota needs. Shared for small utilities |
| Business units | Under *Landing zones* only if policy or administration truly differs |
| Regions | Not a management group dimension. Use policy parameters |
| Regulated workloads | A dedicated child group (for example *Corp-PCI* or *Confidential*) with additional policy initiatives is a legitimate reason for another group |
| Sandbox openness | Loose on what can be created, strict on network isolation, budget and data |

**Brownfield reality:** most clients already have subscriptions scattered under the root. The practical route is to build the target hierarchy alongside, assign policies in audit mode, move subscriptions in waves, remediate, then switch to deny.

## Hands-on

Requires permission to manage management groups; otherwise do the design part only.

```bash
# View the current hierarchy
az account management-group list --query "[].{Name:name, Display:displayName}" -o table
az account management-group show --name <root-or-group-id> --expand --recurse \
  --query "{name:displayName, children:children[].{name:displayName, type:type}}"

# Build a small hierarchy (lab prefixes to avoid clashing with anything real)
az account management-group create --name lab-root   --display-name "Lab"
az account management-group create --name lab-platform --display-name "Platform"      --parent lab-root
az account management-group create --name lab-lz       --display-name "Landing zones" --parent lab-root
az account management-group create --name lab-corp     --display-name "Corp"          --parent lab-lz
az account management-group create --name lab-online   --display-name "Online"        --parent lab-lz
az account management-group create --name lab-sandbox  --display-name "Sandbox"       --parent lab-root

# Move a (non-production) subscription and see inherited policy change
az account management-group subscription add --name lab-sandbox --subscription <sub-id>

# Check tenant-level hierarchy settings
az rest --method get \
  --url "https://management.azure.com/providers/Microsoft.Management/managementGroups/<tenant-id>/settings/default?api-version=2023-04-01"

# Clean up (move the subscription back first)
for g in lab-corp lab-online lab-lz lab-platform lab-sandbox lab-root; do
  az account management-group delete --name $g
done
```

**Design exercise:** for the organisation you used in the CAF exercise, draw the management group tree, list every subscription with its owner and environment, and write the three most important policies at each level.

## Security, cost and reliability notes

- **Security:** access at management group scope is far-reaching. Keep standing assignments to Reader and make anything stronger eligible through PIM.
- **Security:** set the default management group so that new subscriptions cannot appear at the root without policy.
- **Cost:** per-workload subscriptions make cost ownership obvious without relying on tags. Apply a budget at vending time.
- **Reliability:** separate production and non-production subscriptions so that a quota exhausted by a load test never blocks a production scale-out.

## Common mistakes

- Assigning policy directly at the tenant root group.
- Prod, Test and Dev management groups.
- A tree that mirrors departments and breaks at the next reorganisation.
- One enormous subscription for everything.
- Hundreds of subscriptions created by hand with inconsistent settings.
- New subscriptions landing at the root, ungoverned.
- An "AI" management group that ignores whether workloads are internal or internet-facing.
- Moving subscriptions without checking the change in inherited policy and access.

## Check yourself

1. Why should you not assign policy at the tenant root group?
2. Why are environments not modelled as management groups?
3. What distinguishes Corp from Online?
4. Give four legitimate reasons to create a new subscription.
5. What does subscription vending automate?
6. Where would you place an internal AI assistant's subscriptions, and where a public chatbot's?
7. What should you check before moving a subscription to another management group?

<details>
<summary>Answers</summary>

1. It affects everything in the tenant with no room to test changes. An intermediate root lets you trial changes in a parallel hierarchy and keeps the true root clean.
2. It multiplies the hierarchy and causes policy drift so production differs from what was tested. Environments are separated by subscriptions under the same group.
3. Corp workloads are internal and connected to the corporate network with no public exposure. Online workloads are internet-facing.
4. Any four of: environment separation, different workload owner, quota or limit isolation, blast radius, billing separation, compliance boundary, different connectivity.
5. Creating the subscription, placing it in a management group, tagging and budgeting it, setting up networking and RBAC, and handing it to the team.
6. Internal assistant under Corp; public chatbot under Online.
7. The differences in inherited policy and role assignments between source and destination, and your permissions on both.

</details>

## References

- [Management groups (landing zone design area)](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/design-area/resource-org-management-groups)
- [Subscription considerations and recommendations](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/design-area/resource-org-subscriptions)
- [Subscription vending](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/design-area/subscription-vending)
- [Subscription vending implementation guidance](https://learn.microsoft.com/azure/architecture/landing-zones/subscription-vending)
- [What are Azure management groups?](https://learn.microsoft.com/azure/governance/management-groups/overview)
- [Manage your management groups: hierarchy settings](https://learn.microsoft.com/azure/governance/management-groups/how-to/protect-resource-hierarchy)
- [Tailor the Azure landing zone architecture](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/tailoring-alz)
