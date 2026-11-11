# Glossary

## Why it matters

Azure and AI conversations are full of acronyms, and many terms mean different things to different audiences. "Agent", "grounding", "PTU" and "landing zone" are common examples. A shared vocabulary speeds up workshops, avoids expensive misunderstandings in statements of work and helps new team members ramp up. This glossary defines the acronyms and terms used across this roadmap and points to the topic where each is explained in depth. For old and new product names, see the [Product Rename Map](02-product-rename-map.md).

## Key concepts

### A–C

| Term | Meaning | Topic |
|---|---|---|
| **A2A** | Agent2Agent protocol: open protocol for agents to discover and communicate with other agents | [19](../19-agent-frameworks-and-protocols/README.md) |
| **ACA** | Azure Container Apps | [07](../07-containers-and-kubernetes/README.md) |
| **ACR** | Azure Container Registry | [07](../07-containers-and-kubernetes/README.md) |
| **ACS** | Azure Communication Services | [37](../37-capstone-projects/04-voice-agent.md) |
| **ADR** | Architecture decision record | [33](../33-solution-architecture/README.md) |
| **Agent** | AI system that uses a model, instructions and tools to pursue goals, taking actions over multiple steps | [18](../18-agents-with-foundry-agent-service/README.md) |
| **AHB** | Azure Hybrid Benefit: use existing Windows Server or SQL Server licences on Azure | [29](../29-cost-management-and-finops/README.md) |
| **AKS** | Azure Kubernetes Service | [07](../07-containers-and-kubernetes/README.md) |
| **ALM** | Application lifecycle management | [27](../27-devops-and-platform-engineering/README.md) |
| **ALZ** | Azure landing zone | [03](../03-governance-and-landing-zones/README.md) |
| **AMA** | Azure Monitor Agent | [26](../26-monitoring-and-observability/README.md) |
| **APIM** | Azure API Management | [11](../11-integration-messaging-and-apis/README.md) |
| **ASR** | Azure Site Recovery; also attack success rate in red teaming | [28](../28-resiliency-backup-and-disaster-recovery/README.md), [22](../22-evaluation-observability-and-genaiops/README.md) |
| **AVM** | Azure Verified Modules: Microsoft-supported IaC modules | [04](../04-tooling-and-infrastructure-as-code/README.md) |
| **azd** | Azure Developer CLI | [04](../04-tooling-and-infrastructure-as-code/README.md) |
| **BCDR** | Business continuity and disaster recovery | [28](../28-resiliency-backup-and-disaster-recovery/README.md) |
| **Bicep** | Azure-native domain-specific language for IaC | [04](../04-tooling-and-infrastructure-as-code/README.md) |
| **CAF** | Cloud Adoption Framework | [03](../03-governance-and-landing-zones/README.md) |
| **Chunking** | Splitting documents into retrievable pieces for RAG | [17](../17-rag-and-knowledge/README.md) |
| **CI/CD** | Continuous integration / continuous delivery or deployment | [27](../27-devops-and-platform-engineering/README.md) |
| **CMK** | Customer-managed key | [25](../25-security/README.md) |
| **Content filter / guardrail** | Controls that detect and block harmful inputs or outputs | [23](../23-responsible-ai-security-and-governance/README.md) |
| **Context window** | Maximum tokens a model can consider in one request | [13](../13-ai-and-generative-ai-foundations/README.md) |
| **CSP** | Cloud Solution Provider (licensing via a partner); also cloud service provider | [35](../35-consulting-practice/10-licensing-and-commercials.md) |
| **CSPM** | Cloud security posture management | [25](../25-security/README.md) |

### D–I

| Term | Meaning | Topic |
|---|---|---|
| **DCR** | Data collection rule (Azure Monitor) | [26](../26-monitoring-and-observability/README.md) |
| **Deployment (model)** | A model made available at an endpoint with a deployment type and quota | [15](../15-foundry-models-and-azure-openai/README.md) |
| **DPIA** | Data protection impact assessment | [23](../23-responsible-ai-security-and-governance/README.md) |
| **DR** | Disaster recovery | [28](../28-resiliency-backup-and-disaster-recovery/README.md) |
| **DSPM** | Data security posture management | [23](../23-responsible-ai-security-and-governance/README.md) |
| **EA** | Enterprise Agreement | [35](../35-consulting-practice/10-licensing-and-commercials.md) |
| **Embedding** | Numeric vector that represents the meaning of text, images or other data | [13](../13-ai-and-generative-ai-foundations/README.md) |
| **Evaluator** | Metric or judge that scores AI outputs (groundedness, relevance, safety, task adherence) | [22](../22-evaluation-observability-and-genaiops/README.md) |
| **FinOps** | Cloud financial management practice | [29](../29-cost-management-and-finops/README.md) |
| **Fine-tuning** | Further training a model on task-specific data | [16](../16-prompt-engineering-and-model-customisation/README.md) |
| **Foundry** | Microsoft Foundry: Azure's platform for models, agents, tools, evaluation and governance | [14](../14-microsoft-foundry-platform/README.md) |
| **Function calling / tool calling** | A model producing structured calls to developer-defined functions | [18](../18-agents-with-foundry-agent-service/README.md) |
| **GA** | General availability | [file 06](06-preview-feature-policy.md) |
| **GenAIOps** | Operational practices for generative AI: evaluation, deployment, monitoring | [22](../22-evaluation-observability-and-genaiops/README.md) |
| **Grounding** | Supplying trusted data to a model so answers are based on it | [17](../17-rag-and-knowledge/README.md) |
| **Groundedness** | Degree to which an answer is supported by the provided context | [22](../22-evaluation-observability-and-genaiops/README.md) |
| **Hallucination / fabrication** | Confident output not supported by facts or context | [13](../13-ai-and-generative-ai-foundations/README.md) |
| **HLD / LLD** | High-level design / low-level design | [35](../35-consulting-practice/07-solution-design-deliverables.md) |
| **Hybrid search** | Combining keyword (BM25) and vector search, often with semantic ranking | [17](../17-rag-and-knowledge/README.md) |
| **IaC** | Infrastructure as code | [04](../04-tooling-and-infrastructure-as-code/README.md) |
| **IDP** | Intelligent document processing | [20](../20-foundry-tools/README.md) |

