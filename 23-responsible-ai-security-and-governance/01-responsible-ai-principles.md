# Responsible AI Principles

## Why it matters

Every AI system makes trade-offs that affect people: who gets a loan, which CV is shortlisted, what a customer is told, what an agent is allowed to do on someone's behalf. Responsible AI is how organisations make those trade-offs deliberately rather than by accident. Regulators (file 08) increasingly require evidence of it, customers ask for it in procurement, and incidents caused by its absence are expensive and public. Microsoft has published principles, an internal **Responsible AI Standard**, impact assessment templates and **transparency notes** for its services, and maps them onto a practical lifecycle of **Discover, Protect, Govern** in Foundry. This file gives every role the shared vocabulary that the rest of the topic builds on.

## Key concepts

### Microsoft's six principles

| Principle | Meaning | Example question |
|---|---|---|
| **Fairness** | Treat all people fairly; avoid allocating opportunities or quality of service unevenly | Does the agent answer as well in Welsh as in English? |
| **Reliability and safety** | Perform reliably and safely in expected and unexpected conditions | What happens when retrieval returns nothing? |
| **Privacy and security** | Protect data and resist attacks | Can a user extract another customer's data? |
| **Inclusiveness** | Empower everyone, including people with disabilities | Does the voice agent work for users with speech impairments? |
| **Transparency** | People understand the system's capabilities, limits and that they're dealing with AI | Is it clear the reply was AI-generated? |
| **Accountability** | People are accountable for AI systems | Who signs off, and who is paged on an incident? |

### From principles to practice

| Artefact | Purpose |
|---|---|
| **Responsible AI Standard** (Microsoft, v2) | Turns principles into goals and requirements for product teams (impact assessment, fitness for purpose, human oversight, data governance, transparency) |
| **Impact assessment** | Documents intended uses, stakeholders, potential harms, benefits and mitigations before building; revisited on significant change |
| **Sensitive uses review** | Escalation for uses affecting life opportunities, physical or psychological harm, or human rights |
| **Transparency notes** | Microsoft's documentation for each AI service: capabilities, intended uses, limitations, evaluation, best practices for responsible deployment |
| **Restricted/Limited Access** | Some capabilities (for example face identification, custom neural voice, modified content filters) require registration and approval |
| **Customer transparency** | Your own user-facing disclosure: AI interaction notices, labelling generated content, explaining limitations |

### Microsoft's lifecycle for Foundry: Discover, Protect, Govern

| Stage | Activities | Where in this roadmap |
|---|---|---|
| **Discover** | Identify and measure quality, safety and security risks before and after deployment (evaluation, red teaming) | [Topic 22](../22-evaluation-observability-and-genaiops/README.md) files 02–04, 09 |
| **Protect** | Guardrails at model output and agent runtime; secure tools and identities; data protection | Files 03–06 |
| **Govern** | Tracing, monitoring, compliance integrations, inventory, policy enforcement | Files 07–10; Topic 22 files 06–07 |

This mirrors the NIST AI RMF functions (Govern, Map, Measure, Manage) covered in file 08.

### Harms to consider

| Harm type | Examples |
|---|---|
| **Allocation** | Unequal access to jobs, credit, services |
| **Quality of service** | Lower accuracy for some groups or languages |
| **Stereotyping and demeaning content** | Biased or offensive outputs |
| **Over-reliance** | Users trusting confident but wrong outputs |
| **Harmful content** | Hate, violence, sexual, self-harm content |
| **Security and privacy** | Data leakage, prompt injection, unauthorised actions |
| **Agentic harms** | Irreversible actions, actions beyond the user's intent, runaway cost |

### Shared responsibility

Microsoft is responsible for the models' base safety training, default guardrails, platform security and transparency documentation. **You** are responsible for the use case, the impact assessment, configuration (guardrails, tools, data), evaluation with your data, user disclosure, human oversight and operations.

## How it works on Azure

| Need | Azure capability |
|---|---|
| Read the limits of the service | Transparency notes for Azure OpenAI, agents, Content Safety, Speech, Vision |
| Measure harms | Foundry evaluators and AI Red Teaming Agent |
| Mitigate | Guardrails (content filters, Prompt Shields, PII, task adherence), safety system messages, tool approvals |
| Monitor | Foundry observability, Defender for Cloud alerts in **Risks + alerts** |
| Predictive ML fairness and explainability | Azure ML Responsible AI dashboard ([Topic 21 file 10](../21-azure-machine-learning-and-mlops/10-responsible-ai-dashboard.md)) |
| Report issues | Microsoft Security Response Center (MSRC) for vulnerabilities or safety problems in Foundry Models |

