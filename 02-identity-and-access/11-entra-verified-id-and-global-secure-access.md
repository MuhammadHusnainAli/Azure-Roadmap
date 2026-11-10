# Entra Verified ID and Global Secure Access

## Why it matters

Two parts of the Entra family extend identity beyond sign-in. **Verified ID** answers "is this really the person, and is this claim about them true?" **Global Secure Access** answers "how do identities reach applications and the internet without a flat, trusted network?" Neither is day-one material for an AI engineer. Both come up in architecture and security conversations, and Global Secure Access now includes controls aimed specifically at how employees and agents use AI services.

## Part 1 · Microsoft Entra Verified ID

### Key concepts

A **verifiable credential** is a digitally signed statement about a person, issued by an organisation and held by the person in a digital wallet. It is built on open standards (W3C Verifiable Credentials and decentralised identifiers, DIDs).

| Role | Does | Example |
|---|---|---|
| **Issuer** | Creates and signs the credential | An employer issues "verified employee"; a university issues a degree |
| **Holder** | Stores it in a wallet (Microsoft Authenticator) and chooses when to present it | The employee |
| **Verifier** | Requests a credential and checks its signature and status | A helpdesk, a partner portal, an onboarding app |

```text
Issuer ──issues credential──► Holder's wallet ──presents──► Verifier
   ▲                                                          │
   └──────── verifier checks signature and revocation ────────┘
```

What makes it different from an ordinary sign-in:

- The verifier does not need a federation relationship with the issuer.
- The holder decides what to share.
- Credentials can be revoked or suspended by the issuer.
- Presentation is cryptographically verifiable.

**Face Check** adds a privacy-respecting biometric match: the user's live selfie is compared with the photo in the credential, and only a match confidence score is returned to the verifier. It raises assurance that the person presenting the credential is its owner.

**Identity verification partners** (document and liveness proofing providers) can issue a credential after checking a government ID, giving you a "verified person" credential without building proofing yourself.

### Use cases

| Scenario | How Verified ID helps |
|---|---|
| **Remote onboarding** | New hire proves identity with a partner-issued credential, then receives a Temporary Access Pass to register a passkey |
| **Account recovery and helpdesk** | Before resetting credentials, the user presents a verified employee credential with Face Check. A direct defence against social-engineering and deepfake-driven helpdesk attacks |
| **High-assurance access** | Entitlement management can require a verified credential before granting an access package |
| **Partner and workforce verification** | A contractor proves employment with their employer without you federating with that company |
| **Customer scenarios** | Proof of age, membership or qualification without over-sharing personal data |

The helpdesk case deserves emphasis. As voice and video impersonation gets easier, "call the helpdesk and sound convincing" is a serious attack path. Verification that does not rely on a human judging a voice is a strong control.

### How it works on Azure

- Set up through the Entra admin centre (a quick setup creates the issuer configuration and a ready-made **Verified Employee** credential).
- Issuance and verification are driven by the **Request Service API**; your app displays a QR code or deep link.
- Keys are managed in Key Vault.
- Core issuance and verification are included with Entra ID. Face Check is a premium feature, included in the Entra Suite or billed per verification.

## Part 2 · Global Secure Access

### Key concepts

Global Secure Access is Microsoft's **Security Service Edge (SSE)**: network security delivered from the cloud and driven by identity. It unifies two products.

| Product | Replaces or augments | What it does |
|---|---|---|
| **Microsoft Entra Private Access** | VPN | Zero Trust Network Access to private applications and resources, per app, with Conditional Access |
| **Microsoft Entra Internet Access** | Secure web gateway and proxy | Identity-aware protection for internet and SaaS traffic, including Microsoft 365 |

**Why it exists.** A traditional VPN authenticates once and puts the device on the network, where it can reach far more than it needs. Zero Trust Network Access grants access to one application at a time, evaluated continuously against identity, device and risk.

### Components

| Component | Role |
|---|---|
| **Global Secure Access client** | Agent on Windows, macOS, iOS and Android that acquires and tunnels traffic to the service |
| **Traffic forwarding profiles** | Which traffic is captured: **Microsoft** traffic, **Private access**, **Internet access** |
| **Private network connectors** | Lightweight outbound-only agents in your network (on-premises or in a VNet) that reach private apps. Shared with application proxy |
| **Remote networks** | Connect a branch office by IPsec tunnel so devices without the client are covered |
| **Enterprise applications / Quick Access** | Define private apps by FQDN, IP range and port |

