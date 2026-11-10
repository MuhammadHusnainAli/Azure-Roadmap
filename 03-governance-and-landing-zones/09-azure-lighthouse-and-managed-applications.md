# Azure Lighthouse and Managed Applications

## Why it matters

Consultants and partners manage Azure environments they do not own. Software vendors ship solutions into customers' subscriptions. Large groups run several tenants. The naive ways to do this, guest accounts in every customer tenant or shared credentials, do not scale and fail security reviews. Azure Lighthouse and Managed Applications are the two platform mechanisms built for it. If you work at a partner, this is how your company should be accessing customer estates.

## Part 1 · Azure Lighthouse

### Key concepts

Azure Lighthouse provides **Azure delegated resource management**: a customer delegates a subscription or resource group to a managing tenant, and users in the managing tenant then work on those resources from their own tenant, with their own identities, without switching directories.

| Term | Meaning |
|---|---|
| **Managing tenant** | The service provider's (or central IT's) tenant |
| **Customer tenant** (managed tenant) | The tenant that owns the delegated resources |
| **Delegation** | The act of onboarding a subscription or resource group |
| **Registration definition** | The offer: managing tenant ID plus the authorisations it requests |
| **Registration assignment** | That definition applied to a specific scope in the customer tenant |
| **Authorisation** | A principal (user, group or service principal in the managing tenant) paired with a built-in Azure role |

```text
Managing tenant                         Customer tenant
┌─────────────────────────┐             ┌──────────────────────────────┐
│ Group: Contoso-Ops      │── Reader ──►│ Subscription A               │
│ Group: Contoso-Engineers│── Contributor (eligible, PIM) ──►          │
│ SP: Monitoring pipeline │── Monitoring Contributor ──► RG app-prod   │
└─────────────────────────┘             └──────────────────────────────┘
        work from here, in one portal, across all customers
```

### What it gives you

- **One sign-in, many customers.** Staff use their home identity, protected by your own Conditional Access and MFA.
- **Cross-tenant management at scale.** Azure Policy, Resource Graph, Azure Monitor, Defender for Cloud, Microsoft Sentinel, Update Manager, Backup and others work across delegated subscriptions in a single view.
- **Customer transparency and control.** The customer sees the delegation in *Service providers*, sees every action in their activity log attributed to the provider's user, and can remove access at any time.
- **Automation.** Service principals in the managing tenant can deploy to delegated subscriptions, so one pipeline serves many customers.
- **No per-customer account lifecycle.** A leaver is removed from one group in your tenant and loses access everywhere.
- **Free.** No charge for Lighthouse itself.

### Authorisations and their limits

Authorisations use **Azure built-in roles**, with restrictions:

| Rule | Detail |
|---|---|
| **Owner is not supported** | The managing tenant can never be Owner |
| **No roles with `DataActions`** | Data-plane roles such as Storage Blob Data Reader cannot be delegated |
| **Custom roles are not supported** | Built-in only |
| **User Access Administrator is limited** | Allowed only for assigning a defined set of roles to **managed identities** in the customer tenant, which is what lets `deployIfNotExists` policy remediation work |
| **Scope** | Subscription or resource group. Management group delegation is not supported; onboard subscriptions individually or by policy |

The data-plane restriction is significant for AI work. Through Lighthouse a partner can deploy and operate a Foundry resource, networking and monitoring, but **cannot be granted roles to call models, query a search index or read blobs** through delegated access. For data-plane access you need something else: a workload identity inside the customer tenant used by pipelines, B2B guest access with PIM, or a jump host inside the customer environment.

**Eligible authorisations** integrate with PIM: provider staff must activate the role, with MFA and optional approval by the provider's approvers, for a limited time. This requires appropriate Entra ID licensing in the managing tenant. Use it for anything above read access.

Always assign to **groups** and **service principals**, never to individual users.

### Onboarding

| Method | Fits |
|---|---|
| **ARM / Bicep template** | Direct onboarding of a specific customer. The customer (an Owner of the subscription) deploys a template containing your tenant ID and authorisations |
| **Marketplace Managed Service offer** | Public or private offers that customers accept; supports many customers and updating terms centrally |

