# Voice Agents

## Why it matters

Voice is the most natural interface for many customer interactions (contact centres, booking lines, field workers, accessibility) and the least forgiving: users notice half-second delays, interruptions that aren't handled, and misheard account numbers. Building voice agents used to mean wiring speech-to-text, an LLM, text-to-speech, turn detection, echo cancellation and telephony together yourself. Foundry now offers **voice-based prompt agents** running on **Voice Live**, with telephony through Teams Phone extensibility or Twilio, avatars, and voice-specific tracing and evaluation. Most of this is in **Preview**, so architects must weigh capability against production readiness.

## Key concepts

### Voice architectures

| Architecture | Flow | Strengths | Trade-offs |
|---|---|---|---|
| **Native speech-to-speech** | Caller audio → realtime speech model (for example `gpt-realtime`) → spoken response | Lowest latency, most natural prosody | Voice and setting choices depend on the model |
| **Cascaded text model** | Caller audio → speech recognition → text model → speech synthesis | Broad text-model choice, Azure voices (including custom neural voice), explicit control of transcription, phrase lists, turn detection, interim responses | Each stage adds latency |

### Building blocks

| Block | Role |
|---|---|
| **Voice-based prompt agent** (agent kind `voice`) | Versioned definition: model, instructions, greeting, audio settings, tools, toolbox |
| **Voice Live** (Azure Speech) | Real-time voice orchestration: speech recognition, turn-taking, model interaction, synthesis |
| **Azure Speech in Foundry Tools** | Transcription and synthesis for cascaded architectures ([Topic 20 file 03](../20-foundry-tools/03-azure-speech.md)) |
| **Realtime models** | `gpt-realtime` and similar ([Topic 15 file 11](../15-foundry-models-and-azure-openai/11-audio-and-real-time-apis.md)) |
| **Telephony** | Teams Phone extensibility (Azure Communication Services + Event Grid) or a Twilio project connection |
| **Application Insights** | Voice traces, latency charts, token breakdowns, evaluations |

### Voice agent configuration

| Setting | Options and guidance |
|---|---|
| **Model** | `managed` (service-managed, for example `gpt-realtime`) or `self_deployed` (your deployment, BYOM where supported) |
| **Instructions and greeting** | Short, speakable sentences; optional session-start greeting |
| **Turn detection** | `server_vad` (silence-based, most business cases), `semantic_vad` (OpenAI end-of-turn model), `azure_semantic_vad` / `_en` / `_multilingual`; tune `threshold`, `prefix_padding_ms`, `silence_duration_ms`, `idle_timeout_ms` |
| **Input audio** | Noise suppression and echo cancellation; transcription for caller speech |
| **Output audio** | Azure standard, custom or personal voices; style; telephone soundscape (SDK only) |
| **Avatar** | Visual avatar over WebRTC in the browser; phone callers hear audio only |
| **Tools and toolbox** | Same tools as other agents; **interim responses** cover tool latency ("Let me check that...") |
| **Persistence** | `store=True` to persist conversations |
| **Response length** | Limit to keep replies short |
| **Subagents** | Delegate or inline subagents for text-heavy work |

Every change creates a new **immutable version** behind the same endpoint, so you can roll back.

### Channels and limits

| Capability | Support (current portal) |
|---|---|
| Browser testing | Agent playground with microphone |
| Preview web app | When the endpoint supports the Responses protocol |
| Phone numbers | **Teams Phone extensibility** and **Twilio** (inbound; outbound via call-job API); SIP integration documented separately |
| Teams / Microsoft 365 Copilot publishing | Not shown for voice agents in the standard channels layout |
| Evaluation | Full-conversation evaluation from existing conversations; no single-turn or simulated evaluation yet |
| Session length | Maximum 60 minutes |
| Retention | Stored voice conversations and Microsoft-managed audio retained 60 days |

Hosted agents can also do voice through the **Invocations (WebSocket)** protocol with Pipecat, LiveKit or Voice Live inside the container (file 04).

## How it works on Azure

Create a managed-model voice agent (Python, `azure-ai-projects` ≥ 2.7, `allow_preview=True`):

```python
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import (VoiceAgentDefinition, VoiceAgentAudioConfig, VoiceAgentAudioOutputConfig,
                                      VoiceModelType, VoiceOutputModality, VoiceType)
from azure.identity import DefaultAzureCredential

definition = VoiceAgentDefinition(
    model_type=VoiceModelType.MANAGED,
    model="gpt-realtime",
    instructions="You are Contoso's booking assistant. Keep replies to one or two sentences. "
                 "Confirm dates and names by repeating them back.",
    audio=VoiceAgentAudioConfig(output=VoiceAgentAudioOutputConfig(
        voice="en-GB-SoniaNeural", voice_type=VoiceType.AZURE_STANDARD)),
    output_modalities=[VoiceOutputModality.AUDIO],
    store=True)

with AIProjectClient(endpoint=PROJECT_ENDPOINT, credential=DefaultAzureCredential(), allow_preview=True) as p:
    v = p.agents.create_version(agent_name="booking-voice", definition=definition)
    print(v.version)
```

