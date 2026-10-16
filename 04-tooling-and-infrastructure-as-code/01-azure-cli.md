# Azure CLI

> Topic: 04 · Tooling and Infrastructure as Code · Tracks: E ● A ○ C ○ · Status checked: 2026-10

## Why it matters

The Azure CLI (`az`) is the quickest way to ask Azure a question or make a change, and it runs the same on Windows, macOS, Linux, Cloud Shell and in pipelines. Every lab in this roadmap uses it. Fluency here is mostly three things: knowing how commands are organised, shaping output with queries, and authenticating correctly when no human is present.

## Key concepts

### Command structure

```text
az  <group>  [<subgroup>]  <command>  [--parameters]

az  storage  account       create     --name stdemo --resource-group rg-demo --sku Standard_LRS
```

| Element | Notes |
|---|---|
| **Groups** | Map to services: `vm`, `network`, `storage`, `cognitiveservices`, `search`, `containerapp`, `aks`, `role`, `ad`, `deployment`, `policy`, `monitor` |
| **Commands** | Mostly `list`, `show`, `create`, `update`, `delete`, `wait`, plus service-specific verbs |
| **Global parameters** | `--subscription`, `--output/-o`, `--query`, `--debug`, `--verbose`, `--only-show-errors`, `--help/-h` |

Finding your way:

```bash
az --help                         # top-level groups
az containerapp --help            # commands in a group
az containerapp create --help     # parameters and examples
az find "create a private endpoint"   # example-based search
az interactive                    # shell with autocomplete and inline docs
```

### Installation and versions

Install with the platform package manager, the MSI on Windows, or run it in a container (`mcr.microsoft.com/azure-cli`). Keep it current with `az upgrade`; new resource features and the mandatory MFA behaviour both depend on a recent version. `az version` shows the core and extension versions.

**Extensions** add commands for newer or specialised services:

```bash
az extension list-available -o table
az extension add --name resource-graph
az extension update --name containerapp
az config set extension.use_dynamic_install=yes_without_prompt   # install on first use
```

Extension commands may be in preview and can change. Pin versions in pipelines.

### Signing in

| Scenario | Command |
|---|---|
| Interactive (browser, or the Windows account broker) | `az login` |
| No browser available | `az login --use-device-code` |
| Specific tenant | `az login --tenant <tenant-id>` |
| Managed identity (on an Azure resource) | `az login --identity` (add `--client-id <id>` for user-assigned) |
| Service principal with federated token (pipelines) | `az login --service-principal -u <app-id> --tenant <tenant> --federated-token "$TOKEN"` |
| Service principal with certificate | `az login --service-principal -u <app-id> --tenant <tenant> --certificate <path>` |
| Service principal with secret (avoid) | `az login --service-principal -u <app-id> -p <secret> --tenant <tenant>` |

In GitHub Actions and Azure DevOps, the `azure/login` action and the Azure CLI task perform the federated sign-in for you.

Because Microsoft enforces MFA on create, update and delete operations for **user** accounts, automation must use a managed identity or service principal. A user account in a script will fail on writes.

### Subscriptions and context

```bash
az account list -o table
az account show --query "{Sub:name, Id:id, Tenant:tenantId, User:user.name}" -o table
az account set --subscription "<name or id>"
```

The active subscription is stored on disk and persists between shells. Running a destructive command against the wrong one is the classic CLI accident. In scripts, pass `--subscription` explicitly or set and verify the context at the top.

### Output formats

| Format | Use |
|---|---|
| `json` (default) | Full fidelity; feeding other tools |
| `jsonc` | Coloured JSON for reading |
| `table` | Human-readable summaries. Shows only top-level scalar fields unless you shape it with `--query` |
| `tsv` | Capturing a value into a shell variable. No quotes, tab-separated |
| `yaml` / `yamlc` | Readable nested output |
| `none` | Suppress output |

```bash
RG_ID=$(az group show -n rg-demo --query id -o tsv)
```

Always use `-o tsv` when capturing into a variable; JSON output would include the quotes.

### Queries with JMESPath

`--query` filters and reshapes the JSON **on the client**, after the service returns it.

| Pattern | Example |
|---|---|
| Select a field | `--query name` |
| Nested field | `--query properties.provisioningState` |
| Array projection | `--query "[].name"` |
| Rename with a multi-select hash | `--query "[].{Name:name, Location:location}"` |
| Filter | `--query "[?location=='swedencentral']"` |
| Filter then project | `--query "[?sku.name=='Standard_LRS'].{Name:name, RG:resourceGroup}"` |
| First element | `--query "[0].id"` |
| Functions | `contains(name,'prod')`, `starts_with(name,'st')`, `length(@)`, `sort_by(@,&name)`, `max_by(@,&sizeGb)`, `to_string()`, `join(',', [].name)` |
| Boolean logic | `[?tags.env=='prod' && properties.publicNetworkAccess!='Disabled']` |
| Pipe | `"[?kind=='AIServices'] \| [0].properties.endpoint"` |

