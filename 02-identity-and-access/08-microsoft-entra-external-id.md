# Microsoft Entra External ID

> Topic: 02 · Identity and Access · Tracks: E ○ A ● C ● · Status checked: 2026-10

## Why it matters

Not everyone who uses your solution is an employee. Partners need to collaborate, suppliers need a portal, and customers need to sign in to the AI assistant on your website. Microsoft Entra External ID covers all of these. The first design question, and the one people get wrong, is which of its two quite different modes applies.

## Key concepts

### Two tenant configurations

| | **Workforce tenant** | **External tenant** |
|---|---|---|
| Purpose | Collaborate with partners and guests inside your organisation's directory | Customer identity and access management (CIAM) for apps you publish to consumers or business customers |
| Users | Guests alongside employees | Consumers and business customers only, isolated from employees |
| Typical apps | Microsoft 365, Teams, SharePoint, internal line-of-business apps, Azure resources | Your public web and mobile apps |
| Sign-up | Invitation, or self-service through entitlement management or user flows | Self-service sign-up with branded user flows |
| Identity providers | The guest's home Entra tenant, Microsoft account, Google, SAML/WS-Fed IdPs, email one-time passcode | Email and password, email one-time passcode, social providers (Google, Facebook, Apple), custom OIDC and SAML providers |
| Branding | Your organisation's sign-in branding | Fully customised per application; custom URL domain |

If the people are external but **work with** you, use B2B collaboration in the workforce tenant. If they are **customers of an application**, create an external tenant. Never put consumers in your corporate directory.

### B2B collaboration (workforce tenant)

A guest is an object in your directory that authenticates somewhere else.

**Lifecycle**

```text
Invite (or self-service request) ─► Redeem invitation ─► Guest user object created
        ─► Access granted via groups, apps, roles ─► Reviewed ─► Removed
```

- The guest signs in with their **home** credentials. You do not manage their password.
- Their `userType` is `Guest`, with restricted default directory permissions.
- They can be assigned to apps, groups, Azure RBAC roles and access packages like any user.

**Cross-tenant access settings**

Control collaboration per partner organisation, in both directions:

| Setting | Controls |
|---|---|
| **Inbound** | Which external users may access what in your tenant |
| **Outbound** | Which of your users may access what in other tenants |
| **Trust settings** | Whether you accept the partner tenant's MFA, compliant device and hybrid-joined device claims |
| **Tenant restrictions** | Stop your users signing in to unapproved external tenants from your network or devices |

Trusting a partner's MFA avoids making guests register a second MFA method in your tenant, and lets device-based Conditional Access policies work for them.

**Related features**

| Feature | What it does |
|---|---|
| **B2B direct connect** | Mutual trust with no guest object; used for Teams shared channels |
| **Cross-tenant synchronisation** | Automatically provisions users between tenants you own; for multi-tenant organisations |
| **Multitenant organisation** | Groups your tenants for a unified people search and collaboration experience |
| **External collaboration settings** | Who can invite guests, guest permission level, allowed and blocked domains |

**Governing guests**

- Restrict who can invite (not every user).
- Use **entitlement management** so guest access is requested, approved and time-limited, and the guest is removed when the last access package expires.
- Assign a **sponsor** accountable for each guest.
- Run **access reviews** on guests quarterly.
- Apply **Conditional Access**: MFA for all guests, and block guests from administrative portals.

### External tenants (CIAM)

A separate tenant dedicated to your application's customers.

**Building blocks**

| Component | Purpose |
|---|---|
| **User flows** | The sign-up and sign-in experience: which identity providers, which attributes to collect, which pages |
| **Identity providers** | Email with password or one-time passcode, social accounts, custom OIDC/SAML federation |
| **Custom attributes** | Extra profile fields collected at sign-up |
| **Company branding** | Logos, colours, layout, text, per language |
| **Custom URL domain** | `login.contoso.com` instead of `contoso.ciamlogin.com` |
| **Custom authentication extensions** | Call your own API at defined points: validate sign-up data, enrich tokens with claims, customise OTP delivery |
| **Native authentication** | SDK-driven sign-in screens built natively in your mobile or desktop app, with no browser redirect |
| **MFA** | Email OTP, SMS, authenticator apps; applied through Conditional Access |
| **Sign-in and sign-up analytics** | Usage and funnel insights |

