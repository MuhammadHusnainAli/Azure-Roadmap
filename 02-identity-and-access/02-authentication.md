# Authentication

## Why it matters

Stolen and phished credentials remain the most common way into cloud environments. Authentication design decides whether an attacker with a password gets in. It is also the first thing a security-conscious client will ask about, and the first thing an AI project inherits: an assistant that can read company data is only as safe as the sign-in in front of it.

## Key concepts

### Authentication vs authorisation

- **Authentication (AuthN):** proving who you are. Entra ID does this.
- **Authorisation (AuthZ):** deciding what you may do. Azure RBAC, app roles and the resource do this.

### Factors

| Factor | Examples |
|---|---|
| Something you know | Password, PIN |
| Something you have | Phone, security key, certificate |
| Something you are | Fingerprint, face |

Multifactor authentication (MFA) requires two or more different factors.

### Authentication methods

| Method | Type | Phishing-resistant | Notes |
|---|---|:-:|---|
| **Passkey (FIDO2)**, device-bound | Passwordless | Yes | Hardware security key, or a passkey in Microsoft Authenticator |
| **Passkey**, synced | Passwordless | Yes | Stored in a provider such as iCloud Keychain or Google Password Manager; easier recovery, weaker device assurance |
| **Windows Hello for Business** | Passwordless | Yes | Biometric or PIN bound to the device TPM |
| **Platform Credential for macOS** | Passwordless | Yes | Secure Enclave-backed |
| **Certificate-based authentication** | Passwordless | Yes | Smart cards, PIV/CAC; common in government |
| **Microsoft Authenticator** push with number matching | MFA | No | Good, but vulnerable to real-time phishing proxies |
| **Authenticator passwordless phone sign-in** | Passwordless | No | |
| **OATH TOTP** (software or hardware token) | MFA | No | |
| **Temporary Access Pass** | Bootstrap | n/a | Time-limited code to register a first strong credential or recover |
| **SMS / voice call** | MFA | No | Weakest; subject to SIM swap and interception. Avoid |
| **Password** | Single factor | No | Never sufficient alone |
| **External authentication method** | MFA | Depends | Third-party MFA provider integrated with Entra ID |

**Phishing-resistant** means the credential is cryptographically bound to the legitimate site, so a fake sign-in page cannot capture and replay it. This is the target state for all users and the minimum for administrators.

### Authentication strengths

A Conditional Access control that specifies *which* methods satisfy a policy, not merely "MFA". Three are built in:

| Strength | Accepts |
|---|---|
| **Multifactor authentication** | Any MFA combination |
| **Passwordless MFA** | Authenticator phone sign-in, passkeys, Windows Hello, CBA |
| **Phishing-resistant MFA** | Passkeys, Windows Hello for Business, CBA |

You can define custom strengths, for example to require a specific security key model. Typical use: require phishing-resistant MFA for administrators and for access to the Azure portal.

### Passwords, where they remain

- **Password protection:** blocks known-weak and custom-banned passwords, on-premises too through an agent.
- **Smart lockout:** throttles attackers without locking out the genuine user.
- **No forced periodic expiry:** current guidance is to change passwords on evidence of compromise, not on a calendar.
- **Self-service password reset (SSPR):** users reset their own password after verifying with registered methods. With **password writeback**, the change flows to on-premises AD.

### Combined registration and the methods policy

Users register MFA and SSPR methods once at `aka.ms/mysecurityinfo`. Administrators control which methods are allowed, and for which groups, in the **Authentication methods policy**. The older per-user MFA and legacy SSPR policies have been retired in favour of this single policy.

A **registration campaign** nudges users toward stronger methods such as passkeys in Authenticator.

### Tokens

Entra ID issues JSON Web Tokens (JWTs).

| Token | Audience | Purpose | Typical lifetime |
|---|---|---|---|
| **ID token** | The client application | Tells the app who signed in | 1 hour |
| **Access token** | A resource API | Authorises calls to that API | 60 to 90 minutes |
| **Refresh token** | Entra ID | Gets new access tokens silently | Up to 90 days, sliding |
| **Primary Refresh Token (PRT)** | The device | Device-wide SSO on joined or registered devices | 90 days, renewed |

