# Compute Decision Guide

> Topic: 06 · Compute · Tracks: E ● A ● C ● · Status checked: 2026-10

## Why it matters

"Where should this run?" is asked on every project, and the wrong answer is costly to reverse: too low-level and a small team drowns in operations; too high-level and they hit a wall the platform will not let them through. Azure has a dozen plausible compute services. This guide gives you a repeatable way to choose, the comparison you need to defend the choice, and specific guidance for the pieces of an AI application. Topics 07 and 10 cover the container and application platforms in depth; this is the map between them.

## Key concepts

### The governing principle

**Choose the highest level of abstraction that meets the requirements.**

```text
More control, more operations                                   Less control, less operations
◄──────────────────────────────────────────────────────────────────────────────────────────►
 Virtual Machines   AKS   Container Apps / App Service   Functions   Managed AI services (Foundry)
 (IaaS)             (managed Kubernetes)   (PaaS / serverless containers)   (FaaS)   (model as a service)
```

Each step right hands more to the platform: patching, scaling, availability, networking plumbing. Step left only for a specific, named requirement. "We might need the flexibility" is not one.

### The candidates

| Service | What you deploy | Scaling | You manage | Billing |
|---|---|---|---|---|
| **Virtual Machines / Scale Sets** | An OS image and everything on it | Manual or autoscale rules | OS, runtime, patching, HA, scaling | Per VM second |
| **Azure Kubernetes Service (AKS)** | Containers, as Kubernetes manifests | Pod and node autoscaling | Cluster configuration, node pools, upgrades, add-ons, workloads. Less with **AKS Automatic** | Per node VM; optional control-plane tiers |
| **Azure Container Apps** | Containers | Automatic, event-driven (KEDA), **to zero** | The container and scaling rules | Per second of vCPU and memory used (consumption), or dedicated profiles |
| **Azure App Service** | Code or a container | Manual or autoscale within a plan | App code and configuration | Per plan instance, always on |
| **Azure Functions** | Functions (code) triggered by events | Automatic, event-driven, to zero on consumption plans | Function code | Per execution and resource time, or a premium/dedicated plan |
| **Azure Container Instances** | A container group | None built in (one instance per group) | The container | Per second |
| **Static Web Apps** | A static front end plus optional API | Automatic | Front-end code | Free and standard tiers |
| **Azure Batch** | Jobs and tasks | Pool autoscale formulas | Job definitions, pools | Per node VM |
| **Azure Red Hat OpenShift** | Containers on OpenShift | OpenShift autoscaling | Cluster workloads; platform jointly managed | Per node plus OpenShift fee |
| **Azure Logic Apps** | Workflows (designer or code) | Automatic | Workflow definitions | Per action, or a plan |
| **Foundry (Models, Agent Service)** | Prompts, agent definitions, or agent code | Managed | Application logic and configuration | Per token, per provisioned unit, or per agent runtime usage |

### The decision questions

Ask them in order. The first decisive answer usually settles it.

1. **Can a managed service do this entirely?** Model inference: Foundry Models. Agent runtime: Foundry Agent Service. Workflow integration: Logic Apps. Static site: Static Web Apps. If yes, stop.
2. **Is it a lift-and-shift of an existing server, or does it need OS-level control, special drivers or licensed software tied to a machine?** → Virtual Machines.
3. **Is the workload event-driven, short-lived functions?** (React to a queue message, a blob, a timer, an HTTP call.) → Azure Functions.
4. **Is it a web application or API from a mainstream stack, with no need for containers?** → App Service.
5. **Is it containerised (or should it be), without a need for the Kubernetes API?** → Azure Container Apps.
6. **Do you need Kubernetes itself**: the API, custom operators, a service mesh of your choosing, specific networking, very large scale, or an existing Kubernetes practice? → AKS.
7. **Is it a large batch or HPC job?** → Batch, Container Apps jobs, or CycleCloud (files 08 and 09).

### Decision tree

