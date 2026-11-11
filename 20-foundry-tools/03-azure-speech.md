# Azure Speech

## Why it matters

Speech is how many customers and employees actually interact: phone calls, meetings, field work, accessibility. **Azure Speech in Foundry Tools** converts speech to text and text to speech, translates spoken language, identifies languages, assesses pronunciation, and powers **Voice Live** for real-time voice agents ([Topic 18 file 11](../18-agents-with-foundry-agent-service/11-voice-agents.md)). Microsoft uses it in Teams captions, Office dictation and Edge Read Aloud. Call-centre analytics, captioning, voice bots and accessible apps all depend on choosing the right recognition mode, voice type and customisation, and on handling voice data responsibly.

## Key concepts

### Speech to text (STT)

| Mode | Use |
|---|---|
| **Real-time transcription** | Streaming audio (microphones, calls, live captions) with interim and final results |
| **MAI-Transcribe Streaming API** | Low-latency transcription of continuous streams |
| **Fast transcription** | Synchronous, faster-than-real-time transcription of audio files |
| **Azure LLM Speech** (enhanced mode on fast transcription) | LLM-powered transcription and translation of recordings with contextual understanding and prompt tuning |
| **MAI-Transcribe API** | Fast transcription of audio files with MAI models |
| **Batch transcription** | Asynchronous processing of large volumes (call archives), REST API |
| **Custom speech** | Train acoustic, language and pronunciation models for noise and jargon; deploy custom endpoints |

Features include diarisation (who spoke when), multichannel audio, profanity filtering, phrase lists, word-level timestamps and language identification.

### Text to speech (TTS)

| Voice option | Description |
|---|---|
| **Standard (neural) voices** | Hundreds of natural voices across languages and styles |
| **MAI-Voice** | Expressive voices from Microsoft AI models |
| **Custom voice** | Brand voice trained on recordings of voice talent; **professional custom voice is Limited Access** with consent and disclosure requirements |
| **Personal voice** | Voice created from a short sample, for approved scenarios |
| **Text-to-speech avatar** | Photorealistic talking avatar video, real-time or batch (custom avatar Limited Access) |

**SSML** (Speech Synthesis Markup Language) controls pitch, rate, pronunciation, pauses, styles and multiple voices.

### Other capabilities

| Capability | Use |
|---|---|
| **Speech translation** | Real-time speech-to-text and speech-to-speech translation; video translation |
| **Language identification** | Detect spoken language from candidates, alone or with STT/translation |
| **Pronunciation assessment** | Accuracy, fluency, completeness and prosody scoring for language learning and training |
| **Speaker recognition** | Speaker verification/identification (Limited Access) |
| **Voice Live** | Real-time speech-to-speech orchestration for voice agents (recognition, turn-taking, model, synthesis) |

### Delivery

| Option | Notes |
|---|---|
| Cloud (Foundry resource or Speech resource) | Full feature set |
| **Containers** | STT, custom STT, neural TTS (GA); fast transcription (Preview); connected or disconnected |
| Sovereign clouds | Azure Government, Azure operated by 21Vianet (subset) |
| Tools | Speech Studio (no-code), Speech SDK (many languages), Speech CLI, REST APIs (batch, fast transcription) |

## How it works on Azure

Keyless real-time recognition with the Speech SDK (Python):

```python
# pip install azure-cognitiveservices-speech azure-identity
import azure.cognitiveservices.speech as speechsdk
from azure.identity import DefaultAzureCredential

config = speechsdk.SpeechConfig(token_credential=DefaultAzureCredential(),
                                endpoint="https://contoso-foundry.cognitiveservices.azure.com/")
config.speech_recognition_language = "en-GB"
recognizer = speechsdk.SpeechRecognizer(speech_config=config,
                                        audio_config=speechsdk.audio.AudioConfig(filename="call.wav"))
result = recognizer.recognize_once()
print(result.text)
```

Text to speech with SSML:

```python
synth = speechsdk.SpeechSynthesizer(speech_config=config)
ssml = """<speak version='1.0' xml:lang='en-GB'>
  <voice name='en-GB-SoniaNeural'>Your order <say-as interpret-as='characters'>CX</say-as> 4471
  will arrive <emphasis>today</emphasis> before five p.m.</voice></speak>"""
synth.speak_ssml_async(ssml).get()
```

