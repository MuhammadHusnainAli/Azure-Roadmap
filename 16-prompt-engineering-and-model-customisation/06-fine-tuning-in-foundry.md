# Fine-Tuning in Foundry

## Why it matters

When prompting and retrieval have been exhausted (file 05), fine-tuning lets you change the model itself: teach a consistent output format, sharpen classification boundaries, improve tool-calling accuracy or make a small model good enough for a narrow task. Foundry runs the training on managed infrastructure, so you never provision GPUs, but you still own the data, the method choice, the evaluation and the bill. Engineers need the mechanics; architects need to know which models, training types and deployment options are allowed by residency and cost constraints.

## Key concepts

### What fine-tuning does

Fine-tuning adapts a pretrained model through additional training on task-specific data. It adjusts weights (often via a low-rank adapter, LoRA) rather than adding examples to each prompt. Benefits are not guaranteed; you must compare the result with your baseline on held-out data.

### Training approaches

| Approach | How it works | Your control | Status |
|---|---|---|---|
| **Managed Training** | Submit data and settings; Foundry runs the job | Data, supported hyperparameters, RFT graders | GA for supported models (some models Preview) |
| **Programmed Training** | Write a Python training loop; Foundry executes training and sampling operations | Losses, rewards, rollouts, gradient accumulation, checkpoints | Preview, requires access approval |

### Customisation methods (Managed Training)

| Method | Training signal | When it fits |
|---|---|---|
| **Supervised fine-tuning (SFT)** | Prompts and desired responses | You can demonstrate the behaviour |
| **Direct preference optimisation (DPO)** | Preferred and rejected responses to the same input | Quality depends on preferences (tone, completeness) rather than a single right answer |
| **Reinforcement fine-tuning (RFT)** | Prompts plus a **grader** that scores generated responses | Multiple valid solutions exist and you can score them reliably |
| **Vision fine-tuning** | SFT with image inputs | Visual understanding for your domain (gpt-4o 2024-08-06, gpt-4.1 2025-04-14) |
| **Tool-calling fine-tuning** | SFT examples that include tool definitions and calls | Better tool selection and arguments |

### Training types

Training types decide **where training runs** and are separate from deployment types.

| Training type | Where | Residency | Notes |
|---|---|---|---|
| **Standard** | The Foundry resource's region | Yes, regional | |
| **Data zone** | Within a data zone (US only today) | Within the zone | |
| **Global** | Capacity beyond your region; data and weights copied to the training region | **No** | Cheaper per token, faster queues |
| **Developer** | Idle capacity; jobs can be pre-empted and resumed | **No** | Big savings, no SLA; for experiments |

### Supported models (October 2026 snapshot)

Support changes often; always check the overview table before planning. Highlights:

| Model | Methods | Notes |
|---|---|---|
| gpt-4.1, gpt-4.1-mini, gpt-4.1-nano (2025-04-14) | SFT, DPO | All four training types; serverless deployment |
| gpt-4o (2024-08-06) | SFT, DPO | Also vision |
| gpt-4o-mini (2024-07-18) | SFT | |
| o4-mini (2025-04-16) | RFT | |
| gpt-5 (2025-08-07) | RFT | **By invitation** via your account team; Standard deployment only |
| gpt-oss-20b | SFT | Global Standard serverless |
| gpt-oss-120b, Qwen3.6-35B-A3B, Qwen3.8-27B, Muse-Glimmer-30B | SFT, RFT; Programmed Training | Preview; managed compute or Fireworks (provisioned) deployment |
| Llama-3.3-70B-Instruct, Qwen3-32B, Ministral-3B | SFT | Global Standard serverless |
| MAI-Code-1.1-Flash | RFT | Preview |

### Data format

SFT uses JSONL in the Chat Completions message format, one conversation per line, with separate `training.jsonl` and `validation.jsonl`:

