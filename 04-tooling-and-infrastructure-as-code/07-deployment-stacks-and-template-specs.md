# Deployment Stacks and Template Specs

## Why it matters

Bicep has no state, so an ordinary deployment cannot tell that you removed a resource from the file, and it cannot stop someone changing what you deployed. Deployment stacks close both gaps natively in Azure. Template specs solve a different problem: storing and sharing templates as governed Azure resources. Together they are also the migration target for Azure Blueprints, which retires in January 2027.

## Part 1 · Deployment stacks

### Key concepts

A **deployment stack** is an Azure resource that represents a deployment and the set of resources it manages. You create it from a Bicep file or ARM template exactly as you would a deployment, and Azure then tracks those resources as a unit.

| Capability | What it gives you |
|---|---|
| **Managed resource tracking** | The stack records every resource the template created |
| **Clean-up on removal** | When a resource leaves the template, the stack can detach or delete it |
| **Deny settings** | A deny assignment protects managed resources from modification or deletion outside the stack |
| **Whole-stack delete** | Remove the stack and, optionally, everything it manages in one operation |
| **Cross-scope management** | A stack at subscription or management group scope can manage resources across resource groups and subscriptions |

### Scopes

| Stack created at | Can deploy to | CLI group |
|---|---|---|
| Resource group | That resource group | `az stack group` |
| Subscription | The subscription and its resource groups | `az stack sub` |
| Management group | Subscriptions and resource groups beneath it | `az stack mg` |

A stack at a higher scope than its resources is useful for protection: developers working in a resource group cannot see or alter a stack defined at subscription scope.

### Action on unmanage

What happens to a resource that is no longer in the template, or when the stack is deleted:

| Setting (`--action-on-unmanage`) | Resources | Resource groups | Management groups |
|---|---|---|---|
| `detachAll` | Detached (left in place, no longer managed) | Detached | Detached |
| `deleteResources` | **Deleted** | Detached | Detached |
| `deleteAll` | **Deleted** | **Deleted** | **Deleted** |

`deleteResources` gives Bicep the "remove from code means remove from Azure" behaviour that Terraform has by default. `detachAll` is the safe starting point while you learn.

### Deny settings

| Mode (`--deny-settings-mode`) | Effect on managed resources |
|---|---|
| `none` | No protection |
| `denyDelete` | Cannot be deleted; can be modified |
| `denyWriteAndDelete` | Cannot be modified or deleted |

Refinements:

| Option | Purpose |
|---|---|
| `--deny-settings-excluded-principals` | Object IDs exempt from the deny, such as a break-glass group or an operations pipeline |
| `--deny-settings-excluded-actions` | Specific operations still permitted, for example `Microsoft.Compute/virtualMachines/restart/action` |
| `--deny-settings-apply-to-child-scopes` | Extend the deny to child resources created later |

The protection is an Azure **deny assignment**, which overrides role assignments, including Owner. It applies to the **control plane**. A `denyWriteAndDelete` on a storage account does not stop data being written to it.

To manage stacks and their deny settings, the built-in roles are *Azure Deployment Stack Contributor* (manage stacks, not deny settings) and *Azure Deployment Stack Owner* (including deny settings).

### Commands

```bash
# Create or update (idempotent)
az stack group create \
  --name stack-hrassist-prod \
  --resource-group rg-hrassist-prod \
  --template-file main.bicep \
  --parameters main.prod.bicepparam \
  --action-on-unmanage deleteResources \
  --deny-settings-mode denyWriteAndDelete \
  --deny-settings-excluded-principals <ops-group-object-id> \
  --yes

# Inspect
az stack group show  -n stack-hrassist-prod -g rg-hrassist-prod --query "{state:provisioningState, deny:denySettings.mode}"
az stack group show  -n stack-hrassist-prod -g rg-hrassist-prod --query "resources[].{id:id, status:status, deny:denyStatus}" -o table
az stack group list  -g rg-hrassist-prod -o table

# Validate before creating
az stack group validate -n stack-hrassist-prod -g rg-hrassist-prod -f main.bicep -p main.prod.bicepparam \
  --action-on-unmanage deleteResources --deny-settings-mode none

# Export the template the stack was built from
az stack group export -n stack-hrassist-prod -g rg-hrassist-prod

# Delete the stack and choose what happens to its resources
az stack group delete -n stack-hrassist-prod -g rg-hrassist-prod --action-on-unmanage detachAll --yes
```

