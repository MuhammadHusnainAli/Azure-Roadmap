# Built-In Evaluators

## Why it matters

Writing your own judge prompts for every quality dimension is slow and hard to calibrate. Microsoft Foundry ships a catalogue of **built-in evaluators** for general quality, textual similarity, RAG, risk and safety, and agent behaviour (task adherence, tool use, intent resolution), several of them drawn from Microsoft's experience with GitHub Copilot and Bing. Knowing what each evaluator measures, what inputs it needs and how it scores lets you assemble a credible evaluation suite in hours. The AI-103 exam expects you to choose the right evaluators for a scenario.

## Key concepts

### Families

| Family | Evaluators | Judge |
|---|---|---|
| **General purpose** | Coherence, Fluency | LLM |
| **Textual similarity** | Similarity (AI-assisted), F1, BLEU, GLEU, ROUGE, METEOR | LLM or deterministic (need ground truth) |
| **RAG** | Retrieval, Document Retrieval, Groundedness, Groundedness Pro (Preview), Relevance, Response Completeness (Preview) | LLM, deterministic, or Content Safety service |
| **Risk and safety** | Hate and Unfairness, Sexual, Violence, Self-Harm, Protected Materials, Indirect Attack (XPIA), Code Vulnerability, Ungrounded Attributes, Prohibited Actions, Sensitive Data Leakage | Hosted Foundry evaluation service models |
| **Agent** | Task Adherence, Task Completion, Customer Satisfaction, Intent Resolution, Task Navigation Efficiency, Tool Call Accuracy, Tool Selection, Tool Input Accuracy, Tool Output Utilization, Tool Call Success | LLM (several Preview) |
| **Batched** (Preview) | Quality Grader, Output Quality, Tool Use Quality | One LLM call covering several dimensions |
| **Rubric** (Preview) | Rubric | LLM with weighted custom criteria (file 04) |
| **Azure OpenAI graders** | Model Labeler, String Checker, Text Similarity, Model Scorer | LLM or string logic (file 04) |

### Agent evaluators in detail

| Evaluator | Question |
|---|---|
| **Task Adherence** | Did the agent follow its instructions and constraints for the identified task? |
| **Task Completion** | Did it complete the requested task end to end? |
| **Intent Resolution** | Did it correctly identify and address what the user wanted? |
| **Customer Satisfaction** | Holistic satisfaction across helpfulness, completeness, clarity, tone, resolution, adaptability |
| **Task Navigation Efficiency** | Did the sequence of steps match an optimal or expected path? |
| **Tool Selection** | Were the most appropriate tools chosen? |
| **Tool Input Accuracy** | Were parameters correct (grounded, typed, formatted, complete)? |
| **Tool Output Utilization** | Were tool results interpreted and used correctly? |
| **Tool Call Success** | Did tool calls execute without technical failure? |
| **Tool Call Accuracy** | Overall quality of tool calls |

Agent evaluators act like **unit tests for agentic systems**: they take agent messages (including tool calls and results) and return **pass/fail**, or a scaled score converted to pass/fail with a threshold. Tool definitions can be supplied to improve judgement.

### Safety evaluators in detail

| Evaluator | Detects |
|---|---|
| Hate and Unfairness, Sexual, Violence, Self-Harm | Harmful content with severity |
| Protected Materials | Copyrighted or protected content in outputs |
| **Indirect Attack (XPIA)** | Whether the response followed instructions injected via retrieved context |
| **Code Vulnerability** | Security issues in generated code |
| **Ungrounded Attributes** | Fabricated inferences about people (for example emotional state, demographics) |
| **Prohibited Actions** | Agent behaviours that violate explicitly disallowed actions |
| **Sensitive Data Leakage** | Agent exposing sensitive information |

Safety evaluators use hosted models in the Foundry evaluation service (no judge deployment of your own) and are available in supported regions only.

### Inputs and scoring

| Pattern | Inputs |
|---|---|
| Quality on single turn | `query`, `response` (+ `context` for groundedness, `ground_truth` for similarity/completeness) |
| Retrieval | `query`, `context` or `retrieval_ground_truth` and `retrieved_documents` |
| Agent | Messages (or agent output items), optional `tool_definitions` |
| Safety | `query`, `response` (and conversation for agents) |

Most AI-assisted quality evaluators produce a **1–5** score with an explanation and a pass/fail against a configurable **threshold**; safety evaluators report severity or defect rates.

Evaluator names in the SDK follow the pattern `builtin.<name>`, for example `builtin.task_adherence`, `builtin.violence`, `builtin.indirect_attack`, `builtin.tool_use_quality`.

## How it works on Azure

Testing criteria for a cloud evaluation of an agent (SDK, OpenAI-compatible evals API):

```python
judge = "gpt-5-mini"
testing_criteria = [
    {"type": "azure_ai_evaluator", "name": "task_adherence", "evaluator_name": "builtin.task_adherence",
     "initialization_parameters": {"deployment_name": judge},
     "data_mapping": {"query": "{{item.query}}", "response": "{{sample.output_items}}"}},
    {"type": "azure_ai_evaluator", "name": "tool_use", "evaluator_name": "builtin.tool_use_quality",
     "initialization_parameters": {"deployment_name": judge},
     "data_mapping": {"query": "{{item.query}}", "response": "{{sample.output_items}}"}},
    {"type": "azure_ai_evaluator", "name": "violence", "evaluator_name": "builtin.violence",
     "data_mapping": {"query": "{{item.query}}", "response": "{{sample.output_text}}"}},
    {"type": "azure_ai_evaluator", "name": "xpia", "evaluator_name": "builtin.indirect_attack",
     "data_mapping": {"query": "{{item.query}}", "response": "{{sample.output_text}}"}},
]
```

