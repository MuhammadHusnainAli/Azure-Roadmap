# Static Web Apps

> Topic: 10 · Application Platform · Tracks: E ○ A ○ C – · Status checked: 2026-10

## Why it matters

Most modern front ends (React, Angular, Vue, Svelte, Blazor WebAssembly, static site generators) compile to static files plus calls to APIs. Azure Static Web Apps hosts those files globally, builds them from GitHub or Azure DevOps, adds authentication and routing, links them to an API back end, and creates a preview environment for every pull request. For AI chat front ends, documentation sites and internal tools it is often the simplest and cheapest option.

## Key concepts

### What Static Web Apps provides

| Feature | Detail |
|---|---|
| Global static hosting | Files served from a globally distributed network |
| **CI/CD integration** | GitHub Actions or Azure Pipelines workflow generated for you; builds the front end (and managed API) on every push |
| **Preview environments** | Each pull request gets its own URL; deleted when the PR closes |
| **API back ends** | **Managed functions** (Functions built and hosted with the app) or **linked back ends**: an existing Function app, Container App, App Service or API Management instance |
| **Authentication** | Built-in providers (Microsoft Entra ID, GitHub) and custom OpenID Connect providers (Standard plan) |
| **Authorisation** | Role-based route rules in `staticwebapp.config.json`; role assignment by invitation or a role function |
| **Routing** | Rewrites, redirects, SPA fallback, headers, custom error pages |
| Custom domains | Free TLS certificates |
| **Database connections** | Data API builder integration exposing REST and GraphQL over Azure SQL, PostgreSQL, MySQL and Cosmos DB (check current status) |
| Private endpoint | Standard plan |

### Plans

| | Free | Standard |
|---|---|---|
| Purpose | Personal projects, prototypes | Production |
| Custom domains | Limited | More |
| Custom authentication providers | – | ✓ |
| **Bring your own back end** (linked Functions, Container Apps, App Service, APIM) | – | ✓ |
| Private endpoints | – | ✓ |
| SLA | – | ✓ |
| Staging environments | Fewer | More |

### Request flow

```text
Browser → https://app.contoso.com
   ├─ /                     → static files (index.html, JS, CSS) from the global edge
   ├─ /api/*                → managed functions or linked back end (same origin: no CORS)
   ├─ /.auth/login/aad      → built-in authentication
   └─ route rules           → allowedRoles checks before serving
```

Serving the API under the same origin avoids CORS configuration and lets the platform pass the authenticated user's identity to the API in the `x-ms-client-principal` header.

### Configuration file

```json
{
  "navigationFallback": { "rewrite": "/index.html", "exclude": ["/assets/*", "/api/*"] },
  "routes": [
    { "route": "/admin/*", "allowedRoles": ["admin"] },
    { "route": "/api/*",   "allowedRoles": ["authenticated"] },
    { "route": "/login",   "redirect": "/.auth/login/aad" }
  ],
  "responseOverrides": { "401": { "redirect": "/.auth/login/aad", "statusCode": 302 } },
  "globalHeaders": { "Content-Security-Policy": "default-src 'self'" }
}
```

## How it works on Azure

### An AI chat front end

```text
Static Web Apps (Standard): React chat UI, Entra sign-in
   └─ linked back end: Container Apps "chat-api" (or Function app)
          → Foundry Models / Agent Service with managed identity
          → streaming responses (SSE) or Web PubSub for real-time (file 09)
```

Linked back ends keep the API independently deployable and scalable while the front end gets preview environments and global hosting.

### Create and deploy

```bash
# Using the SWA CLI for local development and deployment
npm install -g @azure/static-web-apps-cli
swa init                     # detects framework and build settings
swa start                    # local emulator with auth and API proxy
swa deploy --env production

# Or create the resource linked to a GitHub repo (generates the GitHub Actions workflow)
az staticwebapp create -g $RG -n $SWA -l westeurope \
  --source https://github.com/<org>/<repo> --branch main \
  --app-location "/" --api-location "api" --output-location "dist" --login-with-github
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Static Web Apps vs Blob static website + Front Door | SWA for integrated CI/CD, auth, APIs and previews; Blob + Front Door for simple static content under full CDN and WAF control |
| Static Web Apps vs App Service | SWA for SPA and static front ends; App Service for server-rendered apps (for example Next.js with heavy SSR, though SWA has hybrid Next.js support) |
| Managed functions vs linked back end | Managed functions for small APIs tied to the front end; linked back end (Standard) for real services, private networking, independent scaling |
| Auth | Built-in Entra for internal apps; custom OIDC for external identity (Entra External ID) |
| Private | Standard plan with private endpoint for internal-only apps |

## Hands-on

1. Scaffold a front end (`npm create vite@latest chat-ui -- --template react`) and a `api` folder with a Python HTTP function returning a greeting.
2. Run locally with `swa start dist --api-location api` and log in with the emulated auth.
3. Push to GitHub and create the Static Web App linked to the repo; watch the GitHub Actions run.
4. Open a pull request with a visible change and test the preview URL.
5. Add `staticwebapp.config.json` restricting `/api/*` to authenticated users, and test signing in with Entra ID.

## Security, cost and reliability notes

- **Security:** restrict routes with roles; validate the `x-ms-client-principal` claims in your API rather than trusting client-side checks.
- **Security:** preview environments are public by default; protect them with password protection or auth, especially if they use real data.
- **Security:** set security headers (CSP, HSTS) in the config file.
- **Cost:** Free plan for prototypes; Standard is a small fixed monthly cost per app. Back ends bill separately.
- **Reliability:** static content is globally distributed; back-end reliability depends on the linked service and its region.

## Common mistakes

- Putting secrets in front-end code or build environment variables that end up in the bundle.
- Forgetting the SPA navigation fallback, causing 404s on deep links.
- Public preview environments exposing unreleased features or data.
- Calling APIs on other origins and fighting CORS instead of using linked back ends.
- Using managed functions for APIs that need VNet access (use a linked back end).

## Check yourself

1. What does Static Web Apps add over hosting files in Blob Storage?
2. What is the difference between managed functions and a linked back end?
3. How are routes protected by role?
4. What happens when you open a pull request on a connected repository?
5. Which plan is needed for private endpoints and custom OIDC providers?

<details>
<summary>Answers</summary>

1. Integrated CI/CD, preview environments, built-in authentication and authorisation, routing rules, same-origin APIs, custom domains with free certificates.
2. Managed functions are built and hosted with the app (limited features); a linked back end connects an existing Function app, Container App, App Service or API Management instance that you manage separately (Standard plan).
3. With `allowedRoles` on route rules in `staticwebapp.config.json`, using built-in roles (`anonymous`, `authenticated`) and custom roles.
4. A preview environment with its own URL is built and deployed, and removed when the PR closes.
5. The Standard plan.

</details>

## References

- [Azure Static Web Apps overview](https://learn.microsoft.com/azure/static-web-apps/overview)
- [Hosting plans](https://learn.microsoft.com/azure/static-web-apps/plans)
- [API support](https://learn.microsoft.com/azure/static-web-apps/apis-overview)
- [Authentication and authorization](https://learn.microsoft.com/azure/static-web-apps/authentication-authorization)
- [Configuration file](https://learn.microsoft.com/azure/static-web-apps/configuration)
- [Preview environments](https://learn.microsoft.com/azure/static-web-apps/review-publish-pull-requests)
- [Database connections (Data API builder)](https://learn.microsoft.com/azure/static-web-apps/database-overview)
- [Static Web Apps CLI](https://azure.github.io/static-web-apps-cli/)