Skeleton of an onboarding template:

```bicep
targetScope = 'subscription'

param managedByTenantId string
param offerName string = 'Contoso Managed Services'

resource definition 'Microsoft.ManagedServices/registrationDefinitions@2022-10-01' = {
  name: guid(offerName)
  properties: {
    registrationDefinitionName: offerName
    description: 'Operations and monitoring by Contoso'
    managedByTenantId: managedByTenantId
    authorizations: [
      {
        principalId: '<object id of Contoso-Ops group in the managing tenant>'
        principalIdDisplayName: 'Contoso Operations'
        roleDefinitionId: 'acdd72a7-3385-48ef-bd42-f606fba81ae7'   // Reader
      }
      {
        principalId: '<object id of Contoso-Ops group>'
        principalIdDisplayName: 'Contoso Operations'
        roleDefinitionId: '91c1777a-f3dc-4fae-b103-61d183457e46'   // Managed Services Registration assignment Delete Role
      }
    ]
    eligibleAuthorizations: [
      {
        principalId: '<object id of Contoso-Engineers group>'
        principalIdDisplayName: 'Contoso Engineers'
        roleDefinitionId: 'b24988ac-6180-42a0-ab88-20f7382dd24c'   // Contributor
        justInTimeAccessPolicy: {
          multiFactorAuthProvider: 'Azure'
          maximumActivationDuration: 'PT4H'
        }
      }
    ]
  }
}

resource assignment 'Microsoft.ManagedServices/registrationAssignments@2022-10-01' = {
  name: guid(offerName, subscription().id)
  properties: {
    registrationDefinitionId: definition.id
  }
}
```

Including the *Managed Services Registration assignment Delete Role* lets the provider remove its own access cleanly at the end of an engagement.

### Lighthouse inside one organisation

Lighthouse is also useful for enterprises with **several tenants** (after acquisitions, or with separate tenants per region or brand): a central team manages subscriptions in all of them from one tenant. For new estates, a single tenant with management groups is still simpler.

### What Lighthouse is not

| Need | Use |
|---|---|
| Manage **Azure resources** in customer subscriptions | **Azure Lighthouse** |
| Administer a customer's **Microsoft 365 / Entra tenant** | **Granular delegated admin privileges (GDAP)** through Partner Center |
| Unified Microsoft 365 management for managed service providers | **Microsoft 365 Lighthouse** (a different product with a similar name) |
| Collaborate with individual external users | **B2B guest access** |
| Sell Azure and receive partner credit | CSP relationship plus Partner Admin Link (PAL) or the relevant attribution mechanism |

Partners commonly use GDAP and Azure Lighthouse together. They solve different halves of the problem.

## Part 2 · Azure Managed Applications

### Key concepts

A managed application is a packaged solution deployed into a customer's subscription, where the **publisher keeps management access** to the deployed resources and the **customer's access is restricted**.

```text
Customer subscription
├── Resource group: rg-customer-app            ← the "application" resource (what the customer sees)
└── Managed resource group: mrg-app-<id>       ← the actual resources
        • publisher: full access (as defined in the offer)
        • customer: read-only by default, enforced with a deny assignment
```

| Component | Purpose |
|---|---|
| **`mainTemplate.json`** | The ARM template (Bicep compiled to JSON) defining the resources |
| **`createUiDefinition.json`** | The portal form the customer fills in at deployment |
| **`viewDefinition.json`** (optional) | Custom views and actions on the application in the portal |
| **Managed resource group** | Holds the deployed resources; locked against customer changes |
| **Publisher authorisations** | Which publisher identities get which roles on the managed resource group |
| **Deny assignment** | What stops the customer modifying the resources. Publishers can allow specific customer actions |

### Publishing routes

| Route | Audience |
|---|---|
| **Service catalog** | Internal: a central IT team publishes approved solutions for the organisation's own teams |
| **Microsoft Marketplace** | External: an ISV sells to customers, with billing through Microsoft |

### Why use them

**For ISVs:** deliver software as a service-like experience while it runs in the customer's subscription, which many enterprise customers prefer for data residency and network reasons. The publisher can update, patch and support it. Marketplace handles transactions, and purchases can count toward the customer's Azure consumption commitment.

