# Cloud Computing Concepts

> Topic: 01 · Cloud and Azure Fundamentals · Tracks: E ● A ● C ● · Status checked: 2026-10

## Why it matters

Every Azure conversation starts with a choice about how much you want to manage yourself. An engineer makes that choice when picking a hosting option. An architect makes it when trading control for speed. A consultant makes it when explaining to a finance director why the bill is now monthly and variable. If the vocabulary here is shaky, everything built on it is too.

## Key concepts

### What cloud computing is

Cloud computing is renting computing resources (servers, storage, databases, networking, software, AI models) over the internet, paying for what you use, and letting the provider run the physical infrastructure.

The standard definition (NIST SP 800-145) lists five characteristics. They are worth knowing because they separate real cloud from "someone else's data centre":

| Characteristic | Meaning | Azure example |
|---|---|---|
| On-demand self-service | You provision without asking a human | `az vm create` |
| Broad network access | Reachable over standard networks | Portal, REST API, SDKs |
| Resource pooling | Many customers share physical hardware, logically isolated | Multitenant hosts and hypervisor isolation |
| Rapid elasticity | Capacity grows and shrinks quickly | Scale sets, Container Apps scaling to zero |
| Measured service | Usage is metered and billed | Per-second VM billing, per-token model billing |

### Service models

The service model says where the line sits between what the provider runs and what you run.

| Layer | On-premises | IaaS | PaaS | SaaS |
|---|:-:|:-:|:-:|:-:|
| Application | You | You | You | Provider |
| Data | You | You | You | You* |
| Runtime | You | You | Provider | Provider |
| Middleware | You | You | Provider | Provider |
| Operating system | You | You | Provider | Provider |
| Virtualisation | You | Provider | Provider | Provider |
| Servers, storage, network | You | Provider | Provider | Provider |
| Data centre | You | Provider | Provider | Provider |

\* In SaaS you still own your data and who can access it, even though you don't run the system it lives in.

- **Infrastructure as a Service (IaaS):** raw compute, storage and networking. Most control, most work. *Virtual Machines, Virtual Network, Managed Disks.*
- **Platform as a Service (PaaS):** a managed platform to run your code or data. You bring the app, the provider patches and scales the platform. *App Service, Azure SQL Database, Azure Kubernetes Service, Microsoft Foundry.*
- **Software as a Service (SaaS):** a finished application you configure and use. *Microsoft 365, Dynamics 365, Microsoft 365 Copilot.*

Three terms sit alongside these:

- **Serverless:** PaaS where you don't size or see servers at all, capacity scales automatically (often to zero), and you pay per execution or per second of use. *Azure Functions, Container Apps, Logic Apps, serverless SQL.*
- **Containers as a Service (CaaS):** you hand over a container image and the platform runs it. *Container Apps, Container Instances.*
- **Models as a Service (MaaS):** you call a hosted AI model through an API and pay per token. *Foundry Models standard deployments.* It behaves like serverless PaaS: no GPUs to manage.

Real solutions mix models. A typical AI application uses MaaS for the model, PaaS for search and the database, serverless for the API, and perhaps IaaS for one legacy integration server.

### Deployment models

| Model | What it is | When it fits |
|---|---|---|
| Public cloud | Provider-owned infrastructure shared by many customers | The default for new workloads |
| Private cloud | Infrastructure dedicated to one organisation | Strict isolation, existing data centre investment |
| Hybrid cloud | Public and private connected and managed together | Data that must stay on-premises, low-latency edge, phased migration |
| Multicloud | More than one public cloud provider | Acquisitions, best-of-breed services, regulatory diversification |

Azure's hybrid and multicloud story is **Azure Arc** (manage resources anywhere through the Azure control plane) and **Azure Local** (run Azure services on your own hardware). Both are covered in topic 31.

### The economics: CapEx and OpEx

