# App Registrations and Service Principals

> Topic: 02 · Identity and Access · Tracks: E ● A ● C ○ · Status checked: 2026-10

## Why it matters

The moment your application needs users to sign in, or needs to call an API on a user's behalf, you are working with the Microsoft identity platform. For AI this is not optional detail: a RAG assistant that must return only documents the signed-in user is allowed to see, or an agent that acts with the user's permissions, depends entirely on getting OAuth flows right.

## Key concepts

### Two objects, one application

| Object | What it is | Where it lives | Portal location |
|---|---|---|---|
| **Application object** (app registration) | The global definition of the app: IDs, redirect URIs, credentials, permissions it requests, APIs it exposes | The home tenant only | *App registrations* |
| **Service principal** (enterprise application) | The local instance of the app in a tenant: what it has been granted, who is assigned | Every tenant that uses the app | *Enterprise applications* |

Think of the application object as a class and the service principal as an instance. A multitenant SaaS app has one application object and one service principal per customer tenant.

IDs you will meet:

| ID | Belongs to | Used for |
|---|---|---|
| Application (client) ID | App registration; same in every tenant | `client_id` in auth requests |
| Object ID (application) | App registration | Managing the registration through Graph |
| Object ID (service principal) | Enterprise application; differs per tenant | **Role assignments** |
| Directory (tenant) ID | The tenant | Authority URL |

Azure RBAC assignments go to the **service principal object ID**. Using the app registration's object ID is one of the most common errors.

### Account types

| Setting | Who can sign in |
|---|---|
| Single tenant | Accounts in your tenant only. The default for internal apps |
| Multitenant | Accounts in any Entra tenant |
| Multitenant + personal | Any Entra tenant and personal Microsoft accounts |
| Personal only | Personal Microsoft accounts |

### Platform types and redirect URIs

| Platform | Client type | Can keep a secret |
|---|---|:-:|
| Web | Confidential | Yes |
| Single-page application | Public | No |
| Mobile and desktop | Public | No |

Tokens are returned only to registered **redirect URIs**. Keep the list tight; a stray wildcard or leftover localhost entry is an open door.

### Credentials for confidential clients

In order of preference:

1. **Federated identity credential** or **managed identity**: nothing stored.
2. **Certificate**: private key stays in Key Vault or an HSM.
3. **Client secret**: a string. Easiest to leak, hardest to track. Maximum lifetime is limited, and tenants can restrict secrets entirely with app management policies.

### Permissions

| | **Delegated** | **Application** |
|---|---|---|
| Acts as | The signed-in user | The app itself |
| User present | Yes | No |
| Effective access | Intersection of what the app was granted and what the user can do | Whatever the app was granted, across the tenant |
| Token claim | `scp` | `roles` |
| Consent | User or admin, depending on the permission | Admin only |
| Example | `Files.Read` reads the user's files | `Files.Read.All` reads every user's files |

Prefer delegated permissions whenever a user is involved. An application permission such as `Sites.Read.All` or `Mail.Read` gives one credential access to everyone's data. For SharePoint, `Sites.Selected` lets you grant an app access to specific sites instead.

### Consent

- **User consent:** a user approves low-risk delegated permissions for themselves.
- **Admin consent:** an administrator approves on behalf of the organisation; required for application permissions and high-impact delegated ones.
- **Admin consent workflow:** users request, designated reviewers approve.

Illicit consent grants, where a user is tricked into approving a malicious app, are a real attack. Restrict user consent to verified publishers and low-risk permissions.

### OAuth 2.0 flows

| Flow | Use for | Notes |
|---|---|---|
| **Authorisation code + PKCE** | Web apps, SPAs, mobile and desktop | The standard interactive flow |
| **Client credentials** | Daemons and services with no user | Application permissions only |
| **On-behalf-of (OBO)** | A middle-tier API calling another API as the user | Preserves the user's identity down the chain |
| **Device code** | CLIs and devices without a browser | Phishable; block where not needed |
| **Managed identity / federated credential** | Azure-hosted and federated workloads | Preferred over client credentials with a secret |
| ~~Implicit~~ | Legacy SPAs | Replaced by auth code + PKCE |
| ~~ROPC~~ | App collects username and password | Incompatible with MFA; do not use |

### On-behalf-of in an AI application

This is the flow that makes permission-aware AI possible.