### K–P

| Term | Meaning | Topic |
|---|---|---|
| **KQL** | Kusto Query Language (Log Analytics, Resource Graph, Data Explorer) | [26](../26-monitoring-and-observability/README.md) |
| **LLM / SLM** | Large language model / small language model | [13](../13-ai-and-generative-ai-foundations/README.md) |
| **MACC** | Microsoft Azure Consumption Commitment | [35](../35-consulting-practice/10-licensing-and-commercials.md) |
| **MCA** | Microsoft Customer Agreement | [35](../35-consulting-practice/10-licensing-and-commercials.md) |
| **MCP** | Model Context Protocol: open protocol for connecting AI clients to tools and data | [19](../19-agent-frameworks-and-protocols/README.md) |
| **MLOps** | Operational practices for machine learning models | [21](../21-azure-machine-learning-and-mlops/README.md) |
| **Multi-agent** | Several specialised agents coordinated by an orchestration pattern | [18](../18-agents-with-foundry-agent-service/README.md), [34](../34-ai-solution-architecture/04-agentic-architecture.md) |
| **NFR** | Non-functional requirement | [33](../33-solution-architecture/03-requirements-and-constraints.md) |
| **NSG / ASG** | Network security group / application security group | [05](../05-networking/README.md) |
| **OBO** | On-behalf-of flow: a service calls another API with the user's identity | [02](../02-identity-and-access/README.md) |
| **OIDC** | OpenID Connect; workload identity federation for pipelines | [02](../02-identity-and-access/README.md), [27](../27-devops-and-platform-engineering/README.md) |
| **OpenTelemetry (OTel)** | Open standard for traces, metrics and logs | [26](../26-monitoring-and-observability/README.md) |
| **Orchestration** | Coordinating models, tools and agents (sequential, concurrent, group chat, handoff, magentic) | [19](../19-agent-frameworks-and-protocols/README.md) |
| **PaaS / IaaS / SaaS** | Platform / infrastructure / software as a service | [01](../01-cloud-and-azure-fundamentals/README.md) |
| **PIM** | Privileged Identity Management | [02](../02-identity-and-access/README.md) |
| **PoC** | Proof of concept | [35](../35-consulting-practice/05-proof-of-concept-to-production.md) |
| **Private endpoint** | Private IP in your VNet for a PaaS service (Private Link) | [05](../05-networking/README.md) |
| **Prompt injection** | Malicious instructions in user input (direct) or in content the model reads (indirect) | [23](../23-responsible-ai-security-and-governance/README.md) |
| **Prompt Shields** | Content Safety capability that detects prompt injection attacks | [23](../23-responsible-ai-security-and-governance/README.md) |
| **PTU** | Provisioned throughput unit: reserved model capacity | [15](../15-foundry-models-and-azure-openai/README.md) |

### R–Z