```text
Start
 │
 ├─ Is it model inference or an agent, and does a Foundry managed option fit?
 │     └─ Yes → Foundry Models / Foundry Agent Service
 │
 ├─ Lift-and-shift, OS control, specialised hardware, or machine-tied licence?
 │     └─ Yes → Virtual Machines (scale sets for multiples)
 │              VMware / Oracle / SAP estates → file 11
 │
 ├─ Short event-driven code units?
 │     └─ Yes → Azure Functions      (Flex Consumption by default)
 │
 ├─ Static front end?
 │     └─ Yes → Static Web Apps
 │
 ├─ Standard web app / API, code deployment, steady traffic?
 │     └─ Yes → App Service
 │
 ├─ Containers, microservices, APIs, background workers, jobs; want scale to zero and little ops?
 │     └─ Yes → Azure Container Apps
 │
 ├─ Need Kubernetes API, operators, custom networking, GPUs at scale, multi-team platform?
 │     └─ Yes → AKS   (Automatic unless you need Standard's control)
 │
 └─ Large parallel batch / HPC?
       └─ Yes → Batch / Container Apps jobs / CycleCloud
```

### Comparison on the dimensions that matter

| Dimension | VMs | AKS | Container Apps | App Service | Functions |
|---|---|---|---|---|---|
| **Operational effort** | Highest | High (medium with Automatic) | Low | Low | Lowest |
| **Control over environment** | Full | High | Moderate | Moderate | Low |
| **Scale to zero** | No | Nodes can, slowly | **Yes** | No | **Yes** (consumption plans) |
| **Start-up / cold start** | Minutes | Seconds for pods; minutes for new nodes | Seconds (longer with large images) | None when always on | Milliseconds to seconds, plan-dependent |
| **Long-running processes** | Yes | Yes | Yes | Yes | Limited by plan timeouts; Durable Functions for orchestration |
| **Background workers, queue consumers** | Yes | Yes | Yes, with event-driven scaling | WebJobs, limited | Native |
| **Scheduled / one-off jobs** | Cron on a VM | Jobs, CronJobs | **Jobs** | WebJobs | Timer triggers |
| **GPU** | Any N-series | GPU node pools | Serverless GPUs (limited sizes) | No | No |
| **Windows containers / .NET Framework** | Yes | Yes | No (Linux only) | Yes | Yes (Windows plans) |
| **VNet integration and private ingress** | Native | Native | Yes | Yes | Yes (plan-dependent) |
| **Built-in ingress, TLS, revisions, traffic splitting** | Build it | Add-ons / ingress controllers | **Built in** | Slots, built-in TLS | Built in |
| **Service discovery, sidecars, Dapr** | Build it | Yes | **Built in** | Limited (sidecars supported) | Bindings |
| **Portability** | Image-level | High (standard Kubernetes) | Medium (containers; platform-specific config) | Low to medium | Low to medium |
| **Team skill required** | Sysadmin | Kubernetes | Containers | Web development | Web development |
| **Cost at low/spiky traffic** | Poor | Poor to fair | **Good** | Fair | **Good** |
| **Cost at steady high traffic** | Good with commitments | Good with commitments | Good on dedicated profiles | Good | Fair to good on premium plans |

### The frequent head-to-heads

**Container Apps or AKS?**

| Choose Container Apps when | Choose AKS when |
|---|---|
| You want to run containers, not operate a platform | You need the Kubernetes API or ecosystem: operators, custom resource definitions, Helm charts that assume a cluster |
| The team has no Kubernetes specialists | A platform team already runs Kubernetes well |
| Scale to zero and per-second billing matter | You need fine control over nodes, networking (CNI choices, network policy engines), storage, scheduling |
| Microservices, APIs, workers and jobs with standard needs | Very large or specialised workloads: many GPU node pools, stateful systems with operators, service mesh requirements |
| You want built-in ingress, revisions, Dapr, KEDA | You must stay portable across clouds and on-premises with identical manifests |

Container Apps is itself built on Kubernetes, with the cluster hidden. Start there; move to AKS when you can name what you need that it lacks.

**App Service or Container Apps?**

