# Terraform on Azure

> Topic: 04 · Tooling and Infrastructure as Code · Tracks: E ● A ○ C – · Status checked: 2026-10

## Why it matters

Terraform is the most widely used infrastructure-as-code tool across clouds, and a large share of Azure estates are managed with it. Many client teams you join will already have years of Terraform. Compared with Bicep it adds a state file, a plan-and-apply workflow, a huge provider ecosystem and the ability to manage Azure, Entra ID, GitHub, Kubernetes and other clouds from one codebase.

## Key concepts

### How Terraform works

```text
*.tf configuration ──► terraform plan ──► compares with ──► state file ──► and real resources
                                │
                                └──► execution plan (create / update / destroy)
                                           │
                                  terraform apply ──► provider ──► Azure APIs ──► state updated
```

| Concept | Meaning |
|---|---|
| **Configuration** | `.tf` files in HashiCorp Configuration Language (HCL) describing desired state |
| **Provider** | A plugin that knows how to manage a platform's resources |
| **State** | Terraform's record of what it manages and the last known attributes |
| **Plan** | The diff between configuration, state and reality |
| **Apply** | Executing the plan |
| **Module** | A reusable folder of configuration |

Because Terraform tracks state, **removing a resource from configuration destroys it** on the next apply. That is the opposite of Bicep's default and the main behavioural difference between the two.

### Providers for Azure

| Provider | Manages | Notes |
|---|---|---|
| **`azurerm`** | Azure resources through curated, validated resource types | The default. Friendly schema, good validation. New Azure features can lag by weeks or months |
| **`azapi`** | Any Azure resource and API version through the ARM REST API | Day-zero coverage, including previews. Use to fill gaps or as the primary provider |
| **`azuread`** | Entra ID: applications, service principals, groups | |
| **`msgraph`** | Microsoft Graph resources | Newer, API-shaped alternative for Graph |
| `azuredevops`, `github`, `kubernetes`, `helm`, `random`, `time` | Surrounding tooling | |

**AzureRM 5.0** (generally available July 2026) is a breaking major release. The change you will notice first: `resource_provider_registrations` now defaults to `none`, so the provider no longer registers a large set of Azure resource providers on start-up. You declare the registrations you need or handle them separately. It also removes long-deprecated resources such as `azurerm_app_service` and `azurerm_function_app`, and adds opt-in preflight validation. Estates on 3.x or 4.x need a planned upgrade; read the upgrade guide.

**AzAPI** resources take a `type` with an API version and a `body` written as an HCL object:

```hcl
resource "azapi_resource" "project" {
  type      = "Microsoft.CognitiveServices/accounts/projects@2025-06-01"
  name      = "proj-hrassist"
  parent_id = azapi_resource.foundry.id
  location  = var.location
  identity { type = "SystemAssigned" }
  body = {
    properties = { displayName = "HR assistant" }
  }
  response_export_values = ["properties.endpoints"]
}
```

AI services move quickly, so AzAPI is common in AI workloads for anything `azurerm` does not yet expose. Other AzAPI tools: `azapi_update_resource` (patch a property on a resource managed elsewhere), `azapi_resource_action` (call an action), and `azapi_data_plane_resource`.

### Configuration basics

```hcl
terraform {
  required_version = ">= 1.9"
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 5.0" }
    azapi   = { source = "Azure/azapi",       version = "~> 2.0" }
  }
  backend "azurerm" {}                      # values supplied at init
}

provider "azurerm" {
  features {}
  subscription_id                 = var.subscription_id      # required
  storage_use_azuread             = true                     # data-plane calls with Entra ID, not keys
  resource_provider_registrations = "none"
  resource_providers_to_register  = ["Microsoft.CognitiveServices", "Microsoft.Search"]
}

variable "workload"  { type = string }
variable "env" {
  type = string
  validation {
    condition     = contains(["dev", "test", "prod"], var.env)
    error_message = "env must be dev, test or prod."
  }
}
variable "location" { type = string, default = "swedencentral" }

locals {
  suffix = substr(sha1(azurerm_resource_group.this.id), 0, 5)
  tags   = { application = var.workload, environment = var.env, managedBy = "terraform" }
}

resource "azurerm_resource_group" "this" {
  name     = "rg-${var.workload}-${var.env}"
  location = var.location
  tags     = local.tags
}

resource "azurerm_storage_account" "this" {
  name                            = lower("st${var.workload}${var.env}${local.suffix}")
  resource_group_name             = azurerm_resource_group.this.name
  location                        = azurerm_resource_group.this.location
  account_tier                    = "Standard"
  account_replication_type        = var.env == "prod" ? "ZRS" : "LRS"
  shared_access_key_enabled       = false
  allow_nested_items_to_be_public = false
  min_tls_version                 = "TLS1_2"
  tags                            = local.tags
}

data "azurerm_client_config" "current" {}

output "storage_id" { value = azurerm_storage_account.this.id }
```