A layered mitigation model helps when explaining controls to stakeholders:

```text
Model (base safety training, model choice)
  └─ Safety system (guardrails, Prompt Shields, PII detection)
       └─ System message and grounding (instructions, safety templates, RAG)
            └─ User experience (disclosure, citations, human review, feedback)
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **When to do an impact assessment** | Every production AI use case; light-touch for low-risk internal tools, full for customer-facing or consequential decisions. |
| **Human oversight** | Human-in-the-loop for consequential decisions; human-on-the-loop (monitoring, override) for lower risk. |
| **Disclosure** | Always tell users they're interacting with AI; label generated media. |
| **Fitness for purpose** | Don't deploy a model for a task its transparency note warns against without your own evidence. |
| **Proportionality** | Match controls to risk tier (file 09); avoid both under- and over-governance. |
| **Inclusiveness testing** | Test across languages, accents, accessibility needs and demographic groups relevant to your users. |

## Hands-on

1. Read the Azure OpenAI and Foundry agents transparency notes; list five limitations relevant to your use case.
2. Draft an impact assessment for an agent: intended uses, out-of-scope uses, stakeholders, harms, mitigations, residual risk, owner.
3. Map each identified harm to a Discover, Protect or Govern activity and an Azure capability.
4. Write the user-facing AI disclosure and the escalation path to a human.
5. Open **Risks + alerts** in your Foundry project and review recommendations.

## Security, cost and reliability notes

- Responsible AI and security overlap: prompt injection is both a security and a safety issue; treat them in one risk register.
- Mitigations have costs (latency of guardrails, evaluation spend, human review); budget for them in the business case ([Topic 22 file 10](../22-evaluation-observability-and-genaiops/10-measuring-business-value.md)).
- Reliability: define fallback behaviour when guardrails block or the model is unavailable.

## Common mistakes

| Mistake | Fix |
|---|---|
| Principles poster with no process | Impact assessments, gates and owners |
| Assessment done once at the start | Revisit on model, data or scope change |
| Relying only on Microsoft's default filters | Configure, evaluate and red team with your scenarios |
| No user disclosure | Explicit AI notice and limitations |
| Fairness considered only for ML models | Also test LLM apps across languages and groups |

## Check yourself

1. List Microsoft's six responsible AI principles.
2. What is a transparency note, and how should you use it?
3. What does an impact assessment contain?
4. Describe Discover, Protect and Govern.
5. Name four types of harm to consider.
6. Split responsibilities between Microsoft and the customer.

<details>
<summary>Answers</summary>

1. Fairness, reliability and safety, privacy and security, inclusiveness, transparency, accountability.
2. Microsoft's documentation of a service's capabilities, intended uses, limitations and responsible-deployment practices; use it to check fitness for purpose and design mitigations.
3. Intended and out-of-scope uses, stakeholders, potential harms and benefits, mitigations, residual risk and accountable owner.
4. Discover: identify and measure risks (evaluation, red teaming). Protect: guardrails, secure tools, data protection. Govern: tracing, monitoring, compliance, inventory and policy.
5. Any four: allocation, quality of service, stereotyping/demeaning, over-reliance, harmful content, security/privacy, agentic harms.
6. Microsoft: base model safety, default guardrails, platform security, transparency notes. Customer: use case, impact assessment, configuration, evaluation, disclosure, oversight, operations.

</details>

## References

- [Responsible AI for Microsoft Foundry](https://learn.microsoft.com/azure/foundry/responsible-use-of-ai-overview)
- [Transparency note for Azure OpenAI](https://learn.microsoft.com/azure/foundry/responsible-ai/openai/transparency-note)
- [Transparency note for Foundry agents](https://learn.microsoft.com/azure/foundry/responsible-ai/agents/transparency-note)
- [What is responsible AI? (Azure ML)](https://learn.microsoft.com/azure/machine-learning/concept-responsible-ai)
- [Safety system message templates](https://learn.microsoft.com/azure/foundry/openai/concepts/system-message)
- [Responsible AI business principles (training)](https://learn.microsoft.com/training/paths/responsible-ai-business-principles/)
