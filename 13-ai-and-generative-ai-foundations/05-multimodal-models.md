# Multimodal Models

## Why it matters

Business information isn't just text. It's scanned invoices, product photos, diagrams, recorded calls, meeting audio and video. **Multimodal models** take in and/or produce more than one modality: text, images, audio and video. They enable document understanding, visual inspection, voice agents, image generation for marketing, and video generation. Consultants need to know what's realistic; architects need to know which service handles which modality on Azure, and at what latency and cost.

## Key concepts

### Modalities and directions

| Direction | Examples | Typical models and services on Azure (checked 2026-11) |
|---|---|---|
| **Image → text** (vision understanding) | Describe a photo, read a chart, answer questions about a screenshot, compare images | GPT-4o, GPT-4.1, GPT-5 and GPT-6 families (text + image input), Mistral, Llama 4, Phi multimodal models |
| **Document → structured data** | Extract fields from invoices, contracts, forms | **Azure Content Understanding**, **Document Intelligence** (Foundry Tools), vision LLMs; OCR models such as Mistral OCR |
| **Text → image** / image editing | Marketing visuals, product mock-ups, inpainting | `gpt-image-1` family (`gpt-image-1`, `-1-mini`, `-1.5`, `gpt-image-2`...), MAI-Image models (preview), Black Forest Labs FLUX models |
| **Text (+ image) → video** | Short marketing or training clips | `sora-2` (Preview) |
| **Speech → text** (transcription) | Call transcripts, captions, diarisation | `gpt-4o-transcribe`, `gpt-4o-mini-transcribe`, `gpt-4o-transcribe-diarize`, `whisper`; **Azure Speech** (fast, batch and real-time transcription; MAI-Transcribe) |
| **Text → speech** | Read-aloud, IVR, voice agents | `gpt-4o-mini-tts`, `tts`, `tts-hd`; **Azure Speech** neural voices and MAI-Voice |
| **Speech ↔ speech** (real-time) | Voice assistants and agents with low latency, interruptions | `gpt-realtime` family via the **Realtime API** (WebRTC or WebSocket); **Voice Live API** in Azure Speech |
| **Audio generation in chat** | Spoken responses from chat completions | `gpt-audio` family |
| **Video → text** | Summarise or search video, extract insights | Content Understanding (video analysers), Video Indexer, frame sampling with vision LLMs |

Model names change quickly; the patterns (understanding, generation, speech in and out) are stable.

### How multimodal models work (intuition)

- **Vision encoders** turn images into patch embeddings (like tokens) that a transformer can attend to alongside text tokens. Images therefore **consume tokens**: more pixels and higher detail settings mean more tokens and cost.
- **Native multimodal models** (for example GPT-4o and later) were trained jointly on text, images and audio, so they can reason across modalities in one model.
- **Image generation** commonly uses diffusion or autoregressive image-token approaches conditioned on text.
- **Real-time speech models** process audio tokens directly, avoiding a separate speech-to-text → LLM → text-to-speech chain, which cuts latency and keeps tone and emotion.

### Pipeline vs end-to-end voice

| Approach | Chain | Pros | Cons |
|---|---|---|---|
| **Cascaded** | Speech-to-text → LLM → text-to-speech | Swap components independently; easier to log and moderate text; custom voices and vocabularies | Higher latency; loses tone |
| **End-to-end real-time** | Speech-in, speech-out model (Realtime API) | Low latency, natural turn-taking and interruptions | Fewer component choices; audio tokens cost more; harder to inspect |
| **Managed voice agent** | Azure Speech **Voice Live API** (speech + model + avatar options in one service) | Integrated, enterprise voices, noise suppression | Check model and region availability |

### Practical limits of vision understanding

| Works well | Be careful with |
|---|---|
| Describing scenes, reading clear text, charts and screenshots, comparing images | Small text in large images, dense tables, handwriting, precise counting, spatial measurements |
| Classifying images with instructions | Medical or safety-critical judgement without validation |
| Extracting a few fields from a clean document | High-volume structured extraction, where purpose-built document services are cheaper and more consistent |

## How it works on Azure

### Image understanding (Python, keyless)

```python
import base64
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import OpenAI

tp = get_bearer_token_provider(DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default")
client = OpenAI(base_url="https://<resource>.openai.azure.com/openai/v1/", api_key=tp)

b64 = base64.b64encode(open("damaged-parcel.jpg", "rb").read()).decode()
r = client.responses.create(
    model="<vision-capable-deployment>",
    input=[{"role": "user", "content": [
        {"type": "input_text", "text": "Is the parcel damaged? Answer yes/no and describe the damage in one sentence."},
        {"type": "input_image", "image_url": f"data:image/jpeg;base64,{b64}", "detail": "low"},
    ]}])
print(r.output_text)
```

`detail: "low"` uses fewer image tokens; `"high"` reads fine detail at higher cost.

### Image generation (Python)

```python
img = client.images.generate(model="<gpt-image deployment>", prompt="Flat illustration of a parcel delivery drone over a city, brand colours teal and white", size="1024x1024")
open("drone.png", "wb").write(base64.b64decode(img.data[0].b64_json))
```

Generated images pass through content filtering. Some image and video models need access approval or are region-limited, so check the model page.