| Block | Purpose |
|---|---|
| `terraform` | Version constraints, required providers, backend |
| `provider` | Provider configuration |
| `variable` | Inputs, with types, defaults, validation, `sensitive` |
| `locals` | Named expressions |
| `resource` | A managed resource |
| `data` | Read something that exists |
| `module` | Call a module |
| `output` | Return values |

Language features you will use: `count` and `for_each` for multiples, `for` expressions, `dynamic` blocks, conditional expressions, `depends_on` (rarely), and `lifecycle` (`prevent_destroy`, `ignore_changes`, `create_before_destroy`, `replace_triggered_by`).

Prefer `for_each` over `count` for collections: items are keyed by name, so removing one from the middle does not renumber and recreate the rest.

### State

State maps configuration to real resources and caches attributes. It is essential, sensitive and shared.

**Remote state on Azure** uses a storage account blob container:

```hcl
# backend.hcl  (passed with: terraform init -backend-config=backend.hcl)
resource_group_name  = "rg-tfstate-prod"
storage_account_name = "sttfstateprodswc001"
container_name       = "tfstate"
key                  = "hrassist/prod.tfstate"
use_azuread_auth     = true          # RBAC on the container, no account key
use_oidc             = true          # federated credentials in CI
```

| Property | Detail |
|---|---|
| **Locking** | A blob lease prevents concurrent applies |
| **Encryption** | At rest by Azure Storage; add customer-managed keys if required |
| **Access** | *Storage Blob Data Contributor* on the container for the identities that run Terraform. Disable shared keys and public access |
| **Recovery** | Enable blob versioning and soft delete |
| **Contents** | May include secrets in plain text (connection strings, generated passwords). Treat the state as a secret |

**Splitting state.** One state per deployable unit and environment: for example `platform/network`, `hrassist/dev`, `hrassist/prod`. Smaller states plan faster, limit blast radius and allow different permissions. Avoid one giant state for a whole organisation.

**State operations**

| Need | Mechanism |
|---|---|
| Adopt an existing resource | `import` block (declarative, reviewed in a plan), or `terraform import` |
| Rename or move within configuration | `moved` block |
| Stop managing without destroying | `removed` block with `lifecycle { destroy = false }` |
| Inspect | `terraform state list`, `terraform state show` |
| Generate configuration from existing Azure resources | Azure Export for Terraform (`aztfexport`) |

```hcl
import {
  to = azurerm_resource_group.this
  id = "/subscriptions/<sub>/resourceGroups/rg-legacy"
}

moved {
  from = azurerm_storage_account.old_name
  to   = azurerm_storage_account.this
}
```

**Workspaces** give multiple states for one configuration (`terraform workspace new dev`). They suit lightweight variation. For real environment separation, most teams prefer distinct backend keys or directories with their own variable files, so that environments can differ in backend, permissions and pinned versions.

### Authentication

| Context | Method |
|---|---|
| Local development | `az login`; the provider uses the CLI credential |
| GitHub Actions, Azure DevOps | **OIDC federation** (`use_oidc = true` or `ARM_USE_OIDC=true`) with `ARM_CLIENT_ID`, `ARM_TENANT_ID`, `ARM_SUBSCRIPTION_ID` |
| Self-hosted runner on Azure | **Managed identity** (`use_msi = true`) |
| Fallback | Service principal with certificate or secret |

No stored secret is needed in any modern setup. Use separate identities for **plan** (read access plus state access) and **apply** (write access), so that code on a pull request cannot change infrastructure.

### Workflow