Quoting matters. In Bash, wrap the query in double quotes and use single quotes for string literals inside. Numbers and booleans as literals need backticks: ``[?properties.replicaCount > `1`]``. In PowerShell, wrap in single quotes and double any inner single quotes, or use double quotes with escaped backticks.

Prefer **server-side** filters when they exist (`--resource-group`, `--filter`, `--resource-type`): they reduce the data returned. `--query` only trims what you see.

### Configuration and defaults

```bash
az config set defaults.group=rg-demo defaults.location=swedencentral
az config set core.output=table
az config set core.only_show_errors=true
az config get
az config unset defaults.group
```

Defaults are convenient interactively and dangerous in shared scripts. Be explicit in automation.

### Long-running operations

Most `create` and `delete` commands wait for completion.

```bash
az group delete -n rg-demo --yes --no-wait          # return immediately
az group wait -n rg-demo --deleted                  # block until a condition
az vm wait -g rg -n vm1 --custom "instanceView.statuses[?code=='PowerState/running']"
```

### Calling the REST API directly

When a feature has no CLI command yet, `az rest` sends an authenticated request to any Azure endpoint:

```bash
SUB=$(az account show --query id -o tsv)

az rest --method get \
  --url "https://management.azure.com/subscriptions/$SUB/providers/Microsoft.CognitiveServices/accounts?api-version=2024-10-01" \
  --query "value[].{Name:name, Kind:kind}" -o table

az rest --method post --url "https://graph.microsoft.com/v1.0/invitations" --body @invite.json
```

It picks the right token audience for ARM, Microsoft Graph and several other endpoints; pass `--resource` for others.

### Getting tokens

```bash
az account get-access-token --resource https://cognitiveservices.azure.com --query accessToken -o tsv
az account get-access-token --scope https://search.azure.com/.default
```

Handy for testing a data-plane API with `curl`. Treat the output as a secret.

### Debugging

| Flag | Shows |
|---|---|
| `--verbose` | More detail on what the CLI is doing |
| `--debug` | Full HTTP requests and responses, including URLs, API versions and headers. Tokens are redacted but review before sharing |
| `--only-show-errors` | Suppresses warnings, useful in pipelines |

`--debug` is the quickest way to learn which REST call a command makes and which API version it uses.

## How it works on Azure

### Writing scripts that survive

```bash
#!/usr/bin/env bash
set -euo pipefail                     # stop on error, unset variables and failed pipes

SUBSCRIPTION="${SUBSCRIPTION:?set SUBSCRIPTION}"
RG="rg-demo-dev-swc-001"
LOC="swedencentral"

az account set --subscription "$SUBSCRIPTION"

# Idempotent: create is a PUT, safe to repeat
az group create -n "$RG" -l "$LOC" --tags owner=platform env=dev --only-show-errors -o none

# Check before acting when a command is not idempotent
if [[ "$(az group exists -n "$RG")" == "true" ]]; then
  echo "Resource group ready"
fi

# Capture outputs for later steps
ST_ID=$(az storage account create -n "stdemo$RANDOM" -g "$RG" -l "$LOC" \
          --sku Standard_LRS --allow-shared-key-access false --query id -o tsv)

# Retry on eventual consistency (role assignment propagation, new identities)
for i in {1..10}; do
  az role assignment create --assignee-object-id "$PRINCIPAL" --assignee-principal-type ServicePrincipal \
     --role "Storage Blob Data Reader" --scope "$ST_ID" -o none && break
  sleep 10
done
```

Practices:

- **Be explicit:** subscription, resource group, location, `--assignee-principal-type`.
- **Prefer object IDs** to names for principals; it avoids a Graph lookup and its permission requirement.
- **Never echo secrets.** Do not pass them as plain arguments where they land in process lists and logs; use Key Vault, environment variables, or `@file` syntax.
- **Pin** the CLI and extension versions in CI.
- **Use the CLI for glue, IaC for infrastructure.** A long imperative script that creates twenty resources should be a Bicep or Terraform file.

### Commands you will use for AI work

```bash
# Foundry / AI services resources and deployments
az cognitiveservices account list --query "[].{Name:name, Kind:kind, Endpoint:properties.endpoint}" -o table
az cognitiveservices account deployment list -n <account> -g <rg> \
  --query "[].{Deployment:name, Model:properties.model.name, Version:properties.model.version, Sku:sku.name, Capacity:sku.capacity}" -o table
az cognitiveservices model list -l swedencentral --query "[?kind=='OpenAI'].model.name" -o tsv | sort -u
az cognitiveservices usage list -l swedencentral -o table

# Azure AI Search
az search service list -g <rg> -o table

# Container Apps
az containerapp logs show -n <app> -g <rg> --follow
az containerapp revision list -n <app> -g <rg> -o table

# Role assignments and identities
az role assignment list --assignee <object-id> --all -o table
az identity list -g <rg> --query "[].{Name:name, ClientId:clientId, PrincipalId:principalId}" -o table

# Resource Graph
az graph query -q "Resources | where type =~ 'microsoft.cognitiveservices/accounts' | project name, kind, location"
```

## Design decisions and trade-offs

