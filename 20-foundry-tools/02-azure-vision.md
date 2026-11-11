# Azure Vision

## Why it matters

Images and video carry information that text pipelines miss: product photos, scanned forms, shelf images, ID documents, site inspection pictures. **Azure Vision in Foundry Tools** provided the classic computer vision APIs (image analysis, OCR, face, spatial analysis, custom models). The landscape has shifted sharply: **Image Analysis (v3.2 and v4.0) and Custom Vision are deprecated and retire on 25 September 2028**, spatial analysis has been retired, and Microsoft points customers to **Document Intelligence** for OCR, the **Face API** for faces, **multimodal embedding models** in Foundry for image search, and **multimodal LLMs or Content Understanding** for everything else. Architects must design new solutions on the successors and plan migrations for existing ones.

## Key concepts

### Current status (late 2026)

| Capability | Status | Successor or guidance |
|---|---|---|
| **Image Analysis 4.0** (captions, dense captions, tags, objects, people, smart crop, Read) | Deprecated; **retires 25 Sep 2028** (cloud and containers) | See migration table below |
| **Image Analysis 3.2** (brands, landmarks, celebrities, adult content, colour, image type) | Deprecated; same retirement | Same |
| **Image Analysis 4.0 model customisation** | Deprecated | Azure ML AutoML, Content Understanding classifiers, multimodal LLMs |
| **OCR / Read API** (legacy) | Legacy; Read container retires 28 Sep 2028 | **Document Intelligence Read** model |
| **Custom Vision** | Retirement announced; full support until **25 Sep 2028** | **Azure ML AutoML** (classification, detection), Foundry multimodal models, **Content Understanding** custom classification |
| **Spatial analysis** | Retired | Content Understanding video analysis; custom models |
| **Face** | Active; **Limited Access** for identification and verification | Face API |
| **Multimodal embeddings** (Vision vectorise) | Used in AI Search skills/vectorisers (Preview) | Also Cohere Embed v4 and other catalogue models |

### Migration paths (Microsoft guidance)

| Image Analysis scenario | Recommended alternative |
|---|---|
| OCR and Read | **Azure Document Intelligence** (Read model) |
| Face detection and attributes | **Face API** |
| Image embeddings and image-text retrieval | **Cohere Embed v4** in Foundry; open models such as SigLIP |
| Captioning, tagging, object description, other scenarios | **GPT-family multimodal models** in Foundry, or **Azure Content Understanding** (managed generative image analysis and classifiers) |

### Face service

| Capability | Access |
|---|---|
| Face detection (bounding boxes, landmarks, some attributes) | Generally available |
| **Liveness detection** (anti-spoofing) | Available per service terms |
| **Verification** (1:1) and **identification** (1:N), face IDs | **Limited Access**: Microsoft managed customers and partners, approved use cases via intake form |

Responsible AI constraints: biometric data obligations (notice, consent, retention, destruction) are the customer's; Microsoft doesn't allow use by US police departments for facial recognition; some attributes (emotion, gender, age inference) were retired earlier under the Responsible AI Standard.

### Multimodal LLMs as the new default

For open-ended visual understanding ("what's wrong in this inspection photo?", "extract every product and price from this shelf image as JSON"), Foundry's **vision-capable models** (GPT-family, Claude, others) with **structured outputs** are now the main tool ([Topic 15 file 10](../15-foundry-models-and-azure-openai/10-vision-image-and-video-generation.md)). They're flexible but non-deterministic and priced per token; evaluate accuracy and cost against task-specific services.

## How it works on Azure

| Task | Build with |
|---|---|
| Read text in images and documents | Document Intelligence Read/Layout ([file 06](06-azure-document-intelligence.md)) |
| Face detection, liveness, verification | Face API (Limited Access for recognition) |
| Image description, tagging, visual Q&A | Multimodal model (Responses API with image input and structured outputs) |
| Managed image classification and extraction | Content Understanding analysers and classifiers ([file 07](07-azure-content-understanding.md)) |
| Custom object detection/classification | Azure ML AutoML for images ([Topic 21](../Roadmap.md)); or fine-tuned vision models |
| Image search (text-to-image, image-to-image) | Multimodal embeddings + Azure AI Search ([Topic 17 file 11](../17-rag-and-knowledge/11-multimodal-and-structured-data-rag.md)) |
| Video | Video Indexer ([file 08](08-azure-ai-video-indexer.md)) or Content Understanding video |

Image limits for the classic APIs: JPEG, PNG, GIF or BMP; under 4 MB; larger than 50 × 50 pixels (Read: up to 10,000 × 10,000).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **New projects** | Don't start on Image Analysis or Custom Vision; use the successors. |
| **OCR** | Document Intelligence for accuracy, layout, tables and handwriting; LLMs only for light, ad hoc OCR. |
| **Multimodal LLM vs Content Understanding** | LLM for flexible reasoning in apps and agents; Content Understanding for managed, schema-driven extraction with confidence and grounding at scale. |
| **Custom detection** | Azure ML AutoML when you need bounding boxes and controlled training; LLMs struggle with precise localisation. |
| **Face** | Only with an approved Limited Access use case, consent and retention policy; consider whether face is needed at all. |
| **Migration timing** | Microsoft asked customers to plan transitions by 25 Sep 2026 and complete by 25 Sep 2028; start with inventory and evaluation datasets. |

