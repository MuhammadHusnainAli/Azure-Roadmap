# CI/CD for Applications

## Why it matters

The goal of application CI/CD is boring, frequent, low-risk releases: every change is built once, tested automatically, packaged as an immutable artefact, scanned, and promoted through environments to production with safe rollout and fast rollback. Each Azure hosting platform has its own deployment mechanics (App Service slots, Container Apps revisions, AKS manifests or GitOps), but the pipeline shape is the same. Getting this right shortens lead time, reduces change failure rate and makes incidents easier to roll back, the core DORA metrics.

## Key concepts

### Pipeline shape

```text
Commit ─► CI: restore, build, unit test, lint, SAST/SCA, build image/package once ─► publish artefact (versioned, signed)
       ─► CD: deploy to dev ─► integration/E2E tests ─► staging (prod-like) ─► approval ─► production (progressive) ─► verify ─► (auto) rollback
```

| Principle | Meaning |
|---|---|
| **Build once, deploy many** | The same artefact (zip or image digest) moves through environments; only configuration changes |
| **Immutable artefacts** | Container images referenced by **digest**; packages versioned |
| **Configuration outside the artefact** | App settings, App Configuration, Key Vault references |
| **Environment parity** | Staging mirrors production topology |
| **Automated verification** | Smoke tests and health checks after each deployment |
| **Fast rollback** | Slot swap back, revision traffic shift, previous image tag |

### Platform deployment mechanics

| Platform | Mechanism | Safe rollout |
|---|---|---|
| **App Service** | Zip deploy / run-from-package, container image; `azure/webapps-deploy`, `AzureWebApp@1` | **Deployment slots**: deploy to `staging`, warm up, **swap** (with preview); roll back by swapping again; traffic routing % to slots |
| **Azure Functions** | Zip deploy / Flex Consumption deployment | Slots on supported plans |
| **Container Apps** | New image creates a **revision**; `azure/container-apps-deploy-action`, `az containerapp update` | Multiple revision mode with **traffic splitting** and labels |
| **AKS** | Kubernetes manifests, Helm, Kustomize via `kubectl`/`helm` or **GitOps** (Flux extension, Argo CD); **AKS automated deployments** generate pipelines | Rolling updates, canary with service mesh/ingress, Argo Rollouts/Flagger |
| **Static Web Apps** | GitHub/Azure DevOps integration | Preview environments per PR |

### Container build and registry

- Build images in CI (Docker build, `az acr build`), tag with commit SHA, push to **Azure Container Registry**, record digest.
- Scan images (Defender for Containers), sign (Notation) and generate SBOM (Topic 25 file 09).
- Pull with managed identity (`AcrPull`); no admin user.
- Use ACR geo-replication for multi-region and ACR tasks for base image updates.

### Testing pyramid in the pipeline

| Level | When |
|---|---|
| Unit tests | Every commit (fast) |
| Contract/API tests | PR and dev |
| Integration tests | Dev/test environment with real dependencies |
| End-to-end (Playwright) | Staging (file 08) |
| Load tests | Before major releases and on schedule (file 08) |
| Smoke tests | After every deployment, including production |

## How it works on Azure

Container Apps deployment with GitHub Actions and a canary revision:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions: { id-token: write, contents: read }
    outputs: { image: "${{ steps.img.outputs.image }}" }
    steps:
      - uses: actions/checkout@<sha>
      - uses: azure/login@<sha>
        with: { client-id: "${{ vars.AZURE_CLIENT_ID }}", tenant-id: "${{ vars.AZURE_TENANT_ID }}", subscription-id: "${{ vars.AZURE_SUBSCRIPTION_ID }}" }
      - id: img
        run: |
          az acr build -r acrcontoso -t orders:${{ github.sha }} src/Orders
          echo "image=acrcontoso.azurecr.io/orders:${{ github.sha }}" >> $GITHUB_OUTPUT
  production:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    permissions: { id-token: write, contents: read }
    steps:
      - uses: azure/login@<sha>
        with: { client-id: "${{ vars.AZURE_CLIENT_ID }}", tenant-id: "${{ vars.AZURE_TENANT_ID }}", subscription-id: "${{ vars.AZURE_SUBSCRIPTION_ID }}" }
      - run: |
          az containerapp update -g rg-orders-prod -n ca-orders --image ${{ needs.build.outputs.image }} \
            --revision-suffix ${GITHUB_SHA::7}
          NEW=$(az containerapp revision list -g rg-orders-prod -n ca-orders --query "[0].name" -o tsv)
          az containerapp ingress traffic set -g rg-orders-prod -n ca-orders \
            --revision-weight latest=10 --label-weight stable=90
