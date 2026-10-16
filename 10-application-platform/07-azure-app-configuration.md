# Azure App Configuration

> Topic: 10 · Application Platform · Tracks: E ● A ○ C – · Status checked: 2026-10

## Why it matters

As applications grow into many services and environments, configuration scattered across app settings, environment variables and files becomes inconsistent and risky to change. Azure App Configuration centralises settings and feature flags, lets you change them without redeploying, and references secrets in Key Vault. For AI applications it also manages **AI configuration**: model deployments, prompts and parameters that teams want to tune or roll out gradually without shipping new code.

## Key concepts

### Key-values

| Element | Detail |
|---|---|
| **Key** | Hierarchical by convention: `ChatApi:Search:IndexName` or `chat-api/search/index-name` |
| **Value** | String; content type can mark JSON, Key Vault references, feature flags or AI configuration |
| **Label** | A dimension such as environment (`dev`, `prod`) or version; the same key can have different values per label |
| **Tags** | Metadata for filtering |
| **Revisions** | History of changes; point-in-time read and restore |
| **Snapshots** | Immutable, named sets of key-values for safe releases and rollbacks |
| **Key Vault references** | Values that point to Key Vault secrets; the app resolves them with its own identity |

Apps load configuration at start and can **refresh dynamically** (polling a sentinel key or using change events) so changes apply without restart.

### Feature flags

| Concept | Detail |
|---|---|
| Flag | On or off, with a description |
| **Feature filters** | Conditional activation: percentage of users, time window, targeting (users, groups, with rollout percentages), custom filters |
| **Variant feature flags** | Flags that return variants (for example A/B/C configurations), with allocation by percentage or audience |
| **Telemetry** | Flag evaluation events to Application Insights for experimentation analysis |
| Libraries | Microsoft.FeatureManagement (.NET), feature management libraries for Python, JavaScript, Java, Go |

Feature flags separate **deployment** from **release**: ship code dark, enable for internal users, then 10%, then everyone, and turn it off instantly if needed.

### AI configuration

App Configuration supports configuration designed for AI apps (check current status and portal experience):

| Use | Example |
|---|---|
| Model selection | `ChatApi:Model:Deployment = gpt-x-mini` per label or variant |
| Parameters | Temperature, max tokens, top-p |
| Prompts | System prompts as configuration values, versioned through revisions and snapshots |
| Experiments | **Variant feature flags** to send 10% of users to a new prompt or model, with telemetry to compare quality, latency and cost |
| Kill switch | Flag to disable a tool or feature if it misbehaves |

This lets prompt and model changes follow the same controlled rollout and rollback as code, without redeployment. Pair it with evaluation (topic 22) before widening a rollout.

### Tiers

| Tier | Fit |
|---|---|
| Free | Learning; limited requests and storage, no SLA |
| Developer | Development use with more capacity (check current tiers) |
| **Standard** | Production; SLA, private endpoints, geo-replication, higher limits |
| **Premium** | Higher throughput, larger storage, availability zones and higher SLA (check current tiers) |

### Integrations

- **Kubernetes provider**: syncs configuration and Key Vault references into ConfigMaps and Secrets on AKS.
- **Container Apps, App Service, Functions**: load via SDK providers or App Service app settings references (`@Microsoft.AppConfiguration(Endpoint=...; Key=...)`).
- **Geo-replication**: replicas in other regions for resilience and lower latency.
- **Event Grid** events on key-value changes.
- **Import and export** from files, other stores, and CI/CD pipelines (configuration as code).

## How it works on Azure

### Python: load configuration, Key Vault references and feature flags

```python
from azure.identity import DefaultAzureCredential
from azure.appconfiguration.provider import load, SettingSelector, WatchKey
from featuremanagement import FeatureManager

cred = DefaultAzureCredential()
config = load(
    endpoint="https://<store>.azconfig.io",
    credential=cred,
    selects=[SettingSelector(key_filter="ChatApi:*", label_filter="prod")],
    trim_prefixes=["ChatApi:"],
    keyvault_credential=cred,                 # resolves Key Vault references
    feature_flag_enabled=True,
    refresh_on=[WatchKey("ChatApi:Sentinel", label="prod")],
    refresh_interval=30,
)

features = FeatureManager(config)

def handle(user_id: str, prompt: str):
    config.refresh()                          # cheap; respects refresh_interval
    model = config["Model:Deployment"]
    if features.is_enabled("NewRetrieval", user_id):
        ...
```

The identity needs **App Configuration Data Reader** on the store (and Key Vault secret access for references). Disable access keys (`disableLocalAuth`) and use Entra only.

### Layout conventions

