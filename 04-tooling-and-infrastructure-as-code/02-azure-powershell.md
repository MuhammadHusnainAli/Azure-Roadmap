# Azure PowerShell

## Why it matters

Azure PowerShell does the same job as the Azure CLI with a different philosophy: commands return **objects**, not text. For teams with a Windows or operations background it is the natural tool, and a great deal of existing automation (runbooks, operational scripts, migration tooling) is written in it. You should be able to read it, and choose deliberately between it and the CLI.

## Key concepts

### The Az module

Azure PowerShell is the **Az** module, a roll-up of per-service modules (`Az.Accounts`, `Az.Resources`, `Az.Compute`, `Az.Network`, `Az.Storage`, `Az.CognitiveServices`, and so on). It runs on **PowerShell 7** on Windows, macOS and Linux, and is preinstalled in Cloud Shell.

```powershell
Install-Module Az -Scope CurrentUser -Repository PSGallery
Update-Module Az
Get-InstalledModule Az
```

Legacy modules you may meet and should not use for new work:

| Retired or deprecated | Replacement |
|---|---|
| `AzureRM` | `Az` |
| `AzureAD`, `MSOnline` | `Microsoft.Graph` PowerShell SDK, or the `Microsoft.Entra` module |

### Cmdlet naming

`Verb-AzNoun`: `Get-AzVM`, `New-AzResourceGroup`, `Set-AzStorageAccount`, `Remove-AzResourceGroup`.

```powershell
Get-Command -Module Az.CognitiveServices
Get-Command *-AzRoleAssignment*
Get-Help New-AzResourceGroupDeployment -Examples
```

### Signing in and context

| Scenario | Command |
|---|---|
| Interactive | `Connect-AzAccount` |
| Device code | `Connect-AzAccount -UseDeviceAuthentication` |
| Managed identity | `Connect-AzAccount -Identity` (add `-AccountId <client-id>` for user-assigned) |
| Service principal, federated token | `Connect-AzAccount -ServicePrincipal -ApplicationId <id> -Tenant <tenant> -FederatedToken $token` |
| Service principal, certificate | `Connect-AzAccount -ServicePrincipal -ApplicationId <id> -Tenant <tenant> -CertificateThumbprint <thumb>` |

A **context** is the combination of account, tenant and subscription.

```powershell
Get-AzContext
Get-AzSubscription
Set-AzContext -Subscription "<name or id>"

# Work against another subscription without changing the default
$ctx = Get-AzSubscription -SubscriptionName "prod" | Set-AzContext -Scope Process
Get-AzResourceGroup -DefaultProfile $ctx
```

As with the CLI, user accounts must satisfy MFA for write operations; use workload identities in automation.

### Objects and the pipeline

This is what distinguishes PowerShell. Output is structured, so you filter, sort and reshape with standard cmdlets instead of a query language.

```powershell
# Filter, project, sort
Get-AzResourceGroup |
  Where-Object { $_.Tags.owner -eq $null } |
  Select-Object ResourceGroupName, Location |
  Sort-Object Location

# Group and count
Get-AzResource | Group-Object ResourceType | Sort-Object Count -Descending | Select-Object -First 10 Count, Name

# Pipe objects into other cmdlets
Get-AzResourceGroup -Tag @{ env = 'lab' } | Remove-AzResourceGroup -Force -AsJob

# Calculated properties
Get-AzVM -Status | Select-Object Name, @{ n = 'Size'; e = { $_.HardwareProfile.VmSize } }, PowerState

# Export
Get-AzStorageAccount | Select-Object StorageAccountName, Location, @{ n='Sku'; e={$_.Sku.Name} } | Export-Csv accounts.csv -NoTypeInformation
```

Where a cmdlet offers a server-side filter (`-ResourceGroupName`, `-Tag`, `-ResourceType`, `-Filter`), use it before `Where-Object`.

### Useful language features

