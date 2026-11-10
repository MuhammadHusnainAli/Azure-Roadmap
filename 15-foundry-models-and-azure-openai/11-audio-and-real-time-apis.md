# Audio and Real-Time APIs

## Why it matters

Voice is becoming a primary interface: contact-centre agents, in-car assistants, field-worker copilots, accessibility features and meeting tools. Foundry offers two styles. **Request/response audio APIs** do transcription, translation and text-to-speech. The **Realtime API** gives low-latency, speech-in/speech-out conversations over WebRTC, WebSocket or SIP. Choosing the right one, and implementing ephemeral authentication and turn detection correctly, decides whether a voice experience feels natural or frustrating.

## Key concepts

### Audio model families (checked 2026-11)

| Purpose | Models | API |
|---|---|---|
| **Speech to text** | `gpt-4o-transcribe`, `gpt-4o-mini-transcribe`, `gpt-4o-transcribe-diarize` (speaker labels), `whisper` | `/audio/transcriptions` (and `/audio/translations` to English for whisper) |
| **Text to speech** | `gpt-4o-mini-tts` (steerable voice style), `tts`, `tts-hd` | `/audio/speech` |
| **Audio in chat** | `gpt-audio`, `gpt-audio-1.5`, `gpt-audio-mini` | Chat Completions with audio input/output |
| **Realtime speech-to-speech** | `gpt-realtime`, `gpt-realtime-mini`, `gpt-realtime-1.5`, `gpt-realtime-2`, `gpt-realtime-2.1`, `gpt-realtime-2.1-mini` | Realtime API (WebRTC, WebSocket, SIP) |
| **Realtime specialised** | `gpt-realtime-translate` (live translation), `gpt-realtime-whisper` and `gpt-live-transcribe` (live transcription) | Realtime API |

Realtime models support **global deployments**; check region availability. Azure **Speech** (Foundry Tools) adds batch and fast transcription, custom speech, neural voices, MAI-Voice, MAI-Transcribe and the **Voice Live API** (topic 20).

### Realtime API transports

| Transport | Use | Microsoft guidance |
|---|---|---|
| **WebRTC** | Browser and mobile clients streaming audio | **Recommended in most cases**: lower latency, built-in codecs, jitter and packet-loss handling |
| **WebSocket** | Server-to-server or server-to-client streaming (for example a backend bridging a telephony stream) | Higher latency than WebRTC for audio |
| **SIP** | Telephony integration | Connect phone calls directly (check current support) |

Use the **`/openai/v1`** path for the GA Realtime API. Preview-era endpoints and event shapes have a migration guide.

### Ephemeral tokens (WebRTC)

Browsers must never hold an API key or a long-lived Entra token. The pattern is:

```text
Browser ──(user signs in to your app)──► Your token service (backend, managed identity)
          Token service ── POST https://<res>.openai.azure.com/openai/v1/realtime/client_secrets
                           (Entra ID or key; body sets session config: model deployment, instructions, voice)
          ◄── ephemeral client secret (short-lived)
Browser ── WebRTC SDP offer ──► https://<res>.openai.azure.com/openai/v1/realtime/calls  (Bearer <ephemeral>)
          ◄── SDP answer → audio flows peer-to-peer with the service; events on a data channel
```

More secure variant: the token service also **proxies the SDP negotiation**, so the browser never sees the ephemeral token. You can also open an **observer WebSocket** to monitor or record a session server-side.

### Session configuration (main fields)

| Field | Purpose |
|---|---|
| `instructions` | System-style instructions for the voice agent |
| `voice` | Output voice (model-dependent list) |
| `turn_detection` | Voice activity detection: **server VAD** (silence-based) or **semantic VAD** (detects when the user has finished speaking based on content); thresholds and silence duration; or `null` for push-to-talk |
| `input_audio_transcription` | Also transcribe user audio (for logs and UI captions) |
| `tools` | Function tools the realtime model can call during the conversation |
| `input/output audio format` | PCM16 and other supported formats |
| Noise reduction | Model-dependent options for noisy environments |

Events flow both ways (for example `session.update`, `input_audio_buffer.append`, `response.create`, `response.output_audio.delta`, `conversation.item.input_audio_transcription.completed`). Barge-in (the user interrupting) is handled by VAD plus cancelling the in-flight response.

## How it works on Azure

### Transcription (request/response)

```python
with open("call.wav", "rb") as f:
    t = client.audio.transcriptions.create(model="<gpt-4o-transcribe deployment>", file=f, language="en")
print(t.text)
```

Audio file uploads are limited (for example 20 MB per file in the audio API, and default speech-to-text quota of 3 RPM), so use Azure Speech batch transcription for large archives.

### Text to speech

```python
speech = client.audio.speech.create(
    model="<gpt-4o-mini-tts deployment>", voice="alloy",
    input="Your parcel will arrive tomorrow between 9 and 11 am.",
    instructions="Speak warmly and at a calm pace.")
open("notice.mp3", "wb").write(speech.content)
```

### Token service for WebRTC (Python, keyless)

