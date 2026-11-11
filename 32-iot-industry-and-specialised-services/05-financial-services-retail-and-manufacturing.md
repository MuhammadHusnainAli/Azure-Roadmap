# Financial Services, Retail and Manufacturing

## Why it matters

Industry context changes architecture. A bank's AI assistant must satisfy model risk management, operational resilience regulation and auditability; a retailer's personalisation engine must handle seasonal peaks, POS integration and PCI DSS; a manufacturer's predictive maintenance solution must bridge OT networks and run at the edge. Generic cloud designs miss these constraints. Microsoft publishes industry guidance, reference architectures, data models and compliance offerings (Microsoft Cloud for Financial Services, Retail and Manufacturing guidance, Architecture Center industry pages, Fabric industry data solutions), and consultants are expected to speak the industry's language and map regulations to controls. This file summarises the patterns and concerns for three major industries.

## Key concepts

### Financial services

| Theme | Considerations |
|---|---|
| **Use cases** | Fraud detection, AML and KYC, credit decisioning, customer service agents, advisor copilots, document processing (loans, claims), risk analytics, regulatory reporting |
| **Regulation** | Operational resilience (EU DORA, UK PRA/FCA rules), outsourcing and cloud guidelines (EBA), model risk management (SR 11-7, PRA SS1/23), PCI DSS, GDPR, EU AI Act (credit scoring is high-risk), local banking secrecy |
| **Architecture** | Multi-region resilience, exit plans, strict network isolation, CMK and HSMs, immutable audit, explainability for decisions, human oversight |
| **Data** | Customer 360, transactions, market data; lineage and retention |
| **Microsoft resources** | Microsoft Cloud for Financial Services guidance and data models, Architecture Center finance scenarios, Compliance Manager templates, Purview |

### Retail and consumer goods

| Theme | Considerations |
|---|---|
| **Use cases** | Personalisation and recommendations, demand forecasting, inventory optimisation, store analytics and computer vision, shopping assistants, product content generation, supply chain visibility |
| **Regulation** | PCI DSS for payments, consumer privacy (GDPR/CCPA), accessibility, marketing consent, EU AI Act transparency for chatbots/generated content |
| **Architecture** | Elastic scale for peaks (Black Friday), CDN and global routing, edge in stores (Azure Local, IoT), offline POS resilience, real-time data |
| **Data** | Product catalogue, customer, transactions, inventory, clickstream; Fabric retail data solutions and industry data models |
| **Microsoft resources** | Microsoft Cloud for Retail guidance, Dynamics 365 Commerce and Customer Insights, Architecture Center retail scenarios |

### Manufacturing

| Theme | Considerations |
|---|---|
| **Use cases** | Predictive maintenance, quality inspection with vision, OEE dashboards, production scheduling, digital twins, field service copilots, engineering knowledge assistants, supply chain risk |
| **Regulation** | Product safety and quality standards (ISO 9001, industry-specific), OT security (IEC 62443), export controls, EU AI Act for safety components, NIS2 for critical sectors |
| **Architecture** | OT/IT convergence via Azure IoT Operations on Arc-enabled Kubernetes, edge AI, offline operation, Purdue-model network segmentation, Digital Twins, Fabric Real-Time Intelligence |
| **Data** | Time-series sensor data, MES/ERP (SAP), quality records, engineering documents |
| **Microsoft resources** | Microsoft Cloud for Manufacturing guidance, Architecture Center manufacturing scenarios, IoT Operations, SAP on Azure |

### Cross-industry patterns

| Pattern | Applies to |
|---|---|
| Intelligent document processing | Loans, claims, invoices, quality certificates (Topic 34 file 10) |
| Customer service agents | Banking, retail, utilities (Topic 24, Topic 34) |
| Real-time analytics on events | Fraud, store operations, production lines (Topic 12) |
| Edge AI | Stores, factories (Topic 31 file 07) |
| SAP integration | Retail and manufacturing ERP (Topic 34 file 11) |

## How it works on Azure

Example: bank customer-service agent with regulatory controls:

