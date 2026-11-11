# GitHub Actions for Azure

## Why it matters

GitHub Actions is the most common CI/CD engine for teams building on Azure: it lives next to the code, has a huge marketplace, and integrates with GitHub security features and Copilot. Used well, it deploys to Azure without a single stored secret, enforces approvals for production, and lets a platform team publish reusable, governed workflows that every application team consumes. Used carelessly, it's a supply chain risk with long-lived credentials and unpinned third-party actions. This file covers workflows, **OIDC federation** to Azure, **environments** and protection rules, and **reusable workflows**.

## Key concepts

### Building blocks

| Concept | Meaning |
|---|---|
| **Workflow** | YAML in `.github/workflows/`, triggered by events (`push`, `pull_request`, `workflow_dispatch`, `schedule`, `release`) |
| **Job** | Runs on a runner; jobs run in parallel unless `needs:` creates dependencies |
| **Step** | Shell command or action |
| **Action** | Reusable unit (`azure/login`, `actions/checkout`, `azure/webapps-deploy`) |
| **Runner** | GitHub-hosted (Ubuntu, Windows, macOS, larger runners) or self-hosted (including in Azure VNets) |
| **Environment** | Named deployment target with protection rules, secrets/variables and deployment history |
| **Reusable workflow** | Workflow called with `uses:` from other workflows; centralises standards |
| **Composite action** | Bundle of steps reused as one action |

### OIDC federation (no secrets)

The workflow requests a short-lived OIDC token from GitHub; Microsoft Entra trusts it through a **federated identity credential** on an app registration or **user-assigned managed identity**, and exchanges it for an Azure token.

| Federated credential field | Value |
|---|---|
| Issuer | `https://token.actions.githubusercontent.com` |
| Audience | `api://AzureADTokenExchange` |
| Subject | Must match the token's `sub` claim exactly, for example `repo:contoso/orders:environment:production` or `repo:contoso/orders:ref:refs/heads/main` |

Bind production credentials to the **environment** subject so only jobs running in the protected `production` environment can authenticate.

### Environments and protection

| Protection | Use |
|---|---|
| Required reviewers | Human approval before production deployment |
| Wait timer | Delay for bake time |
| Deployment branches/tags | Only `main` or release tags can deploy |
| Environment secrets/variables | Per-environment configuration (prefer variables + OIDC over secrets) |
| Custom deployment protection rules | Gate on external checks (change management, monitoring) |

### Security hardening

| Practice | Why |
|---|---|
| `permissions:` least privilege (default `contents: read`) | Limit `GITHUB_TOKEN` |
| Pin third-party actions to full commit SHAs | Prevent tag hijacking |
| Allow-list actions at organisation level | Supply chain control |
| Avoid `pull_request_target` with untrusted code | Prevents secret exposure to forks |
| Don't echo secrets; use masking | Log leakage |
| Ephemeral, isolated self-hosted runners | Prevent persistence across jobs |
| Dependabot for actions | Keep actions updated |

## How it works on Azure

Create the identity and federated credential:

```bash
az identity create -g rg-cicd -n id-gh-orders-prod
az identity federated-credential create -g rg-cicd --identity-name id-gh-orders-prod -n gh-prod \
  --issuer https://token.actions.githubusercontent.com \
  --subject repo:contoso/orders:environment:production \
  --audiences api://AzureADTokenExchange
az role assignment create --assignee $(az identity show -g rg-cicd -n id-gh-orders-prod --query principalId -o tsv) \
  --role "Website Contributor" --scope /subscriptions/<sub>/resourceGroups/rg-orders-prod
```

Reusable deployment workflow published by the platform team (`contoso/platform-workflows/.github/workflows/deploy-webapp.yml`):

```yaml
on:
  workflow_call:
    inputs:
      environment: { required: true, type: string }
      app-name:    { required: true, type: string }
      artifact:    { required: true, type: string }
permissions: { id-token: write, contents: read }
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/download-artifact@<sha>
        with: { name: "${{ inputs.artifact }}" }
      - uses: azure/login@<sha>
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - uses: azure/webapps-deploy@<sha>
        with: { app-name: "${{ inputs.app-name }}", package: . }
```

