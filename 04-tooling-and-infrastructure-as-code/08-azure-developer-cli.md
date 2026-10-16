# Azure Developer CLI

> Topic: 04 · Tooling and Infrastructure as Code · Tracks: E ● A ○ C ○ · Status checked: 2026-10

## Why it matters

The Azure Developer CLI (`azd`) takes an application from a repository to running in Azure with one command: it provisions the infrastructure, builds the code, deploys it and wires the configuration together. Nearly every Microsoft AI sample and solution accelerator ships as an `azd` template, so it is the fastest way to stand up a working RAG app or agent to study, demo or adapt. For consultants, that is a credible proof of concept in an afternoon.

## Key concepts

### What azd is, and is not

| Tool | Level | Thinks in |
|---|---|---|
| **Azure CLI (`az`)** | Control plane | Individual resources and operations |
| **Bicep / Terraform** | Infrastructure | Desired state of resources |
| **Azure Developer CLI (`azd`)** | Application | Services, environments and the workflow that gets them running |

`azd` does not replace the others. It **orchestrates** them: it calls Bicep or Terraform to provision, your language's toolchain or Docker to build, and Azure APIs to deploy.

### Anatomy of an azd project

```text
my-app/
├── azure.yaml              the project definition: services, hosts, hooks
├── infra/                  infrastructure as code (Bicep by default, or Terraform)
│   ├── main.bicep
│   ├── main.parameters.json
│   └── modules/…
├── src/
│   ├── api/                application code for each service
│   └── web/
├── .azure/                 local environment state (git-ignored)
│   └── dev/.env
└── .github/workflows/azure-dev.yml   generated CI/CD (optional)
```

`azure.yaml`:

```yaml
name: hr-assistant
metadata:
  template: hr-assistant@0.1.0

infra:
  provider: bicep            # or terraform
  path: infra

services:
  api:
    project: ./src/api
    language: python
    host: containerapp
  web:
    project: ./src/web
    language: ts
    host: staticwebapp

hooks:
  postprovision:
    shell: sh
    run: ./scripts/seed-index.sh
    continueOnError: false
```

| Key | Meaning |
|---|---|
| `services` | Each deployable unit: where its code is, its language, and its **host** |
| `host` | Target platform: `containerapp`, `appservice`, `function`, `staticwebapp`, `aks`, and AI-specific hosts added through extensions |
| `infra` | Which IaC provider and where the files are |
| `hooks` | Scripts run before or after lifecycle events |

**How services find their resources.** `azd` tags the resources it deploys to: IaC applies an `azd-service-name: <service>` tag to the host resource, and `azd-env-name: <env>` to the resource group. Without those tags `azd deploy` cannot find the target.

### Core commands

| Command | Does |
|---|---|
| `azd auth login` | Sign in (separate from `az login`, though it can reuse it) |
| `azd init` | Create a project: from a template (`--template <name>`), from existing code (it detects languages and proposes services), or minimal |
| `azd up` | **Package → provision → deploy**, in one step |
| `azd provision` | Infrastructure only. `--preview` shows a what-if |
| `azd package` | Build artefacts or container images |
| `azd deploy` | Deploy application code only (`azd deploy api` for one service) |
| `azd down` | Delete the resources. `--purge` also purges soft-deleted resources |
| `azd env new / list / select / get-values / set / refresh` | Manage environments |
| `azd pipeline config` | Set up CI/CD with federated credentials |
| `azd monitor` | Open Application Insights views |
| `azd show` | Show the project's services and endpoints |
| `azd template list` | Browse templates |

The inner loop is usually `azd up` once, then `azd deploy` after code changes and `azd provision` after infrastructure changes.

### Environments

An environment is a named set of values: subscription, region, and outputs from provisioning.

```bash
azd env new dev
azd env set AZURE_LOCATION swedencentral
azd env set CHAT_MODEL_CAPACITY 30
azd env new prod && azd env select prod
azd env get-values          # prints KEY=value, including IaC outputs
```

- Stored locally in `.azure/<name>/.env`. **Do not commit it.**
- Every **output** of `infra/main.bicep` becomes an environment value after provisioning. That is how endpoint URLs and resource names reach your application configuration and hook scripts.
- `main.parameters.json` (or a `.bicepparam`) can read environment values with `${AZURE_ENV_NAME}`-style substitution.
- Environments can be backed by remote state for team sharing.

### Hooks

| Event | Typical use |
|---|---|
| `preprovision` | Validate prerequisites, check quota, register providers |
| `postprovision` | Seed data, build a search index, create an agent, assign roles to the developer, write a local `.env` |
| `prepackage` / `postpackage` | Code generation, front-end build |
| `predeploy` / `postdeploy` | Database migrations, smoke tests, print URLs |
| `predown` / `postdown` | Export data, clean up external registrations |

Hooks exist at project level and per service, and can have separate `posix` and `windows` scripts. In AI templates the `postprovision` hook commonly does the most interesting work: uploading documents, creating the index, and creating agents.

