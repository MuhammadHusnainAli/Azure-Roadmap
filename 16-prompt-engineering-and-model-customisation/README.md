# 16 · Prompt Engineering and Model Customisation

> Tracks: E ● A ○ C ○ · Part D · AI on Azure

Before you reach for a bigger model, a vector index or a training job, the cheapest improvement is usually in what you send the model. This topic covers the full spectrum of shaping model behaviour on Microsoft Foundry: designing prompts and system messages, prompting reasoning models with effort and verbosity controls, engineering the context window (retrieval, memory, tool results, compaction), managing prompts as versioned, tested code, and deciding when to move on to fine-tuning or distillation. It then covers Foundry's Managed Training (SFT, DPO, RFT, vision), Programmed Training, synthetic data generation, and how to deploy, evaluate and operate fine-tuned models without runaway hosting costs.

## What you will be able to do

- Write structured system messages with role, rules, output contract and fallback behaviour.
- Prompt reasoning models with goals, constraints and an output contract; tune `reasoning_effort` and `verbosity`; handle reasoning tokens and incomplete responses.
- Decide what goes into the context window and apply truncation, summarisation and compaction, service-side or client-side.
- Version prompts and agents in Git, and gate changes with Foundry evaluations in CI.
- Choose between prompting, RAG, fine-tuning and distillation and defend the choice with cost and quality data.
- Run SFT, DPO and RFT jobs in Foundry and pick training types that respect data residency.
- Generate and filter synthetic training data and distil a large model into a smaller one.
- Deploy fine-tuned models, gate promotion on evaluations, and plan retraining.

## Prerequisites

[Topic 13](../13-ai-and-generative-ai-foundations/README.md) (LLMs, tokens), [Topic 14](../14-microsoft-foundry-platform/README.md) (Foundry projects, roles, SDK) and [Topic 15](../15-foundry-models-and-azure-openai/README.md), especially file 07 (Responses API), file 08 (structured outputs) and file 12 (caching and batch).

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Prompt Design Fundamentals](01-prompt-design-fundamentals.md) | 45 min | Python, structured outputs |
| 02 | [Prompting Reasoning Models](02-prompting-reasoning-models.md) | 40 min | Python, effort comparison |
| 03 | [Context Engineering](03-context-engineering.md) | 50 min | Python, compaction |
| 04 | [Prompt Management](04-prompt-management.md) | 45 min | Prompty, GitHub Actions |
| 05 | [Customisation Decision Guide](05-customisation-decision-guide.md) | 35 min | Cost and quality spike |
| 06 | [Fine-Tuning in Foundry](06-fine-tuning-in-foundry.md) | 60 min | Python SDK, azd |
| 07 | [Distillation and Synthetic Data](07-distillation-and-synthetic-data.md) | 45 min | Python, Data Generation |
| 08 | [Deploying and Operating Fine-Tuned Models](08-deploying-and-operating-fine-tuned-models.md) | 40 min | Bicep, evaluation |

**AI engineers:** all of it, with 01, 02, 03 and 06 in depth. **Architects:** 03, 05 and 08 in depth, skim the rest. **Consultants:** 01, 05 and the cost sections of 06 and 08.

## Completion checklist

- [ ] I have written a modular system message with an output contract and a "when unsure" rule, and measured it on a test set.
- [ ] I can explain reasoning tokens, choose an effort level, and handle `incomplete` responses.
- [ ] I know which history model my agent uses (service-managed or application-managed) and how context is compacted.
- [ ] My prompts live in Git, and a pull request that changes them runs an evaluation.
- [ ] I can argue for or against fine-tuning on a real use case with cost, quality and lifecycle data.
- [ ] I have run a fine-tuning job, read its loss curves and compared checkpoints.
- [ ] I have generated and filtered synthetic or teacher data and trained a student model.
- [ ] I have deployed a fine-tuned model, compared it with production, and deleted the loser.

## Maps to

- **AI-103**: implement prompt engineering, manage prompts, fine-tune models, and evaluate customised models.
- **AI-300**: model customisation, fine-tuning lifecycle and GenAIOps practices.
- **AZ-305**: weigh customisation options against cost, residency and operations.
- **AI-901**: prompt engineering and customisation concepts (foundational).

## Next

17 · [Retrieval-Augmented Generation and Knowledge](../17-rag-and-knowledge/README.md)