Stacks support a what-if style preview in recent tooling versions. Where it is unavailable in your version, run `az deployment group what-if` with the same template to preview resource changes, remembering it will not show what the stack would unmanage or delete.

### Limitations to know

- **Implicitly created resources are not managed.** If the platform creates something as a side effect (for example the node resource group of an AKS cluster), the stack does not track it.
- **Deny assignments do not cover everything.** Some resource types and operations do not honour them; check the documentation for your resources.
- **Key Vault secrets and similar child data** have special handling; deletion behaviour can differ.
- **Soft-deleted resources** (Key Vault, Cognitive Services accounts, API Management) remain soft-deleted after stack deletion and can block re-creation with the same name until purged.
- **Limits:** up to 800 stacks per scope and 2,000 deny assignments per scope.
- **A resource should belong to one stack.** If two stacks claim the same resource, the last deployment wins and the other's view becomes inaccurate.
- **Terraform users do not need stacks.** State already provides lifecycle management. Stacks are for Bicep and ARM.

### When to use a stack

| Situation | Use a stack? |
|---|---|
| Workload environments where code should be the single source of truth | Yes, `deleteResources` |
| Platform-provided resources that workload teams must not change (network, policy, shared services) | Yes, at a higher scope with `denyWriteAndDelete` |
| Ephemeral environments (per pull request, sandboxes) | Yes, for one-command teardown with `deleteAll` |
| Replacing Azure Blueprints | Yes, with template specs |
| A quick one-off deployment | An ordinary deployment is fine |
| Resources that another process legitimately modifies | Use `denyDelete` or exclusions, or leave unprotected |

## Part 2 · Template specs

### Key concepts

A **template spec** is an Azure resource (`Microsoft.Resources/templateSpecs`) that stores an ARM template, with versions, in a resource group.

| Property | Benefit |
|---|---|
| Stored in Azure | No need to host template files at a URL reachable by ARM |
| **Azure RBAC** | Control who can read (deploy) and who can write (publish) |
| **Versioned** | Consumers pin a version; publishers release new ones safely |
| Deployable from the portal, CLI, PowerShell, or as a Bicep module | Works for people and pipelines |
| Bicep supported as input | Bicep is compiled to JSON on publish; modules are bundled in |

```bash
# Publish a version
az ts create --name ts-storage-standard --version 1.2.0 \
  --resource-group rg-templates --location swedencentral \
  --template-file modules/storage.bicep \
  --display-name "Standard storage account" --description "Company baseline storage"

# List and show
az ts list -g rg-templates -o table
TS_ID=$(az ts show -n ts-storage-standard -g rg-templates --version 1.2.0 --query id -o tsv)

# Deploy it
az deployment group create -g rg-demo --template-spec $TS_ID --parameters name=stdemo001

# Or wrap it in a stack
az stack group create -n stack-storage -g rg-demo --template-spec $TS_ID \
  --action-on-unmanage deleteResources --deny-settings-mode denyDelete
```

Used as a Bicep module:

```bicep
module storage 'ts:<subscription-id>/rg-templates/ts-storage-standard:1.2.0' = {
  params: { name: 'stdemo001' }
}
```

Add an alias in `bicepconfig.json` to shorten the reference.

A template spec can include a **UI definition** so the portal presents a custom form, which makes it suitable for self-service by people who will never open a Bicep file.

### Template specs or a Bicep registry?

| | Template specs | Bicep registry (Azure Container Registry) |
|---|---|---|
| Stores | Complete, deployable templates (compiled JSON) | Bicep modules (OCI artifacts) |
| Access control | Azure RBAC on the resource | Registry roles (`AcrPull`, `AcrPush`) |
| Portal deployment | Yes, with optional custom form | No |
| Developer experience | Good | Better: IntelliSense, restore, familiar package workflow |
| Network isolation | Reached through ARM | Needs network path to the registry (private endpoint supported) |
| Best for | Approved, ready-to-deploy solutions for consumers; Blueprints replacement | Building-block modules shared between engineering teams |

Many organisations use both: a registry for modules that engineers compose, and template specs for finished offerings.

## How it works on Azure

### Replacing Azure Blueprints

Azure Blueprints never left preview and **retires on 31 January 2027**, with a phased wind-down that began in July 2026. Microsoft's recommended replacement is deployment stacks with template specs.

