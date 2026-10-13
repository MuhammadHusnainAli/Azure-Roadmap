# Accounts, Subscriptions and Offers

> Topic: 01 · Cloud and Azure Fundamentals · Tracks: E ○ A ● C ● · Status checked: 2026-10

## Why it matters

How an organisation buys Azure decides who can create subscriptions, where invoices go, which discounts apply and who you have to ask for a quota increase. Engineers mostly need enough to get a subscription and understand its limits. Architects need it for subscription design. Consultants need it on day one of an engagement, because "how do you buy Azure today?" shapes the commercial side of every proposal.

## Key concepts

### Three things that get confused

| Term | What it is |
|---|---|
| **Account** (user account) | An identity in Microsoft Entra ID that signs in |
| **Billing account** | The commercial agreement under which Azure is purchased and invoiced |
| **Subscription** | A container for resources, linked to one billing account and trusting one Entra tenant |

A person signs in with a user account, creates resources in a subscription, and the usage is charged to a billing account.

### Billing account types

There are four. Check yours under **Cost Management + Billing → Properties → Type**.

| Type | Who gets it | How it is bought |
|---|---|---|
| **Microsoft Online Services Program (MOSP)** | Individuals and small organisations signing up on the website | Credit card or invoice, pay-as-you-go rates |
| **Enterprise Agreement (EA)** | Large organisations with an enrolment | Negotiated, typically with an annual monetary commitment |
| **Microsoft Customer Agreement (MCA)** | Organisations buying directly or through a Microsoft seller; also some website sign-ups | Modern, evergreen agreement |
| **Microsoft Partner Agreement (MPA)** | Cloud Solution Provider (CSP) partners, to manage their customers | Customer buys Azure from the partner |

Microsoft is steadily moving direct customers from EA to MCA. Many organisations you meet will be on one, mid-migration, or have both.

### Billing hierarchies

Each agreement has its own structure above the subscription. This is separate from the management group hierarchy: one is about money, the other about governance.

**MOSP**

```text
Billing account
└── Subscription        ← invoice generated here
```

**Enterprise Agreement**

```text
Billing account (enrolment)      ← invoice generated here
└── Department (optional)
    └── Account (one account owner)
        └── Subscription
```

**Microsoft Customer Agreement**

```text
Billing account
└── Billing profile              ← one invoice per profile
    └── Invoice section          ← groups costs on the invoice
        └── Subscription
```

**Microsoft Partner Agreement (CSP)**

```text
Billing account (partner)
└── Billing profile (per currency)
    └── Customer
        └── Subscription (Azure plan)
```

Subscription limits by agreement:

| Agreement | Subscription limit |
|---|---|
| MOSP | 5 for a new billing account (more by request, case by case) |
| MCA for an individual | 5 |
| EA | 5,000 per EA account; unlimited accounts per enrolment |
| MCA for an enterprise | 10,000 |
| MPA | 10,000 |

Deleted and disabled subscriptions count toward the limit.

### Billing roles vs Azure roles

Billing has its own role system, separate from Azure RBAC.

| Agreement | Key billing roles |
|---|---|
| EA | Enterprise Administrator, Department Administrator, Account Owner |
| MCA | Billing account, billing profile and invoice section Owner, Contributor, Reader; Azure subscription creator |
| MOSP | Account Administrator |

Being Owner of a subscription does not let you see the invoice. Being a billing administrator does not let you see the resources. This separation is deliberate, and it is a frequent source of "I can't see costs" tickets. Azure RBAC roles such as *Cost Management Reader* give cost visibility at subscription scope without billing-account access.

### Common subscription offers

| Offer | Purpose | Notes |
|---|---|---|
| **Azure free account** | Try Azure | USD 200 credit for 30 days, 20+ popular services free for 12 months, 65+ services with always-free amounts |
| **Pay-as-you-go** | General use without commitment | Standard rates, monthly billing |
| **Azure for Students** | Learning | Credit without a credit card, restricted regions and services |
| **Visual Studio subscriber** | Individual dev/test | Monthly credit, not for production |
| **Enterprise Dev/Test** and **Pay-As-You-Go Dev/Test** | Team dev/test | Discounted rates, no Windows licence charge on VMs; requires Visual Studio subscribers; no production use and no SLA |
| **Azure plan** | Subscriptions under MCA and CSP | The standard modern offer |
| **Azure Sponsorship** | Programmes, start-ups, events | Fixed credit and end date |