### Entra Private Access

- Publish any TCP or UDP application, not only web apps: RDP, SSH, SMB, databases, legacy client-server.
- **Per-app access** with Conditional Access per application: require MFA, a compliant device, or a risk check to open an RDP session.
- **Quick Access** for a fast VPN replacement; **per-app segments** for least privilege.
- **Private DNS** resolves internal names.
- Extends modern authentication controls to **domain controllers** and legacy protocols, for example requiring MFA before Kerberos access to sensitive resources.
- Complements, and does not replace, Azure networking. Private endpoints and VNets still isolate the resources; Private Access is how users reach them.

### Entra Internet Access

- **Web content filtering** by category and FQDN, applied through Conditional Access so policy can differ by user, group, device and risk.
- **TLS inspection** to see inside encrypted traffic.
- **Threat intelligence** filtering of malicious destinations.
- **Universal tenant restrictions:** stop users signing in to unapproved external tenants and exfiltrating data to them.
- **Compliant network check:** a Conditional Access condition that requires traffic to arrive through Global Secure Access, which blocks stolen-token replay from elsewhere.
- **Source IP restoration:** the user's real IP is preserved in logs and for policy.
- **Microsoft 365 traffic** receives optimised handling and enriched logs.

### Controls for AI usage

Because all web traffic passes through an identity-aware gateway, it becomes a place to govern AI use:

| Control | Purpose |
|---|---|
| **Discover and filter AI applications** | See which generative AI services employees use; allow sanctioned ones and block others by category |
| **Inspect prompts and uploads** | With TLS inspection and data protection integration (Microsoft Purview), detect and block sensitive content being sent to AI services |
| **Prompt injection protection** | Network-level inspection of traffic to AI applications to block malicious prompts before they reach the model, complementing protections in the application |
| **Agent traffic** | Apply network security policy to traffic from agents, including calls to tools and MCP servers |

These capabilities are recent and expanding; some are in preview. Verify current status and supported applications. They complement, and do not replace, application-level controls such as Azure AI Content Safety and Purview's AI-specific policies (topic 23).

### Licensing

Private Access and Internet Access are licensed per user, individually or through the **Microsoft Entra Suite**. Handling of Microsoft traffic is included with Entra ID P1.

## How it works on Azure

### Where Global Secure Access sits

```text
User device ──GSA client──► Microsoft's global edge ──► Internet / SaaS   (Internet Access)
                                   │
                                   └──► Private network connector ──► Private apps in VNets / on-premises
                                                                      (Private Access)
                 Conditional Access evaluated per application, continuously
```

### Admin access to private AI environments

A network-isolated AI landing zone has no public endpoints. Developers and operators still need to reach the Foundry portal's private resources, jump hosts and data stores. Options:

| Option | Fit |
|---|---|
| **Azure Bastion + jump box** | Simple, well understood, good for small teams |
| **Point-to-site VPN** | Broad network access; less granular |
| **Entra Private Access** | Per-app access with Conditional Access; no network-level trust |
| **Windows 365 or Azure Virtual Desktop in the VNet** | Developer workstations already inside the boundary (Dev Box is in maintenance mode) |

Topic 05 covers the networking side.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| VPN vs Private Access | Private Access for user-to-app. Keep site-to-site VPN or ExpressRoute for network-to-network connectivity |
| Application proxy vs Private Access | Application proxy for a handful of web apps. Private Access for everything else and for a full VPN replacement |
| Internet Access vs existing secure web gateway | Evaluate coverage and maturity for your needs. Many organisations run both during transition |
| TLS inspection | Strong visibility; needs certificate deployment, privacy review and careful exclusions (banking, health) |
| Verified ID scope | Start with one high-value flow, usually helpdesk recovery or onboarding |
| Face Check | Use where impersonation risk justifies it; brief users and review privacy obligations first |

## Hands-on

**Verified ID** (Entra admin centre, needs appropriate admin role)

