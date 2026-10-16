# Azure Container Registry

> Topic: 07 · Containers and Kubernetes · Tracks: E ● A ● C ○ · Status checked: 2026-10

## Why it matters

Every container deployment on Azure pulls from a registry, and Azure Container Registry (ACR) is the default private one. Its tier decides whether you can use private endpoints and geo-replication, its authentication model decides whether you leak credentials, and its build and import features decide how controlled your software supply chain is. Getting ACR right is a prerequisite for a secure AKS or Container Apps platform.

## Key concepts

### Registry, repository, artefact

```text
myregistry.azurecr.io                ← registry (login server)
  └─ apps/chat-api                   ← repository (namespaces with "/" are allowed)
       ├─ 1.4.2   → sha256:ab12…     ← tag → manifest
       ├─ 1.4.3   → sha256:cd34…
       └─ sha256:cd34… ← signature, SBOM (referrers)
  └─ helm/chat-api                   ← Helm chart as an OCI artefact
```

ACR is OCI-compliant, so it stores images, Helm charts, signatures, SBOMs and other artefacts.

### Service tiers

| Feature | Basic | Standard | Premium |
|---|---|---|---|
| Included storage and throughput | Lowest | Medium | Highest |
| Webhooks | Few | More | Most |
| **Geo-replication** | – | – | ✓ |
| **Private endpoints** and firewall rules | – | – | ✓ |
| Availability zones (zone redundancy) | ✓ in supported regions | ✓ | ✓ |
| Customer-managed keys | – | – | ✓ |
| Connected registry (on-premises or edge) | – | – | ✓ |
| Artifact streaming | – | – | ✓ |
| Repository-scoped tokens | ✓ | ✓ | ✓ |

Zone redundancy is now enabled by default in regions with availability zones, across tiers. Check current limits on the tier page; included storage and throughput have been raised over time. **Any production registry that must be private needs Premium.** Tier changes are in place and online.

### Authentication options

| Method | Use |
|---|---|
| **Microsoft Entra identity + Azure RBAC** | Default. Users via `az acr login`, workloads via managed identity |
| **AKS kubelet identity** with `AcrPull` | AKS nodes pull using the cluster's managed identity (`az aks update --attach-acr`) |
| **Container Apps / App Service managed identity** | Configure the registry with `identity: system` or a user-assigned identity |
| **Repository-scoped tokens** | Non-Entra clients (edge devices, external CI) with access to specific repositories |
| **Admin user** | A single shared username and password. Disable it |
| **Anonymous pull** | Public content only. Rarely appropriate |

Built-in data roles: `AcrPull`, `AcrPush`, `AcrDelete`, `AcrImageSigner`, plus `Container Registry Repository Reader/Writer/Contributor` roles for **attribute-based (ABAC) repository permissions**, which scope access to individual repositories. Check that ABAC is enabled on the registry before relying on repository-level roles.

### ACR Tasks

ACR Tasks builds and maintains images in Azure, with no local Docker.

| Task type | What it does |
|---|---|
| **Quick task** (`az acr build`) | Build and push a Dockerfile from local or a Git context, once |
| **Triggered task** | Rebuild on a Git commit, pull request or on a schedule |
| **Base image update trigger** | Rebuild app images automatically when their base image is patched |
| **Multi-step task** (YAML) | Build, test, push, run commands, as a mini pipeline |

Base image update triggers are the most valuable feature: when `python:3.12-slim` gets a security fix in your imported copy, every dependent image rebuilds.

### Importing and caching public content

- **`az acr import`** copies an image from Docker Hub, MCR or another registry into ACR, server-side.
- **Artifact cache** (cache rules) transparently caches upstream repositories such as Docker Hub or MCR on first pull, so workloads pull from ACR rather than the internet.

Workloads should never pull directly from public registries in production. Import or cache, scan, then use.

### Geo-replication

