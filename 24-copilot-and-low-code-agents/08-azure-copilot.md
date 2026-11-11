# Azure Copilot

## Why it matters

Azure has hundreds of services and thousands of resource types, and operators spend hours writing Resource Graph queries, reading cost reports, decoding error messages and translating "make this resilient" into concrete changes. **Azure Copilot** (formerly Copilot in Azure) is the AI assistant built into the Azure portal and mobile app that answers questions, generates queries and scripts, and performs tasks on your behalf, using **your** permissions. With **agents in Azure Copilot** (troubleshooting, deployment, optimisation, resiliency, migration, observability) it becomes an agentic operations interface. Architects need to know what it can and can't do, how it's governed and how it fits alongside the Azure SRE Agent (file 09) and GitHub Copilot (file 07).

## Key concepts

### What it does

| Category | Examples |
|---|---|
| **Understand your environment** | Resource Graph queries in natural language, service health, cost analysis and estimates, network topology, attack surface queries, Azure Firewall IDPS investigation |
| **Work with services** | Execute commands, deploy and manage VMs, discover workload templates, AKS operations, Azure Monitor metrics and logs, storage management and migration, SQL Database insights |
| **Write and optimise code** | Azure CLI and PowerShell scripts, Bicep and Terraform, API Management policies, Kubernetes YAML |
| **Troubleshoot** | Resource issues, App Service problems, disk performance |
| **Recommendations and navigation** | Advisor-style recommendations, navigating to the right blade, portal settings |

### Agents in Azure Copilot

| Agent | Status | Where invoked |
|---|---|---|
| **Troubleshooting** | GA | Azure Copilot chat |
| **Deployment** | See docs | Azure Copilot chat |
| **Optimization** | Preview | Azure Copilot chat |
| **Resiliency** | Preview | Azure Copilot chat |
| **Migration** | Preview | Azure Migrate experience |
| **Observability** | GA | Azure Monitor experience (usage-based charges) |

Agents propose actions and produce artefacts such as scripts; **no action runs without your approval**. Access is managed at tenant level and rolled out gradually.

### Security and governance model

- Azure Copilot acts **on behalf of the signed-in user** and can only see and do what that user's Azure RBAC allows.
- By default all users in a tenant can access it; Global Administrators can control access.
- Covered by the Azure Copilot section of the Microsoft Product Terms; responsible AI FAQ published.
- Not available in national clouds (Azure Government, 21Vianet).
- Requires websocket connections to `https://directline.botframework.com`.

### Limits worth knowing

| Limit | Value |
|---|---|
| Conversation continuation | 24 hours |
| Bulk actions | More than 10 resources must be done outside Azure Copilot |
| Lists | Some responses show the top five items |
| Language | 19 languages; agents fully supported in English only |
| Cost | Included at no extra cost except the Observability Agent (usage-based) |

## How it works on Azure

Example prompts by role:

| Role | Prompt |
|---|---|
| Operator | "Which VMs in rg-prod have had CPU above 90% in the last 24 hours?" |
| FinOps | "Why did my costs increase last month in subscription Contoso-Prod? Show the top five drivers." |
| Security | "List storage accounts that allow public blob access." |
| Developer | "Generate a Bicep file for a Container App with a user-assigned managed identity and private ingress." |
| Network | "Show the network topology for vnet-hub and explain why vm-app01 can't reach sql-prod." |
| Architect | "Use the resiliency agent to assess zone redundancy for my production resource group." |

Generated Resource Graph query (typical output):

```kusto
resources
| where type == "microsoft.storage/storageaccounts"
| where properties.allowBlobPublicAccess == true
| project name, resourceGroup, location, subscriptionId
```

Always review generated scripts and queries before running them; Copilot shows the query so you can check and reuse it in automation.

### Where it fits

