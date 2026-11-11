# CI/CD for Infrastructure

## Why it matters

Infrastructure changes are among the riskiest changes you make: a wrong NSG rule, a deleted subnet or a disabled policy can take down every workload at once or open a security hole. Writing infrastructure as code (IaC) is only half the job ([Topic 04](../04-tooling-and-infrastructure-as-code/README.md)); the other half is delivering it safely through pipelines that **validate**, **preview** (`what-if` or `plan`), require **approval** for meaningful changes, **deploy** with least-privilege identities and **detect drift** when someone changes resources outside the pipeline. This file shows the pattern for Bicep and Terraform on GitHub Actions and Azure Pipelines.

## Key concepts

### Pipeline stages

| Stage | Bicep | Terraform |
|---|---|---|
| **Lint and format** | `bicep lint` (linter rules in `bicepconfig.json`), `az bicep build` | `terraform fmt -check`, `terraform validate`, TFLint |
| **Security scan** | IaC scanning (Defender for Cloud DevOps security, Checkov, PSRule for Azure) | Same; plus policy-as-code (OPA/Conftest) |
| **Preview** | `az deployment group what-if` | `terraform plan -out=tfplan` |
| **Approval gate** | Review what-if output; environment approval | Review plan; environment approval |
| **Deploy** | `az deployment group create` or **deployment stacks** | `terraform apply tfplan` (the reviewed plan) |
| **Test** | Smoke tests, Azure Resource Graph assertions, PSRule post-deploy | Terratest or `terraform test`, smoke tests |
| **Drift detection** | Scheduled what-if; deployment stacks deny settings | Scheduled `plan -detailed-exitcode` |

### Bicep specifics

| Feature | Use |
|---|---|
| **what-if** | Shows create/modify/delete/no-change per resource before deploying (some false positives from noisy properties) |
| **Deployment stacks** | Manage a group of resources as a unit; `actionOnUnmanage` deletes or detaches removed resources; **deny settings** block out-of-band changes or deletes |
| **Azure Verified Modules** | Tested modules for consistency ([Topic 04 file 06](../04-tooling-and-infrastructure-as-code/06-azure-verified-modules.md)) |
| **Template specs / registries** | Publish versioned modules for teams |

### Terraform specifics

| Feature | Use |
|---|---|
| **Remote state** | Azure Storage backend with Entra auth, blob versioning, soft delete and state locking (lease) |
| **Plan files** | Apply exactly what was reviewed; plan artefacts are sensitive |
| **Workspaces or directories** | Separate state per environment |
| **Providers** | AzureRM for most resources; AzAPI for new or preview APIs |
| **Exit codes** | `plan -detailed-exitcode` returns 2 when changes exist (useful for drift detection) |

### Drift

Drift is the difference between code and reality. Sources: portal hotfixes, other automation, Azure Policy `modify`/`deployIfNotExists` effects, platform defaults. Detect it on a schedule; decide to codify or revert; prevent with RBAC (read-only for humans in production), deny assignments (stacks) and policy.

## How it works on Azure

GitHub Actions: Bicep with what-if on PR and deploy on merge:

```yaml
name: infra
on:
  pull_request: { paths: [ "infra/**" ] }
  push: { branches: [ main ], paths: [ "infra/**" ] }
permissions: { id-token: write, contents: read, pull-requests: write }
jobs:
  validate:
    runs-on: ubuntu-latest
    environment: infra-plan            # identity with Reader + what-if permissions
    steps:
      - uses: actions/checkout@<sha>
      - uses: azure/login@<sha>
        with: { client-id: "${{ vars.AZURE_CLIENT_ID }}", tenant-id: "${{ vars.AZURE_TENANT_ID }}", subscription-id: "${{ vars.AZURE_SUBSCRIPTION_ID }}" }
      - run: az bicep lint --file infra/main.bicep
      - run: |
          az deployment group what-if -g rg-orders-prod -f infra/main.bicep -p infra/prod.bicepparam \
            --result-format ResourceIdOnly > whatif.txt
          cat whatif.txt
  deploy:
    if: github.event_name == 'push'
    needs: validate
    runs-on: ubuntu-latest
    environment: infra-production      # required reviewers
    steps:
      - uses: actions/checkout@<sha>
      - uses: azure/login@<sha>
        with: { client-id: "${{ vars.AZURE_CLIENT_ID }}", tenant-id: "${{ vars.AZURE_TENANT_ID }}", subscription-id: "${{ vars.AZURE_SUBSCRIPTION_ID }}" }
      - run: |
          az stack group create -n orders-infra -g rg-orders-prod \
            -f infra/main.bicep -p infra/prod.bicepparam \
            --action-on-unmanage detachAll --deny-settings-mode denyWriteAndDelete \
            --deny-settings-excluded-principals "${{ vars.BREAKGLASS_OBJECT_ID }}" --yes
```