| Choose App Service when | Choose Container Apps when |
|---|---|
| A conventional web app or API in a supported stack, deployed as code | The app is containerised, or there are several services |
| Windows or .NET Framework | Background workers, event-driven scaling, jobs |
| The team values the simplest deployment and mature features: slots, built-in auth, diagnostics | Scale to zero, per-second billing, revisions with traffic splitting |
| Steady traffic on an always-on plan | Microservices needing service discovery, sidecars, Dapr |

**Functions or Container Apps?**

| Choose Functions when | Choose Container Apps when |
|---|---|
| The logic is naturally a set of event handlers | A long-running service or custom server process |
| You want the triggers and bindings model and Durable Functions for orchestration | You want full control of the container and runtime |
| Short executions, bursty load | Requests or jobs that run long |

They also combine: Functions can be hosted on Container Apps, getting the programming model on that platform.

**VMs or anything else?** VMs when you must: lift-and-shift, OS-level needs, specialised hardware not offered elsewhere. If the workload can be containerised, it almost always should be.

### What else shapes the decision

| Factor | Effect |
|---|---|
| **Team skills** | The strongest practical constraint. A platform the team cannot operate is the wrong choice whatever its merits |
| **Existing standards** | An organisation with a mature AKS platform should use it. One with none should not build one for a single app |
| **Compliance and isolation** | All options support private networking. Dedicated compute, confidential computing or host-level control push toward AKS or VMs |
| **Traffic shape** | Spiky and idle-heavy favours scale-to-zero. Steady favours reserved capacity |
| **State** | Stateless is easy everywhere. Stateful workloads belong in managed data services, or in AKS or VMs if they must be self-hosted |
| **Startup time** | Large images and model loading lengthen cold starts. Keep minimum replicas for interactive paths |
| **Portability** | Containers keep options open. Proprietary programming models tie you in more |
| **Hybrid and edge** | AKS enabled by Arc, Azure Local, Container Apps on Arc |

### Mixing is normal

Most solutions use several: a front end on Static Web Apps, an API on Container Apps, event handlers on Functions, a workflow in Logic Apps, models in Foundry, and one legacy component on a VM. Choose per component, but keep the **number of distinct platforms** small enough that the team can operate them all well.

## How it works on Azure

### Compute for the parts of an AI application

| Component | Characteristics | Recommended | Alternatives |
|---|---|---|---|
| **Model inference (hosted models)** | No compute to manage | **Foundry Models** | – |
| **Model inference (open or custom models)** | GPU, large images, slow start | **Foundry managed compute / Azure ML endpoints**, or **Container Apps serverless GPU** for bursty use | AKS with KAITO; GPU VMs (file 07) |
| **Agent runtime** | Stateful conversations, tool calls, long-running | **Foundry Agent Service** (prompt or hosted agents) | Your own agent code on Container Apps or AKS |
| **Chat / orchestration API** | HTTP, streaming, I/O-bound, async | **Container Apps** | App Service; AKS; Functions for simple cases |
| **Web front end** | Static assets plus API calls | **Static Web Apps**, or served from the same container app | App Service |
| **MCP servers and agent tools** | Small HTTP services, often idle, need auth | **Azure Functions** or **Container Apps** | App Service; behind API Management |
| **Document ingestion pipeline** | Event-driven, bursty, parallel | **Functions** (blob or queue triggers, Durable Functions for fan-out) or **Container Apps jobs** | Batch for very large corpora; Fabric or Databricks for data-heavy pipelines |
| **Scheduled jobs** (re-indexing, evaluation runs) | Periodic, finite | **Container Apps jobs** or timer-triggered **Functions** | AKS CronJobs; pipeline schedules |
| **Code execution sandbox for agents** | Untrusted code, strong isolation, fast start | **Container Apps dynamic sessions**, or the Foundry code interpreter tool | – |
| **Long-running workflows with human approval** | Durable state, waits | **Durable Functions**, **Logic Apps**, or Agent Framework workflows on a durable host | – |
| **AI gateway** | Policy, routing, token limits | **API Management** | – |
| **Batch inference at scale** | Asynchronous, large volume | **Foundry batch deployments** for hosted models | Batch or AKS on GPUs for self-hosted |
| **Training and fine-tuning** | GPU, long-running, checkpointed | **Foundry fine-tuning** (managed) | Azure ML compute; AKS; CycleCloud |

