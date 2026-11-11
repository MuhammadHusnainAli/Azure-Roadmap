# Bot Service, Health Bot and Immersive Reader

## Why it matters

Three specialised services round out the Foundry Tools family and still appear in real estates and exam questions. **Azure AI Bot Service** is the channel connectivity layer that carries conversations between your agent and Teams, web chat and other channels. The **Microsoft healthcare agent service** (formerly Health Bot) is a compliance-focused platform for healthcare copilots. **Azure AI Immersive Reader** is an inclusive reading tool for education and accessibility. Architects need to know where each fits next to Foundry agents, Copilot Studio and the Microsoft 365 Agents SDK, and which older approaches (Bot Framework SDK, LUIS-based bots) to move away from.

## Key concepts

### Azure AI Bot Service

| Aspect | Description |
|---|---|
| **Role** | Registers your agent (bot) and connects it to **channels**: Microsoft Teams, Microsoft 365 Copilot, web chat, Direct Line, email, SMS and others |
| **Protocol** | The **Activity Protocol**: normalised messages and events exchanged between channels and your agent |
| **Identity** | An Entra app registration or (recommended) **user-assigned managed identity** for the bot; single-tenant where possible |
| **Building agents for it** | **Microsoft 365 Agents SDK** (C#, JavaScript, Python) is now the recommended code-first SDK ([Topic 19 file 08](../19-agent-frameworks-and-protocols/08-microsoft-365-agents-sdk-and-toolkit.md)); **Copilot Studio** for low-code; **Bot Framework WebChat** to embed chat in web apps |
| **Foundry integration** | Foundry publishing to Teams/Copilot uses Bot Service channel authorisation schemes (BotServiceRbac, BotServiceTenant) without you creating a separate bot resource ([Topic 18 file 12](../18-agents-with-foundry-agent-service/12-publishing-agents.md)) |
| **Tools** | Agents Toolkit, Agents Playground, Azure Bot CLI |

The older **Bot Framework SDK** is superseded by the Agents SDK for new work; Microsoft's Bot Service documentation now leads with the Agents SDK and Copilot Studio. Migrate existing Bot Framework bots incrementally, reusing business logic.

### Microsoft healthcare agent service (formerly Azure Health Bot)

| Aspect | Description |
|---|---|
| **Purpose** | Build compliant generative AI **healthcare copilots** and virtual health assistants for providers, pharma, telemedicine and insurers |
| **Capabilities** | Healthcare-adapted LLM **orchestrator**; built-in healthcare sources; **healthcare safeguards** (clinical-aware safety checks, evidence, disclaimers); built-in **triage and symptom checker**; healthcare-specific configuration; management portal; integration with customer data sources and plugins |
| **Channels** | Text and voice via Bot Service channels (web, Teams, telephony) |
| **Use cases** | Clinician access to protocols and documentation, patient self-service (appointments, coverage, triage), administrative workload reduction |
| **Compliance** | Designed around healthcare regulatory requirements; customers remain responsible for clinical governance and regulatory obligations |

Choose it when a healthcare organisation wants a governed, healthcare-specialised starting point rather than building safeguards from scratch on Foundry.

### Azure AI Immersive Reader

| Aspect | Description |
|---|---|
| **Purpose** | Inclusive reading for new readers, language learners and people with learning differences (for example dyslexia) |
| **Features** | Isolate content (focus mode), picture dictionary, parts-of-speech highlighting, read aloud (text to speech), real-time translation, syllabification, line focus |
| **How it works** | A standalone web app that opens over your app in an iframe via the client SDK; your app passes content (text, maths, rich content) and a token |
| **Data** | Immersive Reader doesn't store customer data |
| **Where used** | Microsoft Learning Tools in Word, OneNote, Teams, Edge; education platforms and accessibility features in custom apps |

## How it works on Azure

| Service | Typical architecture |
|---|---|
| **Bot Service** | Agent code (Agents SDK) on App Service, Container Apps or Functions → Azure Bot resource (identity, messaging endpoint `/api/messages`) → channels; brain calls Foundry agents or models |
| **Healthcare agent service** | Service instance configured in its portal → scenarios, data connections and safeguards → channels via Bot Service; optionally Foundry models and your clinical content |
| **Immersive Reader** | Immersive Reader resource (Entra auth) → your backend issues a token → front end launches the reader with content |

All support keyless or token-based authentication patterns; prefer managed identity for backend calls.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Foundry publishing vs Bot Service + Agents SDK** | Foundry publishing for straightforward Teams/Copilot delivery; Bot Service + Agents SDK for custom channel behaviour, non-Microsoft channels or complex conversation handling. |
| **Copilot Studio vs code** | Copilot Studio for makers and Power Platform integration; code for engineering-heavy agents ([Topic 24](../Roadmap.md)). |
| **Healthcare agent service vs Foundry agent** | Healthcare agent service for built-in healthcare safeguards, triage and compliance features; Foundry for general-purpose flexibility (you build safeguards and evaluation). |
| **Immersive Reader vs custom TTS/translation** | Immersive Reader for a ready-made, accessible reading experience; Speech and Translator APIs for custom UIs. |
| **Legacy bots** | Plan migration from Bot Framework SDK and LUIS/QnA Maker to Agents SDK with Foundry or CLU/question answering. |

## Hands-on

1. **Bot Service**: create an Azure Bot resource with a user-assigned managed identity; point it at an Agents SDK echo agent (from Topic 19 file 08) on App Service; enable the **Web Chat** channel and test it; then enable **Teams**.
2. Replace the echo with a call to a Foundry agent and test in Teams.
3. **Healthcare agent service**: create an instance in a test subscription, explore the built-in triage scenario and healthcare safeguards, and note what you'd have to build yourself on Foundry.
4. **Immersive Reader**: add the Immersive Reader button to a simple web page using the JavaScript SDK and a token endpoint; test read aloud, translation and syllables.

## Security, cost and reliability notes

- **Bot identity**: use managed identity or single-tenant apps; secure the messaging endpoint; validate channel tokens (the SDK does this).
- **Channel data**: messages traverse Microsoft channels; check compliance and residency for each channel.
- **Healthcare**: protected health information requires appropriate agreements, access controls, audit and clinical oversight; the service doesn't replace clinical judgement.
- **Immersive Reader**: no customer data stored; still protect tokens and content in transit.
- **Cost**: Bot Service standard channels are free, premium channels (for example Direct Line App Service extension) are billed; healthcare agent service has plan-based pricing; Immersive Reader is metered per use.

## Common mistakes

| Mistake | Fix |
|---|---|
| Starting new bots on Bot Framework SDK | Use the Microsoft 365 Agents SDK |
| Creating separate bot resources when Foundry publishing suffices | Publish from Foundry |
| Multi-tenant bot apps with client secrets in config | Managed identity or single-tenant with federated credentials |
| Building healthcare triage from scratch without clinical safeguards | Evaluate the healthcare agent service |
| Reinventing accessible reading features | Use Immersive Reader |

## Check yourself

1. What does Azure AI Bot Service do in an agent architecture?
2. Which SDK is recommended for new code-first bots and agents?
3. What identity options should a bot use?
4. What does the healthcare agent service add over a generic agent?
5. Name five Immersive Reader features.
6. Does Immersive Reader store customer data?
7. When would you use Bot Service with the Agents SDK instead of Foundry's Teams publishing?
8. What should happen to existing Bot Framework and LUIS-based bots?

<details>
<summary>Answers</summary>

1. It registers the agent and connects it to channels (Teams, Copilot, web chat, Direct Line and others) using the Activity Protocol.
2. The Microsoft 365 Agents SDK.
3. A user-assigned managed identity or a single-tenant Entra app registration (with federated credentials rather than secrets).
4. A healthcare-adapted orchestrator, built-in healthcare sources and safeguards, triage and symptom checker, healthcare configuration and compliance-oriented design.
5. Any five: focus mode/isolate content, picture dictionary, parts-of-speech highlighting, read aloud, translation, syllabification, line focus.
6. No.
7. When you need custom conversation handling, Adaptive Cards, proactive messages, non-Microsoft channels or full control of the channel layer.
8. Plan migration to the Agents SDK with Foundry or current Language features (CLU, question answering), reusing business logic.

</details>

## References

- [Azure Bot Service documentation](https://learn.microsoft.com/azure/bot-service/)
- [Choose the right chatbot solution](https://learn.microsoft.com/azure/bot-service/bot-overview)
- [What is the Microsoft 365 Agents SDK?](https://learn.microsoft.com/microsoft-365/agents-sdk/agents-sdk-overview)
- [Healthcare agent service overview](https://learn.microsoft.com/azure/health-bot/overview)
- [What is Azure AI Immersive Reader?](https://learn.microsoft.com/azure/ai-services/immersive-reader/overview)