```text
ChatApi:Model:Deployment          label=dev   → small model
ChatApi:Model:Deployment          label=prod  → production model
ChatApi:Model:Temperature         label=prod  → 0.2
ChatApi:Prompt:System             label=prod  → "You are …"   (content type text/plain)
ChatApi:Search:Endpoint           label=prod  → https://…search.windows.net
ChatApi:Storage:ConnectionSecret  label=prod  → Key Vault reference
ChatApi:Sentinel                  label=prod  → bump to trigger refresh
.appconfig.featureflag/NewRetrieval            → targeting: internal group 100%, others 10%
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| App Configuration vs app settings | App settings for a few per-app values; App Configuration for shared, multi-environment, dynamically changed configuration and flags |
| Secrets | In Key Vault, referenced from App Configuration; never as plain values |
| Labels vs separate stores | Labels for environments in non-production; separate stores for production isolation and access control |
| Refresh | Sentinel key or change events; avoid polling every key frequently |
| Prompt management | Prompts as configuration with snapshots for release, or in source control for review; many teams combine both (author in Git, deploy to App Configuration) |
| Experiments | Variant flags plus telemetry and offline evaluation |

## Hands-on

```bash
RG=rg-appconfig-lab; LOC=swedencentral; STORE=appcs-lab-$RANDOM
az group create -n $RG -l $LOC -o none
az appconfig create -g $RG -n $STORE -l $LOC --sku Standard --disable-local-auth true -o none
ME=$(az ad signed-in-user show --query id -o tsv)
az role assignment create --assignee $ME --role "App Configuration Data Owner" \
  --scope $(az appconfig show -g $RG -n $STORE --query id -o tsv) -o none

# Key-values with labels
az appconfig kv set -n $STORE --auth-mode login --key "ChatApi:Model:Deployment" --value "small-model" --label dev -y
az appconfig kv set -n $STORE --auth-mode login --key "ChatApi:Model:Deployment" --value "large-model" --label prod -y
az appconfig kv set -n $STORE --auth-mode login --key "ChatApi:Sentinel" --value "1" --label prod -y

# Feature flag
az appconfig feature set -n $STORE --auth-mode login --feature NewRetrieval -y
az appconfig feature filter add -n $STORE --auth-mode login --feature NewRetrieval \
  --filter-name Microsoft.Targeting --filter-parameters Audience='{"Users":["alex@contoso.com"],"DefaultRolloutPercentage":10}' -y

az appconfig kv list -n $STORE --auth-mode login --label prod -o table

# Snapshot for a release
az appconfig snapshot create -n $STORE --auth-mode login --snapshot-name release-2026-10-15 \
  --filters '{"key":"ChatApi:*","label":"prod"}'

az group delete -n $RG --yes --no-wait
```

Then run the Python sample, change the model value, bump the sentinel, and watch the app pick it up without restarting.

## Security, cost and reliability notes

- **Security:** Entra authentication with local auth disabled; private endpoints for production; separate stores or RBAC for production.
- **Security:** prompts and model settings are part of your security posture (they can weaken safety behaviour); restrict who can change production values and audit changes.
- **Cost:** request-based quotas per tier; cache configuration in the app and refresh with a sentinel rather than reading on every request.
- **Reliability:** apps should start with cached or default values if the store is unavailable; use geo-replicas for multi-region apps.

## Common mistakes

- Storing secrets as plain key-values.
- Reading configuration from the store on every request.
- Feature flags that are never removed, accumulating dead code.
- Changing production prompts or models with no rollout, evaluation or rollback plan.
- Access keys in connection strings instead of Entra auth.

## Check yourself

1. What are labels used for?
2. How does dynamic refresh with a sentinel key work?
3. What is a Key Vault reference in App Configuration, and who resolves it?
4. How do variant feature flags support AI experimentation?
5. What are snapshots for?
6. Why should prompt and model changes be governed like code changes?

<details>
<summary>Answers</summary>

1. To hold different values for the same key across environments, versions or other dimensions.
2. The app watches one sentinel key; when its value changes, the app reloads all selected configuration, so a set of changes applies together.
3. A value pointing to a Key Vault secret URI; the application resolves it using its own identity and Key Vault permissions.
4. They allocate users to different configurations (prompts, models, parameters) by percentage or audience, with telemetry to compare outcomes.
5. Immutable, named sets of configuration that can be referenced for releases and rolled back to.
6. They change application behaviour, quality, safety and cost as much as code does, so they need review, staged rollout, evaluation and rollback.

</details>

## References

- [Azure App Configuration overview](https://learn.microsoft.com/azure/azure-app-configuration/overview)
- [Key-values and labels](https://learn.microsoft.com/azure/azure-app-configuration/concept-key-value)
- [Feature management](https://learn.microsoft.com/azure/azure-app-configuration/feature-management-overview)
- [Variant feature flags](https://learn.microsoft.com/azure/azure-app-configuration/concept-variant-feature-flags)
- [AI configuration](https://learn.microsoft.com/azure/azure-app-configuration/concept-ai-configuration)
- [Key Vault references](https://learn.microsoft.com/azure/azure-app-configuration/use-key-vault-references-dotnet-core)
- [Python configuration provider](https://learn.microsoft.com/azure/azure-app-configuration/reference-python-provider)
- [Snapshots](https://learn.microsoft.com/azure/azure-app-configuration/concept-snapshots)