### Templates

An `azd` template is a repository with `azure.yaml`, `infra/` and code. Sources:

- **Awesome azd** gallery (`azure.github.io/awesome-azd`), searchable by language, service and tag.
- **Microsoft Foundry templates** (AI Solution Templates / `aka.ms/foundry/templates`): RAG chat, agents, multi-agent, voice, document processing, and more.
- **Azure-Samples** on GitHub.
- Your own organisation's templates, listed through a custom template source.

```bash
azd init --template Azure-Samples/azure-search-openai-demo
azd up
```

Treat a template as a starting point. Before using one beyond a demo, review it for: public network access, key-based authentication, SKU and capacity choices, region and model assumptions, logging of prompts, and missing governance tags.

### Extensions and AI workflows

`azd` has an extension model that adds commands and service hosts. Extensions relevant here include support for **Foundry agents** (initialising an agent project from a template or manifest, deploying hosted agents) and for model fine-tuning workflows. Extension names and commands are changing quickly; list what is available with `azd extension list` and check the Foundry documentation.

Related convenience features:

- **`azd add`** (compose): add a database, AI model, messaging service or storage to a project interactively, generating the infrastructure for you.
- **Automatic infrastructure generation**: for simple projects `azd` can synthesise Bicep in memory from `azure.yaml`; `azd infra gen` writes it to disk when you want to take control.
- **Aspire**: `azd` understands Aspire app hosts and deploys what they describe.

### CI/CD

```bash
azd pipeline config            # GitHub Actions by default; --provider azdo for Azure DevOps
```

This creates a service principal or managed identity, adds **federated credentials** for the repository, grants roles, stores the needed variables in the repository, and commits a workflow that runs `azd provision` and `azd deploy`. Review the roles it grants; narrow them for production.

In the workflow, `azd` signs in with OIDC:

```yaml
- uses: Azure/setup-azd@v2
- run: azd auth login --client-id "$AZURE_CLIENT_ID" --federated-credential-provider github --tenant-id "$AZURE_TENANT_ID"
- run: azd provision --no-prompt
- run: azd deploy --no-prompt
```

## How it works on Azure

### What `azd up` does

```text
azd up
 ├─ 1. package     build each service (pip/npm/dotnet/…​ or docker build)
 ├─ 2. provision   run hooks → deploy infra/ at subscription scope (creates rg-<env>)
 │                 → capture outputs into the environment
 └─ 3. deploy      push images to the registry / upload packages
                   → update each host resource (found by the azd-service-name tag)
                   → run hooks → print endpoints
```

Provisioning creates a subscription-scoped ARM deployment named after the environment. `azd down` deletes the resource groups it created.

### A minimal infrastructure file for azd

```bicep
targetScope = 'subscription'

@minLength(1) @maxLength(64)
param environmentName string
param location string

var tags = { 'azd-env-name': environmentName }

resource rg 'Microsoft.Resources/resourceGroups@2024-03-01' = {
  name: 'rg-${environmentName}'
  location: location
  tags: tags
}

module app './app.bicep' = {
  scope: rg
  params: {
    location: location
    tags: tags
    apiServiceTags: union(tags, { 'azd-service-name': 'api' })   // lets azd find the host
  }
}

// Outputs become environment values
output AZURE_OPENAI_ENDPOINT string = app.outputs.openAiEndpoint
output AZURE_SEARCH_ENDPOINT string = app.outputs.searchEndpoint
output SERVICE_API_URI string = app.outputs.apiUri
```

### Local development against cloud resources

After `azd provision`, the environment holds every endpoint. A hook or `azd env get-values > .env` gives your local process the same configuration the deployed app uses, and `DefaultAzureCredential` picks up your `azd auth login` or `az login` session. Templates usually grant the signed-in developer the same data roles as the app's managed identity so local runs work keylessly.

### From template to production

`azd` templates optimise for getting started. The gap to production is usually:

| Area | Template default | Production need |
|---|---|---|
| Network | Public endpoints | Private endpoints, landing zone spoke, controlled egress |
| Identity | Often keyless already; sometimes keys | Keyless everywhere, local auth disabled |
| Capacity | Small, single region | Sized, zone-redundant, quota confirmed |
| Governance | Minimal tags | Required tags, policy compliance |
| Environments | One | Dev, test, production in separate subscriptions |
| Pipeline | Single-stage, broad role | Staged with approvals, least-privilege identities, evaluation gates |
| Observability | Basic Application Insights | Tracing, evaluation, alerts, dashboards |
| Data | Sample documents | Real ingestion pipeline with access control |

Many Microsoft templates offer flags (set with `azd env set`) to turn on private networking or disable keys. Read the template's README for them.