```python
import requests
from azure.identity import DefaultAzureCredential

cred = DefaultAzureCredential()

def mint_realtime_secret(resource: str, deployment: str) -> str:
    token = cred.get_token("https://cognitiveservices.azure.com/.default").token
    body = {"session": {"type": "realtime", "model": deployment,
                        "instructions": "You are a helpful, brief support agent.",
                        "audio": {"output": {"voice": "marin"}}}}
    r = requests.post(f"https://{resource}.openai.azure.com/openai/v1/realtime/client_secrets",
                      headers={"Authorization": f"Bearer {token}"}, json=body, timeout=10)
    r.raise_for_status()
    return r.json()["value"]          # short-lived secret returned to an authenticated browser session
```

The session body schema and voice names depend on the model and API version, so copy the current shape from the Realtime WebRTC how-to. Protect this endpoint with your app's user authentication and rate limiting.

### Voice agent architecture

```text
Browser / mobile (WebRTC) ──► Realtime model deployment (Foundry)
        ▲                          │ tool calls (function events)
        │                          ▼
  Token service (Container Apps / Functions, managed identity)
        │                     Tool back ends (APIs, MCP via APIM) with per-user authorisation
        └── observer WebSocket → transcripts → Event Hubs / storage for QA and analytics
```

For contact centres, combine Azure Communication Services (telephony, call automation) with the Realtime API or Voice Live API (topic 10 file 08).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Realtime vs cascaded STT → LLM → TTS | Realtime for natural, low-latency conversation and barge-in; cascaded when you need custom vocabularies, specific neural voices, text moderation between steps, or cheaper text tokens |
| WebRTC vs WebSocket | WebRTC for clients; WebSocket for server-side bridges and observers |
| Realtime API vs Voice Live API | Realtime API for direct model control; Voice Live (Azure Speech) for an integrated voice-agent stack (noise suppression, Azure voices, avatars, multiple model options) |
| Turn detection | Semantic VAD for conversational naturalness; server VAD with tuned silence for noisy or scripted flows; push-to-talk for kiosks |
| Transcripts | Enable input transcription for audit, QA and accessibility; store with consent and retention rules |

## Hands-on

1. Transcribe a 2-minute recording with `gpt-4o-transcribe` and `gpt-4o-transcribe-diarize`; compare speaker labelling.
2. Generate the same sentence with `gpt-4o-mini-tts` using two different `instructions` styles.
3. Build the token service and a minimal browser page from the WebRTC how-to; hold a conversation and test interrupting the model.
4. Add one function tool (for example `get_order_status`) to the realtime session and invoke it by voice.
5. Open an observer WebSocket to capture the transcript of a session.

## Security, cost and reliability notes

- **Security:** never ship API keys to browsers. Use ephemeral secrets minted by an authenticated backend, or proxy SDP entirely.
- **Security:** voice recordings and transcripts are personal data; obtain consent, minimise retention, and secure storage.
- **Security:** voice input can carry prompt-injection; apply the same tool authorisation and approval controls as text agents.
- **Cost:** audio tokens cost more than text; idle open sessions and long system prompts add cost. Close sessions promptly and keep instructions concise.
- **Reliability:** handle network changes and reconnection; set session timeouts; monitor latency (time to first audio).

## Common mistakes

- Using WebSockets from the browser for audio when WebRTC is recommended.
- Exposing the resource key or a broad Entra token to the client.
- Leaving default VAD settings in noisy environments, causing constant interruptions.
- No transcription, so conversations can't be audited or evaluated.
- Using the audio REST API for large call archives instead of Azure Speech batch transcription.

## Check yourself

1. Which realtime transport does Microsoft recommend for browser clients, and why?
2. Describe the ephemeral token flow for WebRTC.
3. What's the difference between server VAD and semantic VAD?
4. Name three request/response audio models and their purposes.
5. When would you choose a cascaded pipeline instead of the Realtime API?
6. What path should GA Realtime API requests use?

<details>
<summary>Answers</summary>

1. WebRTC, for lower latency, built-in media codecs, and handling of jitter and packet loss.
2. An authenticated backend requests an ephemeral client secret from `/openai/v1/realtime/client_secrets` (with Entra ID or a key, setting the session config), returns it to the browser (or proxies SDP), and the browser connects to `/openai/v1/realtime/calls` with that short-lived secret.
3. Server VAD detects turn ends from silence; semantic VAD uses the content of speech to decide when the user has finished.
4. Any three of: `gpt-4o-transcribe` (speech to text), `gpt-4o-transcribe-diarize` (with speaker labels), `whisper` (transcription and translation), `gpt-4o-mini-tts` (steerable text to speech), `tts`/`tts-hd` (text to speech).
5. When you need custom vocabularies or specific voices, text-level moderation or processing between steps, or lower cost per interaction.
6. The `/openai/v1` path.

</details>

## References

- [Realtime API via WebRTC](https://learn.microsoft.com/azure/foundry/openai/how-to/realtime-audio-webrtc)
- [Realtime API via WebSockets](https://learn.microsoft.com/azure/foundry/openai/how-to/realtime-audio-websockets)
- [Realtime API overview](https://learn.microsoft.com/azure/foundry/openai/how-to/realtime-audio)
- [Migrate to the GA Realtime API](https://learn.microsoft.com/azure/foundry/openai/how-to/realtime-audio-preview-api-migration-guide)
- [Foundry Models sold by Azure: audio models](https://learn.microsoft.com/azure/foundry/foundry-models/concepts/models-sold-directly-by-azure#audio-models)
- [Voice Live API](https://learn.microsoft.com/azure/ai-services/speech-service/voice-live)