## Hands-on

Migrate an image tagging workflow from Image Analysis to a multimodal model with structured output.

```python
import base64
from openai import OpenAI
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

client = OpenAI(base_url="https://<foundry>.openai.azure.com/openai/v1/",
                api_key=get_bearer_token_provider(DefaultAzureCredential(), "https://ai.azure.com/.default"))

img = base64.b64encode(open("shelf.jpg", "rb").read()).decode()
schema = {"type": "json_schema", "name": "shelf", "strict": True, "schema": {
    "type": "object", "additionalProperties": False, "required": ["caption", "products"],
    "properties": {
        "caption": {"type": "string"},
        "products": {"type": "array", "items": {"type": "object", "additionalProperties": False,
            "required": ["name", "price", "out_of_stock"],
            "properties": {"name": {"type": "string"}, "price": {"type": ["number", "null"]},
                           "out_of_stock": {"type": "boolean"}}}}}}}

r = client.responses.create(model="gpt-5-mini", text={"format": schema}, input=[{
    "role": "user", "content": [
        {"type": "input_text", "text": "Describe the shelf and list every product with its price."},
        {"type": "input_image", "image_url": f"data:image/jpeg;base64,{img}"}]}])
print(r.output_text)
```

Then:

1. Run the same 30 images through your old Image Analysis call (if available) and the new approach; compare tag quality, latency and cost.
2. Extract printed text from the same images with Document Intelligence Read and compare OCR accuracy with the LLM.
3. Write a migration plan for one retiring capability (inventory, alternative, evaluation, cut-over date).

## Security, cost and reliability notes

- **Biometric data**: face templates and images are sensitive personal data; minimise, consent, secure and delete per policy.
- **Images can contain hidden instructions** (text in images) that steer multimodal LLMs; treat model outputs as untrusted.
- **Cost**: classic APIs bill per image; multimodal LLMs bill per token (images consume many tokens; use lower detail where possible).
- **Accuracy drift**: generative outputs vary; validate with evaluation sets and schema checks.
- **Retirement risk**: track retirement dates in your architecture register; containers retire too.

## Common mistakes

| Mistake | Fix |
|---|---|
| Starting new solutions on Image Analysis or Custom Vision | Use Document Intelligence, Face API, multimodal models, Content Understanding or AutoML |
| Using LLMs for high-volume OCR | Document Intelligence |
| Expecting LLMs to return precise bounding boxes | Use detection models (AutoML) |
| Using face identification without Limited Access approval | Apply first; consider alternatives |
| Sending full-resolution images when low detail is enough | Reduce detail and size |

## Check yourself

1. Which Vision capabilities are deprecated, and when do they retire?
2. What does Microsoft recommend instead of Image Analysis for OCR, faces, embeddings and other scenarios?
3. What are the migration paths from Custom Vision?
4. Which Face capabilities are Limited Access?
5. When would you choose a multimodal LLM over Content Understanding?
6. Why is Azure ML AutoML still relevant for vision?
7. What are the image requirements for the classic Vision APIs?
8. Name two security concerns specific to vision workloads.

<details>
<summary>Answers</summary>

1. Image Analysis v3.2 and v4.0 (including customisation) and Custom Vision retire on 25 September 2028; the Read container retires 28 September 2028; spatial analysis is already retired.
2. Document Intelligence (OCR), Face API (faces), Cohere Embed v4 or SigLIP (embeddings), GPT-family multimodal models or Content Understanding (other scenarios).
3. Azure ML AutoML for classification and detection, Foundry multimodal models, and Content Understanding custom classification.
4. Identification, verification and face ID features (managed customers and approved use cases).
5. For flexible, conversational or agentic reasoning about images inside an app; Content Understanding for managed, schema-driven extraction at scale.
6. It trains precise custom classification and object detection models with bounding boxes, which LLMs don't do reliably.
7. JPEG, PNG, GIF or BMP; under 4 MB; larger than 50 × 50 pixels (Read up to 10,000 × 10,000).
8. Biometric data obligations for faces; prompt injection via text embedded in images (plus data residency of image content).

</details>

## References

- [What is Azure Vision in Foundry Tools?](https://learn.microsoft.com/azure/ai-services/computer-vision/overview)
- [What is Image Analysis? (deprecation notice)](https://learn.microsoft.com/azure/ai-services/computer-vision/overview-image-analysis)
- [Migrate from Image Analysis](https://learn.microsoft.com/azure/ai-services/computer-vision/migration-options)
- [What is Custom Vision? (retirement)](https://learn.microsoft.com/azure/ai-services/custom-vision-service/overview)
- [What is the Azure Face service?](https://learn.microsoft.com/azure/ai-services/face/overview-identity)
- [Limited Access features](https://learn.microsoft.com/azure/ai-services/cognitive-services-limited-access)
