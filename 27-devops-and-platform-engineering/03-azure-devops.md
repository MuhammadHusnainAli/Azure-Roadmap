# Azure DevOps

## Why it matters

Many enterprises, especially those with long Microsoft histories, run their delivery on **Azure DevOps**: Boards for work tracking, Repos for Git, Pipelines for CI/CD, Artifacts for packages and Test Plans for manual and exploratory testing. It's mature, integrates deeply with Entra and Azure, supports on-premises (Azure DevOps Server) and fits regulated environments that need traceability from requirement to deployment. Architects frequently design for Azure DevOps or hybrid setups (GitHub repos with Azure Boards or Pipelines). Key modern practices are YAML pipelines, workload identity federation service connections, environments with checks, templates for reuse, and **Managed DevOps Pools** for custom agents.

## Key concepts

### Services

| Service | Purpose |
|---|---|
| **Azure Boards** | Work items (epics, features, user stories, bugs, tasks), backlogs, sprints, Kanban, queries, dashboards; GitHub integration |
| **Azure Repos** | Git repositories with branch policies, PRs (TFVC legacy) |
| **Azure Pipelines** | CI/CD with YAML (or classic) pipelines for any language, cloud or platform; GitHub repos supported |
| **Azure Artifacts** | Feeds for NuGet, npm, Maven, Python, Cargo and universal packages; **upstream sources** proxy public registries |
| **Azure Test Plans** | Manual and exploratory testing, test suites, traceability |
| **Advanced Security for Azure DevOps** | Secret, dependency and code scanning for Azure Repos |

### Pipelines concepts

| Concept | Meaning |
|---|---|
| **Stages → jobs → steps** | Structure of a YAML pipeline |
| **Templates** | `extends` (enforce a required structure) and `template` includes for stages, jobs, steps, variables |
| **Environments** | Deployment targets with **approvals and checks** (manual approval, business hours, Azure Monitor alerts query, required template, invoke REST/Function) and deployment history |
| **Service connections** | Authenticate to Azure; use **workload identity federation** (no secrets) |
| **Variable groups** | Shared variables, linkable to Key Vault |
| **Agents** | Microsoft-hosted, self-hosted, VM scale set pools, or **Managed DevOps Pools** |

### Managed DevOps Pools

Evolution of VM scale set agent pools, managed as an Azure resource:

- custom images (including the same software as Microsoft-hosted agents) and more powerful SKUs;
- agents in the region closest to dependencies; VNet injection for private resources;
- scale up/down by configuration; agent state kept up to seven days for faster cached builds;
- jobs up to two days long;
- visibility into agent provisioning and reimaging.

### GitHub + Azure DevOps