```bash
terraform fmt -check -recursive        # formatting
terraform init -backend-config=backend.hcl
terraform validate                     # syntax and internal consistency
terraform plan  -var-file=prod.tfvars -out=tfplan
terraform show  tfplan                 # review; in CI, post to the pull request
terraform apply tfplan                 # apply exactly the reviewed plan
terraform destroy -var-file=dev.tfvars # tear down (non-production)
```

Saving the plan and applying that file guarantees that what was reviewed is what runs.

Supporting tools: `tflint`, security scanners such as Checkov or Trivy, `terraform test` for module tests, and `terraform-docs`.

### Modules

```hcl
module "storage" {
  source  = "Azure/avm-res-storage-storageaccount/azurerm"
  version = "~> 0.6"

  name                = "sthrassistprod001"
  resource_group_name = azurerm_resource_group.this.name
  location            = var.location
  shared_access_key_enabled = false
}

module "network" {
  source = "./modules/network"
  cidr   = "10.10.0.0/16"
}
```

Sources: local paths, the public Terraform Registry, private registries (HCP Terraform, or Git with tags). Pin versions. A good module has a small, typed interface with validation, sensible secure defaults and few outputs.

### Ecosystem notes

- **Licensing.** Terraform moved to the Business Source License in 2023. **OpenTofu** is the open-source fork under the Linux Foundation and is largely compatible. Some organisations choose it for licensing reasons; check provider and feature compatibility for your versions.
- **HCP Terraform / Terraform Enterprise** provide managed state, remote runs, policy as code (Sentinel, OPA) and a private registry. Azure Storage plus your CI system is a perfectly good self-managed alternative.
- **Azure landing zones.** The classic `caf-enterprise-scale` module has been archived. Its successor is the set of Azure Verified Modules for platform landing zones (file 06).

## How it works on Azure

### Bicep and Terraform compared

| | Bicep | Terraform |
|---|---|---|
| Scope | Azure (plus a few extensions) | Azure, Entra ID, other clouds, SaaS |
| State | None; Azure is the state | State file you must store and protect |
| Removing a resource from code | Not deleted (use deployment stacks) | Destroyed on apply |
| Drift detection | `what-if` | `plan` (refreshes against reality) |
| New Azure features | Immediately | Immediately with `azapi`; later with `azurerm` |
| Preview before change | `what-if` (some noise) | `plan` (generally precise) |
| Import existing resources | Decompile or export, then deploy | `import` blocks, `aztfexport` |
| Language | Concise, Azure-specific | General HCL, larger ecosystem |
| Tooling cost | Free | Free CLI; optional paid platform |
| Skills market | Azure specialists | Broad, cross-cloud |
| Microsoft support | First-party | Provider supported by HashiCorp with Microsoft; `azapi` and AVM by Microsoft |

Neither is wrong. Choose on what the team already knows, whether you need to manage non-Azure resources, and how you feel about owning state. Do not run both for the same resources.

### A Foundry workload in Terraform

```hcl
resource "azurerm_user_assigned_identity" "app" {
  name                = "id-${var.workload}-${var.env}"
  resource_group_name = azurerm_resource_group.this.name
  location            = var.location
}

resource "azapi_resource" "foundry" {
  type      = "Microsoft.CognitiveServices/accounts@2025-06-01"
  name      = "aif-${var.workload}-${var.env}-${local.suffix}"
  parent_id = azurerm_resource_group.this.id
  location  = var.location
  identity { type = "SystemAssigned" }
  body = {
    kind = "AIServices"
    sku  = { name = "S0" }
    properties = {
      allowProjectManagement = true
      customSubDomainName    = "aif-${var.workload}-${var.env}-${local.suffix}"
      disableLocalAuth       = true
      publicNetworkAccess    = "Disabled"
    }
  }
  response_export_values = ["properties.endpoint"]
}

resource "azapi_resource" "chat" {
  type      = "Microsoft.CognitiveServices/accounts/deployments@2025-06-01"
  name      = "chat"
  parent_id = azapi_resource.foundry.id
  body = {
    sku = { name = "DataZoneStandard", capacity = 50 }
    properties = {
      model                = { format = "OpenAI", name = "gpt-4.1-mini", version = "2025-04-14" }
      versionUpgradeOption = "NoAutoUpgrade"
    }
  }
}

resource "azurerm_role_assignment" "openai_user" {
  scope                = azapi_resource.foundry.id
  role_definition_name = "Cognitive Services OpenAI User"
  principal_id         = azurerm_user_assigned_identity.app.principal_id
  principal_type       = "ServicePrincipal"
}

output "foundry_endpoint" { value = azapi_resource.foundry.output.properties.endpoint }
```

