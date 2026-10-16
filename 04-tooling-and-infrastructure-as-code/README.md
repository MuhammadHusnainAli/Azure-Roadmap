# 04 · Tooling and Infrastructure as Code

> Tracks: E ● A ○ C ○ · Part A · Foundations

Everything in Azure is an API call. This topic covers the tools that make those calls for you: command lines for exploring and scripting, declarative languages for building environments repeatably, SDKs for application code, and the newer agent-driven tools that put an AI assistant in front of all of it. An engineer lives here daily. Architects and consultants need enough to read a template, review a plan and judge a team's delivery practice.

## What you will be able to do

- Script Azure confidently with the CLI or PowerShell, including queries and authentication for automation.
- Read ARM JSON and write Bicep, including modules, parameters and `what-if`.
- Run Terraform on Azure with remote state, federated authentication and the right provider for the job.
- Compose infrastructure from Azure Verified Modules instead of hand-writing every resource.
- Manage the lifecycle of deployed resources with deployment stacks.
- Take an AI application from repository to running in Azure with one `azd up`.
- Call Azure from code with the SDKs, handling identity, retries, paging and long-running operations.
- Use AI coding agents against Azure safely.

## Prerequisites

[Topic 01](../01-cloud-and-azure-fundamentals/README.md) (Azure Resource Manager) and [Topic 02](../02-identity-and-access/README.md) (RBAC, managed identities, federation). A terminal, Azure CLI, VS Code with the Bicep extension, and a subscription you can create resources in.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Azure CLI](01-azure-cli.md) | 60 min | CLI |
| 02 | [Azure PowerShell](02-azure-powershell.md) | 40 min | PowerShell |
| 03 | [ARM Templates](03-arm-templates.md) | 30 min | Read |
| 04 | [Bicep](04-bicep.md) | 90 min | Bicep |
| 05 | [Terraform on Azure](05-terraform-on-azure.md) | 90 min | Terraform |
| 06 | [Azure Verified Modules](06-azure-verified-modules.md) | 45 min | Bicep or Terraform |
| 07 | [Deployment Stacks and Template Specs](07-deployment-stacks-and-template-specs.md) | 45 min | CLI |
| 08 | [Azure Developer CLI](08-azure-developer-cli.md) | 60 min | azd |
| 09 | [Azure SDKs and REST API](09-azure-sdks-and-rest-api.md) | 60 min | Python |
| 10 | [Azure MCP Server and AI-Assisted Operations](10-azure-mcp-server-and-ai-assisted-operations.md) | 45 min | VS Code |

Pick **one** of Bicep or Terraform to learn deeply first, and read the other file for literacy. Most teams standardise on one.

## Completion checklist

- [ ] I can write a JMESPath query that filters and reshapes CLI output.
- [ ] I can sign in non-interactively with a managed identity and with a federated credential.
- [ ] I have deployed a multi-resource Bicep file with a module and a `.bicepparam` file, after reviewing `what-if`.
- [ ] I have run `terraform plan` and `apply` against Azure with state in a storage account and no stored secret.
- [ ] I have consumed an Azure Verified Module and pinned its version.
- [ ] I have created a deployment stack, removed a resource from the template and seen it deleted.
- [ ] I have deployed a template with `azd up` and torn it down with `azd down`.
- [ ] I have written code that lists resources with paging and waits on a long-running operation.
- [ ] I can state three guardrails for letting an AI agent act on Azure.

## Maps to

- **AZ-104**: deploy and manage Azure resources with ARM templates and Bicep; manage with CLI and PowerShell.
- **AI-103 / AI-200**: provision AI resources with IaC; use SDKs and keyless authentication.
- **AZ-400**: infrastructure as code strategy and implementation.
- **AZ-305**: design a deployment and automation approach.

## Next

[05 · Networking](../05-networking/README.md)