| Question | Guidance |
|---|---|
| CLI or PowerShell | CLI for Bash users and cross-platform scripts; PowerShell when you want objects and are on a Windows-centric team. Both are fully supported; pick one per team |
| CLI or IaC | CLI for queries, one-off operations, and pipeline glue. IaC for anything that should exist tomorrow |
| CLI in application code | No. Use the SDKs |
| `az rest` vs waiting for a command | Fine for scripts and exploration; in IaC use the AzAPI provider or a newer Bicep API version instead |
| Interactive defaults | Convenient; never rely on them in shared automation |

## Hands-on

```bash
# 1. Orientation
az version
az account show -o jsonc
az config get

# 2. Query practice (adapt to your subscription)
az group list --query "[].{Name:name, Location:location, Owner:tags.owner}" -o table
az group list --query "[?starts_with(name,'rg-')].name" -o tsv
az group list --query "length([?tags.owner==null])"
az resource list --query "sort_by([], &type)[].{Type:type, Name:name}" -o table | head -20
az vm list-sizes -l swedencentral --query "[?numberOfCores==\`4\` && memoryInMb>=\`16384\`].name" -o tsv | head

# 3. Capture, reuse, and see the REST call
RG=rg-cli-lab
az group create -n $RG -l swedencentral -o none
RG_ID=$(az group show -n $RG --query id -o tsv); echo "$RG_ID"
az group show -n $RG --debug 2>&1 | grep -E "Request URL|Response status"

# 4. Call ARM directly
az rest --method get --url "https://management.azure.com${RG_ID}?api-version=2022-09-01" --query "{name:name, state:properties.provisioningState}"

# 5. Tag and query
az group update -n $RG --set tags.env=lab tags.owner=me
az group list --query "[?tags.env=='lab'].{Name:name, Owner:tags.owner}" -o table

# 6. Clean up without waiting, then wait
az group delete -n $RG --yes --no-wait
az group wait -n $RG --deleted && echo gone
```

Exercises:

1. List every storage account that still allows shared key access, with its resource group.
2. Write a one-liner that prints the endpoint of the first Foundry resource in a resource group.
3. Turn the script above into one that takes the resource group name as an argument and fails clearly if it is missing.

## Security, cost and reliability notes

- **Security:** the CLI caches tokens under `~/.azure`. On shared machines and build agents, run `az logout` and `az account clear`, or use ephemeral agents.
- **Security:** `--debug` output and shell history can contain sensitive values. Scrub before pasting into tickets or chat.
- **Security:** give pipeline identities only the roles their scripts need, at the narrowest scope.
- **Cost:** the CLI is free. What it creates is not; `az group delete` at the end of every lab.
- **Reliability:** ARM is eventually consistent. Retry reads after writes, especially for identities and role assignments.

## Common mistakes

- Running against the wrong subscription.
- Capturing JSON output (with quotes) into a variable instead of `-o tsv`.
- Unquoted or wrongly quoted JMESPath, especially with numbers and booleans.
- Assuming `--query` reduces the work the service does.
- Scripts without `set -e`, which carry on after a failed command.
- Automating with a user account and hitting MFA enforcement.
- Passing secrets on the command line.
- Building whole environments in imperative scripts.
- An outdated CLI missing parameters that the documentation shows.

## Check yourself

1. Which output format should you use to store a single value in a shell variable, and why?
2. Write a query that lists the names of resource groups in `westeurope`.
3. Where does `--query` filtering happen?
4. How does a GitHub Actions workflow sign in without a stored secret?
5. What does `az rest` do, and when is it useful?
6. Which flag shows the REST requests a command makes?
7. Why will a script that signs in as a user fail on `az group create` today?

<details>
<summary>Answers</summary>

1. `tsv`, because it emits the raw value without JSON quotes.
2. `az group list --query "[?location=='westeurope'].name" -o tsv`
3. On the client, after the full response is returned.
4. With workload identity federation: the workflow's OIDC token is exchanged for an Entra token (`azure/login`, or `az login --service-principal --federated-token`).
5. It sends an authenticated HTTP request to an Azure endpoint; useful when no CLI command exists for a feature or API version.
6. `--debug`.
7. Mandatory MFA applies to create, update and delete operations for user accounts; automation must use a workload identity.

</details>

## References

- [Azure CLI documentation](https://learn.microsoft.com/cli/azure/)
- [Sign in with Azure CLI](https://learn.microsoft.com/cli/azure/authenticate-azure-cli)
- [How to query Azure CLI command output using a JMESPath query](https://learn.microsoft.com/cli/azure/use-azure-cli-successfully-query)
- [Output formats for Azure CLI commands](https://learn.microsoft.com/cli/azure/format-output-azure-cli)
- [Azure CLI configuration options](https://learn.microsoft.com/cli/azure/azure-cli-configuration)
- [Use the Azure REST API with Azure CLI](https://learn.microsoft.com/cli/azure/use-azure-cli-rest-command)
- [Tips for using the Azure CLI successfully](https://learn.microsoft.com/cli/azure/use-azure-cli-successfully-tips)
- [JMESPath specification and tutorial](https://jmespath.org/tutorial.html)