1. *Verified ID → Setup*: run the quick setup.
2. Open `myaccount.microsoft.com`, choose *Get my Verified ID*, and add the Verified Employee credential to Microsoft Authenticator.
3. Use Microsoft's public verification test app (linked from the setup page) to present the credential.
4. Open the credential definition and read the display and rules definitions.

**Global Secure Access** (trial licence required)

1. *Global Secure Access → Get started*: activate the service.
2. *Connect → Traffic forwarding*: enable the Microsoft traffic profile for a test group.
3. Install the client on a test Windows device and confirm it connects.
4. *Secure → Web content filtering policies*: create a policy blocking one category and attach it to a security profile linked from a Conditional Access policy.
5. *Monitor → Traffic logs*: find your own traffic.
6. (Optional) Install a private network connector in a VNet and publish a VM's RDP endpoint as a per-app access application with MFA required.

## Security, cost and reliability notes

- **Security:** Verified ID does not replace authentication. It strengthens the moments around it: onboarding, recovery and step-up.
- **Security:** the compliant network condition is one of the more effective defences against token theft, because a stolen token used from outside the service is rejected.
- **Security:** private network connectors are sensitive infrastructure. Harden and monitor them.
- **Cost:** per-user licensing for Private and Internet Access; Face Check is premium. Weigh against VPN concentrators and proxies being retired.
- **Reliability:** deploy at least two connectors per connector group. Plan for client failure modes and a break-glass access path.
- **Privacy:** inspecting web traffic and using biometrics both need legal and works-council review in many jurisdictions.

## Common mistakes

- Treating Private Access as a substitute for private endpoints and network isolation of resources.
- Publishing an entire subnet through Quick Access and calling it Zero Trust.
- Enabling TLS inspection without communicating it or excluding sensitive categories.
- Assuming network-level AI controls remove the need for application-level safety.
- Rolling out Verified ID broadly before proving one flow.
- Relying on a helpdesk agent's judgement to verify a caller.

## Check yourself

1. Name the three roles in a verifiable credential exchange.
2. How does Verified ID help defend against helpdesk social engineering?
3. What is the difference between Entra Private Access and Entra Internet Access?
4. How does Zero Trust Network Access differ from a traditional VPN?
5. What does the compliant network Conditional Access condition protect against?
6. How can Global Secure Access help govern employees' use of third-party AI services?
7. Does Private Access replace private endpoints? Explain.

<details>
<summary>Answers</summary>

1. Issuer, holder, verifier.
2. The user must present a cryptographically verifiable credential, optionally with Face Check, before credentials are reset, instead of an agent judging a voice or answers to questions.
3. Private Access gives identity-based access to private applications (VPN replacement). Internet Access secures traffic to the internet and SaaS (secure web gateway).
4. A VPN grants network-level access after one authentication. ZTNA grants access per application, evaluated continuously against identity, device and risk.
5. Use of stolen tokens from outside the organisation's secured network path.
6. By discovering which AI applications are used, filtering unsanctioned ones, and inspecting prompts and uploads for sensitive data or malicious content.
7. No. Private endpoints isolate the resource on the network. Private Access is how authorised users reach resources that are already private.

</details>

## References

- [Introduction to Microsoft Entra Verified ID](https://learn.microsoft.com/entra/verified-id/decentralized-identifier-overview)
- [Face Check with Microsoft Entra Verified ID](https://learn.microsoft.com/entra/verified-id/using-facecheck)
- [Verified ID helpdesk and onboarding scenarios](https://learn.microsoft.com/entra/verified-id/plan-verification-solution)
- [What is Global Secure Access?](https://learn.microsoft.com/entra/global-secure-access/overview-what-is-global-secure-access)
- [Microsoft Entra Private Access](https://learn.microsoft.com/entra/global-secure-access/concept-private-access)
- [Microsoft Entra Internet Access](https://learn.microsoft.com/entra/global-secure-access/concept-internet-access)
- [Universal Conditional Access through Global Secure Access](https://learn.microsoft.com/entra/global-secure-access/concept-universal-conditional-access)
- [Microsoft Entra Suite](https://learn.microsoft.com/entra/fundamentals/whats-new-entra-suite)
