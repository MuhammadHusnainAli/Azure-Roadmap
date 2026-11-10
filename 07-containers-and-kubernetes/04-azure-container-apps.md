# Azure Container Apps

## Why it matters

Azure Container Apps is a serverless platform for containers built on Kubernetes, KEDA, Dapr and Envoy, without exposing any of them to you. It covers most of what teams want from Kubernetes (microservices, autoscaling, scale to zero, revisions, jobs, private networking) with a fraction of the operational load. For most new APIs, background workers, AI applications and agents on Azure, it is the first container host to consider.

## Key concepts

### The resource model

```text
Container Apps environment   (secure boundary: one VNet, one Log Analytics workspace, shared DNS)
 ├─ Workload profiles: Consumption (serverless) + optional Dedicated profiles (D, E, GPU)
 ├─ Container app: chat-api
 │    ├─ Revision chat-api--v7   (immutable snapshot of template)  ← 80% traffic
 │    │     └─ Replicas 0 … N   (pods)  each with containers + sidecars + init containers
 │    └─ Revision chat-api--v8                                   ← 20% traffic
 ├─ Container app: worker      (no ingress, scales on a queue)
 ├─ Job: nightly-indexer       (scheduled)
 └─ Dapr components, certificates, managed identities, storage mounts
```

| Concept | Meaning |
|---|---|
| **Environment** | Boundary for networking, logging and Dapr. Apps in the same environment reach each other by name |
| **Container app** | A long-running service with optional ingress |
| **Revision** | Immutable version created whenever the template (image, env vars, scale rules) changes |
| **Replica** | A running instance of a revision |
| **Job** | Containers that run to completion: manual, scheduled (cron) or event-driven |
| **Workload profile** | The compute an app runs on: serverless Consumption, or Dedicated VM-backed profiles |

### Revisions and traffic

- **Single revision mode** (default): a new revision replaces the old one once healthy.
- **Multiple revision mode**: several revisions run together; you split traffic by percentage or address each by a **label** (`https://chat-api---canary.<env-domain>`). Use for blue-green and canary releases.

Revision-scope changes (template) create a new revision. Application-scope changes (secrets, ingress, registry credentials) do not, though a restart may be needed to pick up a secret.

### Scaling with KEDA

Each app defines `minReplicas`, `maxReplicas` and **scale rules**:

| Rule type | Scales on |
|---|---|
| HTTP | Concurrent requests per replica |
| TCP | Concurrent connections |
| Azure Queue, Service Bus, Event Hubs, Kafka, Redis, Cosmos DB, and other KEDA scalers | Queue length, lag, or similar metric |
| CPU / memory | Utilisation (cannot scale to zero) |
| Cron | Time windows |

With `minReplicas: 0`, an app scales to zero and you pay nothing for idle Consumption compute. The first request after idle pays a **cold start** (image pull plus app start). Keep `minReplicas: 1` for latency-sensitive APIs. Scale rules can authenticate to Azure sources with **managed identity**.

### Dapr

Dapr (Distributed Application Runtime) runs as a managed sidecar and gives apps HTTP or gRPC APIs for common distributed concerns, backed by pluggable components:

| Building block | Example backing service |
|---|---|
| Service invocation | Other apps, with retries and mTLS |
| State management | Cosmos DB, Redis, Table Storage |
| Pub/sub | Service Bus, Event Hubs, Redis |
| Bindings | Blob, queues, cron, external systems |
| Secrets | Key Vault |
| Workflows | Durable, code-first workflows |

Dapr is optional. It helps polyglot microservice teams; it adds a dependency that simple apps do not need. Dapr also offers agent-oriented capabilities (Dapr Agents) built on its workflow and state primitives.

### Jobs

| Trigger | Use |
|---|---|
| **Manual** | On-demand runs started by API, CLI or a pipeline |
| **Schedule** | Cron expression, for example nightly re-indexing |
| **Event** | KEDA-driven: start one execution per batch of queue messages |

Jobs support parallelism, replica completion counts, retries and timeouts. They replace many uses of ACI, WebJobs and cron VMs.

### Workload profiles

