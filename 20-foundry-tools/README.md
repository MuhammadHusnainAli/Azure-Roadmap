# 20 · Foundry Tools

> Tracks: E ● A ○ C ○ · Part D · AI on Azure

Foundry Tools (formerly Azure AI services and, before that, Cognitive Services) are Microsoft's task-specific AI services: they see, hear, read, translate, extract and moderate. In the LLM era they remain the cheapest, fastest and most auditable way to do well-defined perception tasks at scale, and they increasingly act as tools for agents. This topic covers the Foundry resource, authentication, containers and Limited Access; Azure Vision and its major 2028 retirements (Image Analysis, Custom Vision) with their successors; Azure Speech including LLM Speech and Voice Live; Azure Language's core and legacy features; Azure Translator with its new NMT-or-LLM API; Document Intelligence; Content Understanding for multimodal, schema-driven extraction; Video Indexer in the cloud and at the edge with Azure Arc; Azure AI Content Safety and Prompt Shields; Bot Service, the healthcare agent service and Immersive Reader; and a decision guide for choosing a task-specific service versus an LLM.

## What you will be able to do

- Provision Foundry Tools keylessly, choose between the Foundry resource and single-service resources, and run services in containers.
- Identify retired and retiring services and features, and choose their successors.
- Build speech pipelines with real-time, fast and batch transcription, custom speech and SSML.
- Redact PII and extract entities with Azure Language, and know which features are legacy.
- Translate text and documents with NMT or LLM engines and control terminology.
- Extract structured data with Document Intelligence and Content Understanding, with confidence and review paths.
- Index video and audio with Video Indexer, including edge deployment with Azure Arc.
- Protect applications and agents with Content Safety: harm categories, Prompt Shields, protected material.
- Connect agents to channels with Bot Service, and position the healthcare agent service and Immersive Reader.
- Decide between a Foundry Tool, an LLM or a combination using cost, accuracy and compliance evidence.

## Prerequisites

[Topic 13](../13-ai-and-generative-ai-foundations/README.md) and [Topic 14](../14-microsoft-foundry-platform/README.md) (Foundry resources, authentication). [Topic 15](../15-foundry-models-and-azure-openai/README.md) files 10 and 11 (vision and audio models) help with the comparisons in files 02, 03 and 11.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Foundry Tools Overview](01-foundry-tools-overview.md) | 40 min | CLI, keyless PII call |
| 02 | [Azure Vision](02-azure-vision.md) | 40 min | Image Analysis → multimodal LLM |
| 03 | [Azure Speech](03-azure-speech.md) | 45 min | Speech SDK, batch transcription |
| 04 | [Azure Language](04-azure-language.md) | 40 min | PII redaction |
| 05 | [Azure Translator](05-azure-translator.md) | 35 min | Text and document translation |
| 06 | [Azure Document Intelligence](06-azure-document-intelligence.md) | 50 min | Prebuilt invoice, layout Markdown |
| 07 | [Azure Content Understanding](07-azure-content-understanding.md) | 45 min | Custom analyser |
| 08 | [Azure AI Video Indexer](08-azure-ai-video-indexer.md) | 35 min | Index a video, export insights |
| 09 | [Azure AI Content Safety](09-azure-ai-content-safety.md) | 45 min | Harm analysis, Prompt Shields |
| 10 | [Bot Service, Health Bot and Immersive Reader](10-bot-service-health-bot-and-immersive-reader.md) | 35 min | Bot channels, Immersive Reader |
| 11 | [Foundry Tools vs LLM Decision Guide](11-foundry-tools-vs-llm-decision-guide.md) | 35 min | Head-to-head evaluation |

**AI engineers:** all of it, with 03, 06, 07, 09 and 11 in depth. **Architects:** 01, 02, 07, 09 and 11 in depth; skim the rest. **Consultants:** 01, 02 (retirements), 09 and 11.

## Completion checklist

- [ ] I can list the current Foundry Tools and the retired or retiring ones with their successors and dates.
- [ ] I have created a Foundry resource with local auth disabled and called a tool with Entra ID.
- [ ] I can explain which Foundry Tools run in containers and what disconnected means.
- [ ] I have transcribed audio with batch or fast transcription and synthesised speech with SSML.
- [ ] I have redacted PII from text and transcripts before sending them to an LLM.
- [ ] I have translated a formatted document and controlled terminology.
- [ ] I have extracted invoice fields with Document Intelligence and routed low-confidence fields to review.
- [ ] I have built a Content Understanding analyser with extract, classify and generate fields.
- [ ] I have indexed a video and made it searchable.
- [ ] I have configured guardrails with Prompt Shields and tested an indirect prompt injection.
- [ ] I can position Bot Service, the healthcare agent service and Immersive Reader.
- [ ] I have run a service-vs-LLM comparison and made a cost-and-accuracy recommendation.

## Maps to

- **AI-103**: implement vision, speech, language, translation, document processing, content understanding and content safety solutions.
- **AI-901**: Foundry Tools capabilities and responsible AI (foundational).
- **SC-500**: content safety and prompt injection defences for AI applications.
- **AZ-305**: choose AI services for solution designs.

## Next

21 · Azure Machine Learning and MLOps (see [Roadmap.md](../Roadmap.md))
