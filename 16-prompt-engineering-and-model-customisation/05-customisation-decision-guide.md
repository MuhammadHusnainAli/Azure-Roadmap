# Customisation Decision Guide

## Why it matters

"Should we fine-tune?" is one of the most common questions in AI projects, and the answer is usually "not yet". Fine-tuning adds training cost, hourly hosting cost, a retraining burden and lifecycle risk. Many problems that look like they need it are really prompt, context or retrieval problems. Equally, some workloads (high-volume, narrow, latency-sensitive) are badly served by an ever-growing prompt on a frontier model. This file gives a structured way to choose between **prompting**, **RAG**, **fine-tuning** and **distillation**, and to explain the choice to a client.

## Key concepts

### The four levers

| Lever | Changes | Teaches the model… | Typical effort |
|---|---|---|---|
| **Prompt and context engineering** (files 01–03) | Input only | Nothing permanently; conditions one request | Hours to days |
| **Retrieval-augmented generation** ([Topic 17](../17-rag-and-knowledge/README.md)) | Input, from a knowledge source | Nothing; supplies facts at runtime | Days to weeks |
| **Fine-tuning** (file 06) | Model weights (or adapter) | Behaviour, format, style, tool use, domain patterns | Weeks, plus ongoing |
| **Distillation** (file 07) | Weights of a smaller model, trained on a larger model's outputs | To imitate a stronger model on a narrow task, more cheaply | Weeks, plus ongoing |

### Knowledge vs behaviour

The single most useful rule:

- **Missing or changing knowledge → RAG.** Facts change, need citations and access control, and must be updateable without retraining. Fine-tuning is a poor way to store facts: it is hard to update, cannot be permission-trimmed, and the model may still hallucinate.
- **Inconsistent behaviour → prompting first, then fine-tuning.** Format, tone, classification boundaries, tool-calling accuracy and domain reasoning patterns are behaviours.

Microsoft's fine-tuning overview lists where fine-tuning helps: improving accuracy beyond what fits in a prompt, reducing prompt overhead (fewer examples, lower token cost and latency), aligning style and structure, improving tool use, using retrieved context more effectively, and specialising a smaller model. It also states that fine-tuning **does not replace retrieval for current information or application-level safety controls**, and benefits are not guaranteed.

### They combine

These are not exclusive. A common mature pattern is a **fine-tuned or distilled small model, grounded with RAG, with a concise system prompt**. Fine-tuning can even teach a model to use retrieved context better (ignore irrelevant chunks, cite correctly).

## How it works on Azure

| Option | Foundry capability |
|---|---|
| Prompting | Playgrounds, prompt agents, Prompty, evaluations |
| RAG | Azure AI Search, Foundry IQ, file search tool, Work IQ and Fabric IQ |
| Fine-tuning | **Managed Training** with SFT, DPO and RFT; **Programmed Training** (Preview, access approval required) for custom training loops |
| Distillation | Teacher-generated SFT data; synthetic data generation (Preview) |
| Reinforcement on business workflows | **Frontier Tuning**: announced at Build 2026, private preview through Microsoft's Forward Deployed Engineering team, with Foundry and Copilot Studio availability "upcoming" |

Supported base models, training types (Standard, Data zone, Global, Developer) and deployment options vary by model; see file 06.

## Design decisions and trade-offs

### Decision flow

```text
1. Have you written a clear prompt with an output contract and evaluated it?
      no  → do that first (files 01–04)
2. Does the model lack facts, or need private/current/permissioned data?
      yes → RAG (Topic 17). Then re-evaluate.
3. Does it still fail on behaviour: format, tone, labels, tool calls, domain reasoning?
      → try few-shot examples, structured outputs, a stronger model, or higher reasoning effort
4. Still failing, or prompts are now huge and expensive at your volume?
      → fine-tune (file 06) if you have hundreds+ good examples or a reliable grader
5. Is a large model good enough but too slow/expensive at volume?
      → distil to a smaller model (file 07)
```

### Comparison

| Factor | Prompting | RAG | Fine-tuning | Distillation |
|---|---|---|---|---|
| Up-front cost | Very low | Medium (index, pipeline) | Training jobs and data work | Teacher inference plus training |
| Running cost | Tokens per call (grows with prompt length) | Search service plus extra input tokens | Per-token inference **plus hourly hosting** (for many deployment types) | As fine-tuning, but on a cheaper base model |
| Time to change | Minutes | Re-index | Retrain, re-evaluate, redeploy | Regenerate data, retrain |
| Fresh facts | No | **Yes** | No | No |
| Citations and permissions | No | **Yes** | No | No |
| Consistent format/style | Good with structured outputs | Unaffected | **Very good** | Good |
| Latency | Grows with prompt | Adds retrieval step | Can drop (shorter prompts) | **Lowest** (smaller model) |
| Data needed | A few examples | Your documents | Hundreds to thousands of examples, or a grader for RFT | Prompts plus teacher outputs |
| Lifecycle risk | Low | Low | **Base model retirement forces retraining** | Same |

### Choosing a fine-tuning method