| Profile | Billing | Use |
|---|---|---|
| **Consumption** | Per second of vCPU and memory used, with a monthly free grant; scale to zero | Default for most apps |
| **Dedicated (D-series general purpose, E-series memory optimised)** | Per node per hour | Steady load, larger replicas, predictable cost, isolation |
| **Dedicated GPU (for example NC A100, T4 profiles)** | Per node per hour | Steady GPU inference |
| **Consumption GPU (serverless GPU)** | Per second of GPU use, scale to zero | Bursty GPU inference (file 05) |

An environment with workload profiles can mix Consumption and Dedicated apps. Older "Consumption-only" environments still exist but new deployments should use workload profile environments.

## How it works on Azure

### Networking

| Option | Detail |
|---|---|
| External environment | Apps with external ingress get public endpoints |
| Internal environment | Environment gets a private IP; apps reachable only from the VNet or peered networks |
| Custom VNet | Bring your own subnet (delegated to `Microsoft.App/environments`; a /27 minimum for workload profile environments) |
| Private endpoint | Inbound private access to an environment; disable public network access |
| Ingress | Built-in Envoy: TLS termination, custom domains, managed certificates, IP restrictions, session affinity, CORS |
| Egress control | UDR to Azure Firewall or a NAT gateway for a static outbound IP (workload profile environments) |
| Premium ingress | Higher ingress scale and configuration for large environments |

### Identity and secrets

- System- or user-assigned **managed identity** per app; use it to pull from ACR, read Key Vault, call Foundry, Storage and databases.
- **Secrets** can reference Key Vault secrets by URI, resolved with the app's identity.
- **Built-in authentication** (Easy Auth) with Entra ID and other providers, without code.

### Observability

Console and system logs to Log Analytics (or Azure Monitor via OpenTelemetry), built-in metrics, the **OpenTelemetry agent** in the environment that forwards traces and metrics to Application Insights or other destinations, and Aspire dashboard support.

### Developer flow

`az containerapp up` (from source or image), `azd up` with templates, GitHub Actions, Visual Studio Code, and .NET Aspire all target Container Apps. Azure Functions can also run in a Container Apps environment with Functions triggers and scaling.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Container Apps or AKS | Container Apps unless you need Kubernetes APIs directly (CRDs, operators, custom controllers, service mesh control, specific node configuration) or very large, platform-team-run estates |
| Container Apps or App Service | Container Apps for microservices, event-driven workers, jobs and scale to zero; App Service for classic web apps, Windows apps and teams that do not want containers |
| Consumption or Dedicated | Consumption for spiky or low load; Dedicated when load is steady and cost per hour beats per-second billing, or for larger replicas |
| minReplicas | 0 for workers and internal tools; 1 or more for user-facing APIs; at least 2 for production availability across zones |
| One environment or many | One per application boundary and environment stage; apps that call each other belong together |
| Dapr | Use when you need its building blocks across many services; skip for a single API |
| Zone redundancy | Enable at environment creation (it cannot be added later) with a suitably sized subnet |

## Hands-on

```bash
RG=rg-aca-lab; LOC=swedencentral; ENV=aca-env
az group create -n $RG -l $LOC -o none
az extension add -n containerapp --upgrade -y

# 1. Environment with workload profiles
az containerapp env create -g $RG -n $ENV -l $LOC --enable-workload-profiles -o none

# 2. An app with external ingress that scales to zero on HTTP
az containerapp create -g $RG -n hello --environment $ENV \
  --image mcr.microsoft.com/k8se/quickstart:latest \
  --ingress external --target-port 80 \
  --min-replicas 0 --max-replicas 5 \
  --scale-rule-name http --scale-rule-type http --scale-rule-http-concurrency 20 -o none
URL=https://$(az containerapp show -g $RG -n hello --query properties.configuration.ingress.fqdn -o tsv)
curl -s $URL | head -5

# 3. Load it and watch replicas scale out, then back to zero after a few minutes
for i in $(seq 1 300); do curl -s -o /dev/null $URL & done; wait
az containerapp replica list -g $RG -n hello -o table

# 4. Canary: multiple revision mode and a traffic split
az containerapp revision set-mode -g $RG -n hello --mode multiple
az containerapp update -g $RG -n hello --set-env-vars VERSION=v2 --revision-suffix v2 -o none
az containerapp revision list -g $RG -n hello --query "[].name" -o tsv
LATEST=$(az containerapp revision list -g $RG -n hello --query "[-1].name" -o tsv)
FIRST=$(az containerapp revision list -g $RG -n hello --query "[0].name" -o tsv)
az containerapp ingress traffic set -g $RG -n hello --revision-weight $FIRST=80 $LATEST=20

# 5. A scheduled job
az containerapp job create -g $RG -n nightly --environment $ENV \
  --trigger-type Schedule --cron-expression "0 2 * * *" \
  --image mcr.microsoft.com/k8se/quickstart-jobs:latest \
  --cpu 0.25 --memory 0.5Gi --replica-timeout 600 --replica-retry-limit 1 -o none
az containerapp job start -g $RG -n nightly
az containerapp job execution list -g $RG -n nightly -o table

az group delete -n $RG --yes --no-wait
```