`azurerm` also has resources for AI services accounts and deployments; whether they cover the newest Foundry properties depends on the provider version. Mixing `azurerm` for stable resources and `azapi` for fast-moving ones is normal. Verify API versions, SKU names and model versions before use.

### Repository layout

```text
infra/
├── modules/                 reusable modules (network, ai, data, app)
├── stacks/
│   ├── network/             one root module = one state
│   │   ├── main.tf  variables.tf  outputs.tf  providers.tf
│   │   └── env/ dev.tfvars  prod.tfvars  dev.backend.hcl  prod.backend.hcl
│   └── hrassist/
└── .tflint.hcl
```

Share values between states with outputs read through `terraform_remote_state`, or better, by looking resources up with data sources or passing IDs as variables, which avoids coupling to another state's layout.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| `azurerm` or `azapi` | `azurerm` where it covers the need; `azapi` for new or preview features. Some teams standardise on `azapi` alone for consistency with ARM |
| State granularity | Per workload per environment, plus separate platform states |
| Environment separation | Separate backend keys and variable files; separate subscriptions and identities |
| Version pinning | Pin providers with `~>` and commit `.terraform.lock.hcl`. Upgrade deliberately |
| Protecting critical resources | `prevent_destroy`, Azure locks, and policy. A mistaken `destroy` is Terraform's sharpest edge |
| Portal changes | Treated as drift and reverted on the next apply. Use `ignore_changes` sparingly for properties legitimately managed elsewhere |
| Secrets | Avoid generating them. If unavoidable, mark outputs `sensitive`, use ephemeral or write-only values where supported, and remember state may still hold them |

## Hands-on

```bash
mkdir tf-lab && cd tf-lab

# 1. State storage (once), RBAC-only
RG=rg-tfstate-lab; LOC=swedencentral; ST=sttfstate$RANDOM
az group create -n $RG -l $LOC -o none
az storage account create -n $ST -g $RG -l $LOC --sku Standard_LRS \
  --allow-shared-key-access false --allow-blob-public-access false -o none
az storage account blob-service-properties update -n $ST -g $RG --enable-versioning true -o none
ME=$(az ad signed-in-user show --query id -o tsv)
az role assignment create --assignee-object-id $ME --assignee-principal-type User \
  --role "Storage Blob Data Contributor" --scope $(az storage account show -n $ST -g $RG --query id -o tsv) -o none
sleep 60
az storage container create --account-name $ST -n tfstate --auth-mode login -o none

cat > backend.hcl <<EOF
resource_group_name  = "$RG"
storage_account_name = "$ST"
container_name       = "tfstate"
key                  = "lab/dev.tfstate"
use_azuread_auth     = true
EOF

# 2. Configuration
cat > main.tf <<'EOF'
terraform {
  required_providers { azurerm = { source = "hashicorp/azurerm", version = "~> 5.0" } }
  backend "azurerm" {}
}
provider "azurerm" {
  features {}
  subscription_id                 = var.subscription_id
  storage_use_azuread             = true
  resource_provider_registrations = "none"
}
variable "subscription_id" { type = string }
variable "containers"      { type = set(string), default = ["docs", "archive"] }

resource "azurerm_resource_group" "this" {
  name     = "rg-tf-lab"
  location = "swedencentral"
  tags     = { managedBy = "terraform" }
}
resource "azurerm_storage_account" "this" {
  name                      = "sttflab${substr(sha1(azurerm_resource_group.this.id), 0, 8)}"
  resource_group_name       = azurerm_resource_group.this.name
  location                  = azurerm_resource_group.this.location
  account_tier              = "Standard"
  account_replication_type  = "LRS"
  shared_access_key_enabled = false
}
resource "azurerm_storage_container" "this" {
  for_each           = var.containers
  name               = each.key
  storage_account_id = azurerm_storage_account.this.id
}
output "storage_name" { value = azurerm_storage_account.this.name }
EOF

export TF_VAR_subscription_id=$(az account show --query id -o tsv)

# 3. Workflow
terraform init -backend-config=backend.hcl
terraform fmt && terraform validate
terraform plan -out=tfplan
terraform apply tfplan
terraform state list

# 4. Drift: add a tag in the portal or with the CLI, then plan again and read the diff
az group update -n rg-tf-lab --set tags.drift=yes -o none
terraform plan

# 5. Removal destroys: remove "archive" and plan
terraform plan -var='containers=["docs"]'

# 6. Tear down
terraform destroy -auto-approve
az group delete -n $RG --yes --no-wait
```