| Blueprints concept | Replacement |
|---|---|
| Blueprint definition | Bicep file or template spec |
| Versioned, published blueprint | Template spec version (or registry module version) |
| Artifacts: ARM templates, resource groups | Resources and modules in the Bicep file |
| Artifacts: policy assignments | `Microsoft.Authorization/policyAssignments` resources in Bicep |
| Artifacts: role assignments | `Microsoft.Authorization/roleAssignments` resources in Bicep |
| Blueprint assignment | Deployment stack at subscription or management group scope |
| Resource locking (*Read Only*, *Do Not Delete*) | Stack deny settings (`denyWriteAndDelete`, `denyDelete`) |
| Assignment update to a new version | Update the stack with the new template version |

Migration steps in outline: export each blueprint's artifacts, convert them to Bicep, publish as a template spec or module, create a stack in a non-production subscription with `detachAll`, validate, then cut over and remove the blueprint assignment. Test deny settings carefully, because they differ in detail from blueprint locks.

### Platform protection pattern

A platform team vends a subscription and deploys its baseline as a stack at **subscription scope** from a pipeline:

```bash
az stack sub create \
  --name stack-lz-baseline \
  --location swedencentral \
  --template-spec $BASELINE_TS_ID \
  --parameters workload=hrassist env=prod \
  --action-on-unmanage detachAll \
  --deny-settings-mode denyWriteAndDelete \
  --deny-settings-apply-to-child-scopes \
  --deny-settings-excluded-principals $PLATFORM_PIPELINE_OBJECT_ID \
  --deployment-resource-group rg-lz-baseline
```

The baseline (spoke network, route table, diagnostic settings, budget) is protected. The workload team is Owner of their subscription and builds freely, but cannot alter the guardrail resources. This is the Blueprints locking model, implemented natively.

### In a pipeline

Replace `az deployment group create` with `az stack group create`. The command is idempotent, so the same step creates and updates. For pull-request environments:

```bash
az stack group create -n "stack-pr-$PR" -g "rg-pr-$PR" -f main.bicep \
  --action-on-unmanage deleteAll --deny-settings-mode none --yes
# on PR close:
az stack group delete -n "stack-pr-$PR" -g "rg-pr-$PR" --action-on-unmanage deleteAll --yes
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| `detachAll` vs `deleteResources` | Start with `detachAll`; move to `deleteResources` once the team trusts the pipeline. `deleteAll` for disposable environments |
| Deny mode | `denyDelete` is a low-friction safety net. `denyWriteAndDelete` for platform resources only; it blocks legitimate operational changes unless you exclude them |
| Exclusions | Exclude a group or the deploying identity, never individuals. Keep the list short |
| Stack scope | Same scope as resources for workload teams managing their own. Higher scope for platform-enforced baselines |
| Granularity | One stack per lifecycle unit, mirroring how you would split deployments or Terraform states |
| Stacks vs complete mode | Stacks. Complete mode deletes anything in the resource group not in the template, including resources the template never owned |
| Stacks vs locks | Stacks protect and track a managed set with exclusions; locks are a blunt per-resource or per-group switch. They can coexist |

**The stateful step.** A stack makes Bicep behave more like a stateful tool: the deployment now has memory and consequences. That is the point, and it also means a mistaken template change can delete resources. Keep what-if or validation and an approval in the path to production.

## Hands-on

```bash
RG=rg-stack-lab; az group create -n $RG -l swedencentral -o none

cat > main.bicep <<'EOF'
param location string = resourceGroup().location
param deploySecond bool = true
var suffix = take(uniqueString(resourceGroup().id), 6)

resource st1 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: 'ststack1${suffix}'
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}
resource st2 'Microsoft.Storage/storageAccounts@2023-05-01' = if (deploySecond) {
  name: 'ststack2${suffix}'
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}
EOF

# 1. Create the stack with protection
az stack group create -n stack-lab -g $RG -f main.bicep \
  --action-on-unmanage deleteResources --deny-settings-mode denyDelete --yes
az stack group show -n stack-lab -g $RG --query "resources[].{id:id, status:status, deny:denyStatus}" -o table

# 2. Try to delete a managed resource: blocked by the deny assignment, even as Owner
ST1=$(az storage account list -g $RG --query "[0].name" -o tsv)
az storage account delete -n $ST1 -g $RG --yes
az role assignment list --scope $(az group show -n $RG --query id -o tsv) --include-inherited -o none
# Portal: resource group → Access control (IAM) → Deny assignments

