# Developer Platforms

## Why it matters

A new developer's first week often disappears into installing tools, requesting access, finding the right subscription and waiting for a test environment. Multiply that by every project switch and every team, and environment friction becomes a major productivity and security problem (local machines with production credentials, unmanaged test VMs that never get deleted). Developer platforms give developers **ready-to-code workstations** and **self-service environments** within guardrails set by platform and security teams. The Azure portfolio here is changing significantly: **Microsoft Dev Box is retiring** (Windows 365 is the recommended path), **Azure Deployment Environments is retiring**, and **DevTest Labs** remains for VM-based labs. Architects must plan around these lifecycles and know the alternatives, including GitHub Codespaces and IaC-based self-service.

## Key concepts

### Workstations

| Option | Status | Fit |
|---|---|---|
| **Windows 365 Cloud PCs** (developer-focused configurations) | Recommended Microsoft path for managed cloud workstations | Persistent, Intune-managed Windows workstations for developers |
| **Microsoft Dev Box** | **Retiring 18 September 2028**; closing-down period began 14 September 2026 | Existing customers: plan migration to Windows 365 or alternatives; not a one-to-one replacement, so validate scenarios |
| **GitHub Codespaces** | Available | Container-based, repository-defined (`devcontainer.json`) cloud dev environments; browser or VS Code |
| **Dev Containers (local)** | Available | Reproducible tooling on local machines |
| **Azure Virtual Desktop** | Available | Pooled desktops for broader user groups |

### Environments for developers

| Option | Status | Fit |
|---|---|---|
| **Azure Deployment Environments (ADE)** | **Retiring 22 February 2027** | Existing users must move to direct IaC, GitHub/Azure DevOps workflows or partner solutions |
| **IaC templates + pipelines** (Bicep, Terraform, deployment stacks, template specs, AVM) | Recommended replacement pattern | Self-service via workflow dispatch, PR, or a portal (Backstage, internal portal) |
| **Azure DevTest Labs** | Available | Self-service VMs for dev, test, classroom and training with cost controls (quotas, auto-shutdown, claimable VMs, artefacts) |
| **Ephemeral PR environments** | Pattern | Container Apps, App Service slots or Static Web Apps previews per pull request |
| **Sandbox subscriptions** | Pattern (landing zones) | Isolated subscriptions with budgets and policies for experimentation |

### Golden paths

A **golden path** (paved road) is the supported, opinionated way to build and run a type of workload: template repository, `devcontainer.json`, CI/CD workflows, IaC modules, observability and security defaults, docs. Teams can deviate, but the golden path is the easiest route and is kept current by the platform team (file 10).

## How it works on Azure

Repository-defined environment with Codespaces/Dev Containers:

```json
// .devcontainer/devcontainer.json
{
  "name": "orders-api",
  "image": "mcr.microsoft.com/devcontainers/dotnet:9.0",
  "features": {
    "ghcr.io/devcontainers/features/azure-cli:1": {},
    "ghcr.io/azure/azure-dev/azd:0": {},
    "ghcr.io/devcontainers/features/docker-in-docker:2": {}
  },
  "customizations": { "vscode": { "extensions": [ "ms-azuretools.vscode-bicep", "GitHub.copilot" ] } },
  "postCreateCommand": "dotnet restore"
}
```

Self-service environment without ADE: a GitHub workflow that deploys a sandbox environment from approved Bicep with a time-to-live tag:

```yaml
name: create-sandbox
on:
  workflow_dispatch:
    inputs: { name: { required: true }, ttlDays: { default: "3" } }
permissions: { id-token: write, contents: read }
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: sandbox
    steps:
      - uses: actions/checkout@<sha>
      - uses: azure/login@<sha>
        with: { client-id: "${{ vars.AZURE_CLIENT_ID }}", tenant-id: "${{ vars.AZURE_TENANT_ID }}", subscription-id: "${{ vars.SANDBOX_SUBSCRIPTION_ID }}" }
      - run: |
          RG=rg-sbx-${{ inputs.name }}
          az group create -n $RG -l westeurope --tags owner=${{ github.actor }} expires=$(date -d "+${{ inputs.ttlDays }} days" +%F)
          az stack group create -n sbx -g $RG -f templates/webapp-sql/main.bicep --action-on-unmanage deleteAll --deny-settings-mode none --yes
```