If the container data-plane call fails with an authorisation error, grant yourself *Storage Blob Data Contributor* on the lab storage account too. Compare step 5 with the equivalent step in the Bicep lab.

## Security, cost and reliability notes

- **Security:** state is sensitive. Private container, Entra-only access, versioning, soft delete, and least-privilege roles. Never commit state or `.tfvars` containing secrets.
- **Security:** separate plan and apply identities; require approval before apply to production.
- **Security:** scan plans with policy tooling to block public endpoints, missing encryption and similar before they deploy.
- **Cost:** `terraform plan` shows what will be created; tools such as Infracost can estimate the bill in the pull request.
- **Reliability:** a lost or corrupted state is a serious incident. Versioning on the container is your recovery path.
- **Reliability:** an interrupted apply can leave a stale lease. Break it deliberately with `terraform force-unlock` only after confirming nothing else is running.

## Common mistakes

- Local state on a laptop for shared infrastructure.
- Storage account keys for the backend instead of Entra authentication.
- One enormous state.
- Applying without reviewing the plan, or re-planning at apply time.
- Using `count` for collections and recreating resources when the list changes.
- Unpinned providers picking up a breaking major version.
- Jumping to AzureRM 5.0 without handling resource provider registration and removed resources.
- Editing resources in the portal and being surprised when Terraform reverts them.
- Managing the same resource with both Bicep and Terraform.
- Secrets in outputs or variables without `sensitive`, and forgetting they persist in state.

## Check yourself

1. What is the state file, and why must it be protected?
2. What happens when you delete a resource block and apply?
3. When would you use the `azapi` provider instead of `azurerm`?
4. How should Terraform authenticate from a GitHub Actions workflow?
5. What is the benefit of `terraform plan -out` followed by `apply` of that file?
6. Why prefer `for_each` over `count`?
7. What changed with resource provider registration in AzureRM 5.0?
8. Give two differences between Bicep and Terraform.

<details>
<summary>Answers</summary>

1. Terraform's record of the resources it manages and their attributes. It is required to plan correctly and can contain secrets in plain text.
2. Terraform destroys the resource.
3. For resource types, properties or API versions that `azurerm` does not support yet, especially previews, or as a consistent ARM-shaped primary provider.
4. With OIDC workload identity federation, without stored secrets.
5. The exact reviewed plan is what gets applied, with no changes slipping in between review and execution.
6. Items are keyed by a stable identifier, so adding or removing one does not shift indices and recreate others.
7. `resource_provider_registrations` defaults to `none`, so the provider no longer auto-registers a large set of resource providers; you specify what is needed.
8. Any two of: Terraform keeps state and Bicep does not; Terraform destroys removed resources while Bicep does not by default; Terraform is multi-platform while Bicep is Azure-focused; Bicep has day-zero support natively while `azurerm` can lag.

</details>

## References

- [Terraform on Azure documentation](https://learn.microsoft.com/azure/developer/terraform/)
- [AzureRM provider documentation](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [AzureRM provider 5.0 announcement](https://www.hashicorp.com/blog/terraform-azurerm-provider-50-now-generally-available)
- [AzAPI provider overview](https://learn.microsoft.com/azure/developer/terraform/overview-azapi-provider)
- [Store Terraform state in Azure Storage](https://learn.microsoft.com/azure/developer/terraform/store-state-in-azure-storage)
- [Authenticate Terraform to Azure](https://learn.microsoft.com/azure/developer/terraform/authenticate-to-azure)
- [Azure Export for Terraform](https://learn.microsoft.com/azure/developer/terraform/azure-export-for-terraform/export-terraform-overview)
- [Comparing Terraform and Bicep](https://learn.microsoft.com/azure/developer/terraform/comparing-terraform-and-bicep)
- [Terraform language documentation](https://developer.hashicorp.com/terraform/language)
- [OpenTofu](https://opentofu.org/)
