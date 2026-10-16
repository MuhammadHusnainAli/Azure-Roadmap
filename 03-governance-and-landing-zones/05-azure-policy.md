# Azure Policy

> Topic: 03 · Governance and Landing Zones · Tracks: E ● A ● C ● · Status checked: 2026-10

## Why it matters

Azure Policy is how written governance becomes enforced reality. It evaluates every resource against rules you define and can audit, block, alter or fix what does not comply. Engineers meet it as the reason a deployment was denied. Architects use it to guarantee properties of an environment instead of hoping for them. For AI it is the mechanism behind "no keys, no public endpoints, approved models only".

## Key concepts

### Policy vs RBAC

| | Azure RBAC | Azure Policy |
|---|---|---|
| Question | **Who** can do what? | **What** is allowed to exist, and in what state? |
| Focus | User actions | Resource properties |
| Default | Deny unless granted | Allow unless restricted |
| Example | Alice can create storage accounts in this resource group | Storage accounts must not allow public blob access |

They work together. An Owner is still subject to policy.

### Building blocks

| Object | What it is |
|---|---|
| **Policy definition** | One rule: a condition and an effect |
| **Initiative** (policy set definition) | A group of definitions managed as one |
| **Assignment** | A definition or initiative applied to a scope, with parameter values |
| **Exemption** | A documented exception for a scope from an assignment |
| **Remediation task** | A job that brings existing resources into compliance |
| **Compliance state** | The evaluation result per resource |

Definitions and initiatives are **built-in** (maintained by Microsoft; there are several thousand) or **custom** (yours). Custom definitions are stored at a management group or subscription and can be assigned at that scope or below.

### Definition structure

```json
{
  "properties": {
    "displayName": "Storage accounts should disable public blob access",
    "mode": "Indexed",
    "parameters": {
      "effect": {
        "type": "String",
        "allowedValues": ["Audit", "Deny", "Disabled"],
        "defaultValue": "Audit"
      }
    },
    "policyRule": {
      "if": {
        "allOf": [
          { "field": "type", "equals": "Microsoft.Storage/storageAccounts" },
          { "field": "Microsoft.Storage/storageAccounts/allowBlobPublicAccess", "notEquals": false }
        ]
      },
      "then": { "effect": "[parameters('effect')]" }
    }
  }
}
```

| Element | Purpose |
|---|---|
| `mode` | `All` evaluates every resource type including resource groups and subscriptions; `Indexed` only types that support tags and location. Resource provider modes (for example `Microsoft.Kubernetes.Data`, `Microsoft.KeyVault.Data`) evaluate data-plane objects |
| `parameters` | Make the definition reusable. Parameterising the effect is standard practice |
| `if` | Conditions using `field`, `value` or `count`, combined with `allOf`, `anyOf`, `not` |
| `then` | The effect, plus `details` for effects that need them |

**Aliases** map policy fields to resource properties, such as `Microsoft.Storage/storageAccounts/allowBlobPublicAccess`. Array aliases use `[*]`. If a property has no alias, policy cannot evaluate it directly.

### Effects

| Effect | What it does | Acts on |
|---|---|---|
| **audit** | Marks non-compliant; logs a warning | New and existing |
| **auditIfNotExists** | Non-compliant if a related resource is missing or wrong | New and existing |
| **deny** | Blocks the create or update request | New and updated |
| **denyAction** | Blocks a specific action, currently delete | Requests |
| **append** | Adds fields to the request | New and updated |
| **modify** | Adds, replaces or removes properties or tags; can remediate existing resources | New, updated, and existing through remediation |
| **deployIfNotExists** (DINE) | Deploys a related resource if missing; can remediate existing | After creation, and existing through remediation |
| **mutate** | Alters data-plane requests for supported resource provider modes | Requests |
| **addToNetworkGroup** | Adds VNets to an Azure Virtual Network Manager group | Network resources |
| **manual** | Compliance is attested by a person | – |
| **disabled** | Turns the rule off | – |