| Pattern | Use |
|---|---|
| GitHub repos + Azure Boards | Developers on GitHub, planning in Boards (AB#123 links) |
| GitHub repos + Azure Pipelines | Existing pipeline investments |
| Migration to GitHub | GitHub Enterprise Importer for repos; Actions Importer for pipelines |

## How it works on Azure

Workload identity federation service connection (created in Project settings → Service connections → Azure Resource Manager → Workload identity federation, or via CLI/REST), then a multi-stage YAML pipeline with templates and environment checks:

```yaml
# azure-pipelines.yml
trigger: { branches: { include: [ main ] } }
extends:
  template: templates/secure-pipeline.yml@platform     # enforced structure from platform repo
  parameters:
    buildSteps:
      - task: DotNetCoreCLI@2
        inputs: { command: publish, projects: src/Orders/Orders.csproj, arguments: '-c Release -o $(Build.ArtifactStagingDirectory)' }
      - publish: $(Build.ArtifactStagingDirectory)
        artifact: orders
    deployments:
      - environment: orders-staging
        serviceConnection: sc-orders-staging
        appName: app-orders-stg
      - environment: orders-production      # approvals and checks configured on the environment
        serviceConnection: sc-orders-prod
        appName: app-orders-prod
resources:
  repositories:
    - repository: platform
      type: git
      name: Platform/pipeline-templates
      ref: refs/tags/v4
```

Deployment job inside the template:

```yaml
- deployment: deploy
  environment: ${{ d.environment }}
  pool: { name: mdp-contoso-linux }          # Managed DevOps Pool
  strategy:
    runOnce:
      deploy:
        steps:
          - task: AzureWebApp@1
            inputs:
              azureSubscription: ${{ d.serviceConnection }}
              appName: ${{ d.appName }}
              package: $(Pipeline.Workspace)/orders/**/*.zip
```

Configure the production environment with: approvals (release managers), **required template** check (only pipelines extending the approved template can deploy), business-hours check, and an Azure Monitor alerts check that blocks deployment while Sev 0/1 alerts are active.

Azure Artifacts feed with upstream sources gives a curated, cached package source and protection against public registry outages.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **YAML vs classic** | YAML for everything new (versioned, reviewable); migrate classic release pipelines. |
| **Service connections** | Workload identity federation; per-environment connections with least privilege; restrict pipeline permissions on connections. |
| **Agents** | Microsoft-hosted for simplicity; Managed DevOps Pools for private networking, custom images, performance. |
| **Templates** | `extends` templates for security-mandated structure; versioned via tags. |
| **Boards vs GitHub Issues** | Boards for portfolio planning and traceability needs; GitHub Issues/Projects for developer-centric teams. |
| **Stay or migrate** | Hybrid is fine; migrate when consolidating on GitHub Advanced Security and Copilot workflows. |

## Hands-on

1. Create a workload identity federation service connection to a resource group.
2. Build a multi-stage YAML pipeline deploying to staging and production environments.
3. Add approvals, a required template check and an Azure Monitor alerts check on production.
4. Create a Managed DevOps Pool with VNet injection and run a job that reaches a private endpoint.
5. Create an Artifacts feed with npm/PyPI upstreams and consume it in the build.

## Security, cost and reliability notes

- **Service connection scope**: restrict to specific pipelines; never "grant access to all pipelines" for production.
- **Secrets**: Key Vault-linked variable groups; avoid secret variables in YAML.
- **Agent hygiene**: ephemeral or reimaged agents to avoid cross-job contamination.
- **Cost**: parallel jobs for Microsoft-hosted agents; Managed DevOps Pools billed as Azure compute plus service.
- **Reliability**: retry tasks, timeouts, and deployment strategies (rolling, canary) in deployment jobs.

## Common mistakes

| Mistake | Fix |
|---|---|
| Secret-based service principals | Workload identity federation |
| Classic release pipelines with manual edits | YAML with templates |
| One service connection for all environments | Per-environment connections |
| Unrestricted pipeline access to production resources | Pipeline permissions and checks |
| Long-lived self-hosted agents | Managed DevOps Pools with reimaging |

## Check yourself

1. List the Azure DevOps services and their purpose.
2. What do environments with approvals and checks provide?
3. Why use `extends` templates?
4. What advantages do Managed DevOps Pools offer?
5. How should pipelines authenticate to Azure?
6. How can GitHub and Azure DevOps be combined?

<details>
<summary>Answers</summary>

1. Boards (work tracking), Repos (Git), Pipelines (CI/CD), Artifacts (packages), Test Plans (manual/exploratory testing), plus Advanced Security.
2. Gated deployments (manual approval, business hours, alert checks, required template, REST/Function checks) and deployment history.
3. To enforce a mandatory, secure pipeline structure that application pipelines must follow.
4. Custom images and SKUs, regional placement, VNet access, configurable scaling, cached state up to seven days, jobs up to two days, visibility.
5. Workload identity federation service connections with least-privilege roles.
6. GitHub repos with Azure Boards and/or Azure Pipelines, or migration tools to move to GitHub.

</details>

## References

- [What is Azure DevOps?](https://learn.microsoft.com/azure/devops/user-guide/what-is-azure-devops)
- [Managed DevOps Pools overview](https://learn.microsoft.com/azure/devops/managed-devops-pools/overview)
- [Connect to Azure with service connections](https://learn.microsoft.com/azure/devops/pipelines/library/connect-to-azure)
- [Approvals and checks](https://learn.microsoft.com/azure/devops/pipelines/process/approvals)
- [Template usage reference](https://learn.microsoft.com/azure/devops/pipelines/process/templates)