```text
User ──(1) sign in──► Web app
Web app ──(2) access token for API A──► Orchestrator API
Orchestrator API ──(3) OBO: exchange token for one targeting API B──► Entra ID
Orchestrator API ──(4) token for API B, still representing the user──► Search / Graph / data API
```

Downstream services see the **user**, so their normal authorisation applies:

- Azure AI Search can trim results to documents the user may access.
- Microsoft Graph returns only the user's own mail and files.
- A line-of-business API applies its own per-user rules.

The alternative, where the application's own identity fetches everything and the app filters afterwards, means one bug or prompt injection can expose data the user should never see.

Rule of thumb:

| Access to | Use |
|---|---|
| The model endpoint | The application's managed identity |
| User-specific data (documents, mail, records) | The user's identity through OBO |
| Shared reference data | The application's managed identity |

### Exposing your own API

To protect an API you build:

1. Register it and set an **Application ID URI** (for example `api://<client-id>`).
2. Define **scopes** (delegated, such as `Documents.Read`) and/or **app roles** (for applications or users, such as `Documents.Reader`).
3. In the client registration, add the API permission and grant consent.
4. In the API, validate each token: signature, issuer, **audience**, expiry, then check `scp` or `roles`.

**App roles** also work for users and groups: assign people to roles in *Enterprise applications*, and the `roles` claim arrives in their token. This is the clean way to do in-app authorisation without calling Graph for group membership.

**Group claims** can be emitted too, but tokens have a size limit; above roughly 200 groups you get an overage indicator and must query Graph. Prefer app roles, or filter to groups assigned to the application.

### MSAL

The Microsoft Authentication Library handles token acquisition, caching and refresh for you.

| Client | Class |
|---|---|
| Public (SPA, desktop, mobile) | `PublicClientApplication` |
| Confidential (web app, API, daemon) | `ConfidentialClientApplication` |

Always try `acquire_token_silent` first and fall back to an interactive or credential-based call. For server apps, higher-level libraries (Microsoft.Identity.Web for .NET, `msal` plus a framework integration for Python and Node) do most of the work.

### Built-in authentication ("Easy Auth")

App Service, Functions and Container Apps can handle sign-in at the platform level, before requests reach your code. The user's identity and tokens arrive in headers. It is the fastest way to put Entra sign-in in front of an internal tool or prototype and it supports the token store needed for OBO.

## How it works on Azure

### Protecting agents and MCP servers

The same building blocks apply to AI components:

- A **remote MCP server** is an API. Protect it with Entra ID: register it, expose scopes, validate tokens. Clients obtain tokens for it through the standard authorisation code flow.
- **API Management** can validate tokens and apply policy in front of model endpoints and MCP servers.
- **Foundry agents** calling tools on a user's behalf use OBO-style delegated access; agents acting autonomously use their own identity (see [Microsoft Entra Agent ID](10-microsoft-entra-agent-id.md)).

### Managing applications

- **Owners:** every registration should have at least two owners who are people still at the company.
- **Enterprise application settings:** *Assignment required* restricts sign-in to assigned users and groups. Turn it on for internal apps.
- **App management policies:** tenant-wide restrictions on secrets, lifetimes and identifier URIs.
- **Recommendations:** Entra flags unused applications and expiring or unused credentials.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Single vs multitenant | Single unless you are building for other organisations |
| Delegated vs application | Delegated when a user is present. Application permissions only for true background jobs, scoped as narrowly as the API allows |
| OBO vs app identity for data | OBO for per-user data. It is more work and it is the correct model |
| App roles vs groups | App roles: self-contained, small tokens, managed per app |
| One registration or several | Separate registrations for the front end and each API. Separate registrations per environment |
| Easy Auth vs library | Easy Auth for simple gating; a library when you need fine control over flows and token handling |

## Hands-on

