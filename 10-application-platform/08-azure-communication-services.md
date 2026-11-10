# Azure Communication Services

## Why it matters

Customers expect to reach businesses by chat, phone, SMS, email and video, and increasingly to talk to AI agents on those channels. Azure Communication Services (ACS) provides these channels as APIs on the same platform that runs Microsoft Teams. With **Call Automation** and real-time voice models, it is the main Azure building block for **voice agents** in contact centres and service lines. Engineers build on it; architects and consultants need to know what it can do and how it connects to Foundry.

## Key concepts

### Capabilities

| Capability | What it provides |
|---|---|
| **Identity and access tokens** | ACS user identities and tokens for clients, mapped to your own users |
| **Chat** | Threads, messages, read receipts, typing indicators; Teams interop |
| **Voice and video calling** | Browser, mobile and desktop SDKs; group calls; Teams meetings interop |
| **Phone numbers (PSTN)** | Acquire toll-free and geographic numbers (by country availability), or bring your carrier via **direct routing** |
| **Call Automation** | Server-side control of calls: answer, play, recognise speech or DTMF, transfer, record, and **stream audio** in real time |
| **Rooms** | Managed virtual meeting spaces with roles and time windows (for telehealth, interviews) |
| **SMS** | Send and receive SMS on toll-free, short code and local numbers (country dependent) |
| **Email** | Transactional and bulk email from Azure-managed or custom verified domains |
| **Advanced Messaging** | WhatsApp Business messages and templates |
| **Teams interoperability** | ACS users join Teams meetings and call Teams users |
| UI Library | Ready-made calling and chat UI components |

### Voice agent architecture

```text
Caller (PSTN) → ACS phone number
   → Event Grid: IncomingCall → your service (Container Apps / Functions)
   → Call Automation: answer call with bidirectional media streaming (WebSocket)
        ⇅ audio
   Voice agent service
        ├─ Azure AI Speech Voice Live API / realtime model (speech in → reasoning → speech out)
        │    or: speech-to-text → Foundry agent/LLM → text-to-speech
        ├─ tools: CRM lookup, order status, booking (MCP / APIs)
        └─ handoff: transfer to a human agent (Teams, contact centre) with context
   Call recording and transcripts → storage → analytics and evaluation
```

| Design point | Detail |
|---|---|
| Latency budget | People notice gaps over roughly a second. Realtime speech-to-speech models and streaming at every stage keep responses natural |
| **Barge-in** | Callers interrupt; the agent must stop speaking and listen (voice activity detection) |
| Turn detection | Semantic or acoustic end-of-turn detection to avoid cutting people off |
| Handoff | Always offer escalation to a human; pass the conversation summary |
| Telephony features | DTMF input, call transfer, hold music, recording consent |
| Compliance | Recording disclosures, data retention, PCI considerations for card payments (pause recording) |

The **Voice Live API** in Azure AI Speech combines speech recognition, a generative model, and speech synthesis with noise suppression, echo cancellation and turn detection in one low-latency interface, and is designed to plug into ACS Call Automation (topic 20 covers Speech).

### SMS and email for AI applications

| Use | Notes |
|---|---|
| Notifications from agents (appointment confirmations, order updates) | Use templates; respect opt-out |
| Two-way SMS assistants | Inbound SMS via Event Grid → agent → reply |
| Email summaries and reports | ACS Email with verified domains; DKIM, SPF, DMARC configured |
| WhatsApp assistants | Advanced Messaging with approved templates |

Regulations differ by country: toll-free verification, 10DLC registration in the US, sender ID rules, consent requirements. Plan lead time for number provisioning and verification.

### Pricing model

Pay per use: per minute for calling (PSTN rates by destination), per message for SMS and chat, per email and per GB, per number per month, plus AI services (Speech, models) separately.

## How it works on Azure

### Answering a call and streaming audio (outline, Python)

```python
from azure.communication.callautomation import (
    CallAutomationClient, MediaStreamingOptions, MediaStreamingTransportType,
    MediaStreamingContentType, MediaStreamingAudioChannelType, AudioFormat,
)
from azure.identity import DefaultAzureCredential

client = CallAutomationClient("https://<acs>.communication.azure.com", DefaultAzureCredential())

def on_incoming_call(event):            # from Event Grid IncomingCall
    streaming = MediaStreamingOptions(
        transport_url="wss://<your-agent-service>/media",
        transport_type=MediaStreamingTransportType.WEBSOCKET,
        content_type=MediaStreamingContentType.AUDIO,
        audio_channel_type=MediaStreamingAudioChannelType.MIXED,
        start_media_streaming=True,
        enable_bidirectional=True,
        audio_format=AudioFormat.PCM24_K_MONO,
    )
    client.answer_call(
        incoming_call_context=event["data"]["incomingCallContext"],
        callback_url="https://<your-agent-service>/callbacks",
        media_streaming=streaming,
    )
```

