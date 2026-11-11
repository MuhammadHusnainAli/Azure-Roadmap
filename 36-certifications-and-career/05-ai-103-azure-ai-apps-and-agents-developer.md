# AI-103 Azure AI Apps and Agents Developer

## Why it matters

AI-103 (Developing AI Apps and Agents on Azure) leads to **Microsoft Certified: Azure AI Apps and Agents Developer Associate**. It replaced AI-102 (Azure AI Engineer Associate), which retired on 30 June 2026. It's the core certification for the **Azure AI Engineer** track in this roadmap. It validates that you can plan, build, secure, evaluate and operate generative AI apps and agents with **Microsoft Foundry** and Python, plus the vision, speech, text analysis and information extraction capabilities that surround them.

## Key concepts

### Skills measured (study guide, as of 16 April 2026)

| Domain | Weight | Sub-areas |
|---|---|---|
| **Plan and manage an Azure AI solution** | 25–30% | Choose Foundry services for generative AI and agents; set up AI solutions in Foundry; manage, monitor and secure AI systems; implement responsible AI across generative and agentic systems |
| **Implement generative AI and agentic solutions** | 30–35% | Build generative applications with Foundry; build agents with Foundry; optimise and operationalise generative AI systems |
| **Implement computer vision solutions** | 10–15% | Image and video generation and editing; multimodal understanding workflows; responsible AI for multimodal content |
| **Implement text analysis solutions** | 10–15% | Language model text analysis; speech solutions |
| **Implement information extraction solutions** | 10–15% | Retrieval and grounding pipelines; document content extraction |

Examples from the study guide show its scope. It covers choosing between LLMs, small language models and multimodal models, integrating Foundry projects with CI/CD, and managing quotas, rate limits and cost. On the security side, it includes keyless credentials and private networking, guardrails, and auditing with trace logging and approval workflows. For agents, it covers memory, tools, multi-agent orchestration, and governing agent behaviour with oversight modes and tool-access controls.

### AI-102 vs AI-103

| Area | AI-102 (retired) | AI-103 |
|---|---|---|
| Centre of gravity | Individual Azure AI services | Foundry, generative AI and agents |
| Agents | Light coverage | About a third of the exam, plus governance |
| Operations | Limited | CI/CD, quotas, monitoring, tracing, evaluation |
| Language | C# or Python | Python |
| Vision | Image analysis, custom vision | Image and video generation, multimodal understanding |

### Roadmap mapping

| Exam area | Roadmap topics |
|---|---|
| Foundry setup, models, deployments | [14](../14-microsoft-foundry-platform/README.md), [15](../15-foundry-models-and-azure-openai/README.md) |
| Prompting, tuning, optimisation | [16 · Prompt Engineering and Model Customisation](../16-prompt-engineering-and-model-customisation/README.md) |
| Retrieval and grounding | [17 · RAG and Knowledge](../17-rag-and-knowledge/README.md) |
| Agents, tools, multi-agent | [18 · Foundry Agent Service](../18-agents-with-foundry-agent-service/README.md), [19 · Agent Frameworks and Protocols](../19-agent-frameworks-and-protocols/README.md) |
| Vision, speech, language, documents | [20 · Foundry Tools](../20-foundry-tools/README.md) |
| Custom models (awareness) | [21 · Azure Machine Learning and MLOps](../21-azure-machine-learning-and-mlops/README.md) |
| Evaluation, tracing, monitoring | [22 · Evaluation, Observability and GenAIOps](../22-evaluation-observability-and-genaiops/README.md) |
| Responsible AI, security, guardrails | [23 · Responsible AI, Security and Governance](../23-responsible-ai-security-and-governance/README.md) |

## How it works on Azure

### Ten-week study plan (about 8 hours a week)

| Week | Focus | Build |
|---|---|---|
| 1 | Foundry resources, projects, identity, networking | Project with managed identity and keyless access |
| 2 | Model choice and deployment types, quotas | Deploy two models; compare latency, cost and quality |
| 3 | Prompting and parameters; structured output | Prompt templates in source control |
| 4 | RAG | Azure AI Search index with hybrid search and citations |
| 5 | Agents I | Agent with file search, function calling and a code interpreter |
| 6 | Agents II | Multi-agent workflow with approval steps; MCP tool |
| 7 | Vision and multimodal | Image generation and editing; multimodal document Q&A |
| 8 | Speech, language and extraction | Speech-to-text pipeline; Content Understanding extraction |
| 9 | Evaluation, tracing, guardrails | Evaluation run with groundedness and safety evaluators; tracing to Application Insights |
| 10 | CI/CD and revision | GitHub Actions deployment with an evaluation gate; practice assessment |

Use [Topic 37](../37-capstone-projects/README.md) capstones 02, 03 and 08 as larger practice projects.

### Exam tips

- The exam is 120 minutes. Most questions cover GA features, though commonly used Preview features can appear.
- Expect "choose the right service or option" questions with constraints (cost, latency, data residency, security).
- Know the current names of Foundry features. Old names (Azure AI Studio, hubs, Cognitive Services) can mislead you.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **AI-901 first?** | Optional; useful if you're new to Foundry. |
| **AI-103 and AI-200 together** | They complement each other: AI-103 covers the AI layer, AI-200 covers back ends, data and containers. |
| **Preview features** | Learn GA behaviour first; know Preview features exist and what they're for. |

## Hands-on

1. Build a RAG chat app with citations, evaluation and tracing.
2. Build an agent with two tools and a human-approval step.
3. Configure guardrails and test them with adversarial prompts.
4. Deploy the app through CI/CD with keyless authentication and private networking.
5. Take the practice assessment and fix weak areas.

## Security, cost and reliability notes

- Practise managed identity and RBAC rather than API keys.
- Watch quota and token costs in labs; delete provisioned deployments.
- Learn the difference between platform guardrails and app-level authorisation.

## Common mistakes

| Mistake | Fix |
|---|---|
| Studying AI-102 courses | Use AI-103 content and the current study guide |
| Only using the playground | Build in Python with the SDK |
| Ignoring operations | Practise CI/CD, monitoring and quotas |
| Skipping responsible AI | It's spread across every domain |

## Check yourself

1. Which certification does AI-103 lead to, and what did it replace?
2. What are the five domains and which is largest?
3. Name three operational skills AI-103 measures.
4. Which roadmap topics map to the agentic domain?
5. Which programming language does AI-103 assume?

<details>
<summary>Answers</summary>

1. Azure AI Apps and Agents Developer Associate; it replaced the Azure AI Engineer Associate (AI-102).
2. Plan and manage; generative AI and agentic solutions (largest, 30–35%); computer vision; text analysis; information extraction.
3. Any three: CI/CD integration, quota and rate-limit management, monitoring drift and safety events, tracing, evaluation.
4. Topics 18 and 19, supported by 16, 17 and 22.
5. Python.

</details>

## References

- [Exam AI-103](https://learn.microsoft.com/credentials/certifications/exams/ai-103/)
- [AI-103 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-103)
- [Azure AI Apps and Agents Developer Associate](https://learn.microsoft.com/credentials/certifications/azure-ai-apps-and-agents-developer-associate/)
- [Microsoft Foundry documentation](https://learn.microsoft.com/azure/foundry/)