| Term | Meaning | Topic |
|---|---|---|
| **RAG** | Retrieval-augmented generation | [17](../17-rag-and-knowledge/README.md) |
| **RAI** | Responsible AI | [23](../23-responsible-ai-security-and-governance/README.md) |
| **RAID** | Risks, assumptions, issues and dependencies log | [35](../35-consulting-practice/11-delivery-governance.md) |
| **RBAC / ABAC** | Role-based / attribute-based access control | [02](../02-identity-and-access/README.md) |
| **Red teaming** | Adversarial testing to find safety and security weaknesses | [22](../22-evaluation-observability-and-genaiops/README.md) |
| **Reranking / semantic ranker** | Re-scoring retrieved results with a model for relevance | [17](../17-rag-and-knowledge/README.md) |
| **Responses API** | OpenAI-compatible API for stateful, tool-using model interactions; basis for agents | [15](../15-foundry-models-and-azure-openai/README.md) |
| **RPO / RTO** | Recovery point objective (data loss tolerated) / recovery time objective (downtime tolerated) | [28](../28-resiliency-backup-and-disaster-recovery/README.md) |
| **RU** | Request unit (Cosmos DB throughput) | [09](../09-databases/README.md) |
| **SAS** | Shared access signature | [08](../08-storage/README.md) |
| **SCU** | Security Compute Unit (Security Copilot capacity) | [35](../35-consulting-practice/10-licensing-and-commercials.md) |
| **Semantic cache** | Cache that returns stored answers for semantically similar prompts | [37](../37-capstone-projects/06-generative-ai-gateway.md) |
| **SIEM / SOAR** | Security information and event management / security orchestration, automation and response | [25](../25-security/README.md) |
| **SLA / SLO / SLI** | Service level agreement / objective / indicator | [28](../28-resiliency-backup-and-disaster-recovery/README.md) |
| **SoW** | Statement of work | [35](../35-consulting-practice/06-proposals-estimation-and-statements-of-work.md) |
| **Spillover** | Routing overflow traffic from provisioned to standard deployments | [34](../34-ai-solution-architecture/09-capacity-and-performance-planning.md) |
| **STP** | Straight-through processing (no human touch) | [37](../37-capstone-projects/05-intelligent-document-processing-pipeline.md) |
| **Temperature / top-p** | Sampling parameters controlling randomness of model output | [16](../16-prompt-engineering-and-model-customisation/README.md) |
| **Token** | Unit of text a model processes; basis for limits and pricing | [13](../13-ai-and-generative-ai-foundations/README.md) |
| **TPM / RPM** | Tokens per minute / requests per minute (quota measures) | [15](../15-foundry-models-and-azure-openai/README.md) |
| **UDR** | User-defined route | [05](../05-networking/README.md) |
| **Vector index / vector search** | Index and similarity search over embeddings | [17](../17-rag-and-knowledge/README.md) |
| **VNet** | Azure virtual network | [05](../05-networking/README.md) |
| **WAF** | Well-Architected Framework; also web application firewall (context decides) | [33](../33-solution-architecture/README.md), [05](../05-networking/README.md) |
| **Zero Trust** | Security model: verify explicitly, least privilege, assume breach | [25](../25-security/README.md) |
| **Zone redundancy** | Spreading a service across availability zones in a region | [28](../28-resiliency-backup-and-disaster-recovery/README.md) |

## How it works on Azure

Use this glossary in three ways:

1. **Onboarding**: give new team members this page alongside the [Roadmap](../Roadmap.md).
2. **Workshops**: define ambiguous terms (agent, grounding, landing zone) at the start of discovery sessions.
3. **Documents**: add a project-specific glossary to HLDs and SoWs, copied from here and extended with client terms.

Microsoft Learn's own glossaries, such as the Azure glossary and the Foundry concepts pages, give authoritative product definitions.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Precision vs simplicity** | Executives need plain definitions; engineers need precise ones. Keep both where it matters |
| **Ambiguous acronyms** | Spell out on first use (WAF, ASR, CSP) |
| **Maintenance** | Update when products are renamed (file 02) |

## Hands-on

1. Pick ten terms you use daily and explain each in one sentence to a non-technical colleague.
2. Add a client-specific glossary section to a design document.
3. Find three acronyms in this roadmap that have two meanings and note how to disambiguate them.

## Security, cost and reliability notes

- Misunderstood terms (for example "private" meaning a private endpoint versus simply authenticated access) can cause security gaps. Define them in requirements.
- Ambiguous cost terms (PTU, TPM, SCU, capacity units) cause budget errors. Define the units in estimates.

## Common mistakes

| Mistake | Fix |
|---|---|
| Assuming everyone knows acronyms | Spell out on first use |
| Using "agent" for every chatbot | Distinguish chat apps, single agents and multi-agent systems |
| Mixing old and new product names | Use the rename map |

## Check yourself

1. What's the difference between RPO and RTO?
2. What does PTU stand for, and why does it matter?
3. Define grounding and groundedness.
4. What does MCP do?
5. Which two meanings can WAF have?

<details>
<summary>Answers</summary>

1. RPO is the maximum data loss tolerated; RTO is the maximum downtime tolerated.
2. Provisioned throughput unit: reserved model capacity with predictable latency and cost.
3. Grounding supplies trusted data to the model; groundedness measures how well an answer is supported by that data.
4. It standardises how AI clients connect to tools and data sources (servers).
5. Well-Architected Framework, or web application firewall.

</details>

## References

- [Azure glossary](https://learn.microsoft.com/azure/azure-glossary-cloud-terminology)
- [Microsoft Foundry documentation](https://learn.microsoft.com/azure/foundry/)
- [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/)
- [Cloud Adoption Framework](https://learn.microsoft.com/azure/cloud-adoption-framework/)