A sensible default stack for a new AI application with a small team: **Foundry** for models and agents, **Container Apps** for the API and workers, **Functions** for event-driven ingestion and tools, **API Management** when more than one consumer shares models. Reach for AKS when an existing platform or a specific need calls for it.

### Why Container Apps suits AI applications

- **Scale to zero**, useful because AI features often have idle periods and separate dev and test environments.
- **Event-driven scaling** on HTTP concurrency, queue length and custom metrics.
- **Jobs** for ingestion and evaluation alongside the service.
- **Long-lived HTTP connections** for streaming responses, with configurable timeouts.
- **Managed identity** and VNet integration for keyless, private access to Foundry and data.
- **Dynamic sessions** for sandboxed code execution.
- **Serverless GPUs** for the occasional self-hosted model.
- **Revisions and traffic splitting** for safe rollout and A/B comparison of prompt or model changes.

Characteristics of AI workloads to design for, whichever host you choose:

| Characteristic | Implication |
|---|---|
| **I/O-bound, long requests** | Use async frameworks; scale on concurrent requests, not CPU |
| **Streaming** | Disable buffering at every proxy; raise timeouts |
| **Bursty, expensive downstream calls** | Rate limits and back-pressure; queue non-interactive work |
| **Cold starts hurt interactive latency** | Minimum replicas of at least one on user-facing services |
| **State lives elsewhere** | Conversation history and agent state in a database or the agent service, so any replica can serve any request |
| **Long-running agent tasks** | Use a durable or background execution pattern, not a single HTTP request |

### Checking a decision

Before committing, confirm against a short list:

| Check | Question |
|---|---|
| **Requirements** | Does the service meet every hard requirement: runtime, OS, protocol, timeout, GPU, region? |
| **Limits** | Request duration, payload size, concurrency, instance counts, image size |
| **Networking** | Private ingress and egress as the landing zone requires |
| **Identity** | Managed identity support |
| **Scaling behaviour** | Signal, speed, minimum and maximum, cold start |
| **Availability** | Zone redundancy; SLA |
| **Operations** | Who patches, upgrades and is on call? Does the team have the skills? |
| **Cost** | Priced at expected, low and peak load; idle cost |
| **Exit** | How hard is it to move later? |

A one-day spike deploying a thin slice on the candidate platform answers most of these better than a week of reading.

## Design decisions and trade-offs

| Tension | How to resolve it |
|---|---|
| **Control vs effort** | Default to less control. Pay for control only where a requirement demands it |
| **Standardisation vs best fit** | One or two standard application platforms per organisation beat a perfect choice per app. Deviate deliberately |
| **Portability vs productivity** | Containers give most of the portability that matters. Do not accept large operational cost for theoretical portability |
| **Scale to zero vs latency** | Zero for background and non-production; warm minimums for interactive paths |
| **Now vs later** | Choose for the next 12 to 18 months and keep the app containerised so moving is feasible. Do not build for scale you may never have |
| **Managed AI services vs self-hosting** | Managed first. Self-host with evidence (file 07) |

**Migration paths that are realistic:** App Service to Container Apps (containerise), Container Apps to AKS (same containers, new manifests), VMs to containers (often the largest effort). Functions and Logic Apps involve more rewriting to leave. Starting on Container Apps keeps the widest set of exits.

## Hands-on

**Decision exercise.** For each scenario, choose the compute service or combination, and justify it in two sentences. State one requirement that, if it changed, would change your answer.

