# 14 · Microsoft Foundry Platform

> Tracks: E ● A ● C ● · Part D · AI on Azure

Microsoft Foundry is Azure's platform for building, deploying and governing AI applications and agents. This topic covers the platform itself rather than any one capability: what Foundry is and how it was renamed, the resource and project model (and the older hub-based model), access control with the renamed Foundry roles, private networking and network-isolated agents, connections and bring-your-own resources, the SDKs and endpoints, the developer workflow with the Foundry Toolkit for VS Code and `azd`, on-device inference with Foundry Local, fleet governance with Foundry Control Plane, quotas and pricing, and infrastructure as code. Models, RAG, agents and evaluation build on this foundation in topics 15–22.

## What you will be able to do

- Explain what Microsoft Foundry is, its previous names, and when to use Foundry vs Foundry (classic).
- Design a resource and project layout and plan migration from hub-based projects.
- Assign least-privilege roles (Foundry User, Project Manager, Account Owner, Owner, Agent Consumer) and go keyless.
- Design private networking with private endpoints, DNS zones, agent network injection and egress control.
- Create connections with Entra ID and choose between basic and standard (BYO) agent setups.
- Choose the right SDK and endpoint (Foundry SDK, OpenAI v1, Anthropic, Agent Framework, Foundry Tools).
- Run an inner and outer development loop with the Foundry Toolkit, playgrounds and `azd`.
- Run models on device with Foundry Local and design hybrid local/cloud patterns.
- Govern an agent fleet with Foundry Control Plane and the AI gateway.
- Plan regions, quota (tiers, subscription-level pools) and costs, and attribute spend per project.
- Deploy Foundry resources, projects, deployments, connections and roles with Bicep or Terraform (AzAPI).

## Prerequisites

[Topic 02](../02-identity-and-access/README.md) (Entra ID, RBAC, managed identities), [Topic 04](../04-tooling-and-infrastructure-as-code/README.md) (CLI, Bicep, Terraform, azd), [Topic 05](../05-networking/README.md) (private endpoints, DNS) and [Topic 13](../13-ai-and-generative-ai-foundations/README.md).

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Microsoft Foundry Overview](01-microsoft-foundry-overview.md) | 40 min | Portal, Python |
| 02 | [Foundry Resources and Projects](02-foundry-resources-and-projects.md) | 50 min | CLI |
| 03 | [Foundry Access Control](03-foundry-access-control.md) | 50 min | CLI, Python |
| 04 | [Foundry Networking](04-foundry-networking.md) | 60 min | Bicep, templates |
| 05 | [Connections and Bring-Your-Own Resources](05-connections-and-bring-your-own-resources.md) | 45 min | Bicep, CLI, Python |
| 06 | [Foundry SDKs and APIs](06-foundry-sdks-and-apis.md) | 45 min | Python, JS, REST |
| 07 | [Foundry Toolkit for VS Code and Developer Workflow](07-foundry-toolkit-for-vs-code-and-developer-workflow.md) | 45 min | VS Code, azd |
| 08 | [Foundry Local and Edge Inference](08-foundry-local-and-edge-inference.md) | 40 min | Foundry Local, Python |
| 09 | [Foundry Control Plane](09-foundry-control-plane.md) | 40 min | Portal |
| 10 | [Quotas, Regions and Pricing](10-quotas-regions-and-pricing.md) | 45 min | CLI, calculator |
| 11 | [Foundry Infrastructure as Code](11-foundry-infrastructure-as-code.md) | 60 min | Bicep, Terraform, GitHub Actions |

**AI engineers:** all of it, with 03, 05, 06, 07 and 11 in depth. **Architects:** all of it, with 02, 03, 04, 09 and 10 in depth. **Consultants:** 01, 02, 03 (roles), 09 and 10.

## Completion checklist

- [ ] I can name the previous names of Foundry and the current names of related services.
- [ ] I can explain what lives at resource level vs project level.
- [ ] I have created a resource with `--allow-project-management`, a project and a deployment.
- [ ] I can assign Foundry User, Project Manager, Account Owner and Agent Consumer correctly, using role IDs.
- [ ] I have disabled local auth and called a model and an agent with Entra ID.
- [ ] I can list the three private DNS zones and explain agent subnet delegation.
- [ ] I have created an Entra ID connection to AI Search with the right role on the project identity.
- [ ] I know when to use the Foundry SDK vs the OpenAI SDK on `/openai/v1`.
- [ ] I have used the Foundry Toolkit Agent Builder, Agent Inspector and Evaluation.
- [ ] I have run a model with Foundry Local and called it with the OpenAI SDK.
- [ ] I can explain Foundry Control Plane and how it differs from Agent 365.
- [ ] I can explain quota tiers and subscription-level quota pools, and estimate monthly cost for a use case.
- [ ] I have deployed a Foundry resource, project, deployment and role assignments from Bicep or Terraform.

## Maps to

- **AI-103**: plan and manage a Foundry solution (resources, projects, security, networking, deployment, monitoring).
- **AI-901**: Foundry platform concepts.
- **AZ-305**: design AI platform infrastructure, identity and networking.
- **AB-100**: governance and positioning of Foundry in agentic business solutions.

## Next

15 · Foundry Models and Azure OpenAI (see [Roadmap.md](../Roadmap.md))