Terraform with a reviewed plan:

```bash
terraform init -backend-config="resource_group_name=rg-tfstate" -backend-config="storage_account_name=sttfstateprod" \
  -backend-config="container_name=tfstate" -backend-config="key=orders.tfstate" -backend-config="use_azuread_auth=true"
terraform plan -out=tfplan -detailed-exitcode     # exit 2 = changes present
terraform show -no-color tfplan > plan.txt        # post to PR for review
# after approval, in the deploy job using the same artefact:
terraform apply tfplan
```

Scheduled drift detection: nightly job runs `what-if` or `plan -detailed-exitcode` against production and opens an issue or alert when changes appear.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Identities** | Separate plan (read) and apply (write) identities; apply identity only in protected environments. |
| **Granularity** | One deployment/state per workload per environment; platform/landing zone in separate pipelines. |
| **Stacks deny settings** | Use `denyDelete` or `denyWriteAndDelete` for production to stop out-of-band changes; exclude break-glass. |
| **Policy interplay** | Expect policy-driven changes; codify them or exclude from drift noise. |
| **Promotion** | Same templates, different parameter files per environment; promote by merging, not by copying. |
| **Bicep vs Terraform** | Bicep for Azure-only with day-zero API support; Terraform for multicloud and existing skills ([Topic 04](../04-tooling-and-infrastructure-as-code/README.md)). |

## Hands-on

1. Build the Bicep workflow; open a PR and review the what-if output.
2. Deploy with a deployment stack and deny settings; try to delete a resource in the portal.
3. Build a Terraform pipeline with remote state in Azure Storage using Entra auth and plan artefacts.
4. Add PSRule or Checkov scanning to fail on a public storage account.
5. Create a nightly drift detection job and introduce manual drift to test it.

## Security, cost and reliability notes

- **State files** contain secrets and resource details; protect storage with private access, RBAC and versioning.
- **Plan artefacts** are sensitive; short retention.
- **Blast radius**: smaller deployments reduce the impact of mistakes.
- **Cost**: preview environments and test deployments; tear down automatically.

## Common mistakes

| Mistake | Fix |
|---|---|
| Applying without previewing | what-if/plan with review |
| Re-planning at apply time | Apply the reviewed plan |
| Humans with Contributor in production | Read-only humans, pipeline-only writes |
| Ignoring drift | Scheduled detection |
| Terraform state with access keys | Entra auth and locked-down storage |

## Check yourself

1. List the stages of an IaC pipeline.
2. What do what-if and plan provide?
3. What are deployment stacks and deny settings?
4. Why apply a saved plan file?
5. What causes drift, and how do you detect it?
6. How should identities be split?

<details>
<summary>Answers</summary>

1. Lint/format, security scan, preview, approval, deploy, test, drift detection.
2. A preview of creates, modifications and deletions before applying.
3. Stacks manage resources as a unit with lifecycle on removal; deny settings block writes/deletes outside the stack.
4. To deploy exactly what was reviewed and approved.
5. Portal changes, other automation, policy effects; detect with scheduled what-if or `plan -detailed-exitcode`.
6. A read-only identity for validation and preview; a write identity only in protected deployment environments.

</details>

## References

- [Bicep what-if](https://learn.microsoft.com/azure/azure-resource-manager/bicep/deploy-what-if)
- [Deployment stacks](https://learn.microsoft.com/azure/azure-resource-manager/bicep/deployment-stacks)
- [Bicep linter](https://learn.microsoft.com/azure/azure-resource-manager/bicep/linter)
- [Store Terraform state in Azure Storage](https://learn.microsoft.com/azure/developer/terraform/store-state-in-azure-storage)
- [Terraform plan](https://developer.hashicorp.com/terraform/cli/commands/plan)
