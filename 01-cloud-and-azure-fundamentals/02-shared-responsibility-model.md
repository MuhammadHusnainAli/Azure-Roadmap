# Shared Responsibility Model

## Why it matters

Most cloud security incidents are not failures of the provider. They are customer misconfigurations: a storage account left public, a key in source control, an over-privileged identity. The shared responsibility model tells you exactly which problems are yours. For AI workloads the model has an extra layer, and clients regularly assume Microsoft handles things (like what an agent is allowed to do) that it does not.

## Key concepts

### The basic split

Microsoft is responsible for the security **of** the cloud. You are responsible for security **in** the cloud. Where the line sits depends on the service model.

| Responsibility | On-premises | IaaS | PaaS | SaaS |
|---|:-:|:-:|:-:|:-:|
| Information and data | Customer | Customer | Customer | Customer |
| Devices (mobile and PCs) | Customer | Customer | Customer | Customer |
| Accounts and identities | Customer | Customer | Customer | Customer |
| Identity and directory infrastructure | Customer | Customer | Shared | Shared |
| Applications | Customer | Customer | Shared | Microsoft |
| Network controls | Customer | Customer | Shared | Microsoft |
| Operating system | Customer | Customer | Microsoft | Microsoft |
| Physical hosts | Customer | Microsoft | Microsoft | Microsoft |
| Physical network | Customer | Microsoft | Microsoft | Microsoft |
| Physical data centre | Customer | Microsoft | Microsoft | Microsoft |

### What never transfers

Three things stay with the customer in every model:

1. **Data.** Its classification, who may see it, how long it is kept.
2. **Endpoints.** The devices people use to reach the service.
3. **Accounts and access.** Who has an identity and what it can do.

If you remember nothing else: Microsoft will never decide who should have access to your data.

### What "shared" means

"Shared" rows are where confusion lives. It means Microsoft provides the capability and you must configure it.

- **Network controls in PaaS:** Microsoft runs the network and offers private endpoints and firewalls. Whether your storage account accepts traffic from the internet is your setting.
- **Applications in PaaS:** Microsoft patches the runtime. Your code and its dependencies are yours.
- **Identity infrastructure:** Microsoft runs Entra ID. Enabling MFA and Conditional Access is your job.

## How it works on Azure

### Worked examples

**Virtual machine (IaaS)**

| Microsoft | You |
|---|---|
| Data centre, hardware, hypervisor | Guest OS patching and hardening |
| Host OS patching | Antimalware and endpoint protection |
| Physical network | NSG rules, public IP exposure |
| | Disk encryption choices, backups |
| | Applications, data, local accounts |

**App Service web app (PaaS)**

| Microsoft | You |
|---|---|
| Everything in IaaS, plus OS and runtime patching | Application code and library vulnerabilities |
| Platform availability and scaling infrastructure | Authentication configuration |
| | TLS settings, access restrictions, private endpoints |
| | Secrets handling, data, backups of your content |

**Microsoft 365 Copilot (SaaS)**

| Microsoft | You |
|---|---|
| The application, model, orchestration and infrastructure | Who is licensed |
| Enforcing existing permissions at query time | Whether those permissions are correct (oversharing) |
| | Sensitivity labels, DLP, retention |
| | User training and acceptable use |

### The AI shared responsibility model

Microsoft publishes a separate model for AI. It adds three layers above the platform:

| Layer | What it covers | IaaS (your model on GPUs) | PaaS (Foundry) | SaaS (Copilot) |
|---|---|:-:|:-:|:-:|
| **AI usage** | User training, acceptable use policy, accountability for outputs | Customer | Customer | Customer |
| **AI application** | App design, plugins and tools, data connections, safety systems around the app | Customer | Customer and shared | Microsoft and shared |
| **AI platform** | Model safety systems, model hosting, compute | Customer and shared | Microsoft and shared | Microsoft |

What that means in practice when you build on Microsoft Foundry:

| Microsoft handles | You handle |
|---|---|
| Hosting, scaling and patching the model infrastructure | Which model you choose and whether it suits the task |
| Base model safety and default content filters | Tuning filters and adding guardrails for your use case |
| Isolation between tenants | System prompts, grounding data and its access control |
| Not training foundation models on your prompts or completions | What data you send, and whether you are allowed to send it |
| Platform compliance certifications | Your application's compliance with law and policy |
| | What tools an agent can call, with which identity and permissions |
| | Evaluation, red teaming and monitoring of your application |
| | Human oversight, and accountability for what the system outputs |

The row about agents deserves emphasis. An agent with a tool that can send email or update a database acts with whatever permissions you gave it. If it is tricked by a prompt injection into misusing that tool, the platform did what it was configured to do. Least-privilege tool design is a customer responsibility. Topic 23 covers this in depth.

### How Microsoft proves its half

You cannot audit an Azure data centre. Instead you rely on:

