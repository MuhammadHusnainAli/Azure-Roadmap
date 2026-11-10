# Deep Learning and Transformers

> Topic: 13 · AI and Generative AI Foundations · Tracks: E ● A ○ C ○ · Status checked: 2026-11

## Why it matters

Every modern language, vision, speech and embedding model you'll deploy on Foundry is a deep neural network, and almost all of them are **transformers**. You don't need to implement one. You do need to understand why context windows are limited and expensive, why long prompts slow responses, why models hallucinate, what "encoder" and "decoder" mean on a model card, and why bigger models are more capable but cost more. These ideas explain behaviour you'll otherwise find confusing in production.

## Key concepts

### Neural networks in one minute

| Concept | Meaning |
|---|---|
| **Neuron / unit** | Computes a weighted sum of inputs plus a bias, then applies a non-linear **activation** (ReLU, GELU) |
| **Layer** | Many units in parallel; networks stack layers ("deep" = many layers) |
| **Parameters (weights)** | The learned numbers. Modern LLMs have billions to trillions |
| **Loss function** | Measures error (for language models, cross-entropy on the next token) |
| **Backpropagation** | Computes how much each weight contributed to the error |
| **Gradient descent** (Adam, AdamW) | Updates weights a small step in the direction that reduces loss |
| **Epoch / batch** | One pass over the data / a group of examples processed together |
| **GPU / accelerator** | Hardware for the massively parallel matrix multiplications deep learning needs |

### Before transformers

| Architecture | Idea | Limitation |
|---|---|---|
| **CNNs** (convolutional) | Local patterns in grids (images) | Not natural for long-range dependencies in text |
| **RNNs / LSTMs** | Process sequences token by token with a hidden state | Sequential (slow to train) and they forget long-range context |

The 2017 paper *Attention Is All You Need* introduced the **transformer**, which replaced recurrence with **attention**. That made training highly parallel and long-range relationships easier to learn.

### Tokens and embeddings

1. Text is split into **tokens** (sub-word pieces) by a tokenizer.
2. Each token ID is mapped to a vector, its **embedding**.
3. **Positional information** is added (for example rotary position embeddings, RoPE) because attention by itself doesn't know word order.

### Self-attention

For each token, the model creates three vectors from its embedding: a **query (Q)**, a **key (K)** and a **value (V)**. Attention decides how much each token should "look at" every other token:

```text
Attention(Q, K, V) = softmax( Q·Kᵀ / √d_k ) · V
```

- `Q·Kᵀ` scores how relevant each token is to each other token.
- Dividing by `√d_k` (the key dimension) keeps the scores in a stable range.
- `softmax` turns the scores into weights that sum to 1.
- Multiplying by `V` produces a context-aware mix of information.

**Multi-head attention** runs several attention operations in parallel, so different heads can learn different relationships (syntax, coreference, topic). A **transformer block** is attention + a feed-forward network, with residual connections and layer normalisation. Models stack dozens to over a hundred of these blocks.

```text
"The bank raised interest rates because it ..."   → attention lets "it" attend strongly to "bank"
"She sat on the river bank and ..."               → the same word "bank" gets a different contextual vector
```

### Why context windows are costly

Standard self-attention compares every token with every other token, so compute and memory grow roughly with the **square of the sequence length** (O(n²)). During generation, models cache keys and values (the **KV cache**) for all previous tokens, and that memory grows with context length. This is why:

- long prompts increase latency (time to first token) and cost;
- very long context windows (now up to around a million tokens on some models) rely on engineering such as FlashAttention, grouped-query attention and sparse or sliding-window attention;
- **prompt caching** (reusing computation for a repeated prefix) saves money and time (topic 15).

### Encoder, decoder and encoder–decoder

| Architecture | How it attends | Good at | Examples |
|---|---|---|---|
| **Encoder-only** | Bidirectional: every token sees the whole input | Understanding: classification, extraction, **embeddings**, reranking | BERT family, many embedding and cross-encoder reranker models |
| **Decoder-only** | Causal: each token sees only previous tokens; generates one token at a time | **Generation**: chat, code, reasoning, tool calling | GPT family, Llama, Mistral, Phi, most modern LLMs |
| **Encoder–decoder** | Encoder reads input; decoder generates output while attending to it | Sequence-to-sequence: translation, summarisation, speech recognition | T5, BART, Whisper |

Most modern LLMs are decoder-only. Embedding models are often encoder-based or adapted decoders.

### Mixture of experts (MoE)

Some large models replace each feed-forward layer with many "expert" sub-networks and a **router** that activates only a few experts per token. Total parameters can be huge while **active parameters per token** stay much smaller, giving more capability per unit of inference compute. Many frontier and open-weight models (for example gpt-oss, DeepSeek, several Mistral and Llama 4 models) use MoE.

### How LLMs are built

| Stage | What happens |
|---|---|
| **Pre-training** | Self-supervised next-token prediction on trillions of tokens of text and code: general knowledge and language ability |
| **Supervised fine-tuning (SFT) / instruction tuning** | Training on curated prompt → response examples to follow instructions |
| **Preference alignment** | RLHF, DPO and similar methods to prefer helpful, harmless, honest responses |
| **Reinforcement learning for reasoning** | Training on verifiable tasks (maths, code) so models learn to "think" longer before answering (reasoning models, file 03) |
| **Safety training and evaluation** | Red teaming, refusal training, safety classifiers |

### Why scale matters

