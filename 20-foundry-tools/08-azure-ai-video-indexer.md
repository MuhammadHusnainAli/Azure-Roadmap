# Azure AI Video Indexer

## Why it matters

Video and audio archives (training libraries, broadcast media, meeting recordings, security footage, product demos) are some of the least searchable assets organisations own. **Azure AI Video Indexer** runs a stack of AI models over video and audio to produce transcripts, translations, captions, faces, objects, OCR, scenes, topics, keywords, sentiment and generative summaries, making content searchable and reusable. It comes in two forms: a **cloud-based** service and an **Azure Arc-enabled** extension that runs at the edge, including on **live video streams**. Architects use it for media asset management, accessibility, compliance and edge video analytics.

## Key concepts

### Two offerings

| | Cloud-based Video Indexer | Video Indexer enabled by Azure Arc |
|---|---|---|
| Where it runs | Azure cloud (built on Face, Translator, Vision, Speech) | Arc-enabled Kubernetes at the edge or on-premises |
| Sources | Uploaded video and audio | Uploaded files **and live camera streams** |
| Features | Full model catalogue (below) | Indexing presets (transcription, translation, captioning, keyframes, objects, scenes, shots, summarisation); live presets with first-party detections (people, vehicles) and **custom insights** defined in natural language or by image |
| Use when | Media libraries, search, accessibility, content creation | Data can't leave the site (regulation, size, latency); pre-indexing before upload; real-time retail, manufacturing and safety analytics |
| Access | Azure resource | Application required |

### Cloud insights (selection)

| Category | Models |
|---|---|
| **Video** | Face detection; **celebrity identification** and **account-based face identification** (restricted, need authorisation); OCR; visual content moderation; labels; scene, shot and keyframe detection; black frames; rolling credits; editorial shot types; observed people with clothing; object detection and tracking; slate detection |
| **Audio** | Transcription (50+ languages); automatic and multi-language detection; closed captions (VTT, TTML, SRT); two-channel processing; noise reduction; custom language models; speaker enumeration (up to 16) and statistics; textual content moderation; text-based emotion; translation; audio effects (advanced preset) |
| **Combined** | Keywords, named entities (brands, locations, people), topic inference, sentiment |
| **Generative** | Multimodal generative AI for textual summaries of video content |

Some features are restricted for privacy and regulatory reasons and require authorisation (notably face identification and celebrity recognition), in line with Microsoft's Limited Access policies.

### Indexing presets

Choose **basic** or **advanced** video, audio, or audio + video presets per upload; advanced presets add models (for example the full set of audio effects) at higher cost.

### Outputs

- Insights JSON (time-coded), downloadable artefacts (keyframes, face thumbnails, captions).
- Embeddable player and insights widgets.
- Search across the account by spoken words, faces, OCR text, labels and topics.

## How it works on Azure

| Step | How |
|---|---|
| Create | A Video Indexer resource connected to an Azure Storage account (and, for Arc, the extension on an Arc-enabled cluster) |
| Authenticate | Entra ID / ARM-generated access tokens for the Video Indexer API; managed identity for storage |
| Upload | API or web portal from URL or file; choose preset, language and custom models |
| Retrieve | Poll for completion; get index JSON, captions, thumbnails |
| Integrate | Push insights into Azure AI Search for video search ([Topic 17](../17-rag-and-knowledge/README.md)); trigger workflows with Event Grid/Logic Apps; publish captions |

### Video Indexer vs Content Understanding vs multimodal LLMs