1. An internal HR assistant: chat API with streaming, calls a Foundry agent, about 500 users in office hours, nothing at night. Team of three developers, no Kubernetes experience.
2. A document ingestion pipeline: PDFs land in Blob Storage in bursts of up to 50,000, each needs parsing, chunking, embedding and indexing.
3. A fifteen-year-old .NET Framework application on Windows Server with a third-party licensed component that installs as a Windows service.
4. A SaaS platform with 40 microservices, three engineering teams, an existing Kubernetes practice, and customers who require deployment in their own cloud accounts as well.
5. A remote MCP server exposing five tools over an internal API, called a few hundred times a day by agents.
6. A fine-tuned open-weight 8-billion-parameter model used by one feature, with about 2,000 requests a day in two daily bursts.
7. A nightly evaluation run of 5,000 test conversations against the production agent, producing a report.
8. An agent that needs to run Python code it writes itself to analyse uploaded spreadsheets.
9. A public marketing site with a contact form and a small chatbot widget.
10. A risk engine running 200,000 independent simulations every night, each about two minutes of CPU, tolerant of retries.
11. A human-in-the-loop expense approval process: an agent prepares a decision, a manager approves in Teams hours or days later, then systems are updated.
12. An organisation wants models to run inside its own data centre with no internet connection.

<details>
<summary>Suggested answers</summary>

1. **Container Apps** for the API (minimum one replica in office hours, zero at night) with Foundry Agent Service. Low operations, scale to zero, streaming support. Would change with an existing AKS platform standard.
2. **Functions** with blob or queue triggers and Durable Functions fan-out, or **Container Apps jobs** scaled on queue length. Bounded by embedding rate limits, so queue and throttle. For millions of documents, consider Batch or the model's batch API.
3. **Virtual Machines** (or App Service on Windows if the component could be removed). The Windows service and licence tie it to a machine. Would change if the component were replaced.
4. **AKS**. Scale, team structure, existing skills and the multi-cloud portability requirement all point to standard Kubernetes.
5. **Azure Functions** (or Container Apps), behind Entra authentication and optionally API Management. Small, idle most of the time, HTTP.
6. **Container Apps serverless GPU** or a managed endpoint that scales down between bursts. Too little volume for a standing GPU fleet; cold start must be acceptable or a warm replica kept during burst windows. First check whether a hosted model would do.
7. **Container Apps job** on a schedule (or a pipeline job). Finite, periodic, no need for a running service.
8. **Container Apps dynamic sessions** or the Foundry code interpreter tool. Untrusted code needs strong, fast-starting isolation, never the application's own process.
9. **Static Web Apps** with a managed API or linked Functions, the chatbot calling a small API in front of Foundry with rate limiting.
10. **Azure Batch** with Spot nodes and autoscale to zero. Large, embarrassingly parallel, retry-tolerant.
11. **Durable Functions** or **Logic Apps** (or an Agent Framework workflow on a durable host). Long waits with persisted state and an approval step.
12. **Azure Local** with GPU nodes running open models on AKS enabled by Arc, or Foundry Local for device-scale use. A sovereignty and disconnection requirement that managed cloud services cannot meet.

</details>

**Build exercise.** Deploy the same small containerised API (a "hello" endpoint plus a streaming endpoint) to App Service and to Container Apps. Compare: deployment steps, time to first response after idle, how each scales under a simple load test, and the bill for a day of light use. Then delete both.

```bash
RG=rg-compute-compare; LOC=swedencentral
az group create -n $RG -l $LOC -o none

# Container Apps: build from source, scale 0..5 on HTTP concurrency
az containerapp up -g $RG -n ca-compare --source . --ingress external --target-port 8000
az containerapp update -g $RG -n ca-compare --min-replicas 0 --max-replicas 5 \
  --scale-rule-name http --scale-rule-type http --scale-rule-http-concurrency 20

# App Service: same code, always-on plan
az webapp up -g $RG -n app-compare-$RANDOM --runtime "PYTHON:3.12" --sku B1

az group delete -n $RG --yes --no-wait
```

## Security, cost and reliability notes

- **Security:** every option supports managed identity and private networking; use both. Less to manage means less to misconfigure, which is itself a security argument for higher abstraction.
- **Security:** code written by a model must run in an isolated sandbox, never in your application's process or container.
- **Cost:** compare at three load levels (idle, typical, peak), not one. Scale-to-zero services win when idle; reserved, always-on capacity wins when busy.
- **Cost:** include people. A cluster that needs half an engineer to run is not cheaper than a managed platform because its node price is lower.
- **Reliability:** check zone redundancy and minimum replica settings on whatever you choose. A service that scales to zero with no warm instance has a latency and availability characteristic you must accept explicitly.
- **Reliability:** the platform's limits (timeouts, payload size, concurrency) become your application's limits. Read them before designing around them.