Premium registries replicate to other regions. Each replica has a regional endpoint behind the same login server, and clients are routed to the closest one. Benefits: faster, cheaper pulls (no cross-region egress), and resilience if one region fails. Each replica is billed as a Premium registry.

### Artifact streaming

Large images (GPU and AI images often exceed 5 GB) take minutes to pull. **Artifact streaming** (Premium, with AKS) lets a pod start before the full image has downloaded, by lazily loading layers on demand. It can cut start-up time for large images substantially. Enable it on the repository, and on the AKS node pool.

### Supply-chain features

| Feature | Purpose |
|---|---|
| **Signing with Notation (Notary Project)** | Sign images with keys in Azure Key Vault; verify at deploy time on AKS. Docker Content Trust is deprecated in ACR; use Notation for new work |
| **Defender for Containers** | Vulnerability assessment of images in ACR and running in clusters |
| **Soft delete and retention policies** | Recover deleted artefacts; purge untagged manifests automatically |
| **Locking** (`--write-enabled false`) | Make a tag or repository immutable |
| **Purge task** (`acr purge`) | Scheduled clean-up of old tags |

## How it works on Azure

### Network access

| Setting | Effect |
|---|---|
| Public network access enabled, no rules | Reachable from anywhere (authentication still required) |
| Firewall rules (Premium) | Allow listed public IP ranges |
| **Private endpoint** (Premium) | Private IP in your VNet; disable public access. Needs DNS zone `privatelink.azurecr.io`, and one record per **data endpoint** per replica region |
| **Dedicated data endpoints** | Region-specific data endpoint names, so firewalls can allow precise FQDNs |
| Trusted services bypass | Lets services such as ACR Tasks or Defender reach a locked-down registry |

A common trap: the login server works through the private endpoint but image layer downloads fail because the data endpoint (`myregistry.<region>.data.azurecr.io`) does not resolve privately.

### Typical platform pattern

```text
Public sources (Docker Hub, MCR)
   │ artifact cache / az acr import (scheduled)
   ▼
ACR Premium  (private endpoint, public access disabled, zone redundant, geo-replicated to DR region)
   │  ├─ base images  ← ACR Task rebuilds on upstream patch
   │  └─ app images   ← CI pushes with AcrPush (workload identity federation, no secrets)
   │  Notation signatures, SBOMs, Defender scans attached
   ▼
AKS / Container Apps pull with AcrPull managed identity; admission policy verifies signatures
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| One registry or many | One per environment boundary is common (non-prod and prod). Fewer registries mean simpler governance; separate prod for blast radius |
| Tier | Premium for any private or multi-region production estate; Basic for labs |
| Geo-replication | Replicate to every region where you run clusters, and to your DR region |
| Who can push | CI identities only, never humans, for production repositories |
| Retention | Purge untagged manifests and old dev tags; keep release tags per your compliance policy |
| Signing | Sign in CI, verify at admission. Start with audit mode, then enforce |

## Hands-on

```bash
RG=rg-acr-lab; LOC=swedencentral; ACR=acrlab$RANDOM
az group create -n $RG -l $LOC -o none
az acr create -g $RG -n $ACR --sku Standard --admin-enabled false -o none

# 1. Build in the cloud: no local Docker needed
az acr build -r $ACR -t demo/hello:{{.Run.ID}} -t demo/hello:1.0.0 \
  https://github.com/Azure-Samples/aci-helloworld.git

# 2. Import a public base image server-side
az acr import -n $ACR --source mcr.microsoft.com/azurelinux/base/python:3.12 --image base/python:3.12

# 3. Inspect repositories, tags and the digest
az acr repository list -n $ACR -o table
az acr repository show-tags -n $ACR --repository demo/hello -o table
az acr manifest list-metadata -r $ACR -n demo/hello --query "[].{digest:digest, tags:tags}" -o table

