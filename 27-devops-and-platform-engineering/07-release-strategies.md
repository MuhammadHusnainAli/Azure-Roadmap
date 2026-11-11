# Release Strategies

## Why it matters

Most production incidents are caused by change. You can't stop changing, so you limit the blast radius of each change: expose it to a small group first, watch the signals, and be able to undo it in seconds. Release strategies (**blue-green**, **canary**, **rolling**, **ring-based progressive exposure**) and **feature flags** separate *deploying* code from *releasing* functionality, turning risky "big bang" releases into routine, reversible steps. Azure services provide the mechanics at every layer: App Service slots, Container Apps revisions, AKS rollouts, Front Door and Traffic Manager routing, API Management, and Azure App Configuration feature flags.

## Key concepts

### Strategies

| Strategy | How | Rollback | Cost |
|---|---|---|---|
| **Recreate** | Stop old, start new | Redeploy old | Lowest; downtime |
| **Rolling** | Replace instances gradually (AKS default, VMSS) | Roll back the deployment | Low; mixed versions during rollout |
| **Blue-green** | Two full environments; switch traffic at once | Switch back instantly | Double capacity during release |
| **Canary** | Small % of traffic to new version; increase on healthy signals | Shift traffic back | Small extra capacity |
| **Ring-based / progressive exposure** | Internal users → early adopters → regions → everyone | Halt rings | Process overhead |
| **Dark launch / shadow** | New version receives mirrored traffic but responses aren't used | N/A | Extra capacity |
| **A/B testing** | Variants for experimentation on user behaviour | Turn off variant | Analytics needed |

### Feature flags

Feature flags decouple deployment from release. **Azure App Configuration feature management** supports:

| Scenario | Capabilities |
|---|---|
| **Switch** | Instant on/off kill switch; operational toggles (maintenance mode) |
| **Rollout** | Percentage-based gradual release; targeting users, groups or attributes (region, tier); time windows; configuration delivered with the flag |
| **Experiment** | Variants with telemetry to Application Insights for A/B experiments |

Flags need hygiene: owners, expiry dates and removal after full rollout, otherwise they become technical debt.

### Azure mechanics by layer

| Layer | Mechanism |
|---|---|
| **App Service** | Deployment slots, swap with preview, traffic routing percentages to slots |
| **Container Apps** | Revisions with labels and traffic weights |
| **AKS** | Rolling updates; canary with ingress/Gateway API weights, service mesh (Istio add-on), Argo Rollouts or Flagger for automated analysis |
| **Global routing** | Front Door origin group weights; Traffic Manager weighted routing for cross-region blue-green |
| **APIs** | API Management revisions and versions; route by header or percentage |
| **AI agents and models** | Gateway-based traffic split between agent versions or model deployments ([file 06](06-ci-cd-for-ai-applications-and-agents.md)) |
| **Infrastructure** | Deployment stamps; deploy region by region (safe deployment practices) |

### Safe deployment practices

Microsoft deploys Azure itself in stages (canary regions, then paired regions gradually) with health monitoring between stages. Apply the same idea: define stages, **bake time**, health criteria (error rate, latency, saturation, business KPIs) and automatic halt/rollback.

## How it works on Azure

Feature flag in Python with App Configuration (Entra auth):

```python
from azure.appconfiguration.provider import load
from azure.identity import DefaultAzureCredential
from featuremanagement import FeatureManager, TargetingContext

config = load(endpoint="https://appcs-contoso.azconfig.io", credential=DefaultAzureCredential(),
              feature_flag_enabled=True, feature_flag_refresh_enabled=True)
features = FeatureManager(config)

def checkout(user_id: str, groups: list[str]):
    if features.is_enabled("EnableNewCheckout", TargetingContext(user_id=user_id, groups=groups)):
        return new_checkout()
    return legacy_checkout()
```

Flag definition (targeting 10% plus a beta group):

```json
{
  "id": "EnableNewCheckout",
  "enabled": true,
  "conditions": {
    "client_filters": [ {
      "name": "Microsoft.Targeting",
      "parameters": { "Audience": { "Groups": [ { "Name": "beta-testers", "RolloutPercentage": 100 } ], "DefaultRolloutPercentage": 10 } }
    } ]
  }
}
```

Automated canary progression in a pipeline (Container Apps):

```text
deploy new revision → 5% traffic → wait 15 min → query App Insights: error rate < 1% and p95 < SLO?
  yes → 25% → wait → 50% → wait → 100% and relabel stable
  no  → set new revision weight 0, alert owners, fail pipeline
```

Cross-region blue-green with Front Door: deploy to the green stamp, smoke test via its own endpoint, shift origin weights 0→100, keep blue for rollback until bake time passes.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Default strategy** | Canary or progressive exposure for user-facing services; blue-green for big changes needing instant switch-back. |
| **Flags vs deployment strategies** | Use both: deployment strategies limit risk of new binaries; flags control feature exposure. |
| **Health criteria** | Automate decisions with SLO-based metrics, not manual eyeballing. |
| **Stateful changes** | Database migrations must be backward-compatible across versions running side by side. |
| **Flag lifecycle** | Owner, expiry and clean-up policy; track flag count. |
| **Session affinity** | Consider sticky routing during canaries for consistent user experience. |

## Hands-on

1. Add a feature flag with targeting to an app via App Configuration; roll from 0% to 100%.
2. Run a Container Apps canary with automated health checks against Application Insights.
3. Use App Service traffic routing to send 10% to a staging slot.
4. Implement blue-green across two stamps behind Front Door.
5. Define flag hygiene rules and remove a fully rolled-out flag.

## Security, cost and reliability notes

- **Flag configuration is production config**: restrict who can change flags; audit changes.
- **Kill switches** need to work during incidents: test them.
- **Cost**: blue-green doubles capacity briefly; canaries need little extra.
- **Observability**: tag telemetry with version/revision to compare cohorts.

## Common mistakes

| Mistake | Fix |
|---|---|
| Big-bang releases | Canary or rings |
| Manual canary decisions | Automated health gates |
| Flags never removed | Expiry and clean-up |
| Incompatible schema between versions | Expand/contract |
| No version tag in telemetry | Add version/revision attributes |

## Check yourself

1. Compare blue-green, canary and rolling deployments.
2. What do feature flags decouple, and what are the three App Configuration scenarios?
3. How do you run a canary on Container Apps?
4. What are safe deployment practices?
5. What must be true of database changes during canaries?
6. How do you prevent feature flag debt?

<details>
<summary>Answers</summary>

1. Blue-green: two environments, instant switch; canary: small traffic share increased on health; rolling: instances replaced gradually.
2. Deployment from release; Switch, Rollout and Experiment.
3. Multiple revision mode with labels and traffic weights, increasing weight on healthy signals.
4. Staged rollout with bake time, health criteria and automatic halt/rollback between stages.
5. Backward compatible so old and new versions work simultaneously.
6. Owners, expiry dates, tracking and removal after full rollout.

</details>

## References

- [Feature management overview (App Configuration)](https://learn.microsoft.com/azure/azure-app-configuration/concept-feature-management)
- [Deployment slots](https://learn.microsoft.com/azure/app-service/deploy-staging-slots)
- [Container Apps revisions and traffic splitting](https://learn.microsoft.com/azure/container-apps/revisions)
- [Safe deployment practices (Well-Architected)](https://learn.microsoft.com/azure/well-architected/operational-excellence/safe-deployments)
