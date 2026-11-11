# Healthcare

## Why it matters

Healthcare combines some of AI's highest-value opportunities (documentation burden, imaging backlogs, patient access, research) with the strictest constraints: protected health information (PHI), HIPAA and GDPR, medical device regulation, interoperability standards and patient safety. Azure provides healthcare-specific building blocks: **Azure Health Data Services** (FHIR, DICOM and de-identification), the **healthcare agent service** for compliant generative AI copilots, **healthcare AI models** in Foundry for medical imaging, and **healthcare data solutions in Microsoft Fabric**. Architects working with providers, payers or life sciences need to know these services, the interoperability standards they implement, and the lifecycle change that matters most right now: **Azure API for FHIR was deprecated on 30 September 2026**.

## Key concepts

### Azure Health Data Services

| Service | Purpose |
|---|---|
| **FHIR service** | Managed FHIR R4 server: store and exchange structured clinical data via a standard REST API; SMART on FHIR, `$export`, `$import`, search, conversion from HL7v2/C-CDA via `$convert-data` |
| **DICOM service** | DICOMweb-compatible medical imaging store; integrates with FHIR and Data Lake for analytics |
| **De-identification service** | Removes or surrogates PHI in clinical text while preserving clinical relevance (HIPAA-aligned) |
| **Workspace model** | Multiple FHIR and DICOM services in one workspace with shared configuration and networking |

**Azure API for FHIR** (the older single-service offering) was deprecated on 30 September 2026; migrate to the FHIR service in Azure Health Data Services using Microsoft's migration strategies.

### AI services for healthcare

| Service | Purpose | Notes |
|---|---|---|
| **Healthcare agent service** | Build compliant generative AI healthcare copilots/agents with a healthcare-adapted orchestrator, built-in clinical safeguards (provenance, clinical code validation), templates (triage, symptom checking), integration with customer data sources and Foundry models | Successor to Health Bot |
| **Healthcare AI models in Foundry** | Medical imaging foundation models: **MedImageInsight Premium** (embeddings for classification/search across radiology, pathology, ophthalmology, dermatology) and **CxrReportGen Premium** (draft chest X-ray findings), both **Preview** and assistive only; partner models (for example pathology) and open-source models | Not medical devices; human review required |
| **Text Analytics for health** | Extract medical entities and relations from clinical text ([Topic 20](../20-foundry-tools/README.md)) | Part of Azure Language |
| **Microsoft Dragon Copilot** | Clinical documentation and ambient AI (Microsoft Cloud for Healthcare product) | Licensed separately |
| **Healthcare data solutions in Fabric** | Ingest FHIR, DICOM, claims, SDOH into a healthcare data lakehouse with OMOP and analytics | Fabric |

### Regulatory considerations

| Area | Consideration |
|---|---|
| **HIPAA / BAA** | Microsoft offers a Business Associate Agreement for in-scope services; confirm each service is covered |
| **GDPR / national health data laws** | Residency, lawful basis, data subject rights |
| **Medical device rules** | AI used for diagnosis may be a medical device (FDA, EU MDR); healthcare AI models are for research/development and require validation |
| **Clinical safety** | Human oversight, provenance, audit trails, evaluation on representative populations |
| **Interoperability** | FHIR, DICOM, HL7v2, CDA, terminologies (SNOMED CT, LOINC, ICD) |

## How it works on Azure

Reference architecture for a clinical assistant:

```text
EHR (HL7v2/FHIR) → FHIR service ($convert-data for HL7v2) → de-identification for research copies
DICOM modalities → DICOM service → MedImageInsight embeddings for similar-case search (research)
Healthcare agent service (or Foundry agent) → tools: FHIR search (patient-scoped, SMART on FHIR auth), knowledge base of guidelines
Guardrails, evaluation, human review; audit in Purview and Application Insights; private endpoints throughout
Analytics: FHIR $export → Fabric healthcare data solutions → Power BI
```