Clients connect over a **real-time WebSocket** (not individual text requests). Telephony bindings connect a phone number to the agent.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Speech-to-speech vs cascaded** | Speech-to-speech for natural, low-latency conversation; cascaded when you need specific text models, custom voices, phrase lists for domain terms, or detailed transcription control. |
| **Voice-based prompt agent vs hosted voice** | Prompt agent when Foundry-managed orchestration suffices; hosted agent with Invocations WebSocket for custom pipelines (Pipecat, LiveKit). |
| **Turn detection** | Server VAD for most; semantic VAD where callers pause mid-sentence; raise thresholds in noisy environments. |
| **Tools in voice** | Keep tool calls fast; use interim responses; avoid long tool chains during live calls. |
| **Telephony** | Teams Phone extensibility for organisations already on Teams Phone; Twilio when numbers live there. |
| **Production readiness** | Many voice capabilities are Preview; review support, SLA and compliance before customer-facing launches. |
| **Human handoff** | Design escalation to human agents early (transfer, callback). |

## Hands-on

1. In the Foundry portal, create a **voice-based prompt agent** with a managed `gpt-realtime` model, a greeting, and `server_vad`.
2. Test with your microphone in the playground; interrupt the agent mid-sentence and check barge-in behaviour.
3. Add a function tool (`check_availability(date)`) and an interim response; measure perceived latency with and without it.
4. Recreate the agent in code (above), then create version 2 with `azure_semantic_vad` and longer `silence_duration_ms`; compare in a noisy room.
5. If you have a Twilio trial number, create a Twilio project connection and bind it to the agent; call it.
6. Open Application Insights voice dashboards: connection latency, token breakdown per turn.

## Security, cost and reliability notes

- **Personal data in audio**: voice recordings and transcripts are personal (sometimes biometric-adjacent) data; set retention, inform callers, and avoid capturing card numbers (use DTMF or secure payment flows).
- **Telephony secrets**: store Twilio tokens only in project connections, never in requests, logs or screenshots.
- **Safety**: asynchronous content safety and guardrails apply; test voice-specific attacks (spoken prompt injection).
- **Costs**: model audio tokens, speech recognition and synthesis, custom voice training and hosting, avatar, Communication Services calling and numbers, Application Insights ingestion.
- **Latency budget**: target under about a second end-to-end; region placement close to callers and model matters.
- **Preview**: no SLA for preview voice, avatar, WebRTC and telephony features.

## Common mistakes

| Mistake | Fix |
|---|---|
| Long, written-style answers | Instruct for one or two spoken sentences |
| Silent pauses during tool calls | Interim responses |
| Default VAD in noisy call centres | Tune threshold and silence duration |
| Reading back sensitive numbers aloud | Mask and confirm partially |
| No path to a human | Design transfer and callback |
| Treating Preview voice features as production-ready | Validate support and SLAs first |

## Check yourself

1. Compare native speech-to-speech and cascaded voice architectures.
2. What does Voice Live provide?
3. Name four turn detection types and when to use them.
4. What are interim responses for?
5. Which telephony providers are supported, and what Azure services does Teams Phone extensibility need?
6. What is the maximum voice session length and audio retention?
7. How can a hosted agent provide voice?
8. List four cost components of a voice agent.

<details>
<summary>Answers</summary>

1. Speech-to-speech uses one realtime model for audio in and out (lowest latency, natural); cascaded chains speech recognition, a text model and speech synthesis (more control and model/voice choice, more latency).
2. Real-time voice orchestration: speech recognition, turn-taking, model interaction and synthesis, so you don't host it.
3. `server_vad` (most business cases), `semantic_vad` (OpenAI end-of-turn), `azure_semantic_vad_en` (English only), `azure_semantic_vad_multilingual` (callers switch languages).
4. To fill silence with a short spoken update while a tool call runs.
5. Teams Phone extensibility (needs Azure Communication Services and Event Grid for inbound) and Twilio.
6. 60 minutes per session; 60 days retention for stored voice conversations and Microsoft-managed audio.
7. Through the Invocations (WebSocket) protocol with Pipecat, LiveKit or Voice Live in the container.
8. Any four: model audio tokens, speech recognition, speech synthesis, custom voice, avatar, Communication Services calling and numbers, Application Insights.

</details>

## References

- [What is Foundry Agent Service? (voice-based agents)](https://learn.microsoft.com/azure/foundry/agents/overview)
- [Quickstart: create a voice-based prompt agent](https://learn.microsoft.com/azure/foundry/agents/quickstarts/prompt-voice-agent)
- [Configure a voice agent](https://learn.microsoft.com/azure/foundry/agents/how-to/configure-voice-agent)
- [Integrate telephony channels](https://learn.microsoft.com/azure/foundry/agents/how-to/voice-agent-telephony-channels)
- [Best practices for voice-based agents](https://learn.microsoft.com/azure/foundry/agents/concepts/voice-agent-best-practice)
- [Voice Live quickstart](https://learn.microsoft.com/azure/ai-services/speech-service/voice-live-quickstart)