- **Capital expenditure (CapEx):** money spent up front on assets you own, depreciated over years. Buying servers.
- **Operational expenditure (OpEx):** money spent as you go on services you consume. Paying a monthly Azure bill.

Moving to cloud shifts spend from CapEx to OpEx. That is not automatically cheaper. It changes the shape of the cost:

| | On-premises | Cloud |
|---|---|---|
| Up-front cost | High | None or low |
| Capacity planning | Buy for peak, years ahead | Provision for now, adjust |
| Cost of being wrong | Idle hardware or outages | A larger or smaller bill next month |
| Predictability | High once purchased | Variable unless you commit |
| Who controls spend | Procurement | Whoever can deploy |

The last row is the one that surprises organisations. In the cloud an engineer can commit the company to cost with one command, which is why cost governance (topic 29) exists.

Azure softens the variability with **commitment discounts**: reservations and savings plans trade a one or three year commitment for a lower rate. For AI, **provisioned throughput** does the same for model capacity.

### The benefits, stated honestly

| Benefit | What it means | The catch |
|---|---|---|
| High availability | Services stay up through failures | Only if you design for it and choose the right tier |
| Scalability | Handle more load by adding resources | Scale up (bigger) has ceilings; scale out (more) needs stateless design |
| Elasticity | Scale automatically with demand | Needs autoscale rules and quota headroom |
| Reliability | Recover from failure | Multi-zone and multi-region cost more |
| Predictability | Consistent performance and forecastable cost | Requires monitoring and budgets |
| Security | Provider-grade physical and platform security | Your configuration is still your responsibility |
| Governance | Central policy and audit | Must be set up; it is not on by default |
| Manageability | Everything is an API | Without automation it is just a slower data centre |
| Agility | Days to minutes for new environments | Sprawl, if nobody owns clean-up |

**Scalability vs elasticity** is a common exam and interview question. Scalability is the *ability* to handle growth. Elasticity is doing so *automatically and in both directions*. A system you can manually resize is scalable but not elastic.

**Vertical vs horizontal scaling:** vertical (scale up) means a bigger instance and usually a restart. Horizontal (scale out) means more instances behind a load balancer and no downtime, but the application must tolerate requests landing on different instances.

## How it works on Azure

A quick mapping of the concepts to services you will meet later:

| Need | IaaS | PaaS | Serverless |
|---|---|---|---|
| Run code | Virtual Machines | App Service, AKS | Functions, Container Apps |
| Relational data | SQL Server on a VM | Azure SQL Database (provisioned) | Azure SQL Database serverless |
| Messaging | Self-hosted broker on VMs | Service Bus Premium | Event Grid, Service Bus Standard |
| AI model | Open model on GPU VMs | Foundry managed compute, AKS with KAITO | Foundry Models standard deployment |

The AI row matters for this roadmap. Self-hosting a model on GPU VMs gives full control and predictable per-hour cost, but you carry GPU quota, patching, scaling and model updates. A serverless model deployment removes all of that and bills per token. Most projects should start serverless and only move left with a measured reason.

## Design decisions and trade-offs

**Default to the highest abstraction that meets the requirement.** Start at SaaS, then serverless, then PaaS, and drop to IaaS only when something forces you. Typical forcing reasons:

- A dependency that needs OS-level access or a specific kernel or driver.
- Software licensed per host or certified only on VMs.
- A regulator or contract that demands dedicated hardware.
- Sustained, predictable load where reserved IaaS is clearly cheaper than per-use pricing.

**Serverless is not always cheaper.** Per-execution pricing wins for spiky or low traffic. At steady high volume a provisioned plan often costs less. The same curve applies to AI: pay-per-token is cheapest until usage is high and steady, at which point provisioned throughput wins.

**Hybrid is a requirement, not a strategy.** Choose it because data, latency or regulation demands it. Running two environments is always more work than running one.

**Multicloud for resilience rarely pays off.** A multi-region design inside one provider covers almost every realistic failure at far lower complexity.

## Hands-on