When the free account's credit runs out or the 30 days pass, the subscription is disabled until you upgrade to pay-as-you-go. The credit cannot be spent on Marketplace offers.

Free, student and trial subscriptions have **reduced quotas and restricted features**. Expect low or zero quota for GPU VMs and limited access to some AI models. If a lab fails with a quota error on a free subscription, that is usually why.

### Subscription lifecycle

| State | Meaning |
|---|---|
| Active | Normal operation |
| Past due | Payment missed; still running |
| Disabled | Resources stopped; data retained for a period (commonly 90 days) |
| Deleted | Gone after the retention window |
| Expired | Offer ended (free, sponsorship) |

Other lifecycle operations:

- **Transfer billing ownership:** move a subscription to another billing account. Resources and tenant stay put.
- **Change directory:** move a subscription to a different Entra tenant. This **removes all RBAC assignments** and breaks managed identities and Key Vault access policies. Plan it carefully.
- **Cancel:** stops billing and starts the deletion timer.

### Subscription vending

In an enterprise nobody should create subscriptions by hand. A team requests one through a form or pull request, and automation creates it, places it in the right management group, attaches networking, applies RBAC and sets budgets. Microsoft publishes Bicep and Terraform modules for this. Covered in topic 03.

### Commercial levers you should recognise

Detail is in topic 29; you need the names now.

| Lever | What it is |
|---|---|
| **Reservations** | One or three year commitment to a specific resource type for a discount |
| **Savings plan for compute** | Hourly spend commitment across compute services |
| **Azure Hybrid Benefit** | Use existing Windows Server and SQL Server licences in Azure |
| **MACC** (Microsoft Azure Consumption Commitment) | Contractual commitment to spend a total amount over a term |
| **Provisioned throughput reservations** | Commitment discount on reserved AI model capacity |
| **Marketplace** | Third-party software billed through Azure; eligible purchases can count toward MACC |

### Support plans

| Plan | For | Technical support |
|---|---|---|
| Basic | Everyone | Billing and subscription support only |
| Developer | Non-production | Business hours |
| Standard | Production | 24x7, faster response for critical cases |
| Professional Direct | Business-critical | 24x7, faster again, plus advisory |
| Unified (Enterprise) | Whole organisation | Contract-based, includes an account team |

Quota increase requests and billing issues can be raised on any plan, including Basic.

## How it works on Azure

### Buying routes compared

| | Direct (MCA / EA) | Through a partner (CSP) |
|---|---|---|
| Contract with | Microsoft | The partner |
| Invoice from | Microsoft | The partner |
| First-line support | Microsoft (per support plan) | The partner |
| Pricing | List price or negotiated | Set by partner |
| Good for | Large organisations with procurement capability | Organisations wanting managed services bundled |

As a consultant, establish the route early. In CSP the partner controls subscription creation and some reservation purchases, which changes how fast things can happen.

### Where AI spend shows up

- **Foundry Models pay-as-you-go:** billed per token on the subscription that holds the Foundry resource. Models sold directly by Azure appear as normal Azure meters.
- **Partner and community models:** many are billed through Azure Marketplace. That needs Marketplace purchasing enabled on the billing account and the right permissions, and it is blocked on some offer types.
- **Provisioned throughput:** billed hourly per unit, with reservations bought separately for the discount.
- **Microsoft 365 Copilot and Agent 365:** per-user licences bought through Microsoft 365 licensing, not Azure consumption.
- **Copilot Studio:** capacity packs or pay-as-you-go meters linked to an Azure subscription.

A common discovery in workshops is that the team owning the Azure subscription and the team owning Microsoft 365 licensing have never spoken.

## Design decisions and trade-offs

- **One subscription or many?** Start with at least a production and a non-production subscription. Add more for independent workloads, distinct owners or quota isolation.
- **Dev/Test offer for non-production.** Real savings, but only Visual Studio subscribers may use the resources, and there is no financially backed SLA.
- **Free and student subscriptions for learning only.** Don't prototype client work on them; quotas will not reflect production.
- **Decide who can create subscriptions.** Open creation leads to sprawl. Fully manual creation leads to shadow IT. Automated vending with guardrails is the workable middle.
- **Tenant strategy is not a billing decision.** Don't create a tenant to get a separate invoice. Use billing profiles, invoice sections or subscriptions.

## Hands-on

Portal steps (no cost):