**Order of evaluation** for a request: `disabled`, then `append` and `modify`, then `deny`, then `audit`, then `manual`, then `auditIfNotExists`, and `denyAction` last. After the resource provider returns success, `auditIfNotExists` and `deployIfNotExists` run.

Because `modify` runs before `deny`, a modify policy can fix a request so that it then passes a deny policy.

**Layering.** Each assignment is evaluated independently. The combined result is **cumulative and most restrictive**: if any applicable policy denies, the request is denied. A lower scope cannot loosen a policy assigned above; only an exclusion or exemption can.

### Choosing an effect

| Goal | Effect |
|---|---|
| See what would be affected | audit |
| Prevent a bad configuration | deny |
| Add a default or required value without failing deployments | modify or append |
| Ensure a companion resource or setting exists | deployIfNotExists |
| Stop accidental deletion of critical resources | denyAction |
| Track a control that cannot be evaluated automatically | manual |

`deny` gives fast feedback but frustrates teams if overused. `modify` and `deployIfNotExists` "fix it for them" and reduce friction, but they change resources outside the team's IaC, which causes drift unless the template already matches.

### Assignments

| Setting | Meaning |
|---|---|
| **Scope** | Management group, subscription or resource group |
| **Exclusions** (`notScopes`) | Child scopes left out entirely |
| **Parameters** | Values for this assignment |
| **Enforcement mode** | `Default` enforces; `DoNotEnforce` evaluates and reports compliance without applying deny or modify |
| **Resource selectors** | Limit by location or resource type for gradual rollout |
| **Overrides** | Change the effect for selected definitions in an initiative without editing it |
| **Non-compliance message** | The text a user sees on denial. Write one that says what to do |
| **Managed identity** | Required for `modify` and `deployIfNotExists` |

`DoNotEnforce` is the policy equivalent of Conditional Access report-only: it shows what deny would have blocked.

### Exemptions

A time-boxed, documented exception.

| Category | Meaning |
|---|---|
| **Waiver** | The non-compliance is accepted |
| **Mitigated** | The intent is met another way |

Exemptions have an optional expiry date, a description, and can target specific definitions within an initiative. Prefer them to exclusions: an exclusion silently removes a scope from evaluation forever, while an exemption is visible, justified and expires.

### Remediation

`modify` and `deployIfNotExists` apply automatically to new and updated resources. **Existing** non-compliant resources need a **remediation task**.

- The assignment's managed identity performs the change, so it needs the roles listed in the definition's `roleDefinitionIds` at the assignment scope. The portal grants these automatically; in IaC you assign them yourself.
- Tasks can be scoped, limited by resource count, and tuned for parallelism and failure threshold.

### Evaluation timing

| Trigger | When |
|---|---|
| Resource created or updated | Within about 15 minutes |
| New or changed assignment | Starts within about 5 minutes for the scope |
| Standard compliance cycle | Every 24 hours |
| On demand | `az policy state trigger-scan` |

Compliance data is not instantaneous. Wait, or trigger a scan, before concluding a policy is not working.

### Policy at other layers

| Capability | Purpose |
|---|---|
| **Machine configuration** | Audit or configure settings inside VMs and Arc-enabled servers |
| **Azure Policy for Kubernetes** | Gatekeeper-based admission control for AKS and Arc-enabled clusters |
| **Key Vault data-plane policy** | Govern keys, secrets and certificates (for example expiry and key size) |
| **Regulatory compliance initiatives** | Built-in initiatives mapped to standards (file 08) |
| **Defender for Cloud** | Uses policy initiatives, including the Microsoft cloud security benchmark, to compute Secure Score |

### Limits

| Item | Limit |
|---|---|
| Policy definitions per scope | 500 |
| Initiative definitions per scope | 200 (tenant-level maximum higher) |
| Policy or initiative assignments per scope | 200 |
| Definitions in one initiative | 1,000 |
| Exemptions per scope | 1,000 |

