# Azure Resource Hierarchy

> Topic: 01 · Cloud and Azure Fundamentals · Tracks: E ● A ● C ● · Status checked: 2026-10

## Why it matters

Access, policy and cost in Azure all flow down a hierarchy. Get the hierarchy right and governance is mostly inheritance. Get it wrong and you spend years assigning permissions resource by resource. This file covers the building blocks; topic 03 covers how to design them for an enterprise.

## Key concepts

### The five levels

```text
Microsoft Entra tenant
└── Root management group
    └── Management groups          (up to 6 levels below root)
        └── Subscriptions
            └── Resource groups
                └── Resources
```

| Level | What it is | Main purpose |
|---|---|---|
| **Tenant** | An instance of Microsoft Entra ID representing an organisation | Identity boundary |
| **Management group** | A container for subscriptions and other management groups | Apply policy and access to many subscriptions |
| **Subscription** | A logical container tied to a billing account | Billing, quota and scale boundary; access and policy boundary |
| **Resource group** | A container for resources that share a lifecycle | Deploy, manage and delete things together |
| **Resource** | A single manageable item: a VM, a storage account, a Foundry project | The thing that does the work |

### Tenant

A tenant is a dedicated Entra ID directory. It holds users, groups, applications and service principals. Every subscription trusts exactly one tenant for authentication. A tenant can have many subscriptions.

The tenant is an identity boundary, not a resource container. Resources live in subscriptions; identities live in the tenant.

### Management groups

- Every tenant has one **root management group** (the tenant root group). All other groups and subscriptions sit beneath it.
- Up to **10,000** management groups per tenant, and up to **six levels** of depth below the root (not counting the subscription level).
- Each management group and subscription has exactly **one parent**.
- Policy and RBAC assigned at a management group are **inherited** by everything beneath it.

By default no one has access to the root management group. A Global Administrator must elevate their access first, a deliberate safeguard because anything assigned there affects the entire tenant.

### Subscriptions

A subscription is three boundaries at once:

1. **Billing:** usage is metered and invoiced per subscription.
2. **Scale:** most quotas and limits are per subscription (often per subscription per region).
3. **Management:** a natural scope for RBAC and policy.

Because quotas are per subscription, subscriptions are also how you scale. A large AI workload may need its own subscription simply to have its own model quota.

Key limits:

| Item | Limit |
|---|---|
| Resource groups per subscription | 980 |
| Role assignments per subscription | 5,000 |
| Tags applied directly to a subscription | 50 |
| Subscriptions per management group | Unlimited |

### Resource groups

Rules that never change:

- Every resource belongs to **exactly one** resource group.
- Resource groups **cannot be nested**.
- A resource group has a **location**, which is where its metadata is stored. Resources inside can be in any region.
- **Deleting a resource group deletes everything in it.**
- Resources can usually be **moved** between resource groups and subscriptions, with per-service exceptions.
- Resources in different resource groups can interact freely. A resource group is not a network or security boundary by itself.

Key limits:

| Item | Limit |
|---|---|
| Resources per resource group | 800 per resource type (some types exempt) |
| Deployments in history per resource group | 800 (oldest are pruned automatically) |
| Tags per resource or resource group | 50 |
| Management locks per scope | 20 |

Because the resource group's location holds its metadata, a regional outage there can block control-plane changes to the group even when its resources are elsewhere. Put the resource group in the same region as its resources.

### Resources and resource IDs

Every resource has a unique ID that encodes the hierarchy:

```text
/subscriptions/{subscription-id}
  /resourceGroups/{resource-group-name}
    /providers/{resource-provider-namespace}/{resource-type}/{resource-name}
```

For example:

```text
/subscriptions/0000.../resourceGroups/rg-hr-assistant-prod-swc
  /providers/Microsoft.Storage/storageAccounts/sthrassistprodswc
```

Child resources extend the path, such as a blob container under a storage account or a project under a Foundry resource. You will read and write resource IDs constantly in IaC, RBAC assignments and diagnostics.

### Scope and inheritance

**Scope** is the level at which something applies. The four scopes are management group, subscription, resource group and resource. Three things are assigned at a scope and inherited downward:

| Thing | Inherited? | Notes |
|---|---|---|
| **RBAC role assignments** | Yes | Additive. A lower scope cannot remove access granted above (short of deny assignments) |
| **Azure Policy assignments** | Yes | Can be excluded at lower scopes or exempted |
| **Management locks** | Yes | A lock on a resource group applies to all resources in it |
| **Tags** | **No** | Tags on a resource group are not inherited by its resources unless a policy copies them |

The tags row is the one people get wrong.

## How it works on Azure

### How to split things

**What goes in one resource group?** Things that share a lifecycle: deployed together, updated together, deleted together, owned by the same team. A typical unit is one workload in one environment.

```text
rg-hr-assistant-dev-swc      ← everything for the dev environment
rg-hr-assistant-prod-swc     ← everything for production
rg-network-hub-swc           ← shared networking, different owner and lifecycle
```

**When do you need another subscription?** When you need a different billing owner, separate quotas, a hard blast-radius boundary, or a different policy set. Production and non-production are commonly separated at subscription level.

**When do you need a management group?** When several subscriptions should share policy or access. Groups are organised by the kind of workload and its governance needs (for example *corp-connected* vs *internet-facing*), not by the org chart, because org charts change.

**When do you need another tenant?** Rarely. A separate tenant means separate identities and a great deal of duplicated administration. Valid cases are legal entity separation, sandbox isolation for identity testing, and some acquisitions.