**For central IT:** offer a "standard secure storage account" or "approved data science environment" that teams can deploy but not misconfigure.

**Features worth knowing:** just-in-time publisher access (the customer approves each elevation), customer-enabled actions, notifications to the publisher by webhook on lifecycle events, and metering APIs for usage-based billing.

### Managed applications for AI solutions

An ISV shipping an AI application faces a choice:

| Model | Where it runs | Notes |
|---|---|---|
| **SaaS** | Publisher's tenant | Simplest to operate. Customer data leaves the customer's subscription |
| **Managed application** | Customer's subscription | Data, models and network stay under the customer's tenant and policies. Uses the customer's model quota and region choices. Publisher still operates it |
| **Template only (solution template)** | Customer's subscription | Customer owns and operates everything; no ongoing publisher access |

Managed applications suit regulated customers who want an AI solution but will not send data to a vendor's tenant. The complications: the customer's Azure Policy can block parts of your deployment, model quota is the customer's to arrange, and private networking has to integrate with their landing zone. Design the template to be parameterised for those realities and test it against a landing-zone-style policy set.

AI agents and applications can also be listed in Microsoft Marketplace through agent-specific and SaaS offer types (topic 32).

### Lighthouse vs Managed Applications

| | Azure Lighthouse | Managed Applications |
|---|---|---|
| Purpose | Manage a customer's **existing** resources as a service | Deliver and manage a **packaged solution** |
| What is managed | Delegated subscriptions or resource groups | Only the managed resource group of the application |
| Customer access | Unchanged | Restricted by deny assignment |
| Typical user | Managed service providers, central IT across tenants | ISVs, internal platform teams |
| Billing | Free; services sold separately | Marketplace transaction or internal |

They are complementary. A provider might sell a managed application and also operate the customer's wider estate through Lighthouse.

## How it works on Azure

### A partner access model that passes security review

| Layer | Mechanism |
|---|---|
| Azure control plane, read and monitor | Lighthouse, permanent Reader and Monitoring roles to a group |
| Azure control plane, change | Lighthouse eligible authorisations with PIM, MFA and time limit |
| Deployments | A pipeline using a workload identity, either a service principal in the managing tenant through Lighthouse (control plane) or an identity in the customer tenant with federated credentials (when data-plane roles are needed) |
| Data plane (models, search, storage) | Identity in the customer tenant; or named B2B guests with PIM and access reviews, for limited periods |
| Microsoft 365 and Entra administration | GDAP with least-privilege roles and expiry |
| Evidence | Customer's activity log and sign-in logs; provider's PIM audit history |
| Exit | Remove the delegation and GDAP relationship; delete customer-tenant identities |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Lighthouse vs guest accounts | Lighthouse for control-plane management at any scale. Guests only where data-plane or directory access is unavoidable |
| Subscription vs resource group delegation | Resource group when the engagement covers one workload; subscription for full managed service |
| Permanent vs eligible | Permanent for read and monitoring only |
| Template vs Marketplace onboarding | Template for a few customers; Marketplace offer for a repeatable service |
| SaaS vs managed application | SaaS unless customers require the solution inside their own subscription |
| How locked down a managed app is | Enough to protect what you support, with customer actions enabled for legitimate operations |

## Hands-on

**As a customer (any subscription)**

1. Open **Service providers** in the portal. Review *Service provider offers* and *Delegations*. This is where a customer audits and removes access.

**Lighthouse lab (needs two tenants, or reading only)**

1. In the managing tenant, create a security group and note its object ID and the tenant ID.
2. In the customer tenant, deploy the onboarding template at subscription scope:

   ```bash
   az deployment sub create --location swedencentral \
     --template-file lighthouse.bicep --parameters managedByTenantId=<managing-tenant-id>
   ```

3. In the managing tenant, open **My customers** and confirm the delegation.
4. Query across tenants from the managing tenant:

   ```bash
   az account list --query "[].{Name:name, Tenant:tenantId, ManagedBy:managedByTenants[0].tenantId}" -o table
   az graph query -q "Resources | summarize count() by subscriptionId, tenantId" -o table
   ```