```powershell
# Splatting: readable parameter sets
$params = @{
  Name              = 'stdemo12345'
  ResourceGroupName = 'rg-demo'
  Location          = 'swedencentral'
  SkuName           = 'Standard_LRS'
  AllowSharedKeyAccess = $false
}
New-AzStorageAccount @params

# Preview and confirmation
Remove-AzResourceGroup -Name rg-demo -WhatIf
Remove-AzResourceGroup -Name rg-demo -Confirm

# Error handling
$ErrorActionPreference = 'Stop'
try {
  New-AzResourceGroupDeployment -ResourceGroupName rg-demo -TemplateFile main.bicep -ErrorAction Stop
} catch {
  Write-Error "Deployment failed: $($_.Exception.Message)"
  throw
}

# Background jobs
$job = Remove-AzResourceGroup -Name rg-old -Force -AsJob
$job | Wait-Job | Receive-Job
```

`-WhatIf` and `-Confirm` are supported on most cmdlets that change state. For deployments, `-WhatIf` on `New-AzResourceGroupDeployment` runs the ARM what-if operation.

### Deployments, Resource Graph and REST

```powershell
# Bicep / ARM deployment with what-if
New-AzResourceGroupDeployment -ResourceGroupName rg-demo -TemplateFile main.bicep `
  -TemplateParameterFile main.bicepparam -WhatIf

# Resource Graph
Search-AzGraph -Query "Resources | summarize count() by type | order by count_ desc" -First 20

# Raw REST call
Invoke-AzRestMethod -Method GET -Path "/subscriptions/$((Get-AzContext).Subscription.Id)/resourcegroups?api-version=2022-09-01" |
  Select-Object -ExpandProperty Content | ConvertFrom-Json

# Access token for a data-plane API (returned as a SecureString in current versions)
$token = (Get-AzAccessToken -ResourceUrl "https://cognitiveservices.azure.com").Token
```

### Where PowerShell runs in Azure

| Host | Use |
|---|---|
| **Azure Automation runbooks** | Scheduled and event-driven operational tasks, with a managed identity; hybrid workers for on-premises |
| **Azure Functions (PowerShell)** | Event-driven automation with bindings |
| **Cloud Shell** | Ad hoc |
| **GitHub Actions / Azure Pipelines** | `azure/powershell` action, Azure PowerShell task |
| **VM extensions and Run Command** | Configuration inside Windows VMs |

For new long-running or event-driven automation, also consider Logic Apps and Functions in other languages; choose on team skills.

## How it works on Azure

### CLI and PowerShell side by side

| Task | Azure CLI | Azure PowerShell |
|---|---|---|
| Sign in | `az login` | `Connect-AzAccount` |
| Set subscription | `az account set -s X` | `Set-AzContext -Subscription X` |
| Create resource group | `az group create -n rg -l loc` | `New-AzResourceGroup -Name rg -Location loc` |
| List VMs | `az vm list -o table` | `Get-AzVM` |
| Filter | `--query "[?location=='x']"` | `Where-Object Location -eq 'x'` |
| Capture a value | `$(az ... -o tsv)` | `(Get-Az...).Property` |
| Deploy Bicep | `az deployment group create` | `New-AzResourceGroupDeployment` |
| What-if | `az deployment group what-if` | `... -WhatIf` |
| Resource Graph | `az graph query -q` | `Search-AzGraph -Query` |
| REST | `az rest` | `Invoke-AzRestMethod` |

Both call the same Azure Resource Manager API. New features usually reach both, though one may lead by a release.

## Design decisions and trade-offs

| Factor | Favours CLI | Favours PowerShell |
|---|---|---|
| Team background | Linux, developers | Windows, operations |
| Output handling | Quick text and `jq`/JMESPath | Rich object manipulation, reports, CSV and Excel export |
| Complex logic | Becomes awkward in Bash | A full scripting language with error handling and modules |
| Start-up speed in CI | Fast | Slower module import |
| Existing estate | – | Automation runbooks, legacy scripts |
| Microsoft 365 and Entra administration alongside Azure | – | Graph and Exchange modules in the same session |

Use one consistently within a codebase. Mixing both in one script is a maintenance burden, and a script that grows complex in either is usually asking to be IaC or a real program.

## Hands-on

```powershell
# 1. Sign in and inspect
Connect-AzAccount
Get-AzContext | Format-List Account, Subscription, Tenant