```

(Assumes multiple revision mode with a `stable` label on the current revision; promote to 100% after verification.)

App Service slot swap:

```bash
az webapp deploy -g rg-orders-prod -n app-orders-prod --slot staging --src-path app.zip --type zip
curl -fsS https://app-orders-prod-staging.azurewebsites.net/healthz
az webapp deployment slot swap -g rg-orders-prod -n app-orders-prod --slot staging --target-slot production
```

AKS with GitOps: CI updates the image tag in a Git config repo; the Flux extension reconciles the cluster, so cluster credentials never leave Azure and every change is a Git commit.

The **Azure Developer CLI** (`azd`) provides opinionated templates that generate both IaC and pipelines (`azd pipeline config`) for many app types.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Push vs GitOps for AKS** | GitOps for multi-cluster consistency and audit; push for simplicity on single clusters. |
| **Slots vs blue-green resources** | Slots for App Service; separate environments for big infra changes. |
| **Tag vs digest** | Deploy by digest in production. |
| **Database changes** | Backward-compatible migrations (expand/contract) so rollbacks work. |
| **Environment count** | Dev, staging, production at minimum; ephemeral PR environments where cheap. |
| **Configuration** | App Configuration and Key Vault references; no environment-specific builds. |

## Hands-on

1. Build a container image with `az acr build`, push and deploy to Container Apps in multiple revision mode with 10% canary.
2. Deploy an App Service app to a staging slot, run smoke tests and swap; swap back to roll back.
3. Set up Flux GitOps on AKS and deploy by committing an image tag change.
4. Add unit tests, image scanning and SBOM generation to CI.
5. Measure lead time and deployment frequency for a week.

## Security, cost and reliability notes

- **Pull with managed identity**; disable ACR admin user; restrict network access to ACR.
- **Secrets**: never baked into images; Key Vault references or workload identity.
- **Rollback readiness**: test rollback paths regularly.
- **Cost**: slots and canary revisions run extra instances; clean up old revisions and preview environments.

## Common mistakes

| Mistake | Fix |
|---|---|
| Rebuilding per environment | Build once, promote artefact |
| Deploying `latest` tags | Commit SHA tags and digests |
| No post-deploy verification | Smoke tests and health checks |
| Breaking schema changes with app deploy | Expand/contract migrations |
| Cluster admin credentials in CI | GitOps or OIDC with scoped RBAC |

## Check yourself

1. What does "build once, deploy many" mean?
2. How do App Service slots enable safe rollout and rollback?
3. How does Container Apps support canary releases?
4. What are the options for deploying to AKS?
5. Why deploy by image digest?
6. How do you make database changes rollback-safe?

<details>
<summary>Answers</summary>

1. Produce one immutable artefact in CI and promote it unchanged through environments, varying only configuration.
2. Deploy to a staging slot, warm up and verify, swap into production; swap back to roll back; traffic routing for gradual exposure.
3. Revisions with traffic splitting and labels in multiple revision mode.
4. kubectl/Helm/Kustomize from pipelines, AKS automated deployments, or GitOps with Flux or Argo CD.
5. Digests are immutable; tags can be moved.
6. Expand/contract: add compatible schema first, deploy code, remove old schema later.

</details>

## References

- [Deploy to App Service with GitHub Actions](https://learn.microsoft.com/azure/app-service/deploy-github-actions)
- [Deployment slots](https://learn.microsoft.com/azure/app-service/deploy-staging-slots)
- [Container Apps with GitHub Actions](https://learn.microsoft.com/azure/container-apps/github-actions)
- [Container Apps revisions](https://learn.microsoft.com/azure/container-apps/revisions)
- [AKS automated deployments](https://learn.microsoft.com/azure/aks/automated-deployments)
- [Azure Developer CLI](https://learn.microsoft.com/azure/developer/azure-developer-cli/overview)