5. In the customer tenant's activity log, find an action taken by the provider's user.
6. Try to assign *Storage Blob Data Reader* in an authorisation and observe the rejection.
7. Remove the delegation from **Service providers → Delegations**.

**Managed application (service catalog)**

1. Follow the quickstart *Create and publish a managed application definition* with a storage account template.
2. Deploy the application from the service catalog.
3. Open the managed resource group as the customer and try to delete the storage account. Read the deny assignment under *Access control (IAM) → Deny assignments*.

## Security, cost and reliability notes

- **Security:** a compromised account in the managing tenant can reach every customer. Protect it with phishing-resistant MFA, Conditional Access, PIM and privileged access workstations. Supply-chain attacks through service providers are a known pattern.
- **Security:** as a customer, review delegations regularly, prefer providers who use eligible authorisations, and alert on new registration assignments.
- **Security:** Lighthouse intentionally cannot grant Owner or data access. Do not work around it with shared credentials.
- **Cost:** Lighthouse is free. Managed applications may carry Marketplace fees for the publisher.
- **Reliability:** customer policies can break provider automation and managed application updates. Agree on exemptions and test against the customer's policy set.

## Common mistakes

- Using guest accounts in dozens of customer tenants instead of Lighthouse.
- Assigning authorisations to individual users.
- Permanent Contributor for the whole provider team.
- Expecting Lighthouse to grant access to data or to Entra ID and Microsoft 365.
- Confusing Azure Lighthouse with Microsoft 365 Lighthouse.
- Forgetting the delete role, leaving the provider unable to remove its own delegation.
- Shipping a managed application that fails under the customer's landing zone policies.
- No documented offboarding.

## Check yourself

1. What does Azure Lighthouse let a managing tenant do?
2. Name three kinds of role that cannot be granted through Lighthouse.
3. Why does the data-plane restriction matter for a partner operating an AI workload?
4. How does a customer see and revoke a provider's access?
5. What is the difference between Azure Lighthouse and GDAP?
6. What is the managed resource group in a managed application, and who can change it?
7. When would an ISV choose a managed application over SaaS for an AI product?

<details>
<summary>Answers</summary>

1. Manage delegated subscriptions and resource groups in customer tenants from its own tenant, using its own identities, at scale.
2. Owner, any role containing `DataActions`, and custom roles (User Access Administrator is allowed only in a limited form).
3. The partner cannot be delegated roles to call models, query search indexes or read storage, so data-plane tasks need an identity in the customer tenant or guest access.
4. In *Service providers* in the portal they can view delegations and remove them; all provider actions appear in the customer's activity log.
5. Lighthouse delegates Azure resource management. GDAP delegates administration of the customer's Microsoft 365 and Entra tenant through Partner Center.
6. The resource group holding the application's actual resources. The publisher manages it; the customer is restricted by a deny assignment.
7. When customers require data, models and networking to remain in their own subscription and under their own policies, while the ISV still operates the solution.

</details>

## References

- [What is Azure Lighthouse?](https://learn.microsoft.com/azure/lighthouse/overview)
- [Azure Lighthouse architecture](https://learn.microsoft.com/azure/lighthouse/concepts/architecture)
- [Tenants, users, and roles in Azure Lighthouse scenarios](https://learn.microsoft.com/azure/lighthouse/concepts/tenants-users-roles)
- [Onboard a customer to Azure Lighthouse](https://learn.microsoft.com/azure/lighthouse/how-to/onboard-customer)
- [Create eligible authorizations](https://learn.microsoft.com/azure/lighthouse/how-to/create-eligible-authorizations)
- [Recommended security practices for Azure Lighthouse](https://learn.microsoft.com/azure/lighthouse/concepts/recommended-security-practices)
- [Azure Managed Applications overview](https://learn.microsoft.com/azure/azure-resource-manager/managed-applications/overview)
- [Quickstart: create and publish a managed application definition](https://learn.microsoft.com/azure/azure-resource-manager/managed-applications/publish-service-catalog-app)
- [Granular delegated admin privileges (GDAP)](https://learn.microsoft.com/partner-center/customers/gdap-introduction)