Your WebSocket endpoint receives caller audio, forwards it to the Voice Live API or a realtime model, and streams the synthesised reply back into the call. Microsoft samples provide complete implementations; parameter names change between SDK versions.

### Sending an email

```python
from azure.communication.email import EmailClient
from azure.identity import DefaultAzureCredential

email = EmailClient("https://<acs>.communication.azure.com", DefaultAzureCredential())
poller = email.begin_send({
    "senderAddress": "DoNotReply@notify.contoso.com",
    "recipients": {"to": [{"address": "customer@example.com"}]},
    "content": {"subject": "Your appointment", "plainText": "Your appointment is confirmed for 18 Oct at 10:00."},
})
print(poller.result()["status"])
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Build on ACS vs buy a contact centre | ACS for custom experiences and AI-first channels; Dynamics 365 Contact Center or partners for full contact centre suites (they also use ACS underneath) |
| Speech-to-speech realtime vs pipeline (STT → LLM → TTS) | Realtime (Voice Live, realtime models) for natural latency and barge-in; pipeline for maximum control over each stage, specific voices or text-based agent reuse |
| Agent logic | Reuse the same Foundry agent and tools across chat and voice; adapt prompts for spoken style |
| Numbers | Check country availability early; direct routing to reuse existing carriers |
| Human handoff | Design it from day one, including context transfer |

## Hands-on

1. Create an ACS resource, connect an Email Communication Services resource with an Azure-managed domain, and send yourself an email with the sample above.
2. Create user identities and tokens and run the ACS calling quickstart in a browser to make a VoIP call between two tabs.
3. If you can acquire a phone number in your country, follow the Call Automation "voice agent with Voice Live" or "realtime audio" sample from Azure Samples, and call your number to talk to an agent.
4. Write a short design note for a voice agent: latency budget, barge-in, handoff, recording consent and data retention.

```bash
az communication create -g $RG -n acs-lab-$RANDOM --location global --data-location Europe
az communication identity user create --connection-string "<connection-string>"   # lab only; prefer Entra
```

## Security, cost and reliability notes

- **Security:** use Entra authentication for server-side SDKs; issue short-lived ACS user tokens to clients from your own authenticated back end.
- **Security:** voice agents are susceptible to social engineering and prompt injection through speech; apply the same tool restrictions and confirmations as chat agents, plus identity verification before sensitive actions.
- **Security:** recordings and transcripts contain personal data; encrypt, restrict, retain only as needed, and disclose recording.
- **Cost:** PSTN minutes, numbers, and AI audio tokens add up; monitor per-call cost and cap call duration.
- **Reliability:** design for network jitter and model latency spikes; have fallbacks (play a message, transfer to a human) when the AI path fails.

## Common mistakes

- Treating a voice agent as a chat agent with text-to-speech bolted on, with long monologues and no barge-in.
- No path to a human.
- Discovering phone number regulations late in the project.
- Exposing the ACS connection string in client apps.
- Recording calls without disclosure or retention policies.

## Check yourself

1. Name five communication capabilities ACS provides.
2. What does Call Automation do?
3. Describe the flow of an inbound call handled by an AI voice agent.
4. What is barge-in and why does it matter?
5. What does the Voice Live API combine?
6. How should client applications authenticate to ACS?

<details>
<summary>Answers</summary>

1. Any five of: chat, voice and video calling, phone numbers (PSTN), Call Automation, Rooms, SMS, email, WhatsApp messaging, Teams interoperability.
2. It lets a server answer, control and route calls, play audio, recognise speech or DTMF, record, transfer, and stream call audio in real time.
3. A call to an ACS number raises an IncomingCall event; your service answers with Call Automation and bidirectional media streaming; audio flows to a realtime speech/model service with tools; responses stream back; the call can be transferred to a human.
4. The caller interrupting the agent; the agent must stop speaking and listen, or the conversation feels robotic and frustrating.
5. Speech recognition, a generative model and speech synthesis, with audio processing such as noise suppression, echo cancellation and turn detection, in one low-latency API.
6. They receive short-lived ACS user access tokens issued by your own authenticated back end; they never hold the resource's keys.

</details>

## References

- [Azure Communication Services overview](https://learn.microsoft.com/azure/communication-services/overview)
- [Call Automation overview](https://learn.microsoft.com/azure/communication-services/concepts/call-automation/call-automation)
- [Audio streaming with Call Automation](https://learn.microsoft.com/azure/communication-services/concepts/call-automation/audio-streaming-concept)
- [Voice Live API](https://learn.microsoft.com/azure/ai-services/speech-service/voice-live)
- [Phone number availability by country](https://learn.microsoft.com/azure/communication-services/concepts/numbers/sub-eligibility-number-capability)
- [Email overview](https://learn.microsoft.com/azure/communication-services/concepts/email/email-overview)
- [Advanced Messaging for WhatsApp](https://learn.microsoft.com/azure/communication-services/concepts/advanced-messaging/whatsapp/whatsapp-overview)
