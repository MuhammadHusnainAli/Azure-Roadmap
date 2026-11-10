# 13 · AI and Generative AI Foundations

> Tracks: E ● A ● C ● · Part D · AI on Azure

This topic builds the mental model that the rest of Part D depends on. It covers how machine learning and deep learning work, why transformers changed everything, what tokens, context windows and sampling parameters mean for cost and quality, how reasoning models differ, how embeddings power search, what multimodal and small models can do, which solution pattern fits a problem, how Microsoft's AI products fit together, and how to qualify a use case before anyone writes code. It's conceptual first, with short Python examples against Microsoft Foundry using keyless authentication.

## What you will be able to do

- Explain training vs inference, supervised vs unsupervised learning, and choose the right evaluation metric.
- Explain attention, encoder vs decoder architectures, mixture of experts and why scale matters.
- Estimate token counts and costs, set sampling parameters, and use reasoning models correctly (`reasoning_effort`, unsupported parameters, reasoning tokens).
- Choose an embedding model, dimensions, distance metric and vector index algorithm.
- Pick the right Azure option for vision, image generation, speech and video.
- Decide when a small or open-weight model (Phi, gpt-oss, Llama) beats a frontier model, and run one on device with Foundry Local.
- Map a business request to a solution pattern (classification, extraction, RAG, copilot, agent, multi-agent).
- Place Microsoft 365 Copilot, Copilot Studio, Foundry, Fabric and Azure ML correctly and explain how they connect.
- Run an AI use case qualification: value, feasibility, data readiness, risk and adoption.

## Prerequisites

[Topic 01](../01-cloud-and-azure-fundamentals/README.md) and [Topic 02](../02-identity-and-access/README.md). Basic Python helps for the hands-on parts; no maths beyond school algebra is required.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Machine Learning Essentials](01-machine-learning-essentials.md) | 50 min | Python (scikit-learn) |
| 02 | [Deep Learning and Transformers](02-deep-learning-and-transformers.md) | 50 min | Python (transformers) |
| 03 | [Large Language Models](03-large-language-models.md) | 60 min | Python (OpenAI SDK, tiktoken) |
| 04 | [Embeddings and Vector Similarity](04-embeddings-and-vector-similarity.md) | 50 min | Python |
| 05 | [Multimodal Models](05-multimodal-models.md) | 45 min | Python |
| 06 | [Small Language Models and Open Models](06-small-language-models-and-open-models.md) | 40 min | Foundry Local |
| 07 | [AI Solution Patterns](07-ai-solution-patterns.md) | 45 min | Exercise, Foundry portal |
| 08 | [The Microsoft AI Stack](08-the-microsoft-ai-stack.md) | 45 min | Exercise |
| 09 | [AI Use Case Qualification](09-ai-use-case-qualification.md) | 45 min | Workshop exercise |

**AI engineers:** all of it, with 03, 04 and 07 in depth. **Architects:** all of it, with 03, 07 and 08 in depth. **Consultants:** 01 (metrics), 03 (limitations), 07, 08 and 09 in depth; skim 02 and 04.

## Completion checklist

- [ ] I can explain precision, recall and F1 and pick a metric for a business problem.
- [ ] I can explain self-attention and why long contexts cost more.
- [ ] I can estimate tokens and cost for a prompt and response, including reasoning tokens.
- [ ] I know which parameters reasoning models don't accept and how to set `reasoning_effort`.
- [ ] I have embedded text, computed cosine similarity and tested reduced dimensions.
- [ ] I can compare HNSW, IVF and DiskANN.
- [ ] I have sent an image to a vision-capable model and chosen the detail level deliberately.
- [ ] I have run a small model locally with Foundry Local.
- [ ] I can map five business requests to solution patterns with human-in-the-loop levels.
- [ ] I can explain buy vs extend vs build across Microsoft 365 Copilot, Copilot Studio and Foundry.
- [ ] I have completed a use case canvas and weighted score for a real candidate.

## Maps to

- **AI-901** (Azure AI Fundamentals): AI workloads, machine learning principles, generative AI concepts, responsible AI.
- **AI-103**: foundation for planning and building generative AI and agentic solutions.
- **AB-100**: use case qualification and positioning of Microsoft AI products.
- **AZ-305**: working knowledge for AI solution design.

## Next

[14 · Microsoft Foundry Platform](../14-microsoft-foundry-platform/README.md)