### A minimal sensible hierarchy

```text
Tenant root group
├── Platform
│   ├── Identity          (subscription)
│   ├── Management        (subscription)
│   └── Connectivity      (subscription)
├── Landing zones
│   ├── Corp              (subscriptions for internal workloads)
│   └── Online            (subscriptions for internet-facing workloads)
├── Sandbox               (subscriptions for experimentation)
└── Decommissioned
```

This is the shape of the Azure landing zone reference. Topic 03 explains why.

### Where AI resources sit

A Microsoft Foundry resource is an ordinary Azure resource in a resource group. Foundry **projects** are child resources beneath it. Connected resources (Azure AI Search, Storage, Cosmos DB, Key Vault) are separate resources, normally in the same resource group when they share the workload's lifecycle. Model quota is tracked per subscription and region, which is often what drives subscription design for AI platforms.

## Design decisions and trade-offs

| Decision | Fewer, larger containers | More, smaller containers |
|---|---|---|
| Subscriptions | Simpler billing and networking; shared quotas and bigger blast radius | Clear isolation and cost ownership; more to manage, needs automation |
| Resource groups | Fewer objects; mixed lifecycles make deletion risky | Clean lifecycle and RBAC; more cross-group references |
| Management groups | Easy to reason about | Finer policy control; deep trees are hard to debug |

Guidance:

- Keep the management group tree shallow: three or four levels is usually plenty.
- Assign RBAC to **groups** at the **highest sensible scope**, not to individuals at resource level.
- Don't use resource groups as an access-control substitute for subscriptions when you need hard isolation.
- Automate subscription creation ("subscription vending") once you have more than a handful.

## Hands-on

```bash
# Where am I?
az account show --query "{Tenant:tenantId, Subscription:name, Id:id}" -o table

# List subscriptions you can see
az account list --query "[].{Name:name, Id:id, State:state}" -o table

# Create a resource group with tags
az group create \
  --name rg-roadmap-lab-swc \
  --location swedencentral \
  --tags env=lab owner=me costCentre=learning

# Create a resource in it
az storage account create \
  --name stroadmaplab$RANDOM \
  --resource-group rg-roadmap-lab-swc \
  --location swedencentral \
  --sku Standard_LRS

# Inspect the resource ID and confirm tags were NOT inherited
az resource list -g rg-roadmap-lab-swc \
  --query "[].{Id:id, Tags:tags}" -o json

# Lock the group against deletion, try to delete, then remove the lock
az lock create --name no-delete --lock-type CanNotDelete -g rg-roadmap-lab-swc
az group delete -n rg-roadmap-lab-swc --yes        # fails with ScopeLocked
az lock delete --name no-delete -g rg-roadmap-lab-swc

# Clean up: deletes the group and everything inside
az group delete -n rg-roadmap-lab-swc --yes --no-wait
```

If you have permission, also list management groups:

```bash
az account management-group list --query "[].{Name:name, Display:displayName}" -o table
```

## Security, cost and reliability notes

- **Security:** a role at a high scope is powerful. Owner on a management group is Owner on every resource below it. Review high-scope assignments regularly.
- **Cost:** subscriptions and resource groups are the easiest cost-reporting dimensions. Aligning them to owners saves a lot of tagging effort.
- **Reliability:** a `CanNotDelete` lock on production resource groups prevents the most expensive kind of accident. Note that a `ReadOnly` lock can break operations that look like reads but are technically writes (listing storage keys, for example).

## Common mistakes

- Expecting tags to inherit from resource group to resources.
- One giant resource group per subscription, making safe deletion impossible.
- Mirroring the org chart in management groups and then reorganising.
- Putting a resource group in a different region from its resources.
- Granting access to individuals at resource scope until nobody can tell who has what.
- Treating a resource group as a security boundary between workloads that should be isolated.
- Creating new tenants to separate environments.

## Check yourself

1. List the levels of the hierarchy from top to bottom.
2. Which of RBAC, Policy, locks and tags are inherited by child scopes?
3. Can a resource belong to two resource groups? Can resource groups be nested?
4. What three boundaries does a subscription represent?
5. What determines whether resources should share a resource group?
6. Why does the location of a resource group matter?
7. What is the maximum depth of management groups below the root?
8. Why might an AI workload get its own subscription?

<details>
<summary>Answers</summary>

1. Tenant, management groups, subscription, resource group, resource.
2. RBAC, Policy and locks are inherited. Tags are not.
3. No, and no.
4. Billing, scale (quotas) and management (access and policy).
5. A shared lifecycle and owner.
6. It stores the group's metadata, so control-plane operations depend on that region being available, and it may matter for residency.
7. Six levels.
8. To get dedicated model and compute quota, a separate cost boundary and a contained blast radius.

</details>

## References

- [Organize your Azure resources effectively](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/azure-setup-guide/organize-resources)
- [What are Azure management groups?](https://learn.microsoft.com/azure/governance/management-groups/overview)
- [What is a resource group?](https://learn.microsoft.com/azure/azure-resource-manager/management/manage-resource-groups-portal)
- [Azure subscription and service limits, quotas, and constraints](https://learn.microsoft.com/azure/azure-resource-manager/management/azure-subscription-service-limits)
- [Move resources to a new resource group or subscription](https://learn.microsoft.com/azure/azure-resource-manager/management/move-resource-group-and-subscription)
- [Lock your resources to protect your infrastructure](https://learn.microsoft.com/azure/azure-resource-manager/management/lock-resources)