FHIR search example (Entra token with FHIR Data Reader):

```bash
TOKEN=$(az account get-access-token --resource https://hdsws-contoso-fhir-weu.fhir.azurehealthcareapis.com --query accessToken -o tsv)
curl -H "Authorization: Bearer $TOKEN" \
  "https://hdsws-contoso-fhir-weu.fhir.azurehealthcareapis.com/Observation?patient=Patient/123&code=http://loinc.org|8867-4&_sort=-date&_count=5"
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **FHIR as the integration hub** | Use FHIR for structured exchange; avoid custom schemas where FHIR resources fit. |
| **Agent platform** | Healthcare agent service for patient/clinician-facing copilots with built-in healthcare safeguards; Foundry Agent Service for custom agents with your own controls. |
| **Imaging AI** | Use models as assistive research tools; formal validation and regulatory pathways for clinical use. |
| **Data for AI** | De-identify before broad analytics and model development; govern re-identification risk. |
| **Migration** | Move from Azure API for FHIR to the FHIR service now. |

## Hands-on

1. Deploy a Health Data Services workspace with FHIR and DICOM services; load sample Synthea data.
2. Convert an HL7v2 message to FHIR with `$convert-data`.
3. De-identify sample clinical notes with the de-identification service.
4. Build a patient-scoped FHIR tool for an agent with SMART on FHIR authorisation.
5. If applicable, plan an Azure API for FHIR migration.

## Security, cost and reliability notes

- **PHI protection**: private endpoints, CMK where required, Entra RBAC (FHIR Data Reader/Contributor), audit logs.
- **Safety**: clinical outputs need human review, provenance and evaluation across populations.
- **Cost**: storage and API usage for FHIR/DICOM; model endpoints; Fabric capacity.
- **Availability**: clinical systems need DR plans; know service SLAs and regional options.

## Common mistakes

| Mistake | Fix |
|---|---|
| Building new solutions on Azure API for FHIR | FHIR service in Health Data Services |
| Using imaging foundation models diagnostically without validation | Assistive use with regulatory pathway |
| Analytics on identified data by default | De-identification and governance |
| Custom data models instead of FHIR | Standards first |
| Assuming every service is under the BAA | Verify coverage |

## Check yourself

1. Which services make up Azure Health Data Services?
2. What happened to Azure API for FHIR?
3. What does the healthcare agent service provide?
4. Name the premium healthcare AI models and their status.
5. Which regulations and standards matter most?
6. Why de-identify data before analytics?

<details>
<summary>Answers</summary>

1. FHIR service, DICOM service and de-identification service in workspaces.
2. It was deprecated on 30 September 2026; migrate to the FHIR service.
3. A platform for compliant generative AI healthcare copilots with a healthcare-adapted orchestrator, safeguards and templates.
4. MedImageInsight Premium and CxrReportGen Premium, both Preview and assistive only.
5. HIPAA (BAA), GDPR and national health data laws, medical device regulation; FHIR, DICOM, HL7v2, CDA and clinical terminologies.
6. To reduce privacy risk and meet regulatory requirements while keeping data useful.

</details>

## References

- [What is Azure Health Data Services?](https://learn.microsoft.com/azure/healthcare-apis/healthcare-apis-overview)
- [FHIR service](https://learn.microsoft.com/azure/healthcare-apis/fhir/overview)
- [DICOM service](https://learn.microsoft.com/azure/healthcare-apis/dicom/overview)
- [Migration strategies from Azure API for FHIR](https://learn.microsoft.com/azure/healthcare-apis/fhir/migration-strategies)
- [Healthcare agent service](https://learn.microsoft.com/azure/health-bot/overview)
- [Healthcare AI models in Foundry](https://learn.microsoft.com/azure/foundry/how-to/healthcare-ai/healthcare-ai-models)
- [Healthcare data solutions in Fabric](https://learn.microsoft.com/industry/healthcare/healthcare-data-solutions/overview)
