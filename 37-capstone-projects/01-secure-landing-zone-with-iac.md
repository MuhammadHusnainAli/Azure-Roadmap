# Secure Landing Zone with IaC

## Why it matters

Every serious Azure AI programme starts with a landing zone. Without one, teams deploy Foundry, Search and storage with public endpoints, scattered identities and no policy, and security reviews block go-live. In this capstone you build a governed **hub-spoke foundation** with Bicep or Terraform, deployed by CI/CD, ready to host an AI workload. It's the strongest portfolio piece for the **AI Solution Architect** track and the foundation for capstones 02–08.

**Brief:** Contoso Financial wants a secure Azure foundation for its first AI applications. All resources must be private, governed by policy, deployed from code and observable from day one.

## Key concepts

### Requirements

| # | Requirement | Type |
|---|---|---|
| R1 | Management group hierarchy: platform (connectivity, management, identity) and landing zones (corp, online), plus sandbox | Governance |
| R2 | Hub VNet with Azure Firewall, Bastion and private DNS zones; one spoke for an AI workload | Network |
| R3 | Policies: allowed regions, deny public network access for PaaS, require tags, deploy diagnostic settings, Defender for Cloud plans | Governance and security |
| R4 | Central Log Analytics workspace; activity logs and resource diagnostics routed there | Operations |
| R5 | RBAC with Entra groups; PIM for privileged roles; no standing Owner access | Identity |
| R6 | Budgets and cost alerts per subscription | Cost |
| R7 | Everything deployed from Git with pull request review and a `what-if` or `plan` step | DevOps |
| R8 | An AI spoke ready for Foundry, AI Search and Storage with private endpoints | AI readiness |

### Skills practised

Topics [02](../02-identity-and-access/README.md), [03](../03-governance-and-landing-zones/README.md), [04](../04-tooling-and-infrastructure-as-code/README.md), [05](../05-networking/README.md), [25](../25-security/README.md), [26](../26-monitoring-and-observability/README.md), [27](../27-devops-and-platform-engineering/README.md) and [29](../29-cost-management-and-finops/README.md).

## How it works on Azure

### Target architecture

```text
Tenant root
└── mg-contoso
    ├── mg-platform
    │   ├── sub-connectivity: hub VNet (AzureFirewallSubnet, AzureBastionSubnet, GatewaySubnet), private DNS zones
    │   ├── sub-management: Log Analytics, Automation, Defender for Cloud config
    │   └── sub-identity (optional)
    ├── mg-landingzones
    │   ├── mg-corp: sub-ai-prod: spoke VNet (snet-app, snet-pe, snet-agents), UDR → Firewall
    │   └── mg-online
    └── mg-sandbox
```

### Implementation options

| Option | When to use |
|---|---|
| **Azure landing zones accelerator** (Azure Verified Modules-based, Bicep or Terraform) | Recommended starting point; customise rather than rebuild |
| **Custom IaC using Azure Verified Modules** | Learning exercise or smaller estates; you own every decision |

For the portfolio, either deploy the accelerator and **document every decision and customisation**, or build a smaller custom version with AVM modules. The older `ALZ-Bicep` and `caf-enterprise-scale` codebases have been succeeded by the AVM-based accelerators.

### Milestones

| Milestone | Deliverable |
|---|---|
| M1 Design | Architecture diagram, ADRs (IaC tool, network topology, policy approach), naming and tagging standard |
| M2 Bootstrap | Repo, branch protection, OIDC workload identity federation from GitHub Actions or Azure Pipelines, state storage (Terraform) |
| M3 Governance | Management groups, policy assignments (audit first, then deny), RBAC with groups |
| M4 Connectivity | Hub and spoke, Firewall policy, Bastion, private DNS zones linked to the hub |
| M5 Management | Log Analytics, diagnostic settings by policy, Defender plans, budgets |
| M6 AI spoke | Spoke with subnets and NSGs, private endpoints for Foundry, AI Search and Storage resolving through private DNS |
| M7 Validation | Policy compliance report, `what-if` evidence, connectivity tests, cost estimate |