Extension: give the app a system-assigned identity, grant it `AcrPull` on your registry from file 02, and deploy your own image with `--registry-identity system`.

## Security, cost and reliability notes

- **Security:** use internal environments with private endpoints for anything not meant for the internet, and front public apps with Front Door or Application Gateway with WAF.
- **Security:** managed identities for registries and data; Key Vault references for secrets; Easy Auth or your own auth on every endpoint.
- **Security:** Consumption replicas run in a Microsoft-managed, Hyper-V isolated sandbox. Dedicated profiles give you dedicated nodes.
- **Cost:** scale to zero is the main saving. Watch `minReplicas` creep and idle Dedicated nodes. Each environment with workload profiles has a small fixed management charge once Dedicated profiles are used.
- **Reliability:** zone redundancy must be chosen when the environment is created. Run at least two replicas for production and configure health probes (startup, liveness, readiness).
- **Reliability:** cold starts after scale to zero can take seconds; large images make them longer.

## Common mistakes

- Choosing AKS by default when Container Apps would do, then paying for a platform team.
- Scale to zero on a user-facing API with strict latency requirements.
- CPU-based scaling on a queue worker that should scale on queue length.
- Forgetting that environments are VNet boundaries: putting apps that must talk privately in separate environments.
- Creating an environment without zone redundancy and finding it cannot be added later.
- A subnet too small to grow.
- Storing secrets as plain environment variables.

## Check yourself

1. What is the difference between a revision and a replica?
2. What happens when you change an environment variable on a container app?
3. How do you run a canary release on Container Apps?
4. Which scale rule would you use for a Service Bus worker, and can it scale to zero?
5. When would you choose a Dedicated workload profile over Consumption?
6. Name four reasons to choose AKS over Container Apps.
7. What are the three job trigger types?

<details>
<summary>Answers</summary>

1. A revision is an immutable version of the app's template; replicas are the running instances of a revision.
2. It is a revision-scope change, so a new revision is created.
3. Switch to multiple revision mode, deploy a new revision, then split traffic by weight or by label.
4. A KEDA Azure Service Bus rule on queue or subscription message count, authenticated by managed identity. Yes, with `minReplicas: 0`.
5. For steady load where per-node pricing is cheaper, for larger replicas than Consumption allows, for dedicated isolation, or for steady GPU workloads.
6. Need for Kubernetes APIs, CRDs and operators; custom node configuration; full control of the service mesh and networking; very large multi-team platforms; portability of Kubernetes manifests and tooling.
7. Manual, schedule and event.

</details>

## References

- [Azure Container Apps documentation](https://learn.microsoft.com/azure/container-apps/)
- [Revisions](https://learn.microsoft.com/azure/container-apps/revisions)
- [Scaling with KEDA rules](https://learn.microsoft.com/azure/container-apps/scale-app)
- [Workload profiles](https://learn.microsoft.com/azure/container-apps/workload-profiles-overview)
- [Jobs](https://learn.microsoft.com/azure/container-apps/jobs)
- [Dapr integration](https://learn.microsoft.com/azure/container-apps/dapr-overview)
- [Networking](https://learn.microsoft.com/azure/container-apps/networking)
- [Reliability in Azure Container Apps](https://learn.microsoft.com/azure/reliability/reliability-azure-container-apps)
- [Comparing Container Apps with other options](https://learn.microsoft.com/azure/container-apps/compare-options)
