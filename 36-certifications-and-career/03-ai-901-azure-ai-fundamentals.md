# AI-901 Azure AI Fundamentals

## Why it matters

AI-901 (Microsoft Azure AI Fundamentals) replaced AI-900, which retired on 30 June 2026. The certification keeps the name *Microsoft Certified: Azure AI Fundamentals*, but the exam is different. AI-900 tested recognition of Azure AI services. AI-901 expects you to **implement simple AI solutions by using Microsoft Foundry**, and it assumes basic Python knowledge. More than half the exam is hands-on Foundry work. For all three roadmap tracks it's the fastest way to show that you understand modern Azure AI: models, agents, responsible AI and Foundry Tools.

## Key concepts

### Skills measured (study guide, as of 15 April 2026)

| Domain | Weight | Sub-areas |
|---|---|---|
| **Identify AI concepts and capabilities** | 40–45% | Principles of responsible AI; AI model components and configurations; AI workloads |
| **Implement AI solutions by using Microsoft Foundry** | 55–60% | Generative AI apps and agents; text and speech; computer vision and image generation; information extraction |

### AI-900 vs AI-901

| Area | AI-900 (retired) | AI-901 |
|---|---|---|
| Focus | Describe workloads and services | Describe concepts **and** implement with Foundry |
| Platform | Azure AI services, Azure OpenAI, Azure Machine Learning | Microsoft Foundry and Foundry Tools |
| Coding | None expected | Python syntax familiarity expected |
| Agents | Not covered | Covered |
| Expires | No | No |

### Roadmap mapping

| Exam area | Roadmap topics |
|---|---|
| AI concepts, model components, configurations | [13 · AI and Generative AI Foundations](../13-ai-and-generative-ai-foundations/README.md) |
| Responsible AI principles | [23 · Responsible AI, Security and Governance](../23-responsible-ai-security-and-governance/README.md) (files 01–03) |
| Foundry projects, deployments, playgrounds | [14 · Microsoft Foundry Platform](../14-microsoft-foundry-platform/README.md), [15 · Foundry Models](../15-foundry-models-and-azure-openai/README.md) |
| Generative apps and agents | [16 · Prompt Engineering](../16-prompt-engineering-and-model-customisation/README.md), [18 · Agents with Foundry Agent Service](../18-agents-with-foundry-agent-service/README.md) (introductory files) |
| Text, speech, vision, information extraction | [20 · Foundry Tools](../20-foundry-tools/README.md) |

## How it works on Azure

### Five-week study plan (about 5 hours a week)

| Week | Focus | Practical work |
|---|---|---|
| 1 | AI workloads and responsible AI | Read topic 13 and topic 23 files 01–03; map six responsible AI principles to examples |
| 2 | Models and configuration | Create a Foundry project; deploy a chat model; vary temperature, top-p and max tokens in the playground; compare a small and a large model |
| 3 | Apps and agents | Call the deployed model from Python with keyless authentication; create an agent with instructions and a file-search tool |
| 4 | Text, speech, vision, extraction | Try language analysis, speech-to-text, image analysis or understanding with a multimodal model, image generation, and Content Understanding or Document Intelligence on a sample invoice |
| 5 | Revision | Practice assessment on AI Skills Navigator; exam sandbox; review weak areas |

### Minimal Python you should be able to read

```python
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient

project = AIProjectClient(
    endpoint="https://<resource>.services.ai.azure.com/api/projects/<project>",
    credential=DefaultAzureCredential(),
)
client = project.get_openai_client()
response = client.responses.create(model="<deployment-name>", input="Summarise responsible AI in one sentence.")
print(response.output_text)
```

You don't need to write complex code, but you should recognise endpoints, credentials, deployments, and request and response objects. SDK surfaces evolve, so check the current Foundry quickstart.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **AI-901 or go straight to AI-103?** | Developers with Foundry experience can skip it; architects and consultants benefit from the broad, practical coverage. |
| **Lab environment** | Use a personal or sandbox subscription with budget alerts; delete deployments after study. |
| **Depth** | Learn what each Foundry capability does and its basic configuration, not production hardening. |

## Hands-on

1. Deploy a model in a Foundry project and test it in the playground.
2. Call it from Python with `DefaultAzureCredential`.
3. Build a simple agent with grounding on an uploaded file.
4. Run speech-to-text and document extraction samples.
5. Apply a content filter or guardrail configuration and test a blocked prompt.

## Security, cost and reliability notes

- Prefer keyless authentication (Entra ID) even in labs; it builds the right habit.
- Delete model deployments and provisioned resources after labs; some resources bill while idle.
- Never paste personal or client data into lab prompts.

## Common mistakes

| Mistake | Fix |
|---|---|
| Studying AI-900 material | Use the AI-901 study guide and Foundry-based content |
| Skipping Python | Practise reading short SDK snippets |
| Ignoring responsible AI | It's a named sub-area worth significant marks |
| Using retired service names (Cognitive Services, Azure AI Studio) | Learn current names (topic 38 file 02) |

## Check yourself

1. Which exam did AI-901 replace, and when did that exam retire?
2. What are AI-901's two domains and weights?
3. Which platform is the hands-on domain built on?
4. What coding knowledge does AI-901 expect?
5. Does the Azure AI Fundamentals certification expire?

<details>
<summary>Answers</summary>

1. AI-900, which retired on 30 June 2026.
2. Identify AI concepts and capabilities (40–45%); implement AI solutions by using Microsoft Foundry (55–60%).
3. Microsoft Foundry.
4. Knowledge of Python coding syntax and programming techniques.
5. No.

</details>

## References

- [Exam AI-901](https://learn.microsoft.com/credentials/certifications/exams/ai-901/)
- [AI-901 study guide](https://learn.microsoft.com/credentials/certifications/resources/study-guides/ai-901)
- [Microsoft Foundry documentation](https://learn.microsoft.com/azure/foundry/)
- [Exam and assessment lab retirement](https://learn.microsoft.com/credentials/support/retired-certification-exams)