Group policies into initiatives to stay within assignment limits and to keep the picture readable.

## How it works on Azure

### AI governance policies

Built-in definitions cover most needs. Search by service name in the definitions list. Representative examples:

| Requirement | Built-in policy (name may vary) | Effect |
|---|---|---|
| No API keys | *Azure AI Services resources should have key access disabled (disable local authentication)* | Audit / Deny |
| | *Configure Azure AI Services resources to disable local key access* | Modify |
| No public access | *Azure AI Services resources should restrict network access* | Audit / Deny |
| Private endpoints | *Azure AI Services resources should use Azure Private Link* | Audit |
| Approved models only | *Cognitive Services Deployments should only use approved Registry Models* | Audit / Deny |
| Search without keys | *Azure AI Search services should have local authentication methods disabled* | Audit / Deny |
| Search not public | *Azure AI Search services should disable public network access* | Audit / Deny |
| Diagnostics | *Deploy diagnostic settings for Azure AI services to a Log Analytics workspace* | DeployIfNotExists |
| Encryption | *Azure AI Services resources should encrypt data at rest with a customer-managed key* | Audit / Deny |

**Model allow-list.** The approved-models policy takes a list of permitted model asset IDs or publishers. Assigned with `Deny` at the *Landing zones* management group, it stops any deployment of a model not on the list, whichever tool is used. A governance board maintains the list; changing it is a parameter update.

A custom policy can restrict **deployment types**, for example denying Global deployments where in-region processing is required:

```json
{
  "mode": "All",
  "parameters": {
    "allowedSkus": {
      "type": "Array",
      "defaultValue": ["Standard", "DataZoneStandard", "ProvisionedManaged", "DataZoneProvisionedManaged"]
    }
  },
  "policyRule": {
    "if": {
      "allOf": [
        { "field": "type", "equals": "Microsoft.CognitiveServices/accounts/deployments" },
        { "field": "Microsoft.CognitiveServices/accounts/deployments/sku.name", "notIn": "[parameters('allowedSkus')]" }
      ]
    },
    "then": { "effect": "deny" }
  }
}
```

Verify SKU names and aliases against the current resource schema before relying on a rule like this.

### Policy as code

Manage definitions, initiatives, assignments and exemptions in a repository, deployed by pipeline.

| Approach | Notes |
|---|---|
| **Bicep or Terraform** | Policy objects are ordinary resources. Fits naturally with the landing zone accelerators |
| **Enterprise Azure Policy as Code (EPAC)** | Microsoft-supported open-source solution that manages the whole policy estate from JSON/CSV with plan-and-deploy semantics and desired-state handling |
| **Landing zone library** | The accelerators ship their policy set as a versioned library you extend |

A sound workflow:

1. Author or select the definition; test in a sandbox scope.
2. Assign with `DoNotEnforce` or `audit` to the target scope.
3. Review compliance and what would have been denied.
4. Communicate; remediate existing resources.
5. Switch to enforce (`deny`).
6. Handle exceptions with expiring exemptions.

### Designing a non-compliance message

Bad: *"Resource was disallowed by policy."* Good: *"Foundry resources must have key authentication disabled. Set `disableLocalAuth: true` and use managed identity. See https://wiki.contoso.com/ai-guardrails. Contact #cloud-platform for exemptions."* The message is the main interface between the platform team and everyone else.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Built-in vs custom | Built-in first: Microsoft maintains them as resource schemas evolve. Custom only for gaps |
| Deny vs audit | Deny for a small set of non-negotiables; audit for everything else, reviewed regularly |
| Modify/DINE vs deny | Use modify and DINE for hygiene (tags, diagnostics, DNS records). Use deny for security boundaries. Make sure IaC matches what DINE configures |
| Scope of assignment | As high as the rule holds true |
| Initiative design | Group by theme and owner (for example *AI guardrails*, *Network baseline*, *Tagging*) rather than one giant set |
| Exemptions | Allowed, visible, time-boxed, reviewed |
| Number of policies | Each deny is a constraint on every team. Fewer, well-explained policies beat hundreds nobody understands |