Keep `azd` after the prototype if it fits: it works well as the developer workflow and CI entry point over infrastructure you have hardened. Or keep only `infra/` and deploy it with your existing pipeline. Either is fine.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| azd or plain IaC plus pipeline | `azd` for application-centric repositories and fast onboarding. Plain IaC pipelines for platform infrastructure with no application code |
| Bicep or Terraform under azd | Bicep is the default and best supported; Terraform works if that is your standard |
| Template as-is or adapted | As-is for demos and learning. Adapt, or rebuild on AVM, for anything real |
| One repository or split infra | `azd` assumes infrastructure and code together. Where a platform team owns shared infrastructure separately, have `infra/` reference existing resources instead of creating them |
| `azd down` in shared environments | Only for disposable environments. Guard production with locks or stacks |

## Hands-on

```bash
# 1. Install and sign in
azd version
azd auth login

# 2. Browse templates
azd template list | head -30

# 3. Start from a small AI template (pick a current one from the Foundry or awesome-azd gallery)
mkdir azd-lab && cd azd-lab
azd init --template <template-name>         # e.g. a "get started with AI chat/agents" sample
cat azure.yaml
ls infra

# 4. Create an environment and choose a region where the template's models are available
azd env new lab
azd env set AZURE_LOCATION swedencentral

# 5. Preview, then bring it up
azd provision --preview
azd up

# 6. Inspect what happened
azd show
azd env get-values
az resource list -g rg-lab --query "[].{Type:type, Name:name}" -o table
az group show -n rg-lab --query tags

# 7. Change some application code, then redeploy only the code
azd deploy

# 8. Tear down, purging soft-deleted AI resources and key vaults
azd down --purge --force
```

While it is running, review the template with the production table above and list the five changes you would make first.

Exercise: take a small web API of your own, run `azd init` in its folder, accept the detected services, and get it running on Container Apps.

## Security, cost and reliability notes

- **Security:** `.azure/` can contain keys and connection strings written by templates. Keep it out of source control; the default `.gitignore` does this.
- **Security:** `azd pipeline config` may grant Contributor and User Access Administrator at subscription scope. Reduce that for production and consider separate identities per environment.
- **Security:** templates are third-party code that provisions resources and runs hook scripts on your machine. Read `azure.yaml` hooks and `infra/` before running an unfamiliar one.
- **Cost:** AI templates create search services, model deployments and sometimes API Management or Application Gateway. These bill while idle. Run `azd down --purge` when finished.
- **Reliability:** provisioning commonly fails on model quota or regional availability. Check quota first and pick the region deliberately.
- **Reliability:** `azd down` without `--purge` leaves soft-deleted Key Vaults and AI resources that block a redeploy with the same names.

## Common mistakes

- Running a public template in a production subscription unchanged.
- Committing `.azure/`.
- Missing `azd-service-name` tags, so `azd deploy` cannot find the target.
- Forgetting that `azd` has its own sign-in.
- Choosing a region without the template's models or quota.
- Leaving a demo running for weeks.
- Skipping `--purge` and hitting name conflicts on the next run.
- Using `azd down` on an environment with real data.
- Expecting `azd` to replace governance: policy, network design and review still apply.

## Check yourself

1. How does `azd` differ from the Azure CLI and from Bicep?
2. What three stages does `azd up` run?
3. What is `azure.yaml` for?
4. How do infrastructure outputs reach application configuration?
5. How does `azd deploy` know which Azure resource hosts a service?
6. What does `azd pipeline config` set up?
7. Name four things to review before using a template beyond a demo.

<details>
<summary>Answers</summary>

1. The CLI operates on individual resources; Bicep declares infrastructure; `azd` works at application level and orchestrates provisioning, building and deploying using those tools.
2. Package, provision, deploy.
3. It defines the project: its services, their languages and hosts, the IaC provider and lifecycle hooks.
4. Outputs of the main infrastructure file are stored as environment values, available through `azd env get-values`, to hooks and to the deployment.
5. By the `azd-service-name` tag on the host resource (within the resource group tagged with `azd-env-name`).
6. A CI/CD workflow with a federated identity, role assignments and the required repository variables.
7. Any four of: public network access, key-based authentication, SKUs and capacity, region and model choices, prompt logging, tags and governance, pipeline permissions, hook scripts.

</details>

## References

- [Azure Developer CLI documentation](https://learn.microsoft.com/azure/developer/azure-developer-cli/)
- [Azure Developer CLI's azure.yaml schema](https://learn.microsoft.com/azure/developer/azure-developer-cli/azd-schema)
- [Work with environments](https://learn.microsoft.com/azure/developer/azure-developer-cli/manage-environment-variables)
- [Customize workflows using hooks](https://learn.microsoft.com/azure/developer/azure-developer-cli/azd-extensibility)
- [Configure a deployment pipeline](https://learn.microsoft.com/azure/developer/azure-developer-cli/configure-devops-pipeline)
- [Make your project compatible with azd](https://learn.microsoft.com/azure/developer/azure-developer-cli/make-azd-compatible)
- [Awesome azd template gallery](https://azure.github.io/awesome-azd/)
- [Get started with AI templates in Microsoft Foundry](https://learn.microsoft.com/azure/foundry/how-to/develop/ai-template-get-started)