```text
Channels (web, app, Teams) → Front Door WAF → APIM AI gateway (quotas, logging)
→ Foundry agent (zone-redundant, private networking, two regions for DORA resilience)
   tools: account lookup (OBO, read-only), transaction disputes (approval required), KB search (Foundry IQ)
→ Guardrails (prompt shields, PII), evaluation gates, red teaming, continuous monitoring
→ Audit: immutable storage of decisions and transcripts per retention rules; Purview DLP and retention
→ Model risk documentation: model cards, evaluation results, change approvals, exit plan
```

Example: retail peak readiness: load test at 3× expected peak (Topic 27 file 08), pre-scale and reserve capacity, cache catalogue at the edge, queue-based checkout, feature flags to shed non-critical features, real-time dashboards.

Example: manufacturing predictive maintenance: OPC UA → IoT Operations → Fabric eventstream → anomaly detection model → alerts to maintenance (Dynamics 365 Field Service) → technician copilot grounded on manuals.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Industry clouds vs build** | Use Microsoft industry data models and solutions where they fit; build differentiating logic yourself. |
| **Regulatory mapping** | Map regulations to controls early (Compliance Manager templates, MCSB); involve compliance teams. |
| **Resilience** | Financial services often need multi-region and exit strategies; document concentration risk. |
| **Edge** | Retail and manufacturing need local resilience; plan Arc-based fleet management. |
| **AI risk** | High-risk uses (credit, safety components) need EU AI Act conformity and human oversight. |

## Hands-on

1. For each industry, list three AI use cases and the top regulatory constraints.
2. Map EU DORA requirements to Azure controls for a banking workload (resilience testing, incident reporting, third-party risk).
3. Design a retail peak readiness checklist.
4. Draw a manufacturing OT/IT architecture with IoT Operations and Fabric.
5. Review the Architecture Center industry pages and adapt one reference architecture.

## Security, cost and reliability notes

- **Regulated data** (card data, customer financial data, production IP) needs isolation, encryption and auditing.
- **Third-party risk** registers must include cloud services and AI models.
- **Cost**: peak-driven designs need autoscale and commitments for baseline; edge fleets need lifecycle budgets.

## Common mistakes

| Mistake | Fix |
|---|---|
| Generic designs ignoring sector regulation | Map regulations to controls |
| Single-region critical banking services | Resilience per regulation |
| Store systems dependent on WAN | Edge resilience |
| Flat OT/IT networks | Segmentation and brokers |
| Treating credit or safety AI as low risk | EU AI Act high-risk obligations |

## Check yourself

1. Name three regulations relevant to financial services AI.
2. What architecture concerns are specific to retail?
3. How do manufacturers connect OT data to the cloud safely?
4. Which cross-industry patterns recur?
5. When does EU AI Act high-risk classification apply in these industries?
6. What Microsoft resources help with industry designs?

<details>
<summary>Answers</summary>

1. Any three: DORA, EBA outsourcing guidelines, model risk management (SR 11-7/SS1/23), PCI DSS, GDPR, EU AI Act.
2. Seasonal peak scale, edge and offline store operations, PCI DSS, personalisation with consent, real-time inventory.
3. Through Azure IoT Operations on Arc-enabled Kubernetes with segmented networks, brokers and data flows to the cloud.
4. Intelligent document processing, customer service agents, real-time event analytics, edge AI, SAP integration.
5. For example, credit scoring and AI used as safety components of regulated products.
6. Microsoft Cloud industry guidance and data models, Architecture Center industry scenarios, Fabric industry solutions, Compliance Manager templates.

</details>

## References

- [Microsoft Cloud for Financial Services](https://learn.microsoft.com/industry/financial-services/)
- [Microsoft Cloud for Manufacturing](https://learn.microsoft.com/industry/manufacturing/)
- [Architecture Center: finance](https://learn.microsoft.com/azure/architecture/industries/finance)
- [Architecture Center: manufacturing](https://learn.microsoft.com/azure/architecture/industries/manufacturing)
- [PCI DSS on Azure](https://learn.microsoft.com/compliance/regulatory/offering-pci-dss)
- [SAP on Azure](https://learn.microsoft.com/azure/sap/)