# 4. A base-image-triggered task (rebuilds when base/python changes in this registry)
az acr task create -r $ACR -n rebuild-hello \
  --context https://github.com/Azure-Samples/aci-helloworld.git --file Dockerfile \
  --image demo/hello:{{.Run.ID}} --base-image-trigger-enabled true --commit-trigger-enabled false

# 5. Lock a release tag
az acr repository update -n $ACR --image demo/hello:1.0.0 --write-enabled false

# 6. Scheduled purge of untagged manifests older than 7 days
az acr task create -r $ACR -n purge --cmd "acr purge --filter 'demo/.*:.*' --ago 7d --untagged" \
  --schedule "0 3 * * *" --context /dev/null

az group delete -n $RG --yes --no-wait
```

Then try granting a managed identity `AcrPull` and pulling from Container Apps in file 04.

## Security, cost and reliability notes

- **Security:** disable the admin user and use Entra identities. Audit with Azure Policy (`Container registries should have local admin account disabled`).
- **Security:** Premium plus private endpoint plus public network access disabled for production. Remember the data endpoints in DNS.
- **Security:** restrict push rights to CI identities. A compromised push identity can poison every downstream deployment.
- **Cost:** storage grows silently. Use retention and purge tasks. Each geo-replica costs as much as another Premium registry.
- **Cost:** same-region pulls avoid egress charges; geo-replicate rather than pulling across regions.
- **Reliability:** zone redundancy for the primary; geo-replication so pulls continue during a regional outage.
- **Reliability:** registry throttling during mass scale-out is real for large images. Premium has the highest throughput; artifact streaming helps.

## Common mistakes

- Using the admin user, and then sharing its password in pipelines.
- Choosing Standard and later discovering private endpoints need Premium.
- Private endpoint configured without data endpoint DNS records.
- Pulling from Docker Hub in production.
- No retention: terabytes of untagged manifests.
- Overwriting release tags, breaking rollbacks.
- Pulling across regions for every node, paying egress and adding latency.

## Check yourself

1. Which ACR tier is required for private endpoints and geo-replication?
2. How should AKS nodes authenticate to ACR?
3. What does a base image update trigger do, and why is it valuable?
4. What is the difference between `az acr import` and an artifact cache rule?
5. Why can image pulls fail through a private endpoint even when `az acr login` works?
6. What does artifact streaming improve, and for which workloads?
7. What replaces Docker Content Trust for signing?

<details>
<summary>Answers</summary>

1. Premium.
2. With the cluster's kubelet managed identity granted `AcrPull` on the registry, typically via `az aks update --attach-acr`.
3. It rebuilds dependent application images automatically when their base image changes, so OS and runtime patches flow through without manual work.
4. Import is an explicit, server-side copy of specific images. A cache rule transparently pulls and stores upstream content on first request.
5. Layer downloads go to the regional data endpoint, which also needs a private DNS record; without it the client resolves the public address and is blocked.
6. Container start time for very large images, such as GPU and AI images, by loading layers on demand.
7. Notary Project signing with Notation, using keys and certificates in Azure Key Vault.

</details>

## References

- [Azure Container Registry documentation](https://learn.microsoft.com/azure/container-registry/)
- [ACR service tiers](https://learn.microsoft.com/azure/container-registry/container-registry-skus)
- [Authenticate with ACR](https://learn.microsoft.com/azure/container-registry/container-registry-authentication)
- [Automate builds with ACR Tasks](https://learn.microsoft.com/azure/container-registry/container-registry-tasks-overview)
- [Artifact cache](https://learn.microsoft.com/azure/container-registry/artifact-cache-overview)
- [Geo-replication](https://learn.microsoft.com/azure/container-registry/container-registry-geo-replication)
- [Artifact streaming](https://learn.microsoft.com/azure/container-registry/container-registry-artifact-streaming)
- [Private link for ACR](https://learn.microsoft.com/azure/container-registry/container-registry-private-link)
- [Sign images with Notation and Key Vault](https://learn.microsoft.com/azure/container-registry/container-registry-tutorial-sign-build-push)