`{{item.*}}` maps dataset fields; `{{sample.*}}` maps responses generated by the target agent during the run. See [Topic 17 file 12](../17-rag-and-knowledge/12-rag-evaluation-and-tuning.md) for a complete create-and-run example. Data mapping keys differ per evaluator; check each evaluator's reference page.

## Design decisions and trade-offs

### Suggested suites

| Application | Core evaluators |
|---|---|
| **RAG Q&A** | Retrieval (or Document Retrieval with labels), Groundedness, Relevance, Response Completeness, Indirect Attack, harm evaluators |
| **Task agent with tools** | Task Completion, Task Adherence, Intent Resolution, Tool Use Quality (or the individual tool evaluators), Prohibited Actions, Sensitive Data Leakage |
| **Summarisation** | Groundedness, Coherence, Fluency, ROUGE/Similarity against references |
| **Translation** | BLEU/METEOR/GLEU against references, Fluency |
| **Code assistant** | Code Vulnerability, task-specific tests (deterministic), Protected Materials |
| **Customer-facing chat** | Customer Satisfaction, Relevance, harm evaluators, Ungrounded Attributes |

### Other choices

| Decision | Guidance |
|---|---|
| **Individual vs batched evaluators** | Batched (Output Quality, Tool Use Quality, Quality Grader) cut cost and latency; individual evaluators give clearer diagnostics. |
| **Preview evaluators** | Fine for development gates; validate stability before using them as hard production gates. |
| **Thresholds** | Calibrate against human judgements on a sample (file 04). |
| **Reference-free vs reference-based** | Reference-free (groundedness, relevance, task adherence) for production; reference-based (similarity, completeness) for curated test sets. |

## Hands-on

1. Build a 30-row dataset for a tool-using agent (queries plus expected outcomes).
2. Run an evaluation targeting the agent with Task Completion, Intent Resolution, Tool Use Quality and two safety evaluators.
3. Inspect failures: which tool evaluator failed (selection, input accuracy, output utilisation)?
4. Replace Tool Use Quality with the five individual tool evaluators and compare diagnostic value and cost.
5. Add an indirect-attack case (a document with injected instructions) and confirm the XPIA evaluator flags a vulnerable response.

## Security, cost and reliability notes

- **Safety evaluators process content** through the Foundry evaluation service; check region support and data handling for regulated workloads.
- **Judge costs**: AI-assisted evaluators call your judge deployment per row; batched evaluators reduce calls.
- **Variance**: LLM-based scores fluctuate slightly; use thresholds with margins and multiple samples.
- **Preview evaluators** can change behaviour; pin SDK versions and re-baseline after updates.

## Common mistakes

| Mistake | Fix |
|---|---|
| Using groundedness without passing context | Map context (or tool outputs for agents) |
| Only quality evaluators, no safety | Always include safety and XPIA for user-facing apps |
| Using BLEU for open-ended chat | Use reference-free evaluators |
| Treating a 3.5 average as meaningful without a baseline | Compare against previous versions |
| Ignoring tool evaluators for agents | Tool failures are a main agent failure mode |

## Check yourself

1. Name the evaluator families in Foundry.
2. Which evaluators assess tool use, and what does each check?
3. What does the Indirect Attack evaluator measure?
4. What's the difference between Groundedness and Groundedness Pro?
5. Which evaluators need ground truth?
6. Why use batched evaluators, and what do you lose?

<details>
<summary>Answers</summary>

1. General purpose, textual similarity, RAG, risk and safety, agent, batched, rubric, Azure OpenAI graders (and custom).
2. Tool Selection (right tools), Tool Input Accuracy (correct parameters), Tool Output Utilization (correct use of results), Tool Call Success (no technical failures), Tool Call Accuracy (overall).
3. Whether a response followed instructions injected through retrieved context (cross-prompt injection).
4. Groundedness uses an LLM judge with a 1–5 score; Groundedness Pro uses the Azure AI Content Safety service for a binary result without a model deployment.
5. Similarity, F1, BLEU, GLEU, ROUGE, METEOR, Response Completeness and Document Retrieval.
6. Lower cost and latency with one judge call; less granular diagnostics per dimension.

</details>

## References

- [Built-in evaluators reference](https://learn.microsoft.com/azure/foundry/concepts/built-in-evaluators)
- [Agent evaluators](https://learn.microsoft.com/azure/foundry/concepts/evaluation-evaluators/agent-evaluators)
- [Risk and safety evaluators](https://learn.microsoft.com/azure/foundry/concepts/evaluation-evaluators/risk-safety-evaluators)
- [RAG evaluators](https://learn.microsoft.com/azure/foundry/concepts/evaluation-evaluators/rag-evaluators)
- [General purpose evaluators](https://learn.microsoft.com/azure/foundry/concepts/evaluation-evaluators/general-purpose-evaluators)