**Policy and IaC together.** Policy is the safety net; IaC modules that produce compliant resources by default are the paved road. If teams hit deny constantly, fix the modules.

## Hands-on

```bash
SUB=$(az account show --query id -o tsv)
RG=rg-policy-lab; LOC=swedencentral
az group create -n $RG -l $LOC
SCOPE=/subscriptions/$SUB/resourceGroups/$RG

# 1. Explore built-ins
az policy definition list --query "[?policyType=='BuiltIn'] | length(@)"
az policy definition list \
  --query "[?policyType=='BuiltIn' && contains(displayName,'public blob')].{Name:displayName, Id:name}" -o table

# 2. Custom definition: require an 'owner' tag on resource groups' resources
cat > require-owner-tag.json <<'EOF'
{
  "if": { "field": "tags['owner']", "exists": "false" },
  "then": { "effect": "[parameters('effect')]" }
}
EOF
cat > params.json <<'EOF'
{ "effect": { "type": "String", "allowedValues": ["Audit","Deny","Disabled"], "defaultValue": "Audit" } }
EOF
az policy definition create --name require-owner-tag --display-name "Require owner tag" \
  --mode Indexed --rules require-owner-tag.json --params params.json

# 3. Assign in audit, create a resource, check compliance
az policy assignment create --name owner-tag-audit --policy require-owner-tag --scope $SCOPE \
  --params '{"effect":{"value":"Audit"}}'
az storage account create -n stpollab$RANDOM -g $RG -l $LOC --sku Standard_LRS
az policy state trigger-scan -g $RG
az policy state list -g $RG --query "[].{Resource:resourceId, State:complianceState, Policy:policyDefinitionName}" -o table

# 4. Switch to deny with a helpful message and try again
az policy assignment delete --name owner-tag-audit --scope $SCOPE
az policy assignment create --name owner-tag-deny --policy require-owner-tag --scope $SCOPE \
  --params '{"effect":{"value":"Deny"}}'
az policy assignment non-compliance-message create --name owner-tag-deny --scope $SCOPE \
  --message "Every resource needs an 'owner' tag. Add --tags owner=<email>."
az storage account create -n stpollab$RANDOM -g $RG -l $LOC --sku Standard_LRS              # denied
az storage account create -n stpollab$RANDOM -g $RG -l $LOC --sku Standard_LRS --tags owner=me@example.com

# 5. Modify + remediation: inherit a tag from the resource group
az group update -n $RG --set tags.costCentre=learning
INHERIT=$(az policy definition list --query "[?displayName=='Inherit a tag from the resource group if missing'].name | [0]" -o tsv)
az policy assignment create --name inherit-cc --policy $INHERIT --scope $SCOPE \
  --params '{"tagName":{"value":"costCentre"}}' \
  --mi-system-assigned --location $LOC --role "Tag Contributor" --identity-scope $SCOPE
az policy remediation create --name fix-cc -g $RG --policy-assignment inherit-cc
az resource list -g $RG --query "[].{Name:name, Tags:tags}" -o json

# 6. Exemption with expiry
az policy exemption create --name temp-waiver --policy-assignment \
  $(az policy assignment show --name owner-tag-deny --scope $SCOPE --query id -o tsv) \
  --exemption-category Waiver --scope $SCOPE --expires-on "$(date -u -d '+7 days' +%Y-%m-%dT%H:%M:%SZ)" \
  --description "Lab exemption"

# 7. Clean up
az group delete -n $RG --yes --no-wait
az policy definition delete --name require-owner-tag
```

Then, in a sandbox subscription with a Foundry resource, assign the built-in "key access disabled" policy with `Deny` and try to create a resource with keys enabled.

