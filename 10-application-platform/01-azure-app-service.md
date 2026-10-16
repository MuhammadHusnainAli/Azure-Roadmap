# Azure App Service

> Topic: 10 · Application Platform · Tracks: E ● A ● C ○ · Status checked: 2026-10

## Why it matters

Azure App Service is Azure's fully managed platform for web apps and APIs. It runs code (.NET, Java, Node.js, Python, PHP) or containers on Windows and Linux, with built-in scaling, deployment slots, TLS, authentication and monitoring. A large share of enterprise web workloads on Azure run on it, including many AI chat front ends and APIs. It is often the simplest host for a web application, and the easiest landing spot when modernising an IIS or Tomcat estate.

## Key concepts

### Plans and apps

```text
App Service plan  (region, OS, pricing tier, instance size, instance count)
 ├─ Web app "chat-ui"        ← shares the plan's VMs
 ├─ Web app "chat-api"
 │    ├─ slot: production
 │    └─ slot: staging       ← separate app instance on the same plan
 └─ Function app (Dedicated)  ← Functions can also run on an App Service plan
```

The **plan** is the compute you pay for; **apps** run on all instances of their plan. Multiple apps on one plan share CPU and memory.

### Pricing tiers

| Tier | Purpose | Key features |
|---|---|---|
| **Free (F1), Shared (D1)** | Trials | Shared infrastructure, quotas, no SLA |
| **Basic (B1–B3)** | Dev and test, low-traffic production | Dedicated VMs, manual scale (few instances), custom domains and TLS |
| **Standard (S1–S3)** | Legacy production tier | Autoscale, slots, daily backups |
| **Premium v3 (P0v3–P5mv3)** | **Production default** | Faster processors, more memory options, zone redundancy, more slots, more instances, VNet integration, automatic scaling |
| **Premium v4** | Newest generation | Newer hardware and better price-performance where available (check regional availability) |
| **Isolated v2 (I1v2–I6v2)** | **App Service Environment v3** | Single-tenant, fully network-isolated deployment, very high scale |

Use Premium v3 or v4 for production. Standard is effectively superseded on price-performance.

### Runtime options

| Option | Notes |
|---|---|
| Code on Linux | Built-in stacks for .NET, Java, Node.js, Python, PHP; build on deploy (Oryx) |
| Code on Windows | .NET Framework, .NET, Node.js, Java, PHP; IIS-based |
| **Custom containers** | Linux or Windows containers from ACR (pulled with managed identity) |
| **Sidecars** | Additional containers alongside the main app (for example OpenTelemetry collectors, a local small model or cache); available for Linux apps |
| **Managed instance on App Service** | Windows apps with OS-level dependencies (registry, COM components, MSIs, fonts) configured through install scripts, without rewriting; aimed at lift-and-shift of legacy .NET Framework apps (check current status) |

### Scaling

| Type | How |
|---|---|
| Scale up | Change plan tier or size |
| Scale out (manual) | Set instance count |
| **Autoscale** (rules) | Azure Monitor rules on CPU, memory, HTTP queue, schedules |
| **Automatic scaling** (Premium) | Platform-managed, HTTP-driven scaling with pre-warmed instances and per-app maximums, no rules to write |

Apps run on every instance of the plan. Use **per-app scaling** to limit how many instances an individual app uses on a large shared plan.

### Deployment slots

Slots are live apps with their own host names, configuration and identities, running on the same plan.

| Feature | Detail |
|---|---|
| **Swap** | Warm up staging, then swap routing with production; near zero downtime |
| **Swap with preview** | Apply production settings to staging first, validate, then complete |
| **Slot-specific settings** | Mark settings as "deployment slot setting" so they stay with the slot (for example the staging database connection) |
| **Auto swap** | Swap automatically after deployment to a slot |
| **Testing in production** | Route a percentage of traffic to a slot |
| **Rollback** | Swap back |

### Deployment methods

| Method | Use |
|---|---|
| **GitHub Actions / Azure Pipelines** | Standard CI/CD (with OIDC federation, no secrets) |
| `az webapp up` / `az webapp deploy` | Quick deployments |
| **Zip deploy / run from package** | Deploy a zip that runs read-only from the package; consistent and fast |
| Container image | Deploy by updating the image tag or digest |
| `azd up` | Templates with infrastructure and code together |

### Other built-in features

- **Custom domains** and **free managed certificates** (or certificates from Key Vault).
- **Easy Auth** (file 02), **managed identities**, **Key Vault references** in app settings: `@Microsoft.KeyVault(SecretUri=...)`.
- **Health check** path: unhealthy instances are removed from rotation and replaced.
- **Always On**: keeps apps loaded (needed for continuous WebJobs and to avoid idle unloads).
- **Backups** and restore (with storage), **diagnostics** ("Diagnose and solve problems"), log streaming, Application Insights auto-instrumentation.
- **WebJobs**: background jobs on Windows (and Linux with containers); for new work prefer Functions or Container Apps jobs.

## How it works on Azure

### An AI chat app on App Service

```text
Front Door (WAF, custom domain)
  → App Service (Premium v3, zone redundant, 3 instances)
       chat-ui (Node.js) + chat-api (Python, FastAPI)
       managed identity → Foundry Models (keyless), Azure AI Search, Cosmos DB
       app settings: endpoints only; secrets via Key Vault references
       Application Insights with OpenTelemetry (GenAI traces)
```