| Need | Choose |
|---|---|
| Rich, time-coded insights across a media library (faces, OCR, scenes, transcripts, captions) with a player and search | **Video Indexer** |
| Edge or live-stream video analytics | **Video Indexer enabled by Arc** |
| Schema-driven extraction from video (specific fields) with confidence and grounding, alongside documents and audio | **Content Understanding** ([file 07](07-azure-content-understanding.md)) |
| Ad hoc reasoning over a few frames or a short clip inside an app or agent | **Multimodal model** ([Topic 15 file 10](../15-foundry-models-and-azure-openai/10-vision-image-and-video-generation.md)) |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Cloud vs Arc** | Arc when content must stay on-site, archives are too large to move, or you need live-stream analytics; cloud for everything else. |
| **Preset** | Basic presets for transcripts and core visual insights; advanced only where extra models (audio effects, slates) matter. |
| **Faces** | Avoid face identification unless you have authorisation, a lawful basis and consent; face detection and blurring are lower risk. |
| **Custom models** | Custom language models (for jargon) and person models (approved scenarios) improve accuracy. |
| **Search** | Export insights to Azure AI Search for enterprise search and RAG over video. |
| **Cost control** | Index once; store insights; re-index only when models or needs change. |

## Hands-on

1. Create a cloud Video Indexer resource with a storage account.
2. Upload a 5–10 minute training video with the **basic audio + video** preset.
3. Review insights: transcript, keywords, topics, OCR, scenes; download VTT captions.
4. Translate the transcript to another language and add captions to the player.
5. Export the insights JSON and index transcript segments with timestamps into Azure AI Search; ask a RAG question that returns a link to the exact moment.
6. Compare the result with Content Understanding's `prebuilt-videoSearch` on the same file.

## Security, cost and reliability notes

- **Privacy**: video and audio contain faces, voices and personal data; restricted features require authorisation; apply consent, retention and access control.
- **Legal compliance**: Microsoft requires compliance with applicable laws and prohibits harmful use; facial recognition is especially sensitive.
- **Storage**: source media and insights live in your storage account; secure it with private endpoints and managed identity.
- **Cost**: billed per input minute by preset (basic vs advanced, audio vs video); Arc deployments add cluster costs.
- **Throughput**: indexing time depends on duration and preset; queue and parallelise with care for quotas.

## Common mistakes

| Mistake | Fix |
|---|---|
| Advanced presets for every file | Basic unless advanced insights are needed |
| Moving restricted archives to the cloud anyway | Use the Arc extension |
| Face identification without authorisation | Apply for access or avoid |
| Leaving insights only in the portal | Export to search and workflows |
| Re-indexing repeatedly | Persist insights |

## Check yourself

1. What are the two Video Indexer offerings, and how do they differ?
2. Name five video and five audio insights.
3. Which features are restricted?
4. When would you use the Arc-enabled version?
5. What caption formats are produced?
6. How would you make video content searchable for RAG?
7. When should you use Content Understanding instead of Video Indexer?
8. What drives Video Indexer cost?

<details>
<summary>Answers</summary>

1. Cloud-based (uploaded media, full model catalogue) and enabled by Azure Arc (edge/on-premises, uploaded and live streams, presets and custom live insights).
2. Video: face detection, OCR, labels, scenes/shots/keyframes, object detection, observed people, content moderation. Audio: transcription, language detection, captions, speaker enumeration, translation, emotion, audio effects, noise reduction.
3. Celebrity identification and account-based face identification (and other privacy-sensitive features) require authorisation.
4. When content can't leave the site, archives are too large to move, on-premises workflows need low latency, or for live-stream analytics.
5. VTT, TTML and SRT.
6. Export time-coded insights (transcripts, OCR, topics) into Azure AI Search and return links to timestamps.
7. When you need schema-driven field extraction with confidence and grounding across video, documents and audio.
8. Indexed minutes by preset type (basic/advanced, audio/video), plus storage and, for Arc, cluster resources.

</details>

## References

- [Azure AI Video Indexer overview](https://learn.microsoft.com/azure/azure-video-indexer/video-indexer-overview)
- [Video Indexer enabled by Azure Arc](https://learn.microsoft.com/azure/azure-video-indexer/arc/azure-video-indexer-enabled-by-arc-overview)
- [Content Understanding video analysis](https://learn.microsoft.com/azure/ai-services/content-understanding/video/overview)
- [Limited Access features](https://learn.microsoft.com/azure/ai-services/cognitive-services-limited-access)