Important claims in an access token:

| Claim | Meaning |
|---|---|
| `aud` | The API the token is for. An API must reject tokens with the wrong audience |
| `iss` | Issuer (the tenant) |
| `tid` | Tenant ID |
| `oid` | Object ID of the principal |
| `sub` | Subject, unique per app |
| `scp` | Delegated scopes (user present) |
| `roles` | App roles or application permissions |
| `amr` / `acr` | How the user authenticated |
| `exp` | Expiry |

Access tokens are bearer tokens: whoever holds one can use it until it expires. That makes token theft a real attack, and it is why the following two features exist.

### Continuous Access Evaluation

Supporting services (Exchange, SharePoint, Teams, Microsoft Graph, Azure Resource Manager) subscribe to critical events such as account disabled, password changed, MFA enabled, or a network location change. They reject the token within minutes instead of waiting for expiry. In exchange, CAE-capable clients receive longer-lived tokens.

### Token Protection

A Conditional Access session control that binds the sign-in session token to the device, so a stolen token cannot be replayed from another machine. Coverage is expanding by platform and application; check current support before depending on it.

### Single sign-on

One authentication gives access to many applications. Mechanisms:

- **OpenID Connect / OAuth 2.0:** modern applications.
- **SAML 2.0:** many enterprise SaaS applications.
- **Password-based and linked SSO:** legacy fallbacks.
- **Application Proxy / Entra Private Access:** publish on-premises web apps with Entra pre-authentication.
- **PRT:** seamless SSO from joined devices.

## How it works on Azure

### Mandatory MFA for Azure management

Microsoft enforces MFA at the platform level in two phases:

| Phase | Applies to | Operations |
|---|---|---|
| 1 | Azure portal, Entra admin centre, Intune admin centre, Microsoft 365 admin centre | Any sign-in |
| 2 | Azure CLI, Azure PowerShell, mobile app, IaC tools, SDKs, REST API to `management.azure.com` | Create, update, delete. Reads are not affected |

Implications:

- **All user accounts** are in scope, including break-glass accounts and guests. Give break-glass accounts a passkey or certificate.
- **Managed identities and service principals are exempt.**
- User accounts used as service accounts will break. Migrate them to workload identities.
- The **ROPC** flow (username and password sent by the app) is incompatible with MFA. `UsernamePasswordCredential` in the Azure Identity libraries is deprecated.

### A sensible target state

| Population | Requirement |
|---|---|
| Administrators | Phishing-resistant MFA, on a compliant device |
| All employees | MFA at minimum, moving to passkeys |
| Guests | MFA, trusted from their home tenant where appropriate |
| Workloads | Managed identity or federated credential, no passwords |
| Legacy authentication | Blocked for everyone |

## Design decisions and trade-offs

- **Device-bound vs synced passkeys.** Device-bound gives the strongest assurance and suits administrators and regulated roles. Synced passkeys are far easier to roll out and recover, and suit the general workforce.
- **Passwordless roll-out needs a recovery story.** Decide how a user who loses their only credential gets back in: Temporary Access Pass issued by the helpdesk, ideally backed by identity verification (see file 11).
- **Cloud authentication vs federation.** Authenticating in Entra ID directly is simpler and more resilient than federating to AD FS or another IdP. Federate only for a concrete requirement.
- **Session lifetime.** Short sign-in frequency increases prompts and trains users to approve blindly. Prefer risk-based reauthentication and CAE over aggressive timeouts.
- **Frictions cost adoption.** For an internal AI assistant, SSO with the user's existing session is the difference between daily use and abandonment.

## Hands-on

