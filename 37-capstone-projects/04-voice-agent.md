# Voice Agent

## Why it matters

Voice is returning as a primary channel for customer service, booking and field support, now with agents that understand natural speech and take actions in real time. Voice agents add hard constraints that text chat doesn't have: sub-second latency, interruptions (barge-in), telephony integration, noisy audio, and handover to humans. In this capstone you build a **real-time speech agent with telephony and tool calls**. It shows you can engineer for latency and user experience, not just for accuracy.

**Brief:** Tailspin Clinics wants a phone agent that books, moves and cancels appointments, answers common questions, and hands over to reception when needed. Calls must be recorded only with consent, and no clinical advice may be given.

## Key concepts

### Requirements

| # | Requirement |
|---|---|
| R1 | Inbound phone calls (PSTN number) and a web or mobile voice channel |
| R2 | Real-time speech-to-speech with natural interruptions (barge-in) and noise suppression |
| R3 | Tools: look up patient by date of birth plus a verification question; check availability; book, move or cancel |
| R4 | FAQ answers grounded in clinic information (opening hours, parking, preparation instructions) |
| R5 | Handover to a human with context (summary passed to reception) |
| R6 | Guardrails: no medical advice; escalate symptoms that sound urgent to emergency guidance |
| R7 | Latency target: first audio response within about 1 second at p90 (measure your own baseline) |
| R8 | Post-call summary, transcript (with consent) and analytics |

### Skills practised

Topics [20 file 03 · Azure Speech](../20-foundry-tools/03-azure-speech.md), [18](../18-agents-with-foundry-agent-service/README.md), [15](../15-foundry-models-and-azure-openai/README.md), [34 file 10](../34-ai-solution-architecture/10-document-and-voice-solution-architectures.md), [22](../22-evaluation-observability-and-genaiops/README.md) and [23](../23-responsible-ai-security-and-governance/README.md).

## How it works on Azure

### Architecture

```text
PSTN caller ─▶ Azure Communication Services (phone number, Call Automation)
                 └─ bidirectional media streaming (WebSocket) ─▶ Voice service (Container Apps)
Web caller ─▶ WebRTC / WebSocket ─────────────────────────────▶ Voice service
Voice service ─▶ Voice Live API (speech in/out, turn detection, noise suppression, avatar optional)
             └─ connected model or Foundry agent with tools:
                  ├─ verify_patient(), get_slots(), book(), move(), cancel()  → clinic API via API Management
                  └─ knowledge: clinic FAQ (file search / AI Search)
Handover: Call Automation transfer to reception queue + summary to CRM / Teams
Telemetry: per-turn latency, interruptions, tool calls → Application Insights
```

### Two technical approaches

| Approach | Pros | Cons |
|---|---|---|
| **Voice Live API** (managed speech-to-speech with a model or agent) | Lowest integration effort; built-in turn detection, noise suppression and echo cancellation | Less control over individual stages |
| **Cascaded** (Speech to text → LLM → text to speech) | Full control, custom voices, per-stage choices | More latency to manage; more code |
| **Realtime audio models** directly | Very natural conversation | You manage more of the audio pipeline and tooling |

Check current feature status (GA or Preview) and supported regions for Voice Live and realtime models before you commit.

### Milestones

| Milestone | Deliverable |
|---|---|
| M1 | Conversation design: intents, happy paths, verification flow, escalation rules, sample dialogues |
| M2 | Web voice prototype with Voice Live and one tool |
| M3 | All tools with server-side validation; FAQ grounding |
| M4 | ACS phone number, inbound call routing and media streaming to the voice service |
| M5 | Handover with summary; consent prompt and recording policy |
| M6 | Latency measurement and optimisation (region placement, shorter prompts, streaming, filler phrases) |
| M7 | Evaluation on 30 scripted calls, including noisy audio, accents, interruptions and adversarial requests |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Speech-to-speech vs cascaded | Start with Voice Live for speed; cascaded where you need specific voices or per-stage control |
| Verification | Minimal personal data over the phone; never read back full records |
| Region | Co-locate ACS, voice service and model deployment to cut latency |
| Tool latency | Cache availability; use fast APIs; say a short holding phrase while tools run |

## Hands-on

1. Complete milestones M1–M7.
2. Publish a latency breakdown (speech detection, model, tool, speech output) before and after optimisation.
3. Record a demo call that includes an interruption and a handover.
4. Test five adversarial calls (requests for medical advice, someone impersonating a patient) and document the outcomes.
5. Estimate cost per call minute (ACS telephony, speech and model, compute).

## Security, cost and reliability notes

- Comply with call-recording consent laws; disclose that callers are talking to an AI agent.
- Treat transcripts as personal data, and health data where relevant: encryption, retention limits, access controls.
- Write tools validate identity and enforce business rules server-side.
- Plan fallbacks: if the AI service fails, route the call to a human queue.

## Common mistakes

| Mistake | Fix |
|---|---|
| Ignoring latency until the end | Measure from M2 |
| No barge-in handling | Use service turn detection; stop playback on user speech |
| Reading back sensitive data | Minimal verification and confirmation |
| No human fallback | Transfer on failure, request or low confidence |

## Check yourself

1. Which Azure service provides the phone number and call control?
2. What does Voice Live provide over a cascaded pipeline?
3. How would you cut perceived latency during a slow tool call?
4. Which guardrails are mandatory for a clinic voice agent?
5. What happens when the AI service is unavailable?

<details>
<summary>Answers</summary>

1. Azure Communication Services (phone numbers and Call Automation).
2. Managed real-time speech-to-speech with turn detection, noise suppression and echo cancellation, connected to a model or agent.
3. Co-locate services, stream responses, and say a short holding phrase while the tool runs.
4. AI disclosure, recording consent, no medical advice, urgent-symptom escalation and identity verification for actions.
5. The call routes to a human reception queue (fallback transfer).

</details>

## References

- [Voice Live API overview](https://learn.microsoft.com/azure/ai-services/speech-service/voice-live)
- [Call Automation overview](https://learn.microsoft.com/azure/communication-services/concepts/call-automation/call-automation)
- [Speech service overview](https://learn.microsoft.com/azure/ai-services/speech-service/overview)
- [Transparency note for Azure Speech](https://learn.microsoft.com/azure/ai-foundry/responsible-ai/speech-service/text-to-speech/transparency-note)