```json
{"messages": [{"role": "system", "content": "You are a returns assistant."}, {"role": "user", "content": "Can I return a used helmet?"}, {"role": "assistant", "content": "No. Helmets can only be returned unused, within 30 days [Policy 4.2]."}]}
```

DPO uses `input`, `preferred_output` and `non_preferred_output`. RFT records end in a user message plus any fields the grader needs (for example `reference_answer`). Vision examples add `image_url` content parts (JPEG, PNG or WEBP; up to 64 images per example, 10 MB per image, at least 10 examples).

### RFT graders

A grader is the reward function. Types include **string check**, **text similarity**, **model grader** (`score_model`, an LLM judge), **Python grader** (a `grade(sample, item)` function) and **multi-grader** combinations. Microsoft warns that a rising reward doesn't prove better task performance if the model exploits the grader, so test graders against correct, incorrect and adversarial responses.

### Hyperparameters

| Setting | Effect |
|---|---|
| `n_epochs` (`epochs` in azd YAML) | Passes over the data; more can overfit |
| `batch_size` | Examples per update; larger means smoother, less frequent updates |
| `learning_rate_multiplier` | Scales learning rate; smaller can reduce overfitting |

Start with `auto` or the model's defaults and change one thing at a time.

## How it works on Azure

1. **Prepare data** (training, validation and a separate final test set).
2. **Create a job** in the portal (**Build → Fine-tune**), Python (`client.fine_tuning.jobs.create`), REST, or `azd ai finetuning` (extension `azure.ai.finetune`).
3. **Safety evaluation (automatic).** Before training, data is screened for harmful content; if it fails, the job stops and you are not charged. After training, the model is evaluated with simulated conversations; if it exceeds harm thresholds it is marked not deployable.
4. **Monitor** status, events and metrics: `train_loss`, `full_valid_loss`, `train_mean_token_accuracy`, `full_valid_mean_token_accuracy`.
5. **Pick a checkpoint** by validation metrics and held-out task results, not just the last one.
6. **Deploy** (file 08) and evaluate.

Roles: **Foundry User** to train, **Foundry Owner** (or `deployments/write`) to deploy. These were previously Azure AI User and Azure AI Owner.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Which base model?** | The smallest model that reaches your quality bar after tuning. Smaller tuned models often match larger untuned ones on narrow tasks. |
| **SFT vs DPO vs RFT** | SFT if you can write the answers; DPO if you can rank answers; RFT if you can score answers and the task needs reasoning. |
| **Training type** | Standard or Data zone when residency matters; Global for cost; Developer for experiments only. |
| **How much data?** | Quality beats quantity. Start with a few hundred high-quality, diverse examples; grow based on error analysis. |
| **Managed vs Programmed Training** | Managed unless you need custom losses or rollouts and have ML engineers to own the loop. |
| **Data source** | Real production conversations (with consent and redaction), expert-written examples, or teacher-generated data (file 07). |

## Hands-on

SFT on gpt-4.1-mini with the Foundry SDK and keyless auth.

```python
# pip install azure-ai-projects azure-identity openai
import os
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

project = AIProjectClient(
    endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"],
    credential=DefaultAzureCredential(),
)
client = project.get_openai_client()

train = client.files.create(file=open("training.jsonl", "rb"), purpose="fine-tune")
valid = client.files.create(file=open("validation.jsonl", "rb"), purpose="fine-tune")
client.files.wait_for_processing(train.id)
client.files.wait_for_processing(valid.id)

job = client.fine_tuning.jobs.create(
    model="gpt-4.1-mini-2025-04-14",
    training_file=train.id,
    validation_file=valid.id,
    suffix="returns-v1",
    method={"type": "supervised",
            "supervised": {"hyperparameters": {"n_epochs": "auto",
                                               "batch_size": "auto",
                                               "learning_rate_multiplier": "auto"}}},
    extra_body={"trainingType": "Developer"},   # cheap experiment; no residency guarantee
)
print(job.id, job.status)

# later
job = client.fine_tuning.jobs.retrieve(job.id)
for e in client.fine_tuning.jobs.list_events(job.id).data:
    print(e.created_at, e.message)
print(client.fine_tuning.jobs.checkpoints.list(job.id).model_dump_json(indent=2))
```