Application workflow calling it:

```yaml
name: orders
on: { push: { branches: [main] } }
permissions: { id-token: write, contents: read }
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>
      - run: dotnet publish src/Orders -c Release -o out
      - uses: actions/upload-artifact@<sha>
        with: { name: orders, path: out }
  staging:
    needs: build
    uses: contoso/platform-workflows/.github/workflows/deploy-webapp.yml@v3
    with: { environment: staging, app-name: app-orders-stg, artifact: orders }
  production:
    needs: staging
    uses: contoso/platform-workflows/.github/workflows/deploy-webapp.yml@v3
    with: { environment: production, app-name: app-orders-prod, artifact: orders }
```

Replace `<sha>` with the full commit SHA of each action version you've approved.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **GitHub-hosted vs self-hosted runners** | Hosted for simplicity; self-hosted (or larger runners with Azure private networking) for VNet access, custom tooling or compliance. |
| **Identity per environment** | Separate managed identities and federated credentials per environment with least-privilege roles. |
| **Reusable workflows** | Platform team owns standard build/deploy/scan workflows; apps consume pinned versions. |
| **Secrets** | Prefer OIDC and variables; store unavoidable secrets in Key Vault, not GitHub. |
| **Approvals** | Required reviewers on production; automate lower environments. |
| **Monorepo** | Path filters and matrix jobs to build only what changed. |

## Hands-on

1. Create a user-assigned managed identity with a federated credential for a `staging` environment; deploy a web app with `azure/login`.
2. Add a `production` environment with required reviewers and a deployment branch rule.
3. Extract the deployment job into a reusable workflow and call it from two repositories.
4. Set default workflow permissions to read-only and pin all actions to SHAs.
5. Enable Dependabot for GitHub Actions updates.

## Security, cost and reliability notes

- **Subject claims must match exactly**; decode the token claims if login fails (without printing the token).
- **Self-hosted runners** on public repos are dangerous; restrict to private repos and ephemeral runners.
- **Cost**: GitHub-hosted minutes and larger runners are billed; cache dependencies.
- **Reliability**: retries for transient deployment failures; concurrency groups to avoid overlapping deployments.

## Common mistakes

| Mistake | Fix |
|---|---|
| Service principal secret in repository secrets | OIDC federation |
| One identity with Contributor on the subscription | Per-environment, least-privilege identities |
| Actions referenced by `@v1` tags | Pin SHAs |
| Production deploys without approval | Environment protection |
| Copy-pasted workflows across repos | Reusable workflows |

## Check yourself

1. What are workflows, jobs, steps, actions and runners?
2. How does OIDC federation to Azure work, and what three fields must match?
3. What protections can environments enforce?
4. Why use reusable workflows?
5. List four workflow hardening practices.
6. When would you use self-hosted runners?

<details>
<summary>Answers</summary>

1. Workflow: YAML automation triggered by events; job: set of steps on a runner; step: command or action; action: reusable unit; runner: machine executing jobs.
2. GitHub issues a short-lived OIDC token; Entra trusts it via a federated credential and issues an Azure token; issuer, audience and subject must match.
3. Required reviewers, wait timers, deployment branch/tag rules, environment secrets and variables, custom protection rules.
4. Centralise and govern standard build/deploy logic, reduce duplication and update many repos by version.
5. Any four: least-privilege `permissions`, pin actions to SHAs, allow-list actions, avoid `pull_request_target` with untrusted code, mask secrets, ephemeral runners, Dependabot.
6. When jobs need private network access, custom tooling, specific hardware or compliance controls.

</details>

## References

- [Use GitHub Actions to connect to Azure (OIDC)](https://learn.microsoft.com/azure/developer/github/connect-from-azure-openid-connect)
- [Reusing workflows](https://docs.github.com/en/actions/sharing-automations/reusing-workflows)
- [Managing environments for deployment](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment)
- [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