# 3. Remove a resource from the template and update: it is deleted
az stack group create -n stack-lab -g $RG -f main.bicep -p deploySecond=false \
  --action-on-unmanage deleteResources --deny-settings-mode denyDelete --yes
az storage account list -g $RG --query "[].name" -o tsv

# 4. Publish the template as a template spec and deploy from it
az group create -n rg-templates-lab -l swedencentral -o none
az ts create -n ts-lab -g rg-templates-lab -l swedencentral --version 1.0.0 --template-file main.bicep
az ts show -n ts-lab -g rg-templates-lab --version 1.0.0 --query id -o tsv

# 5. Delete the stack and its resources, then the groups
az stack group delete -n stack-lab -g $RG --action-on-unmanage deleteAll --yes
az group delete -n $RG --yes --no-wait
az group delete -n rg-templates-lab --yes --no-wait
```

Compare step 3 with the same change in the Bicep lab (where the resource survived) and the Terraform lab (where it was destroyed).

## Security, cost and reliability notes

- **Security:** deny settings stop even Owners, which is their value and their risk. Make sure a break-glass principal is excluded or that the stack itself can be updated quickly by a trusted pipeline.
- **Security:** restrict who holds *Azure Deployment Stack Owner*; that role can lift the protection.
- **Security:** deny assignments are control-plane only. Protect data with RBAC, network controls and immutability features.
- **Cost:** stacks and template specs are free.
- **Reliability:** `deleteResources` and `deleteAll` are destructive by design. Protect stateful resources with `prevent`-style safeguards: separate stacks for data, resource locks, soft delete and backups.
- **Reliability:** if a stack update fails part-way, resources may be in a mixed state. Re-running the same create command is the recovery path.

## Common mistakes

- Using complete mode when a stack is what was wanted.
- Jumping straight to `deleteAll` in production.
- `denyWriteAndDelete` on resources that operations staff or autoscalers legitimately change.
- Forgetting to exclude the pipeline or a break-glass principal, then being unable to make an urgent fix.
- Assuming deny settings protect data.
- Two stacks managing the same resource.
- Expecting implicitly created resources to be cleaned up.
- Starting new work on Azure Blueprints.
- Leaving soft-deleted resources behind and failing to redeploy with the same name.

## Check yourself

1. What two gaps in ordinary Bicep deployments do deployment stacks close?
2. Explain the three `action-on-unmanage` values.
3. What does `denyWriteAndDelete` protect, and what does it not?
4. Why might you create a stack at subscription scope for resources in one resource group?
5. What is a template spec?
6. When would you choose a template spec over a Bicep registry module?
7. What replaces Azure Blueprints, and how do its concepts map?

<details>
<summary>Answers</summary>

1. Tracking and removing resources that are no longer in the template, and protecting deployed resources from out-of-band change or deletion.
2. `detachAll` leaves removed resources in place unmanaged; `deleteResources` deletes resources but keeps resource groups; `deleteAll` deletes resources, resource groups and management groups.
3. It blocks control-plane modification and deletion of managed resources, even by Owners. It does not block data-plane operations.
4. So people working in the resource group cannot see or modify the stack, which suits platform-enforced baselines.
5. An Azure resource that stores a versioned ARM template (or compiled Bicep) with RBAC, deployable from the portal, CLI or as a module.
6. When you want Azure RBAC-controlled, portal-deployable, ready-made solutions for consumers, or as part of a Blueprints replacement.
7. Deployment stacks with template specs: the definition becomes Bicep or a template spec, artifacts become resources in the file, assignment becomes a stack, and locking becomes deny settings.

</details>

## References

- [Deployment stacks](https://learn.microsoft.com/azure/azure-resource-manager/bicep/deployment-stacks)
- [Quickstart: create and deploy a deployment stack with Bicep](https://learn.microsoft.com/azure/azure-resource-manager/bicep/quickstart-create-deployment-stacks)
- [Azure template specs in Bicep](https://learn.microsoft.com/azure/azure-resource-manager/bicep/template-specs)
- [Create a private registry for Bicep modules](https://learn.microsoft.com/azure/azure-resource-manager/bicep/private-module-registry)
- [Migrate blueprints to deployment stacks](https://learn.microsoft.com/azure/azure-resource-manager/bicep/migrate-blueprint)
- [Azure Blueprints retirement](https://learn.microsoft.com/azure/governance/blueprints/overview)
- [Understand Azure deny assignments](https://learn.microsoft.com/azure/role-based-access-control/deny-assignments)