## Security, cost and reliability notes

- **Security:** the managed identity of a DINE or modify assignment at a management group can hold broad rights. Review the roles each definition requires.
- **Security:** restrict who can create exemptions and assignments (*Resource Policy Contributor*), and alert on new exemptions.
- **Cost:** Azure Policy is free for Azure resources. Machine configuration on Arc-enabled servers is charged. DINE policies deploy resources that cost money (for example diagnostic settings that drive log ingestion).
- **Reliability:** a wrong deny at a high scope can block every deployment in the organisation, including incident fixes. Roll out through audit, use resource selectors for staged rollout, and keep a fast path to disable an assignment.

## Common mistakes

- Enabling deny without first measuring impact in audit or `DoNotEnforce`.
- Using exclusions instead of expiring exemptions.
- Forgetting to grant the assignment's managed identity the roles needed for remediation.
- Expecting existing resources to be fixed without a remediation task.
- Assuming compliance results are immediate.
- Writing custom definitions for things built-ins already cover.
- DINE policies that fight the team's IaC, causing perpetual drift.
- Generic denial messages that send every developer to the platform team's inbox.
- Hundreds of unowned assignments nobody dares remove.

## Check yourself

1. How does Azure Policy differ from Azure RBAC?
2. What is the difference between `audit`, `deny`, `modify` and `deployIfNotExists`?
3. In what order are effects evaluated, and why does `modify` come before `deny`?
4. What does `DoNotEnforce` do?
5. Exclusion vs exemption: which should you prefer and why?
6. Why does a `deployIfNotExists` assignment need a managed identity?
7. How would you enforce an approved list of AI models across all workload subscriptions?
8. A policy was assigned an hour ago and existing resources still show no compliance state. What do you do?

<details>
<summary>Answers</summary>

1. RBAC controls who can perform actions. Policy controls what resource configurations are allowed, regardless of who deploys them.
2. Audit records non-compliance. Deny blocks the request. Modify changes properties or tags on the request and can remediate existing resources. DeployIfNotExists deploys a related resource or setting when missing.
3. Disabled, append and modify, deny, audit, manual, auditIfNotExists, denyAction. Modify runs first because it can alter the request so that it then satisfies deny or audit rules.
4. Evaluates and reports compliance without enforcing deny or modify effects.
5. Exemption: it is visible, documented, categorised and can expire, whereas an exclusion silently removes the scope.
6. It performs deployments on your behalf and needs an identity with the required roles at the assignment scope.
7. Assign the built-in approved-models policy with the Deny effect at the Landing zones management group, with the allow-list as a parameter managed as code.
8. Trigger an on-demand scan or wait for the evaluation cycle, then check compliance state again.

</details>

## References

- [What is Azure Policy?](https://learn.microsoft.com/azure/governance/policy/overview)
- [Azure Policy definition structure](https://learn.microsoft.com/azure/governance/policy/concepts/definition-structure-basics)
- [Azure Policy definitions effect basics](https://learn.microsoft.com/azure/governance/policy/concepts/effect-basics)
- [Azure Policy assignment structure](https://learn.microsoft.com/azure/governance/policy/concepts/assignment-structure)
- [Azure Policy exemption structure](https://learn.microsoft.com/azure/governance/policy/concepts/exemption-structure)
- [Remediate non-compliant resources](https://learn.microsoft.com/azure/governance/policy/how-to/remediate-resources)
- [Azure Policy built-in definitions](https://learn.microsoft.com/azure/governance/policy/samples/built-in-policies)
- [Control model deployments with Azure Policy in Foundry](https://learn.microsoft.com/azure/foundry/how-to/model-deployment-policy)
- [Enterprise Azure Policy as Code](https://azure.github.io/enterprise-azure-policy-as-code/)
- [Design Azure Policy as Code workflows](https://learn.microsoft.com/azure/governance/policy/concepts/policy-as-code)