## Common mistakes

- AKS for a single small application because Kubernetes is familiar or fashionable.
- VMs for a containerisable application, inheriting patching and scaling work.
- Functions for long-running request handlers that hit plan timeouts.
- Scale to zero on an interactive endpoint, then complaints about the first request.
- A different platform for every component, more than the team can run.
- Self-hosted models on GPUs where a managed model would do.
- Running model-generated code in the application container.
- Choosing by a single dimension, usually unit price, and ignoring operations and skills.
- Ignoring platform limits until production traffic finds them.
- Designing for a scale the product does not yet have.
- Treating the first choice as permanent, or the reverse: re-platforming repeatedly without a concrete reason.

## Check yourself

1. State the governing principle for choosing compute.
2. What are the first three questions to ask, in order?
3. Give three reasons to choose AKS over Container Apps.
4. Give three reasons to choose Container Apps over App Service.
5. When is a virtual machine the right answer?
6. Which service would you use for event-driven document ingestion, and which for a scheduled evaluation run?
7. Where should code generated by an agent be executed?
8. Why does a small team's skill set weigh so heavily in the decision?
9. Name four characteristics of AI application workloads that affect how you configure the host.
10. What is a sensible default compute stack for a new AI application built by a small team?

<details>
<summary>Answers</summary>

1. Choose the highest level of abstraction that meets the requirements, and step down only for a specific, named need.
2. Can a managed service do it entirely? Is it a lift-and-shift or does it need OS-level control? Is it event-driven, short-lived code?
3. Any three of: need for the Kubernetes API, operators or custom resources; fine control over nodes, networking and scheduling; very large or specialised workloads; an existing Kubernetes platform and skills; portability across clouds with identical manifests.
4. Any three of: the app is containerised or has multiple services; scale to zero and per-second billing; event-driven scaling and jobs; built-in revisions, traffic splitting, service discovery and Dapr.
5. For lift-and-shift, software needing OS-level access or machine-tied licences, and specialised hardware not offered by higher-level services.
6. Functions (or Container Apps jobs scaled on a queue) for ingestion; a scheduled Container Apps job (or timer-triggered Function) for the evaluation run.
7. In an isolated sandbox such as Container Apps dynamic sessions or the Foundry code interpreter tool.
8. Because a platform the team cannot operate well will be unreliable and insecure regardless of its technical fit, and operational effort is a large part of true cost.
9. Any four of: long, I/O-bound requests; streaming responses; bursty and expensive downstream calls; sensitivity to cold starts; externalised state; long-running agent tasks.
10. Foundry for models and agents, Container Apps for the API and workers, Functions for event-driven ingestion and tools, and API Management when several consumers share models.

</details>

## References

- [Choose an Azure compute service](https://learn.microsoft.com/azure/architecture/guide/technology-choices/compute-decision-tree)
- [Choose an Azure container service](https://learn.microsoft.com/azure/architecture/guide/choose-azure-container-service)
- [Comparing Container Apps with other Azure container options](https://learn.microsoft.com/azure/container-apps/compare-options)
- [Azure Functions hosting options](https://learn.microsoft.com/azure/azure-functions/functions-scale)
- [App Service overview](https://learn.microsoft.com/azure/app-service/overview)
- [What is AKS Automatic?](https://learn.microsoft.com/azure/aks/intro-aks-automatic)
- [Dynamic sessions in Azure Container Apps](https://learn.microsoft.com/azure/container-apps/sessions)
- [Application platform considerations for AI workloads (Well-Architected Framework)](https://learn.microsoft.com/azure/well-architected/ai/application-platform)
- [Microsoft Foundry hosting options for agents](https://learn.microsoft.com/azure/foundry/agents/overview)
- [Choose a compute option for microservices](https://learn.microsoft.com/azure/architecture/microservices/design/compute-options)