1. Go to **Cost Management + Billing**. Under **Billing scopes**, note the billing account type.
2. Open **Properties** and record the agreement type and your role.
3. Open **Subscriptions**, pick one, and note the **Offer** and **Offer ID**.
4. On the subscription, open **Usage + quotas** to see what limits apply.
5. Open **Access control (IAM)** and compare it with your billing role. Note they are different lists.

CLI:

```bash
# Subscription basics
az account show --query "{Name:name, Id:id, Tenant:tenantId, State:state}" -o table

# Offer / quota ID tells you what kind of subscription it is
az rest --method get \
  --url "https://management.azure.com/subscriptions/$(az account show --query id -o tsv)?api-version=2022-12-01" \
  --query "subscriptionPolicies"

# Billing accounts you can see (may be empty without a billing role)
az billing account list --query "[].{Name:displayName, Type:agreementType}" -o table
```

The `quotaId` value (for example `PayAsYouGo_2014-09-01`, `MSDN_2014-09-01`, `EnterpriseAgreement_2014-09-01`, `FreeTrial_2014-09-01`) identifies the offer and explains unexpected restrictions.

## Security, cost and reliability notes

- **Security:** the account that created an MOSP subscription is its Account Administrator. Protect it with MFA and avoid using personal accounts for company subscriptions.
- **Security:** changing a subscription's directory strips access. Treat it as a migration with a rollback plan.
- **Cost:** set a budget and alert on every subscription the day it is created. On pay-as-you-go there is no spending cap.
- **Cost:** credits (free, Visual Studio, sponsorship) have a spending limit that disables the subscription when reached. That is fine for labs and disastrous for anything real.
- **Reliability:** an unpaid invoice disables the subscription and stops workloads. Make sure billing notifications reach a monitored mailbox, not one person.

## Common mistakes

- Running production on a Dev/Test or Visual Studio subscription.
- Assuming a subscription Owner can see invoices.
- Building a prototype on a free subscription and discovering at scale-up that quotas and model access were restricted.
- Changing a subscription's tenant without planning for lost role assignments.
- Using a new tenant to separate costs.
- Forgetting that Marketplace purchasing must be enabled before deploying partner models.
- Assuming Copilot licences come out of the Azure commitment.

## Check yourself

1. Distinguish a user account, a billing account and a subscription.
2. Name the four billing account types.
3. Draw the MCA billing hierarchy. At which level is an invoice generated?
4. What does the Azure free account include, and what happens when the credit is used?
5. Why might someone who is Owner of a subscription be unable to view its invoice?
6. What breaks when you move a subscription to a different Entra tenant?
7. What is the difference between buying direct and through CSP?
8. Where is a Microsoft 365 Copilot licence purchased, compared with Foundry model usage?

<details>
<summary>Answers</summary>

1. A user account is an Entra identity. A billing account is the commercial agreement that gets invoiced. A subscription is a resource container linked to one billing account and one tenant.
2. Microsoft Online Services Program, Enterprise Agreement, Microsoft Customer Agreement, Microsoft Partner Agreement.
3. Billing account → billing profile → invoice section → subscription. The invoice is generated per billing profile.
4. USD 200 credit for 30 days, 20+ services free for 12 months, and 65+ always-free services. When the credit is spent or 30 days pass, the subscription is disabled until upgraded to pay-as-you-go.
5. Billing roles are separate from Azure RBAC roles.
6. All RBAC role assignments are removed, and managed identities and identity-dependent configuration such as Key Vault access break.
7. Direct means the contract, invoice and support are with Microsoft. CSP means they are with the partner, who may bundle managed services and controls subscription provisioning.
8. Copilot is a per-user licence through Microsoft 365 licensing. Foundry model usage is Azure consumption billed to a subscription.

</details>

## References

- [View your billing accounts in the Azure portal](https://learn.microsoft.com/azure/cost-management-billing/manage/view-all-accounts)
- [Azure free account](https://azure.microsoft.com/pricing/purchase-options/azure-account)
- [Create a Microsoft Customer Agreement subscription](https://learn.microsoft.com/azure/cost-management-billing/manage/create-subscription)
- [Understand and work with scopes](https://learn.microsoft.com/azure/cost-management-billing/costs/understand-work-scopes)
- [Transfer an Azure subscription to a different Microsoft Entra directory](https://learn.microsoft.com/azure/role-based-access-control/transfer-subscription)
- [Subscription vending](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/design-area/subscription-vending)
- [Azure support plans](https://azure.microsoft.com/support/plans/)