| Your signal | Method |
|---|---|
| You can demonstrate correct outputs | **SFT** |
| You have preferred vs rejected pairs (tone, completeness) | **DPO** |
| Many correct answers exist but you can **score** them reliably | **RFT** (reasoning models such as o4-mini; GPT-5 by invitation) |
| Images are part of the input | **Vision SFT** (gpt-4o, gpt-4.1) |
| You need custom losses or rollouts | **Programmed Training** (Preview) |

### Consultant framing

When a client asks for fine-tuning, ask:

1. What is failing today, measured on what dataset?
2. Is it a knowledge gap or a behaviour gap?
3. What is the request volume, and what does the current prompt cost per call?
4. Who will own retraining when the base model retires?
5. Does the data contain personal or regulated information that will be embedded in weights?

## Hands-on

Run a structured customisation spike on one task (for example, extracting fields from invoices into JSON):

1. **Baseline.** Zero-shot prompt with structured outputs on 100 labelled examples. Record accuracy, average input tokens, latency and cost per 1,000 calls.
2. **Prompt.** Add 5 few-shot examples and clearer rules. Record the same metrics.
3. **Model.** Try a stronger model or higher reasoning effort.
4. **RAG** (only if the task needs reference data, such as supplier master data).
5. **Fine-tune estimate.** Using the Foundry pricing page, estimate training cost for 1,000 examples, the hourly hosting cost of a Standard deployment, and inference cost. Compare against the prompt-only cost at your monthly volume.

Produce a one-page table: option, accuracy, p95 latency, monthly cost, effort, risks. That table is the deliverable a client needs.

## Security, cost and reliability notes

- **Training data becomes part of the model.** Personal or confidential data in training sets can be reproduced. Clean, minimise and review data before training.
- **Hosting cost is fixed.** Fine-tuned deployments on Standard incur hourly hosting charges even when idle. The **Developer** deployment type removes the hosting fee but is for evaluation only, lasts 24 hours and has no SLA.
- **Global and Developer training types do not guarantee data residency.** Use Standard or Data zone training if residency matters.
- **Safety still belongs in the application.** Keep content filters, Prompt Shields and evaluation regardless of customisation.
- **Lock-in.** Fine-tuned weights are tied to one base model version. Plan for retraining at retirement ([Topic 15 file 14](../15-foundry-models-and-azure-openai/14-model-lifecycle.md)).

## Common mistakes

| Mistake | Fix |
|---|---|
| Fine-tuning to teach facts | Use RAG |
| Fine-tuning before writing a decent prompt | Baseline with prompting and evaluation first |
| Ignoring hosting cost in the business case | Include hourly hosting and retraining in TCO |
| Measuring success on training loss | Evaluate on a held-out test set against the baseline |
| Treating RAG and fine-tuning as either/or | Combine: fine-tune behaviour, retrieve knowledge |
| Distilling without a strong teacher baseline | First prove the teacher is good enough on your evaluation set |

## Check yourself

1. What is the single best rule for choosing between RAG and fine-tuning?
2. List four benefits of fine-tuning given in the Microsoft overview.
3. What does Microsoft say fine-tuning does **not** replace?
4. When would you choose DPO over SFT?
5. When does RFT fit?
6. What is distillation, and when is it worth doing?
7. Which training types do not guarantee data residency?
8. What cost is often missing from fine-tuning business cases?
9. What is Frontier Tuning's availability status?

<details>
<summary>Answers</summary>

1. Missing or changing knowledge → RAG; inconsistent behaviour → prompting, then fine-tuning.
2. Any four of: better accuracy and relevance, reduced prompt overhead, consistent style and structure, better tool use, better use of retrieved context, specialising a smaller model.
3. Retrieval for current information, and application-level safety controls.
4. When quality depends on preferences (tone, completeness) and you have preferred and rejected responses to the same input, rather than one correct answer.
5. When many valid solutions exist but you can reliably score responses with a grader.
6. Training a smaller student model on outputs from a larger teacher; worth it when the teacher is good enough but too slow or expensive at your volume.
7. Global and Developer.
8. Hourly hosting of the fine-tuned deployment (plus retraining when the base model retires).
9. Announced at Build 2026; private preview through Forward Deployed Engineers; Foundry and Copilot Studio availability announced as upcoming.

</details>

## References

- [Fine-tuning in Microsoft Foundry (overview)](https://learn.microsoft.com/azure/foundry/fine-tuning/overview)
- [Supervised fine-tuning how-to](https://learn.microsoft.com/azure/foundry/openai/how-to/fine-tuning)
- [Reinforcement fine-tuning](https://learn.microsoft.com/azure/foundry/openai/how-to/reinforcement-fine-tuning)
- [Generate synthetic data for fine-tuning (Preview)](https://learn.microsoft.com/azure/foundry/fine-tuning/data-generation)
- [Frontier Tuning announcement](https://devblogs.microsoft.com/microsoft365dev/frontier-tuning-teaching-ai-to-work-the-way-you-do/)
- [Plan and manage costs for Microsoft Foundry](https://learn.microsoft.com/azure/foundry/concepts/manage-costs)