# 2. Explore with objects
Get-AzResourceGroup | Get-Member -MemberType Property
Get-AzResourceGroup | Select-Object ResourceGroupName, Location, @{ n='Owner'; e={ $_.Tags.owner } } | Format-Table

# 3. Create with splatting
$rg = @{ Name = 'rg-pwsh-lab'; Location = 'swedencentral'; Tag = @{ env = 'lab'; owner = 'me' } }
New-AzResourceGroup @rg

# 4. Report: resource count by type as CSV
Get-AzResource | Group-Object ResourceType |
  Select-Object @{ n='Type'; e={$_.Name} }, Count | Sort-Object Count -Descending |
  Export-Csv resource-types.csv -NoTypeInformation

# 5. Resource Graph
Search-AzGraph -Query "ResourceContainers | where type =~ 'microsoft.resources/subscriptions/resourcegroups' | where isnull(tags.owner) | project name, subscriptionId"

# 6. Preview, then remove
Remove-AzResourceGroup -Name rg-pwsh-lab -WhatIf
Remove-AzResourceGroup -Name rg-pwsh-lab -Force
```

Exercise: translate the Bash script from the Azure CLI file into PowerShell with `try/catch` and splatting.

## Security, cost and reliability notes

- **Security:** contexts are cached on disk by default. On shared machines use `Disable-AzContextAutosave` or `Connect-AzAccount -Scope Process`, and `Disconnect-AzAccount` when done.
- **Security:** never hard-code credentials. Use managed identities in Automation and Functions; retrieve secrets from Key Vault at run time.
- **Security:** sign scripts and restrict execution policy where organisational policy requires it.
- **Reliability:** set `$ErrorActionPreference = 'Stop'` in automation. By default many errors are non-terminating and the script continues.
- **Reliability:** pin module versions in runbooks and pipelines; a module update can change output shapes.

## Common mistakes

- Using retired modules (`AzureRM`, `AzureAD`) copied from old blog posts.
- Forgetting that errors are non-terminating by default.
- Filtering thousands of objects client-side when a server-side parameter exists.
- Running against the wrong context.
- Treating formatted output (`Format-Table`) as data to pipe onward.
- Classic Run As accounts in Automation, which are retired; use managed identities.
- Mixing CLI and PowerShell in the same script.

## Check yourself

1. What is the key difference between Azure PowerShell and the Azure CLI in how output is handled?
2. Which module replaces `AzureRM`? Which replace `AzureAD` and `MSOnline`?
3. How do you sign in from an Azure Automation runbook without credentials?
4. What does `-WhatIf` do?
5. What is splatting?
6. Why set `$ErrorActionPreference = 'Stop'`?

<details>
<summary>Answers</summary>

1. PowerShell cmdlets return structured objects you manipulate in the pipeline; the CLI returns JSON text you shape with a query.
2. `Az`. The Microsoft Graph PowerShell SDK or the Microsoft Entra PowerShell module.
3. `Connect-AzAccount -Identity`, using the Automation account's managed identity.
4. Shows what a command would change without doing it.
5. Passing a hashtable of parameters to a cmdlet with `@name`, for readability and reuse.
6. So errors terminate the script and can be caught, instead of being reported while execution continues.

</details>

## References

- [Azure PowerShell documentation](https://learn.microsoft.com/powershell/azure/)
- [Sign in with Azure PowerShell](https://learn.microsoft.com/powershell/azure/authenticate-azureps)
- [Azure PowerShell context objects](https://learn.microsoft.com/powershell/azure/context-persistence)
- [Choose the right Azure command-line tool](https://learn.microsoft.com/cli/azure/choose-the-right-azure-command-line-tool)
- [Migrate from AzureRM to Az](https://learn.microsoft.com/powershell/azure/migrate-from-azurerm-to-az)
- [Microsoft Graph PowerShell SDK](https://learn.microsoft.com/powershell/microsoftgraph/overview)
- [Azure Automation runbook types](https://learn.microsoft.com/azure/automation/automation-runbook-types)