No deployment in this file. Do this classification exercise instead: for each service, write down the service model and one thing you still manage. Answers are below.

1. Azure Virtual Machines
2. Azure Kubernetes Service
3. Azure Functions (Flex Consumption)
4. Azure SQL Database
5. Microsoft 365 Copilot
6. A GPT model deployed in Microsoft Foundry
7. Azure Container Apps
8. Azure Blob Storage

<details>
<summary>Answers</summary>

1. IaaS. You patch the OS and everything above it.
2. PaaS (managed Kubernetes). The control plane is managed; you still manage node pools (less so with AKS Automatic), workloads and cluster configuration.
3. Serverless PaaS. You manage your code and its configuration.
4. PaaS. You manage schema, queries, indexes and access.
5. SaaS. You manage who has licences, what data they can reach, and governance policies.
6. Models as a Service (serverless PaaS). You manage prompts, content filter configuration, access and token spend.
7. Serverless containers. You manage the image and scaling rules.
8. PaaS. You manage access, lifecycle rules, redundancy choice and the data itself.

</details>

## Security, cost and reliability notes

- **Security:** the higher the abstraction, the more security work the provider does, but identity, data and configuration are always yours. See [Shared Responsibility Model](02-shared-responsibility-model.md).
- **Cost:** the consumption model rewards switching things off. Dev and test environments that run 24x7 are the most common waste.
- **Reliability:** cloud infrastructure fails routinely by design. Availability comes from redundancy you configure, not from the word "cloud".

## Common mistakes

- Treating cloud as a hosting location and lifting servers across unchanged, then being surprised that it costs more.
- Saying "PaaS means no operations". There is less infrastructure work, but monitoring, cost, access and configuration remain.
- Confusing scalability with elasticity.
- Assuming serverless has no limits. It has quotas, cold starts and timeouts.
- Describing a company as "multicloud" because it uses Microsoft 365 and AWS. That is true but says nothing about workload architecture.
- Presenting the CapEx to OpEx shift as a saving. It is a change in cash-flow shape; savings come from right-sizing and elasticity.

## Check yourself

1. Name the five NIST characteristics of cloud computing.
2. A team wants to run a containerised API with no cluster management and scale to zero overnight. Which service model and which Azure service?
3. What is the difference between scalability and elasticity?
4. Give two legitimate reasons to choose IaaS over PaaS.
5. Why can a serverless service cost more than a provisioned one?
6. A client says "we are moving to the cloud to save money." What would you ask before agreeing?
7. In SaaS, what does the customer remain responsible for?

<details>
<summary>Answers</summary>

1. On-demand self-service, broad network access, resource pooling, rapid elasticity, measured service.
2. Serverless containers (CaaS/PaaS). Azure Container Apps.
3. Scalability is the capacity to handle more load by adding resources. Elasticity is automatic scaling up and down to match demand.
4. Any two of: OS-level access or custom drivers, vendor certification or licensing tied to VMs, dedicated hardware requirements, steady load where reserved VMs are cheaper.
5. Per-use pricing has a higher unit rate. At steady high utilisation the total exceeds a provisioned plan.
6. What the current fully loaded cost is, how variable the load is, whether workloads will be re-architected or just moved, who will own cost governance, and what the business value of agility is.
7. Their data, identities and access, device and account security, and configuration of the service.

</details>

## References

- [What is cloud computing?](https://azure.microsoft.com/resources/cloud-computing-dictionary/what-is-cloud-computing)
- [Describe cloud computing (Microsoft Learn module)](https://learn.microsoft.com/training/modules/describe-cloud-compute/)
- [Describe cloud service types](https://learn.microsoft.com/training/modules/describe-cloud-service-types/)
- [Choose an Azure compute service](https://learn.microsoft.com/azure/architecture/guide/technology-choices/compute-decision-tree)
- [NIST SP 800-145: The NIST Definition of Cloud Computing](https://csrc.nist.gov/pubs/sp/800/145/final)