```bash
# 1. Register an app and create its service principal
az ad app create --display-name app-roadmap-lab --sign-in-audience AzureADMyOrg
APP_ID=$(az ad app list --display-name app-roadmap-lab --query "[0].appId" -o tsv)
APP_OBJ=$(az ad app show --id $APP_ID --query id -o tsv)
az ad sp create --id $APP_ID
SP_OBJ=$(az ad sp show --id $APP_ID --query id -o tsv)

echo "Client ID:            $APP_ID"
echo "App object ID:        $APP_OBJ"
echo "Service principal ID: $SP_OBJ   <-- use this for role assignments"

# 2. Client credentials flow with a short-lived secret (lab only)
SECRET=$(az ad app credential reset --id $APP_ID --display-name lab --end-date "$(date -u -d '+1 day' +%Y-%m-%d)" --query password -o tsv)
TENANT=$(az account show --query tenantId -o tsv)

curl -s -X POST "https://login.microsoftonline.com/$TENANT/oauth2/v2.0/token" \
  -d "client_id=$APP_ID" -d "client_secret=$SECRET" \
  -d "scope=https://management.azure.com/.default" -d "grant_type=client_credentials" \
  | python3 -c "import sys,json,base64; t=json.load(sys.stdin)['access_token'].split('.')[1]; t+='='*(-len(t)%4); print(json.dumps(json.loads(base64.urlsafe_b64decode(t)),indent=2))" \
  | grep -E '"aud"|"appid"|"oid"|"idtyp"|"roles"'

# 3. Expose an API scope and an app role (inspect in the portal afterwards)
az ad app update --id $APP_ID --identifier-uris "api://$APP_ID"

# 4. Clean up
az ad app delete --id $APP_ID
```

Note in step 2 that `oid` matches the service principal object ID and there is no `scp` claim: this is an app-only token.

Then in the portal: open the app registration and walk through *Authentication*, *Certificates & secrets*, *API permissions*, *Expose an API* and *App roles*; then find the same app under *Enterprise applications* and compare.

## Security, cost and reliability notes

- **Security:** audit application permissions regularly. A forgotten app with `Mail.ReadWrite.All` and a never-expiring secret is a breach waiting to happen.
- **Security:** validate `aud` and `iss`, not just the signature. Accepting any Microsoft-issued token is a serious and common bug.
- **Security:** block device code flow with Conditional Access where it is not needed.
- **Security:** never put secrets in SPAs or mobile apps. They are public clients.
- **Reliability:** secrets and certificates expire. Alert before expiry, or better, remove them with federation.
- **Cost:** app registrations are free. Premium protection for workload identities is licensed separately.

## Common mistakes

- Assigning a role to the wrong ID.
- Requesting application permissions because delegated ones were harder to set up.
- Retrieving all data with the app identity and filtering in code or, worse, in the prompt.
- Leaving `http://localhost` and broad redirect URIs on a production registration.
- One registration shared by dev, test and production.
- Calling a downstream API with the incoming token instead of exchanging it through OBO, which fails on audience.
- Building group-based authorisation and hitting the token overage limit.

## Check yourself

1. What is the difference between an app registration and an enterprise application?
2. Which ID do you use in an Azure role assignment for an application?
3. Delegated vs application permissions: which claim carries each, and who can consent?
4. Which flow should a browser-based SPA use?
5. Why is on-behalf-of important for a RAG application over user documents?
6. What must an API check when validating an access token?
7. Why prefer app roles over group claims for in-app authorisation?

<details>
<summary>Answers</summary>

1. The app registration is the global definition in the home tenant. The enterprise application is the service principal, the local instance in each tenant that holds grants and assignments.
2. The service principal's object ID.
3. Delegated: `scp`, consent by user or admin. Application: `roles`, admin consent only.
4. Authorisation code with PKCE.
5. It carries the user's identity to downstream services so they return only what that user is permitted to see, instead of relying on the app to filter.
6. Signature, issuer, audience, expiry, and the required scope or role.
7. They are defined and managed per application, keep tokens small, and avoid the group overage problem.

</details>

## References

- [Application and service principal objects in Microsoft Entra ID](https://learn.microsoft.com/entra/identity-platform/app-objects-and-service-principals)
- [Microsoft identity platform authentication flows](https://learn.microsoft.com/entra/identity-platform/authentication-flows-app-scenarios)
- [OAuth 2.0 on-behalf-of flow](https://learn.microsoft.com/entra/identity-platform/v2-oauth2-on-behalf-of-flow)
- [Permissions and consent overview](https://learn.microsoft.com/entra/identity-platform/permissions-consent-overview)
- [Add app roles and receive them in the token](https://learn.microsoft.com/entra/identity-platform/howto-add-app-roles-in-apps)
- [Overview of the Microsoft Authentication Library](https://learn.microsoft.com/entra/identity-platform/msal-overview)
- [Authentication and authorization in Azure App Service](https://learn.microsoft.com/azure/app-service/overview-authentication-authorization)
- [Security best practices for application properties](https://learn.microsoft.com/entra/identity-platform/security-best-practices-for-app-registration)
