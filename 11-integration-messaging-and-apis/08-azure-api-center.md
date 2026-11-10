# Azure API Center

## Why it matters

Large organisations don't know what APIs they have. APIs are spread across several API Management instances, other gateways, Functions, App Service and SaaS platforms, and now **MCP servers** and agents as well. Duplicates get built, insecure "shadow" APIs go unnoticed, and agent builders can't find the tools they're allowed to use. **Azure API Center** is a design-time **inventory, governance and discovery** service for all of these assets. API Management governs traffic at **runtime**; API Center governs the **catalogue**.

## Key concepts

### API Center vs API Management

| | API Center | API Management |
|---|---|---|
| Purpose | Design-time inventory, governance and discovery | Runtime gateway: security, policies, traffic, observability |
| Sees traffic? | No | Yes |
| Scope | All APIs and MCP servers, wherever they run and whatever manages them | APIs published through that APIM instance |
| Consumer experience | **API Center portal** (organisation-wide catalogue) | **Developer portal** (subscribe and try the instance's APIs) |
| Typical owner | API programme / platform governance team | Platform and API teams |

They work together: link APIM instances to API Center so their APIs (and MCP servers) synchronise automatically into the inventory. **Amazon API Gateway** can also be linked as an API source (`az apic integration create aws`), with its credentials held in Key Vault.

### Inventory model

| Entity | Meaning |
|---|---|
| **API** | A logical API (REST, GraphQL, gRPC, SOAP, WebSocket, webhook, MCP and others) with title, kind, lifecycle stage and metadata |
| **Version** | A version of the API (for example `v1`, `2026-01-01`) with its own lifecycle stage (design, development, testing, preview, production, deprecated, retired) |
| **Definition** | A specification file for a version (OpenAPI 2/3, AsyncAPI, GraphQL schema, WSDL, protobuf...) |
| **Environment** | Where APIs run: an APIM instance, a Kong or AWS gateway, a Kubernetes cluster, with kind (development, testing, staging, production) |
| **Deployment** | A specific version deployed to an environment at a runtime URL |
| **Metadata schema** | Built-in properties plus **custom metadata** you define (owner team, data classification, cost centre, compliance scope), which can be required |
| **Workspace** | The default container for the inventory |

### Governance features

| Feature | What it does |
|---|---|
| **Custom metadata** | Enforce required properties (for example `dataClassification`, `businessOwner`) for every API |
| **API analysis (linting)** | Managed linting of API definitions against a ruleset (Spectral-based; default ruleset or your own) with results per definition |
| **Breaking change detection** | In the VS Code extension, compare definitions to flag breaking changes |
| **Shadow API detection** | With **Dev Proxy**, detect calls from apps to APIs not registered in the inventory |
| **Lifecycle** | Track lifecycle stage per API and version to plan deprecation |

### Discovery features

| Feature | What it does |
|---|---|
| **API Center portal** | A website where developers search and filter the catalogue and view docs. Microsoft Entra sign-in; access granted through a role |
| **VS Code extension** | Register, browse, lint and generate clients from the inventory in the IDE; integrates with GitHub Copilot |
| **MCP server registry** | Register remote MCP servers (including those exposed through APIM) so developers and agent platforms can discover approved tools |
| **Copilot Studio connector** | Make inventory APIs available to Copilot Studio agents |

## How it works on Azure

### Plans

| | Free | Standard |
|---|---|---|
| Use | Evaluation and small-scale use | Enterprise and production |
| Inventory and governance | Limited (see the API Center limits page for counts) | Expanded |
| Microsoft support | **Not available** | Yes |
| Upgrade | To Standard at any time | n/a |

When you **link an API Management instance** in the Standard, Standard v2, Premium or Premium v2 tier, you can use API Center Standard **at no extra cost** while at least one eligible instance stays linked. Check pricing pages before you commit.

Regional availability is limited (for example East US, West Europe, UK South, Sweden Central, Australia East, Central India, Canada Central and France Central as of 2026-11). The inventory can still describe APIs running in any region.

### Create and populate an inventory (CLI)

```bash
az extension add --name apic-extension

az apic create -g rg-gov -n apic-contoso -l westeurope

# Register an API directly from a spec file (creates API, version and definition)
az apic api register -g rg-gov -n apic-contoso --api-location ./orders-openapi.yaml

# Add an environment and link an APIM instance so its APIs sync automatically
az apic environment create -g rg-gov -n apic-contoso --environment-id prod-apim \
  --title "Production APIM" --type production
az apic integration create apim -g rg-gov -n apic-contoso --integration-name apim-prod \
  --azure-apim apim-contoso-prod
```

Command groups and parameter names are those of the `apic-extension`. Run `az apic --help` to confirm, because the extension is updated often.

### Custom metadata example

```json
{
  "type": "object",
  "properties": {
    "dataClassification": { "type": "string", "enum": ["public", "internal", "confidential", "restricted"] }
  },
  "required": ["dataClassification"]
}
```

Assign it to APIs (and optionally environments or deployments) and mark it **required**, so nothing is registered without a classification.

### Registering MCP servers

Register an MCP server as an API of kind **MCP**, with its endpoint (deployment) and, where available, its tool list. Servers exposed through a linked APIM instance can be synchronised automatically. Developers then find approved MCP servers in the API Center portal or VS Code, and agent platforms can use the catalogue as an enterprise **MCP registry**. Check current docs for how API Center surfaces skills and agents, as this area changes often.

### Pipeline integration

```yaml
# GitHub Actions: register or update an API definition on merge to main
- uses: azure/login@v2
  with: { client-id: ${{ vars.AZURE_CLIENT_ID }}, tenant-id: ${{ vars.AZURE_TENANT_ID }}, subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }} }
- run: |
    az extension add --name apic-extension
    az apic api register -g rg-gov -n apic-contoso --api-location ./api/openapi.yaml
```

Use OIDC federated credentials (no secrets) for the pipeline identity, with the **Azure API Center Service Contributor** role (or a narrower custom role) on the API center.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| How many API centers | Usually **one per organisation** (or per major regulatory boundary), so discovery is truly central |
| Population strategy | Automate: link APIM instances, register from CI/CD, and import from other gateways. Manual registration decays quickly |
| Required metadata | Keep it short (owner, classification, lifecycle, support contact). Too many required fields means people stop registering |
| Linting ruleset | Start with the default, then add a company style guide. Report first, block later |
| MCP governance | Expose production MCP servers through APIM (runtime policies) and register them in API Center (discovery and approval). Publish only approved servers to agent builders |
| Free vs Standard | Free for a proof of concept only, since there's no support. Standard for production, which is free with a linked eligible APIM |

## Hands-on

1. Create an API center (Free plan) and add a custom metadata property `dataClassification` (required).
2. Register two APIs from OpenAPI files with the CLI, then add versions and lifecycle stages.
3. Enable API analysis and review linting findings for each definition.
4. Link an existing APIM instance and confirm its APIs appear in the inventory.
5. Register a remote MCP server (for example your Functions MCP server from topic 10) as an MCP API with a deployment.
6. Set up the API Center portal, grant a colleague access, and have them find the MCP server.
7. Install the Azure API Center VS Code extension, browse the inventory and generate a client for one API.

## Security, cost and reliability notes

- **Security:** the inventory itself is sensitive because it maps your attack surface. Restrict portal access with Entra ID and grant data-reader access only to the people who need it.
- **Security:** don't store credentials in API definitions or metadata.
- **Security:** use shadow API detection and linting results as inputs to security reviews.
- **Cost:** Free for evaluation; Standard billed per instance unless covered by a linked eligible APIM instance.
- **Reliability:** API Center isn't in the request path, so an outage doesn't break APIs. It does affect discovery, governance automation and pipelines that register APIs.

## Common mistakes

- Confusing API Center with API Management and expecting it to enforce runtime policies.
- Populating the inventory manually once and never again.
- Too many required metadata fields, so teams avoid registering.
- Running production governance on the Free plan with no support.
- Letting agent builders connect to any MCP server instead of a curated, registered list.

## Check yourself

1. What is the main difference between API Center and API Management?
2. Name the entities in the API Center inventory model.
3. How does API Center help enforce API design standards?
4. What is a shadow API, and how can API Center help find them?
5. How can you get API Center Standard without extra cost?
6. Why register MCP servers in API Center?

<details>
<summary>Answers</summary>

1. API Center is a design-time inventory, governance and discovery service and sees no traffic. API Management is a runtime gateway that enforces policies on traffic.
2. APIs, versions, definitions, environments, deployments, and metadata (built-in and custom).
3. Required custom metadata, managed API analysis (linting against a ruleset), and breaking change detection in VS Code.
4. An API used by applications but not registered or governed. Dev Proxy integration with API Center detects calls to APIs not in the inventory.
5. Link an API Management instance in Standard, Standard v2, Premium or Premium v2; while it stays linked, API Center Standard is included.
6. So developers and agent platforms can discover approved MCP servers from one governed catalogue, with ownership and lifecycle data, instead of connecting to arbitrary tools.

</details>

## References

- [What is Azure API Center?](https://learn.microsoft.com/azure/api-center/overview)
- [API Center key concepts](https://learn.microsoft.com/azure/api-center/key-concepts)
- [Register and discover MCP servers](https://learn.microsoft.com/azure/api-center/register-discover-mcp-server)
- [Synchronise APIs from API Management](https://learn.microsoft.com/azure/api-center/synchronize-api-management-apis)
- [Enable API analysis (linting)](https://learn.microsoft.com/azure/api-center/enable-managed-api-analysis-linting)
- [Set up the API Center portal](https://learn.microsoft.com/azure/api-center/set-up-api-center-portal)
- [API Center extension for VS Code](https://learn.microsoft.com/azure/api-center/build-register-apis-vscode-extension)
- [Discover shadow APIs with Dev Proxy](https://learn.microsoft.com/azure/api-center/discover-shadow-apis-dev-proxy)
- [API Center frequently asked questions](https://learn.microsoft.com/azure/api-center/frequently-asked-questions)