Then compare training and validation loss. If training loss falls while validation loss rises, you are overfitting: reduce epochs or improve data diversity.

Alternative with the Azure Developer CLI:

```bash
azd ext install azure.ai.finetune
azd ai finetuning init -e https://<account>.services.ai.azure.com/api/projects/<project>
azd ai finetuning jobs submit -f job.yaml
azd ai finetuning jobs show -i <job-id>
```

## Security, cost and reliability notes

- **Training data handling.** Redact personal data before upload; uploaded files persist until you delete them (`client.files.delete`).
- **Residency.** Global and Developer training copy data and weights outside your region.
- **Cost components.** Training (per token or per hour, depending on model and method), grader model calls for RFT, evaluation runs, and **hosting** once deployed. Cancelling a job doesn't delete deployments or stop their charges.
- **Safety thresholds** for fine-tuning evaluation are fixed; you can request changes through a form, not configure them.
- **Reproducibility.** Keep dataset versions, job configuration and checkpoint IDs in source control so you can retrain when the base model retires.

## Common mistakes

| Mistake | Fix |
|---|---|
| Validation data overlapping training data | Split before any processing; keep a separate final test set |
| Picking the last checkpoint by default | Compare checkpoints on held-out tasks |
| Inconsistent labels or styles in examples | Review data; one style guide for annotators |
| Training system prompts that differ from production | Use the same system prompt in training and inference |
| RFT grader that can be gamed | Test the grader on adversarial responses first |
| Using Developer training for regulated data | Use Standard or Data zone |

## Check yourself

1. What is the difference between Managed Training and Programmed Training?
2. Match SFT, DPO and RFT to their training signals.
3. Which training types give data residency guarantees?
4. What happens if your training data fails the safety check?
5. What does a rising RFT reward not prove, and why?
6. Which metric pattern indicates overfitting?
7. Which models support vision fine-tuning?
8. Which roles are needed to train and to deploy?

<details>
<summary>Answers</summary>

1. Managed Training runs a predefined job from your data and settings; Programmed Training (Preview) lets you write the training loop in Python and control losses, rewards and checkpoints.
2. SFT: prompts and desired responses. DPO: preferred and rejected responses to the same input. RFT: prompts and a grader that scores generated responses.
3. Standard (regional) and Data zone (within the zone).
4. The job fails with the harm categories listed, and you aren't charged.
5. Better task performance; the model may be exploiting weaknesses in the grader.
6. Training loss falling while validation loss rises.
7. gpt-4o (2024-08-06) and gpt-4.1 (2025-04-14).
8. Foundry User to train; Foundry Owner (or a role with deployments/write) to deploy.

</details>

## References

- [Fine-tuning in Microsoft Foundry (overview)](https://learn.microsoft.com/azure/foundry/fine-tuning/overview)
- [Supervised fine-tuning](https://learn.microsoft.com/azure/foundry/openai/how-to/fine-tuning)
- [Direct preference optimisation](https://learn.microsoft.com/azure/foundry/openai/how-to/fine-tuning-direct-preference-optimization)
- [Reinforcement fine-tuning](https://learn.microsoft.com/azure/foundry/openai/how-to/reinforcement-fine-tuning)
- [Vision fine-tuning](https://learn.microsoft.com/azure/foundry/openai/how-to/fine-tuning-vision)
- [Safety evaluation for fine-tuning](https://learn.microsoft.com/azure/foundry/openai/how-to/fine-tuning-safety-evaluation)
- [Foundry fine-tuning samples](https://github.com/microsoft-foundry/fine-tuning)