Research on **scaling laws** showed that loss falls predictably as you increase **parameters, training data and compute** together. Larger, better-trained models show broader knowledge and new capabilities (in-context learning, multi-step reasoning, tool use). But scale has costs:

| Benefit of scale | Cost of scale |
|---|---|
| Better reasoning, knowledge and instruction following | Higher latency and price per token |
| Better few-shot learning and generalisation | More GPU capacity needed; regional quota limits |
| More robust multilingual and multimodal performance | Harder to self-host; larger memory footprint |

Recent progress also comes from **better data**, **distillation** (training small models from large ones) and **test-time compute** (letting models reason longer), not just bigger parameter counts. That's why small models such as Phi can be competitive on specific tasks (file 06).

## How it works on Azure

You consume transformers on Azure in three ways:

| Approach | Service | You manage |
|---|---|---|
| Model as a service | **Foundry Models** (Azure OpenAI and other models sold directly by Azure; partner models) | Prompts, deployment type, quotas |
| Model on dedicated compute | Foundry / Azure ML **managed compute** (open models on GPU VMs) | Endpoint scaling, GPU quota |
| Your own training or fine-tuning | Foundry fine-tuning, Azure ML, AKS/VMs with GPUs (ND-series) | Data, training code, infrastructure |

### See attention and tokens in practice (Python)

```python
# pip install transformers torch
from transformers import AutoTokenizer, AutoModel

name = "bert-base-uncased"                       # small encoder-only model, runs on CPU
tok = AutoTokenizer.from_pretrained(name)
model = AutoModel.from_pretrained(name, output_attentions=True)

inputs = tok("The bank raised interest rates because it feared inflation.", return_tensors="pt")
out = model(**inputs)
print(tok.convert_ids_to_tokens(inputs["input_ids"][0]))
print(len(out.attentions), "layers;", out.attentions[0].shape)   # (batch, heads, seq, seq)
```

The last line shows the attention matrix is **sequence × sequence** per head and layer, which is the quadratic cost in action.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Large vs small model | Use the smallest model that meets your evaluation bar; route hard requests to larger models (model router, topic 15) |
| Long context vs retrieval | Long contexts are convenient but costly and can dilute attention ("lost in the middle"). Retrieval (RAG) sends only relevant content |
| Encoder models for classification and embeddings | For high-volume classification or similarity, a small encoder or embedding model is far cheaper than a generative LLM |
| MoE models | Strong capability per inference cost, but large memory footprint if self-hosted |

## Hands-on

1. Run the BERT snippet and print the attention weights of one head for the token "it". Which tokens does it attend to most?
2. Tokenise the same sentence with a GPT tokenizer (`tiktoken`) and compare token counts and splits.
3. In the Foundry model catalogue, open three model cards and note architecture (decoder-only, MoE), parameter count where stated, context window and modalities.
4. Measure time to first token for a short prompt vs a prompt padded to 50,000 tokens on the same deployment.

## Security, cost and reliability notes

- **Security:** models can't distinguish instructions from data inside the context window. That is the root of **prompt injection** (topic 23).
- **Cost:** cost scales with tokens processed, and long prompts hit both latency and price. Design prompts and retrieval to keep context tight.
- **Reliability:** generation is probabilistic; the same input can produce different outputs. Design for variability with evaluation, validation and retries.

## Common mistakes

- Assuming a bigger context window means you should fill it.
- Using a generative LLM for a task an encoder classifier or embedding model does better and cheaper.
- Believing the model "knows" facts reliably. It learned statistical patterns, and knowledge is frozen at the training cut-off.
- Confusing parameters (learned weights) with hyperparameters (training settings) or with inference parameters (temperature).

## Check yourself

1. What problem did transformers solve compared with RNNs?
2. Explain query, key and value in self-attention.
3. Why does attention cost grow quadratically with sequence length?
4. Which architecture is best for embeddings, and which for chat?
5. What is a mixture-of-experts model?
6. Name the main stages of building a modern LLM.
7. What did scaling laws show?

<details>
<summary>Answers</summary>

1. They process all tokens in parallel (faster training) and model long-range dependencies directly through attention instead of passing information step by step.
2. Each token's query is compared with every token's key to produce relevance scores; the softmax-normalised scores weight the tokens' values, producing a context-aware representation.
3. Every token attends to every other token, so the score matrix is n × n for n tokens.
4. Encoder-only (bidirectional) architectures for embeddings and classification; decoder-only (causal) architectures for chat and generation.
5. A model where a router activates only a few expert sub-networks per token, so total parameters are large while active compute per token is smaller.
6. Pre-training (next-token prediction), supervised fine-tuning/instruction tuning, preference alignment (RLHF/DPO), reinforcement learning for reasoning, plus safety training and evaluation.
7. Model quality improves predictably as parameters, data and compute scale together, and larger models show broader capabilities.

</details>

## References

- [Attention Is All You Need (Vaswani et al., 2017)](https://arxiv.org/abs/1706.03762)
- [Introduction to large language models (Microsoft Learn training)](https://learn.microsoft.com/training/modules/introduction-large-language-models/)
- [Fundamentals of generative AI (Microsoft Learn training)](https://learn.microsoft.com/training/modules/fundamentals-generative-ai/)
- [Scaling Laws for Neural Language Models (Kaplan et al., 2020)](https://arxiv.org/abs/2001.08361)
- [Foundry Models overview](https://learn.microsoft.com/azure/foundry/concepts/foundry-models-overview)
- [Hugging Face Transformers documentation](https://huggingface.co/docs/transformers/index)