- **Service Trust Portal:** third-party audit reports (SOC 1/2/3, ISO 27001, and many more).
- **Compliance offerings documentation:** which certifications cover which services and regions.
- **Microsoft Defender for Cloud regulatory compliance dashboard:** maps your configuration to standards and shows which controls are Microsoft's and which are yours.

A platform certification does not make your workload compliant. It means the platform will not be the reason you fail an audit.

## Design decisions and trade-offs

- **Moving up the stack shrinks your responsibility.** A team without the capacity to patch operating systems should not choose IaaS. This is a legitimate, often decisive, architecture argument.
- **Less responsibility means less control.** On PaaS you cannot install a kernel module or choose the patch date. Check that the constraint is acceptable before committing.
- **Write the split down.** For every solution, produce a responsibility matrix that names an owner for each customer-side row. Unowned rows become incidents.
- **Third parties add a third column.** A partner model in the Foundry catalogue, a Marketplace SaaS or a managed service provider each bring their own terms. Check who handles data in each.

## Hands-on

A paper exercise that mirrors what you will do for real clients.

**Scenario:** an internal HR assistant. A web front end on App Service calls a Foundry agent. The agent retrieves from an Azure AI Search index of HR policy documents stored in Blob Storage and can call a tool that looks up an employee's leave balance in an HR system.

Build a table with one row per concern and assign it to Microsoft, the customer, or shared: physical security, App Service OS patching, front-end code vulnerabilities, user sign-in and MFA, which documents are indexed, document-level permissions in search results, model hosting, content filter configuration, the leave-balance tool's permissions, logging of prompts and responses, employee guidance on what to ask.

<details>
<summary>Suggested answer</summary>

| Concern | Owner |
|---|---|
| Physical security | Microsoft |
| App Service OS patching | Microsoft |
| Front-end code vulnerabilities | Customer |
| User sign-in and MFA | Customer (configuring a Microsoft-run service) |
| Which documents are indexed | Customer |
| Document-level permissions in results | Customer (the platform provides the mechanism) |
| Model hosting | Microsoft |
| Content filter configuration | Shared: defaults by Microsoft, tuning by customer |
| Leave-balance tool permissions | Customer |
| Logging of prompts and responses | Customer (decide, configure, protect) |
| Employee guidance | Customer |

</details>

## Security, cost and reliability notes

- **Security:** enable Defender for Cloud early. Its recommendations are effectively a list of your unfulfilled responsibilities.
- **Reliability:** shared responsibility applies to resilience too. Microsoft keeps the platform running; you choose zone redundancy, configure backups and design failover. Microsoft does not back up your data unless you configure it.
- **Cost:** the "you" column costs people-time. Factor operations effort into IaaS vs PaaS comparisons, not just the meter price.

## Common mistakes

- "It's in Azure, so it's backed up." Not unless you configured backup.
- "Azure is ISO 27001 certified, so we are." Certification covers the platform only.
- Leaving default network settings on PaaS services, which often allow public access.
- Assuming content filters make an AI application safe. They are one layer; application design, permissions and evaluation are the rest.
- Deploying Microsoft 365 Copilot without reviewing existing SharePoint permissions. Copilot respects permissions, so oversharing becomes instantly discoverable.
- Forgetting that responsibility for agent actions sits with whoever configured the agent's tools.

## Check yourself

1. Which three responsibilities always stay with the customer?
2. Who patches the guest OS of an Azure VM? Of an App Service plan?
3. A storage account is publicly readable and data leaks. Whose failure is it and why?
4. Name the three layers of the AI shared responsibility model.
5. On Foundry, who is responsible for limiting what an agent's tools can do?
6. Your client asks whether using Azure makes them GDPR compliant. How do you answer?
7. Why is "shared responsibility" a valid argument for choosing PaaS over IaaS?

<details>
<summary>Answers</summary>

1. Data, endpoints (devices), and accounts and access.
2. The customer for a VM. Microsoft for App Service.
3. The customer's. Network and access configuration of a PaaS resource is a customer responsibility.
4. AI usage, AI application, AI platform.
5. The customer.
6. No. Azure provides compliant infrastructure and tools; the client must still classify data, control access, handle subject rights and configure services correctly.
7. PaaS moves OS and runtime patching to Microsoft, reducing operational load and the attack surface the team has to manage.

</details>

## References

- [Shared responsibility in the cloud](https://learn.microsoft.com/azure/security/fundamentals/shared-responsibility)
- [Artificial intelligence (AI) shared responsibility model](https://learn.microsoft.com/azure/security/fundamentals/shared-responsibility-ai)
- [Data, privacy, and security for Azure OpenAI in Foundry Models](https://learn.microsoft.com/azure/foundry/responsible-ai/openai/data-privacy)
- [Service Trust Portal](https://servicetrust.microsoft.com/)
- [Azure compliance documentation](https://learn.microsoft.com/azure/compliance/)