Streaming responses with server-sent events work on App Service; keep request timeouts in mind (App Service's front ends time out idle requests after a few minutes; long agent tasks should be asynchronous, see file 04).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| App Service vs Container Apps | App Service for classic web apps and APIs, Windows workloads, teams preferring code deploy; Container Apps for microservices, event-driven workers, scale to zero and jobs |
| Plan sharing | Group apps with similar scaling and lifecycle on one plan; isolate noisy or critical apps |
| Tier | Premium v3/v4 for production with zone redundancy (minimum instance counts apply); Basic for dev |
| Code vs container | Code deploy is simpler; containers for dependency control and consistency with other hosts |
| Scaling | Automatic scaling for HTTP workloads; rules for specific metrics or schedules |
| Slots | Always use a staging slot for production apps |

## Hands-on

```bash
RG=rg-appsvc-lab; LOC=swedencentral; PLAN=plan-lab; APP=app-lab-$RANDOM
az group create -n $RG -l $LOC -o none

# 1. Linux Premium v3 plan and a Python app
az appservice plan create -g $RG -n $PLAN --is-linux --sku P0v3 -o none
az webapp create -g $RG -p $PLAN -n $APP --runtime "PYTHON:3.12" -o none

# 2. Deploy a minimal FastAPI app
mkdir app && cd app
cat > main.py <<'EOF'
import os
from fastapi import FastAPI
app = FastAPI()
@app.get("/")
def root(): return {"version": os.getenv("APP_VERSION", "v1")}
@app.get("/health")
def health(): return {"ok": True}
EOF
echo -e "fastapi\nuvicorn[standard]" > requirements.txt
az webapp config set -g $RG -n $APP --startup-file "uvicorn main:app --host 0.0.0.0 --port 8000" -o none
az webapp up -g $RG -n $APP --runtime "PYTHON:3.12" -o none
curl -s https://$APP.azurewebsites.net/

# 3. Health check and a staging slot
az webapp config set -g $RG -n $APP --generic-configurations '{"healthCheckPath": "/health"}' -o none
az webapp deployment slot create -g $RG -n $APP --slot staging --configuration-source $APP -o none
az webapp config appsettings set -g $RG -n $APP --slot staging --settings APP_VERSION=v2 -o none
curl -s https://$APP-staging.azurewebsites.net/

# 4. Swap and verify, then swap back
az webapp deployment slot swap -g $RG -n $APP --slot staging --target-slot production
curl -s https://$APP.azurewebsites.net/
az webapp deployment slot swap -g $RG -n $APP --slot staging --target-slot production

cd .. && az group delete -n $RG --yes --no-wait
```

Notice that `APP_VERSION` moved with the swap. Mark it as a slot setting (`--slot-settings`) and repeat to see the difference.

## Security, cost and reliability notes

- **Security:** HTTPS only, minimum TLS 1.2, FTP disabled, basic auth publishing credentials disabled (use Entra-based deployment), managed identity for all outbound access.
- **Security:** restrict inbound access (file 02); public apps behind Front Door with WAF.
- **Cost:** you pay for plan instances whether apps are busy or not. Consolidate small apps; scale in at night with schedules; reserved instances and savings plans for steady Premium plans.
- **Cost:** each slot uses plan capacity; idle slots still consume memory.
- **Reliability:** zone redundancy (with at least the minimum instance count), health checks, at least two instances, and Front Door across two regions for critical apps.

## Common mistakes

- Production on Basic or Standard tier without zone redundancy.
- Deploying straight to production instead of a slot.
- Secrets in app settings instead of Key Vault references.
- Too many unrelated apps on one plan, starving each other.
- Long-running synchronous requests that hit the front-end timeout.
- Leaving FTP and basic publishing credentials enabled.
- Forgetting slot-specific settings, so staging points at the production database after a swap (or vice versa).

## Check yourself

1. What is the relationship between an App Service plan and an app?
2. Which tier should production use, and what does Isolated v2 add?
3. How do deployment slots enable near zero-downtime releases?
4. What is a slot setting?
5. What does automatic scaling offer over rule-based autoscale?
6. When would you choose App Service over Container Apps?
7. What are sidecars on App Service used for?

<details>
<summary>Answers</summary>

1. The plan is the set of VMs (region, OS, tier, size, count) you pay for; apps run on every instance of their plan and share its resources.
2. Premium v3 or v4. Isolated v2 runs in an App Service Environment v3, giving single-tenant, fully network-isolated hosting at high scale.
3. You deploy to a staging slot, warm it up, then swap routing so it becomes production; rollback is another swap.
4. A setting marked to stay with its slot during swaps rather than moving with the app.
5. Platform-managed HTTP-based scaling with pre-warmed instances and per-app limits, without writing rules.
6. For classic web apps and APIs, Windows or .NET Framework workloads, and teams that want code-based deployment without containers.
7. Running additional containers alongside the main app, such as telemetry collectors, caches or small local models.

</details>

## References

- [App Service overview](https://learn.microsoft.com/azure/app-service/overview)
- [App Service plans](https://learn.microsoft.com/azure/app-service/overview-hosting-plans)
- [Deployment slots](https://learn.microsoft.com/azure/app-service/deploy-staging-slots)
- [Automatic scaling](https://learn.microsoft.com/azure/app-service/manage-automatic-scaling)
- [Sidecar containers](https://learn.microsoft.com/azure/app-service/overview-sidecar)
- [Managed instance on App Service](https://learn.microsoft.com/azure/app-service/overview-managed-instance)
- [Reliability in App Service](https://learn.microsoft.com/azure/reliability/reliability-app-service)
- [App Service baseline architecture](https://learn.microsoft.com/azure/architecture/web-apps/app-service/architectures/baseline-zone-redundant)
- [Baseline Azure AI Foundry chat reference architecture](https://learn.microsoft.com/azure/architecture/ai-ml/architecture/baseline-azure-ai-foundry-chat)