Entra authentication requires a custom subdomain endpoint and the **Cognitive Services Speech User** (or Contributor) role.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Real-time vs fast vs batch** | Real-time for live interaction and captions; fast transcription for files needing quick turnaround; batch for large archives at lowest operational effort. |
| **Base vs custom speech** | Start with base models plus phrase lists; train custom models when word error rate on your audio (noise, accents, jargon) is unacceptable. Measure WER on a test set. |
| **Azure Speech vs realtime LLM audio** | Speech for accurate transcription, diarisation, custom vocabularies, compliance and cost at scale; realtime LLM models for natural conversational agents (often combined via Voice Live). |
| **Voice** | Standard voices for most apps; custom voice only with a strong brand case, consent and Limited Access approval. |
| **Cloud vs container** | Containers for on-premises audio that can't leave the site, or disconnected sites. |
| **Post-processing** | Combine with Language (PII redaction, summarisation, sentiment) or an LLM for call analytics. |

## Hands-on

Build a mini call-analytics pipeline.

1. Upload five call recordings to Blob Storage.
2. Submit a **batch transcription** job (REST) with diarisation enabled; poll for results.
3. Redact PII from the transcripts with Azure Language ([file 04](04-azure-language.md)).
4. Summarise each call and extract action items with an LLM using structured outputs.
5. Create a **phrase list** with your product names and re-run one file with real-time recognition; compare accuracy on those terms.
6. Synthesise a callback message with SSML in two voices and compare.

## Security, cost and reliability notes

- **Voice data is personal data** (sometimes biometric for speaker recognition); obtain consent, inform callers, set retention, and redact PII from transcripts.
- **Custom voice responsibilities**: voice talent consent, disclosure that voices are synthetic, code of conduct; misuse can enable impersonation.
- **Keyless access** with managed identity; private endpoints for call-centre audio.
- **Cost**: billed per audio hour (STT), per character (TTS), custom model training and hosting, avatar minutes; batch and commitment tiers reduce unit cost.
- **Throughput**: concurrent request limits per resource; scale with multiple resources or containers.
- **Accuracy monitoring**: track WER on sampled audio; accents and audio quality drift.

## Common mistakes

| Mistake | Fix |
|---|---|
| Real-time API for transcribing large archives | Batch or fast transcription |
| Training custom models before trying phrase lists | Phrase lists first; measure WER |
| Plain text where pronunciation matters | Use SSML |
| No consent or disclosure for synthetic voices | Follow Limited Access and disclosure requirements |
| Storing raw transcripts with personal data indefinitely | Redact and set retention |

## Check yourself

1. Name four speech-to-text modes and when to use each.
2. What is Azure LLM Speech?
3. What does custom speech improve, and with what data?
4. What voice options exist for text to speech?
5. What does SSML control?
6. Which Speech capabilities are Limited Access?
7. Which Speech containers are GA?
8. How do you authenticate the Speech SDK without keys?

<details>
<summary>Answers</summary>

1. Real-time (live audio), fast transcription (files, quick turnaround), batch (large volumes asynchronously), MAI-Transcribe streaming/API (low-latency streams, fast file transcription); plus LLM Speech enhanced mode.
2. An LLM-powered speech model available through enhanced mode on the fast transcription endpoint, for higher-quality, context-aware transcription and translation with prompt tuning.
3. Accuracy for noisy audio, accents and domain jargon, using acoustic, language (text) and pronunciation data.
4. Standard neural voices, MAI-Voice, custom voice (professional), personal voice and text-to-speech avatar.
5. Pitch, rate, volume, pronunciation, pauses, emphasis, speaking styles and voice switching.
6. Professional custom neural voice, custom text-to-speech avatar and speaker recognition.
7. Speech to text, custom speech to text and neural text to speech (fast transcription is Preview).
8. `SpeechConfig(token_credential=DefaultAzureCredential(), endpoint=<custom subdomain endpoint>)` with a Speech role assigned.

</details>

## References

- [What is Azure Speech?](https://learn.microsoft.com/azure/ai-services/speech-service/overview)
- [Microsoft Entra authentication with the Speech SDK](https://learn.microsoft.com/azure/ai-services/speech-service/how-to-configure-azure-ad-auth)
- [Batch transcription](https://learn.microsoft.com/azure/ai-services/speech-service/batch-transcription)
- [Custom speech overview](https://learn.microsoft.com/azure/ai-services/speech-service/custom-speech-overview)
- [Speech Synthesis Markup Language](https://learn.microsoft.com/azure/ai-services/speech-service/speech-synthesis-markup)
- [Voice Live overview](https://learn.microsoft.com/azure/ai-services/speech-service/voice-live)