1. Open `aka.ms/mysecurityinfo` and review your registered methods. Add a passkey if your tenant allows it.
2. In the Entra admin centre, open **Protection → Authentication methods → Policies** and note which methods are enabled and for whom.
3. Open **Authentication strengths** and read the three built-in definitions.
4. Inspect a real token:

   ```bash
   # Get an ARM access token and decode its payload (do not paste tokens into websites)
   az account get-access-token --resource https://management.azure.com --query accessToken -o tsv \
     | cut -d. -f2 | tr '_-' '/+' | awk '{l=length($0)%4; if(l==2)$0=$0"=="; else if(l==3)$0=$0"="; print}' \
     | base64 -d | python3 -m json.tool
   ```

   Find `aud`, `tid`, `oid`, `amr` and `exp`. Note that `amr` shows whether you used MFA.
5. Request a token for a different audience and compare `aud`:

   ```bash
   az account get-access-token --resource https://cognitiveservices.azure.com --query expiresOn
   ```

6. In **Sign-in logs**, open your latest sign-in and read the *Authentication Details* tab.

## Security, cost and reliability notes

- **Security:** block legacy authentication (IMAP, POP, SMTP basic auth, older Office clients). It cannot do MFA.
- **Security:** treat tokens as secrets. Never log them or paste them into online decoders.
- **Security:** monitor for MFA fatigue (repeated denied prompts) and enable number matching, which is now the default for Authenticator push.
- **Cost:** hardware security keys have a per-user cost; passkeys in Authenticator and synced passkeys do not.
- **Reliability:** make sure every user has at least two usable methods, and that the helpdesk can issue a Temporary Access Pass.

## Common mistakes

- Calling SMS codes "MFA" and stopping there.
- Enforcing MFA for users but excluding administrators "for convenience".
- Forgetting that break-glass accounts now also need MFA for Azure management.
- Validating a JWT's signature but not its `aud` and `iss` in a custom API.
- Using ROPC or stored user passwords in scripts.
- Rolling out passwordless without a recovery process.
- Setting sign-in frequency to an hour for everyone.

## Check yourself

1. What makes a method phishing-resistant? Name three such methods.
2. What is the difference between an ID token and an access token?
3. What does Continuous Access Evaluation change about token validity?
4. A script that signs in with a username and password starts failing on `az group create` but `az group list` still works. Why?
5. What is an authentication strength, and where is it used?
6. What is a Temporary Access Pass for?

<details>
<summary>Answers</summary>

1. The credential is cryptographically bound to the genuine sign-in origin, so a proxy or fake page cannot capture and replay it. Passkeys (FIDO2), Windows Hello for Business, certificate-based authentication.
2. An ID token tells the client application who the user is. An access token is presented to an API to authorise requests and carries the API's audience.
3. Supporting services can reject a token within minutes of critical events (account disabled, password reset, location change) instead of honouring it until expiry.
4. Phase 2 mandatory MFA applies to create, update and delete operations against Azure Resource Manager for user accounts; reads are not affected. The script should use a workload identity.
5. A Conditional Access grant control that specifies which combinations of methods are acceptable, such as phishing-resistant MFA only.
6. A time-limited passcode that lets a user register their first strong credential or recover access without a password.

</details>

## References

- [Authentication methods in Microsoft Entra ID](https://learn.microsoft.com/entra/identity/authentication/concept-authentication-methods)
- [Passkeys (FIDO2) in Microsoft Entra ID](https://learn.microsoft.com/entra/identity/authentication/concept-authentication-passwordless)
- [Conditional Access authentication strength](https://learn.microsoft.com/entra/identity/authentication/concept-authentication-strengths)
- [Plan a phishing-resistant passwordless deployment](https://learn.microsoft.com/entra/identity/authentication/how-to-plan-prerequisites-phishing-resistant-passwordless-authentication)
- [Tokens and claims overview](https://learn.microsoft.com/entra/identity-platform/security-tokens)
- [Continuous access evaluation](https://learn.microsoft.com/entra/identity/conditional-access/concept-continuous-access-evaluation)
- [Token Protection in Conditional Access](https://learn.microsoft.com/entra/identity/conditional-access/concept-token-protection)
- [Plan for mandatory multifactor authentication for Azure](https://learn.microsoft.com/entra/identity/authentication/concept-mandatory-multifactor-authentication)