### Transcription (Python)

```python
with open("call.wav", "rb") as f:
    t = client.audio.transcriptions.create(model="<gpt-4o-transcribe deployment>", file=f)
print(t.text)
```

For long recordings, high volume, diarisation, custom vocabularies or batch processing of call-centre archives, **Azure Speech** batch or fast transcription and Content Understanding are usually the better fit (topic 20).

### Choosing a service per modality

| Need | First choice |
|---|---|
| Ad hoc reasoning over an image or screenshot | Vision-capable LLM |
| Reliable, high-volume document field extraction | Content Understanding or Document Intelligence |
| Call-centre transcription and analytics | Azure Speech + Content Understanding |
| Low-latency voice agent | Realtime API (`gpt-realtime`) or Voice Live API |
| Branded voices and avatars | Azure Speech (custom neural voice, avatars) |
| Marketing images | `gpt-image` family, MAI-Image or FLUX, with human review |
| Short video generation | `sora-2` (Preview), with human review and provenance labelling |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| LLM vision vs specialised service | Specialised services (Content Understanding, Document Intelligence, Speech) for repeatable, high-volume, structured tasks with confidence scores; LLMs for flexible, ad hoc reasoning |
| Image detail level | Use low detail unless the task needs fine text; resize and crop images before sending |
| Voice architecture | End-to-end real-time for natural conversation; cascaded when you need custom STT or TTS, text moderation or detailed transcripts |
| Generated media | Require human review for external use; apply provenance (Content Credentials / C2PA metadata is added to images generated by Azure OpenAI image models) |

## Hands-on

1. Send three images (a chart, a receipt, a photo) to a vision-capable deployment and compare `low` vs `high` detail answers and token usage.
2. Extract invoice fields with Content Understanding (or Document Intelligence's prebuilt invoice model) and compare with an LLM vision prompt on the same 10 invoices: accuracy, consistency, cost.
3. Generate two images with an image model and inspect the content credentials metadata.
4. Transcribe a 2-minute recording with a transcription model and with Azure Speech; compare accuracy and speaker labels.
5. Try the Realtime API sample (web) in the Foundry playground for a voice conversation and note the latency.

## Security, cost and reliability notes

- **Security:** images and audio can carry **prompt injection** (text in images, spoken instructions). Treat them as untrusted input.
- **Security:** voice and face data are sensitive and often biometric. Check consent, retention and regional regulations, and the limited access policies for some capabilities (for example custom neural voice).
- **Cost:** image tokens, audio tokens and video generation are significantly more expensive than text. Downsample, trim silence, and choose mini models where quality allows.
- **Reliability:** generation models apply content filters, so plan for refusals. Real-time sessions need reconnection logic.
- **Responsible AI:** label AI-generated media, avoid generating real people's likenesses without consent, and keep a human in the loop for public content.

## Common mistakes

- Sending full-resolution images when a thumbnail answers the question.
- Using an LLM for bulk invoice extraction where a document model is cheaper and more consistent.
- Building a cascaded voice pipeline when latency is the main requirement, or vice versa.
- Publishing generated images or video without review or provenance.
- Ignoring the cost of audio tokens in real-time sessions.

## Check yourself

1. Name three directions of multimodality and an Azure option for each.
2. Why do images consume tokens, and how can you reduce the cost?
3. Compare cascaded and end-to-end voice architectures.
4. When would you choose Content Understanding over a vision LLM?
5. What responsible AI practices apply to generated images and video?

<details>
<summary>Answers</summary>

1. Image → text (vision-capable GPT models); text → image (`gpt-image` family, MAI-Image, FLUX); speech ↔ speech (`gpt-realtime` via the Realtime API or the Voice Live API). Others: speech → text (`gpt-4o-transcribe`, Azure Speech), text → video (`sora-2`).
2. Vision encoders convert images into patch tokens processed by the transformer. Reduce cost by resizing or cropping and using `detail: low`.
3. Cascaded (STT → LLM → TTS) is modular and easy to moderate and log, but slower and loses tone. End-to-end real-time models give low latency and natural conversation, with fewer component choices and higher audio-token costs.
4. For repeatable, high-volume, structured extraction where consistency, confidence scores and lower cost matter.
5. Human review, provenance metadata (Content Credentials), clear AI-generated labelling, consent for likenesses, and content filtering.

</details>

## References

- [Foundry Models sold by Azure (image, video, audio models)](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure)
- [Use vision-enabled chat models](https://learn.microsoft.com/azure/foundry/openai/how-to/gpt-with-vision)
- [Image generation how-to](https://learn.microsoft.com/azure/foundry/openai/how-to/dall-e)
- [Video generation with Sora](https://learn.microsoft.com/azure/foundry/openai/concepts/video-generation)
- [Realtime API for speech and audio](https://learn.microsoft.com/azure/foundry/openai/how-to/realtime-audio)
- [Voice Live API overview](https://learn.microsoft.com/azure/ai-services/speech-service/voice-live)
- [What is Azure Content Understanding?](https://learn.microsoft.com/azure/ai-services/content-understanding/overview)
- [Content credentials in Azure OpenAI](https://learn.microsoft.com/azure/foundry-classic/openai/concepts/content-credentials)