| Tool | Best for |
|---|---|
| **Azure Copilot** | Interactive, portal-based questions and guided tasks by humans |
| **Azure SRE Agent** (file 09) | Continuous, automated incident investigation and remediation for specific apps |
| **GitHub Copilot for Azure** (file 07) | Developer workflows in the IDE: code, IaC, deployment |
| **Azure MCP Server** | Letting any agent (yours, Copilot, Claude Code) call Azure tools |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Access** | Leave enabled for most users (RBAC limits it) or restrict to trained groups initially. |
| **Approval discipline** | Treat agent actions like change requests in production: review, and use change management where required. |
| **Preview agents** | Use for assessments and recommendations; validate before applying at scale. |
| **Automation** | Use Copilot to draft queries and scripts, then put them into IaC, Policy or runbooks for repeatable operations. |
| **Network** | Allow the required Direct Line endpoint through proxies. |

## Hands-on

1. Ask Azure Copilot to list resources missing an `owner` tag; copy the Resource Graph query into a workbook.
2. Ask it to explain last month's cost changes for a subscription.
3. Use the troubleshooting agent on a failing App Service or VM; review proposed actions before approving.
4. Generate Bicep for a small workload; validate with `az bicep build` and deploy via pipeline, not directly.
5. Check how your tenant controls access to Azure Copilot and its agents.

## Security, cost and reliability notes

- **Least privilege still applies**: users with broad roles can make broad changes through Copilot; use PIM for elevated roles.
- **Review outputs**: generated scripts and queries can be wrong; test in non-production.
- **Audit**: actions are performed with the user's identity and appear in the Activity Log.
- **Observability Agent** costs: monitor usage-based charges.

## Common mistakes

| Mistake | Fix |
|---|---|
| Expecting Copilot to bypass permissions | It uses your RBAC |
| Running generated scripts in production unchecked | Review and test first |
| Bulk changes through chat | Use IaC, Policy or scripts for more than 10 resources |
| Blocking Direct Line unknowingly | Allow `directline.botframework.com` websockets |
| Confusing Azure Copilot with the SRE Agent | Interactive assistant vs continuous operations agent |

## Check yourself

1. What can Azure Copilot do? Give four categories.
2. List the agents in Azure Copilot and their status.
3. How is Azure Copilot's access to resources controlled?
4. What are three current limitations?
5. When would you use Azure Copilot vs the SRE Agent vs GitHub Copilot for Azure?
6. Which capability has usage-based charges?

<details>
<summary>Answers</summary>

1. Understand the environment (queries, costs, health), work with services (commands, VMs, AKS, monitoring), write and optimise code (CLI, PowerShell, Bicep, Terraform, APIM policies, YAML), troubleshoot and recommend.
2. Troubleshooting (GA), Deployment, Optimization (Preview), Resiliency (Preview), Migration (Preview, in Azure Migrate), Observability (GA, in Azure Monitor).
3. It acts on behalf of the signed-in user with their Azure RBAC; admins control tenant access; agent actions require approval.
4. Any three: 24-hour conversation limit, actions on more than 10 resources must be done elsewhere, lists limited to top five, some tasks need resource IDs, agents fully supported only in English, not in national clouds.
5. Azure Copilot: interactive portal tasks; SRE Agent: continuous automated incident response for apps; GitHub Copilot for Azure: developer IDE workflows.
6. The Observability Agent.

</details>

## References

- [What is Azure Copilot?](https://learn.microsoft.com/azure/copilot/overview)
- [Azure Copilot capabilities](https://learn.microsoft.com/azure/copilot/capabilities)
- [Agents in Azure Copilot](https://learn.microsoft.com/azure/copilot/agents)
- [Manage access to Azure Copilot](https://learn.microsoft.com/azure/copilot/manage-access)
- [Example prompts](https://learn.microsoft.com/azure/copilot/example-prompts)
- [Responsible AI FAQ for Azure Copilot](https://learn.microsoft.com/azure/copilot/responsible-ai-faq)