Applications integrate exactly as with any Entra tenant: register the app, use MSAL, and use the authorisation code flow with PKCE. The authority is `https://<tenant>.ciamlogin.com/`.

**Pricing** is based on monthly active users (MAU), with a free tier covering the first 50,000 MAU. Add-ons such as SMS are charged separately.

### Azure AD B2C: the legacy path

Azure AD B2C is the earlier CIAM product.

- **No longer sold to new customers** since 1 May 2025.
- Existing customers can keep using it; Microsoft has committed to support until at least **May 2030**.
- The P2 tier has been discontinued.

For new work use an External ID external tenant. For existing B2C estates, plan a migration: B2C's custom policies (the Identity Experience Framework XML) do not carry over directly, and their logic is typically re-implemented with user flows plus custom authentication extensions. Confirm feature parity for anything unusual before committing dates.

## How it works on Azure

### Choosing the model

```text
Are the users employees?
 └─ Yes ─► Workforce tenant, member users

Are they partners or suppliers collaborating on your internal resources
(Teams, SharePoint, internal apps, Azure)?
 └─ Yes ─► Workforce tenant, B2B collaboration (guests)

Are they consumers or customers of an app you publish?
 └─ Yes ─► External tenant (CIAM)

Is this another tenant your own organisation owns?
 └─ Yes ─► Cross-tenant synchronisation / multitenant organisation
```

### External identity for AI applications

**Customer-facing assistant.** A public chatbot or agent on your website:

- Users sign in through an external tenant user flow.
- The application calls Foundry with **its own managed identity**. Customers are never given roles on Azure resources.
- Authorisation to customer-specific data (orders, account records) is enforced by **your API** using the customer's ID from the token.
- Rate-limit and budget per user, because a public endpoint in front of a paid model invites abuse.
- Consider requiring sign-in before any model call. Anonymous access to an LLM endpoint is expensive and hard to defend.

**Partner collaboration on an internal AI tool.** Invite partners as guests, assign them to the enterprise application or an access package, and let on-behalf-of flows carry their identity so they see only what they have been granted.

**Guests and Microsoft 365 Copilot.** Copilot respects existing permissions. A guest over-shared on SharePoint will have that reflected in what AI features can surface for them, which is one more reason to review guest access.

**Multi-tenant SaaS.** If you sell an AI application to other organisations, register it as multitenant so each customer signs in with their own Entra tenant, and handle tenant isolation in your data layer (topic 33).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Guests vs separate accounts for partners | Guests. Creating local accounts for external people means managing their passwords and lifecycle |
| Trust partner MFA | Yes for established partners, set per organisation rather than as a default for all |
| Open vs allow-listed invitations | Allow-list partner domains where collaboration is limited to known organisations |
| One external tenant or several | One per brand or distinct customer base. Separate tenants for dev/test and production |
| Social vs local accounts for customers | Offer both where possible; social sign-in raises conversion, local accounts suit privacy-conscious users |
| Passwords vs email OTP for customers | OTP or passkeys remove password storage and reset flows |
| Build CIAM yourself | Don't. Identity is a poor place to save money |

## Hands-on

**B2B (workforce tenant)**

```bash
# Invite a guest (use a personal address you control)
az rest --method post --url "https://graph.microsoft.com/v1.0/invitations" --body '{
  "invitedUserEmailAddress": "you@example.com",
  "inviteRedirectUrl": "https://myapps.microsoft.com",
  "sendInvitationMessage": true
}' --query "{Id:invitedUser.id, Status:status}"

# List guests
az ad user list --filter "userType eq 'Guest'" \
  --query "[].{Name:displayName, Mail:mail, State:externalUserState}" -o table
```