A scheduled clean-up job deletes resource groups whose `expires` tag has passed; Azure Policy enforces allowed SKUs and regions in the sandbox subscription; budgets alert on spend.

DevTest Labs for training: lab with allowed VM sizes, auto-shutdown at 19:00, claimable VMs from custom images, per-user VM limits.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Dev Box customers** | Start migration planning now; assess Windows 365 for persistent Windows needs and Codespaces for repository-centric work. |
| **ADE customers** | Move environment definitions to IaC pipelines or a portal before February 2027; reuse templates. |
| **Cloud vs local** | Cloud workstations for security (no source or credentials on laptops) and onboarding speed; local dev containers for offline and low-cost needs. |
| **Environment lifetime** | Ephemeral by default with TTL; long-lived shared environments only for integration/staging. |
| **Guardrails** | Policies, budgets, allowed SKUs and identities in sandbox subscriptions. |
| **Portal** | Backstage or a lightweight internal portal over pipelines when self-service usage grows (file 10). |

## Hands-on

1. Add a `devcontainer.json` to a repository and open it in Codespaces; measure time to first successful build.
2. Build the sandbox workflow with TTL tags and a scheduled clean-up job.
3. Create a DevTest Lab with auto-shutdown and allowed sizes for a training scenario.
4. If you use Dev Box or ADE, inventory dependencies and draft a migration plan with target dates.
5. Document one golden path end to end.

## Security, cost and reliability notes

- **Credentials**: developers authenticate with Entra; no production access from dev environments.
- **Data**: no production personal data in sandboxes; synthetic or masked data.
- **Cost**: auto-shutdown, TTLs and budgets; idle cloud workstations are expensive.
- **Lifecycle risk**: track retirement dates of developer services in the platform roadmap.

## Common mistakes

| Mistake | Fix |
|---|---|
| Building new solutions on Dev Box or ADE | Use Windows 365, Codespaces and IaC pipelines |
| Sandboxes without expiry | TTL tags and clean-up |
| Different tooling per developer | Dev containers |
| Developers with Owner on shared subscriptions | Scoped sandbox subscriptions |
| No golden path | Templates with defaults |

## Check yourself

1. When does Microsoft Dev Box retire, and what's the recommended path?
2. When does Azure Deployment Environments retire, and what replaces it?
3. What does DevTest Labs provide?
4. What is a dev container, and why use it?
5. How do you implement self-service environments with guardrails?
6. What is a golden path?

<details>
<summary>Answers</summary>

1. 18 September 2028 (closing-down period from 14 September 2026); Windows 365 or another appropriate solution.
2. 22 February 2027; direct IaC (Bicep/ARM), GitHub/Azure DevOps workflows or partner solutions.
3. Self-service VMs for dev, test and training with quotas, auto-shutdown, claimable VMs and artefacts.
4. A repository-defined, containerised development environment giving reproducible tooling locally or in Codespaces.
5. Pipelines or a portal deploying approved IaC into sandbox subscriptions with policies, budgets, TTL tags and clean-up automation.
6. The supported, opinionated, easiest route to build and run a type of workload, with templates and defaults.

</details>

## References

- [What is Microsoft Dev Box? (retirement notice)](https://learn.microsoft.com/azure/dev-box/overview-what-is-microsoft-dev-box)
- [Dev Box retirement guide](https://learn.microsoft.com/azure/dev-box/dev-box-retirement-guide)
- [Azure Deployment Environments retirement guide](https://learn.microsoft.com/azure/deployment-environments/deployment-environments-retirement-guide)
- [Azure DevTest Labs](https://learn.microsoft.com/azure/devtest-labs/devtest-lab-overview)
- [Windows 365 overview](https://learn.microsoft.com/windows-365/enterprise/overview)
- [GitHub Codespaces](https://docs.github.com/en/codespaces/overview)