### Pipeline sketch (GitHub Actions)

```yaml
on: { pull_request: {}, push: { branches: [main] } }
permissions: { id-token: write, contents: read }
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with: { client-id: ${{ vars.AZURE_CLIENT_ID }}, tenant-id: ${{ vars.AZURE_TENANT_ID }}, subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }} }
      - run: az deployment mg what-if --management-group-id mg-contoso -l uksouth -f main.bicep -p main.bicepparam
  deploy:
    if: github.ref == 'refs/heads/main'
    needs: validate
    environment: production   # requires approval
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with: { client-id: ${{ vars.AZURE_CLIENT_ID }}, tenant-id: ${{ vars.AZURE_TENANT_ID }}, subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }} }
      - run: az deployment mg create --management-group-id mg-contoso -l uksouth -f main.bicep -p main.bicepparam
```

## Design decisions and trade-offs

| Decision | Options | Guidance |
|---|---|---|
| IaC tool | Bicep vs Terraform | Bicep for Azure-only teams; Terraform for multicloud or existing Terraform skills |
| Network | Hub-spoke vs Virtual WAN | Hub-spoke for learning and control; Virtual WAN for many branches and regions |
| Policy rollout | Audit then deny vs deny immediately | Audit first in brownfield; deny from day one in greenfield |
| Firewall SKU | Basic, Standard, Premium | Premium for TLS inspection and IDPS in regulated industries; Basic to save lab cost |

Write an ADR for each.

## Hands-on

1. Complete milestones M1–M7.
2. Prove the deny policy works: try to create a storage account with public access and capture the error.
3. From a VM in the spoke, resolve a private endpoint FQDN to a private IP.
4. Produce a compliance screenshot and a cost estimate.
5. Tear everything down and redeploy from scratch to prove repeatability.

## Security, cost and reliability notes

- **Cost:** Azure Firewall, Bastion and VPN or ExpressRoute gateways bill hourly. Deploy for testing, then delete, or use Firewall Basic and Bastion Developer where suitable.
- **Security:** OIDC federation (no secrets in CI), PIM for Owner and User Access Administrator, and break-glass accounts excluded from Conditional Access.
- **Reliability:** zone-redundant Firewall and gateways in production; document the DR stance for the hub.

## Common mistakes

| Mistake | Fix |
|---|---|
| Private endpoints without DNS integration | Link private DNS zones to the hub; use DNS policy to create records |
| Policies deployed without exemptions process | Document exemptions with expiry |
| Client secrets in pipelines | Workload identity federation |
| Building everything custom | Start from the accelerator unless you can justify otherwise |

## Check yourself

1. Why does an AI workload need private DNS zones linked to the hub?
2. Why use audit mode before deny in a brownfield estate?
3. How does OIDC federation improve pipeline security?
4. What evidence proves the landing zone meets R3?
5. When would you choose Virtual WAN over hub-spoke?

<details>
<summary>Answers</summary>

1. So private endpoint FQDNs (for Foundry, Search, Storage) resolve to private IPs from all spokes.
2. To find existing non-compliant resources and avoid breaking workloads before enforcing.
3. No long-lived secrets: the pipeline exchanges a short-lived token for Azure access.
4. Policy compliance report, a captured deny error, and Defender plan status.
5. With many branches, regions or SD-WAN integration, where managed transit routing is valuable.

</details>

## References

- [What is an Azure landing zone?](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/)
- [Azure landing zones documentation and accelerators](https://azure.github.io/Azure-Landing-Zones/)
- [Azure Verified Modules](https://azure.github.io/Azure-Verified-Modules/)
- [Use GitHub Actions to connect to Azure (OIDC)](https://learn.microsoft.com/azure/developer/github/connect-from-azure-openid-connect)
- [Azure Private Endpoint DNS integration](https://learn.microsoft.com/azure/private-link/private-endpoint-dns-integration)