Then in the Entra admin centre:

1. *External Identities → External collaboration settings*: read who can invite and what guests can see.
2. *Cross-tenant access settings*: review the defaults, then add one organisation and look at the inbound trust settings.
3. Redeem the invitation, sign in as the guest, and see what the guest can view in *My Apps*.
4. Delete the guest afterwards.

**External tenant (CIAM)**

1. In the Entra admin centre, create an **external** tenant (a free trial is available).
2. Follow the built-in *Get started* guide: create a sign-up and sign-in user flow with email one-time passcode, add a custom attribute, customise branding.
3. Use the guide's *Run it now* sample to sign up as a customer.
4. Inspect the resulting user in *Users* and the sign-in in *Sign-in logs*.

## Security, cost and reliability notes

- **Security:** guests are a common path for data leakage. Sponsor, expire and review them.
- **Security:** apply bot and fraud protection to customer sign-up (WAF, CAPTCHA, rate limits). Fake sign-ups drive SMS cost and abuse.
- **Security:** customer data in an external tenant is personal data. Plan consent, retention and deletion.
- **Cost:** MAU-based. SMS-based MFA has per-message cost that varies widely by country and is a known fraud target (international revenue share fraud). Prefer email OTP or authenticator methods.
- **Reliability:** your application's sign-in depends on the identity service and any federated providers. Offer more than one sign-in method.

## Common mistakes

- Creating consumer accounts in the corporate workforce tenant.
- Starting a new project on Azure AD B2C.
- Giving guests Azure roles directly and never reviewing them.
- Letting any employee invite anyone.
- Not trusting partner MFA and then wondering why guests are prompted twice.
- Leaving an AI endpoint reachable without sign-in.
- Assigning external customers RBAC roles on Foundry resources instead of fronting the model with an application.

## Check yourself

1. When do you use a workforce tenant with guests, and when an external tenant?
2. Where does a B2B guest authenticate?
3. What do cross-tenant access trust settings allow?
4. What is the status of Azure AD B2C?
5. In a customer-facing AI assistant, whose identity calls the model?
6. Name two controls that keep guest access from accumulating.
7. What is a custom authentication extension?

<details>
<summary>Answers</summary>

1. Workforce tenant with guests for partners collaborating on internal resources. External tenant for customers of applications you publish.
2. In their home identity provider (their own Entra tenant, Microsoft account, Google, or by email one-time passcode).
3. Accepting the partner tenant's MFA and device compliance claims so guests are not challenged again and device policies can apply.
4. End of sale to new customers since 1 May 2025; supported for existing customers until at least May 2030; replaced by External ID external tenants.
5. The application's managed identity. Customers authenticate to the app, not to Azure resources.
6. Any two of: entitlement management with expiry, sponsors, access reviews, restricted invitation rights.
7. A hook that calls your own API during a sign-up or sign-in flow to validate input, add claims or customise behaviour.

</details>

## References

- [Introduction to Microsoft Entra External ID](https://learn.microsoft.com/entra/external-id/external-identities-overview)
- [Workforce and external tenant configurations](https://learn.microsoft.com/entra/external-id/tenant-configurations)
- [B2B collaboration overview](https://learn.microsoft.com/entra/external-id/what-is-b2b)
- [Cross-tenant access overview](https://learn.microsoft.com/entra/external-id/cross-tenant-access-overview)
- [Microsoft Entra External ID in external tenants](https://learn.microsoft.com/entra/external-id/customers/overview-customers-ciam)
- [Custom authentication extensions overview](https://learn.microsoft.com/entra/identity-platform/custom-extension-overview)
- [Native authentication](https://learn.microsoft.com/entra/identity-platform/concept-native-authentication)
- [Azure AD B2C FAQ (end of sale)](https://learn.microsoft.com/azure/active-directory-b2c/faq)
