# 02 · Identity and Access with Microsoft Entra

> Tracks: E ● A ● C ● · Part A · Foundations

Identity is the control plane of Azure security. Every request to Azure, whether from a person, a pipeline, an application or an AI agent, is authenticated by Microsoft Entra ID and authorised by a role assignment. This topic covers how that works and how to design it so that nothing depends on a shared secret.

## What you will be able to do

- Explain tenants, users, groups, service principals and the two separate role systems.
- Choose authentication methods and enforce phishing-resistant sign-in.
- Design least-privilege Azure RBAC, including data-plane roles for AI services.
- Connect services to each other with managed identities and no stored secrets.
- Register an application and choose the right OAuth flow.
- Design Conditional Access policies and privileged access.
- Explain how AI agents get identities and how those identities are governed.

## Prerequisites

[Topic 01](../01-cloud-and-azure-fundamentals/README.md), especially the resource hierarchy and the control plane vs data plane split.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Microsoft Entra ID Fundamentals](01-microsoft-entra-id-fundamentals.md) | 45 min | CLI |
| 02 | [Authentication](02-authentication.md) | 45 min | Portal |
| 03 | [Azure RBAC](03-azure-rbac.md) | 60 min | CLI |
| 04 | [Managed Identities and Workload Identity](04-managed-identities-and-workload-identity.md) | 60 min | CLI, Python |
| 05 | [App Registrations and Service Principals](05-app-registrations-and-service-principals.md) | 60 min | CLI |
| 06 | [Conditional Access and Identity Protection](06-conditional-access-and-identity-protection.md) | 45 min | Portal |
| 07 | [Privileged Identity Management and Identity Governance](07-privileged-identity-management-and-identity-governance.md) | 45 min | Portal |
| 08 | [Microsoft Entra External ID](08-microsoft-entra-external-id.md) | 40 min | – |
| 09 | [Hybrid Identity](09-hybrid-identity.md) | 40 min | – |
| 10 | [Microsoft Entra Agent ID](10-microsoft-entra-agent-id.md) | 45 min | Portal |
| 11 | [Entra Verified ID and Global Secure Access](11-entra-verified-id-and-global-secure-access.md) | 30 min | – |

Engineers: 01, 03, 04, 05 and 10 are essential. Architects and consultants: all of it, with 06, 07 and 10 in depth.

## Completion checklist

- [ ] I can explain the difference between Entra roles and Azure roles.
- [ ] I can name three phishing-resistant authentication methods.
- [ ] I have assigned a data-plane role at resource scope and tested it.
- [ ] I have called an Azure service from code using a managed identity and no key.
- [ ] I can explain delegated vs application permissions and when to use on-behalf-of.
- [ ] I can sketch a baseline Conditional Access policy set, including break-glass handling.
- [ ] I can explain eligible vs active role assignments.
- [ ] I can describe what an agent identity is and how it differs from a service principal.

## Maps to

- **AZ-900 / AI-901**: identity, access and security concepts.
- **AZ-104**: manage Entra identities and governance; manage access to Azure resources.
- **AI-103**: secure AI solutions with keyless authentication and role-based access.
- **AZ-305**: design identity, governance and monitoring solutions.
- **SC-500 / SC-300**: identity protection, Conditional Access, privileged access.

## Next

[03 · Governance and Landing Zones](../03-governance-and-landing-zones/README.md)
