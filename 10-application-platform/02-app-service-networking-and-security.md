# App Service Networking and Security

## Why it matters

By default an App Service app has a public endpoint and reaches dependencies over the internet. Enterprise and AI workloads usually need the opposite: private inbound access, private outbound access to databases and Foundry, and authentication in front of everything. App Service has several networking features that are easy to confuse because they work in different directions. Getting them right is a core skill for architects designing secure web and AI applications.

## Key concepts

### Inbound vs outbound

```text
                INBOUND (who can reach the app)                 OUTBOUND (what the app can reach)
Internet ──► [Access restrictions] ──► App ──► [VNet integration] ──► VNet ──► private endpoints
         ──► [Private endpoint]   ──►     (route all traffic)          ──► NAT gateway / firewall
         ──► [Front Door + WAF]   ──►                                  ──► on-premises (VPN/ER)
```

| Feature | Direction | Purpose |
|---|---|---|
| **Access restrictions** | Inbound | Allow or deny by IP, service tag, VNet subnet (service endpoint) or HTTP header (for example `X-Azure-FDID` from your Front Door) |
| **Private endpoint** | Inbound | Private IP for the app in your VNet; disable public network access |
| **VNet integration** | Outbound | App's outbound traffic enters a delegated subnet in your VNet, reaching private endpoints, peered VNets and on-premises |
| **Route all** (`vnetRouteAllEnabled`) | Outbound | Send all outbound traffic, including internet, through the VNet (for firewalls, NAT gateway, UDRs) |
| **NAT gateway** on the integration subnet | Outbound | Static outbound IP, more SNAT ports |
| **Hybrid Connections** | Outbound | Reach a specific host:port on-premises through a relay agent, no VPN |
| **App Service Environment v3** | Both | App Service deployed entirely inside your VNet |

A fully private app needs **both**: a private endpoint for inbound and VNet integration for outbound.

### VNet integration details

- One subnet per plan, delegated to `Microsoft.Web/serverFarms`, at least /26 recommended so scale-out and slot swaps have addresses.
- Apps on the same plan share the integration.
- DNS: the app uses the VNet's DNS settings, so private DNS zones linked to the VNet resolve private endpoints.
- NSGs and UDRs on the integration subnet apply to outbound traffic.

### App Service Environment v3 (ASE)

| Property | Detail |
|---|---|
| Model | Single-tenant App Service in a subnet of your VNet |
| Access | Internal (ILB) or external VIP |
| Pricing | Isolated v2 plans; no separate stamp fee in v3 |
| Fit | Strict isolation and compliance, very large scale, dedicated hardware, high density of apps |
| Versions | ASE v1 and v2 are retired; v3 is current |

For most workloads, multi-tenant App Service with private endpoints and VNet integration provides sufficient isolation at lower cost.

### Built-in authentication (Easy Auth)

App Service authentication runs as a module in front of your code:

| Feature | Detail |
|---|---|
| Providers | Microsoft Entra ID (workforce and external tenants), Microsoft account, Google, Facebook, X, Apple, any OpenID Connect provider |
| Behaviour | Redirect unauthenticated users to sign in, or return 401 for APIs |
| Token store | Stores tokens for the signed-in user; `/.auth/me` exposes claims; `/.auth/refresh` refreshes |
| Headers to app | `X-MS-CLIENT-PRINCIPAL`, `X-MS-CLIENT-PRINCIPAL-NAME`, and provider tokens |
| Authorisation | Restrict to specific client apps, tenants, groups or roles via app registration and settings; finer authorisation stays in code |

Easy Auth is a fast, robust way to put Entra sign-in in front of internal tools, AI chat UIs, MCP servers and APIs. For complex authorisation (scopes, on-behalf-of to downstream APIs) you still validate tokens and claims in code or use the token store.

### Other security settings

| Setting | Recommendation |
|---|---|
| HTTPS only | On |
| Minimum TLS version | 1.2 or higher (1.3 supported) |
| FTP / FTPS | Disabled |
| Basic auth publishing credentials | Disabled (deploy with Entra identities) |
| Remote debugging | Off |
| Client certificates | For mutual TLS scenarios |
| Managed identity | For Key Vault, Storage, databases, Foundry, ACR |
| Key Vault references | For any secret that cannot be replaced by identity |
| Defender for App Service | Threat detection for the platform and apps |
| SCM (Kudu) site access | Restrict separately; it is a powerful admin endpoint |

## How it works on Azure

### Private web app pattern

```text
Users ──► Front Door Premium (WAF) ──► Private Link to App Service private endpoint
                                         (public network access disabled)
App Service (Premium v3, zone redundant)
   └─ VNet integration (snet-app-int, /26, route all, NAT gateway or UDR to firewall)
          ├─► private endpoint: Azure SQL / Cosmos DB
          ├─► private endpoint: Key Vault
          ├─► private endpoint: Foundry resource / Azure OpenAI
          └─► private endpoint: Storage
Private DNS zones linked to the VNet (or hub DNS resolver)
```

Front Door Premium can connect to App Service through **Private Link**, so the app needs no public endpoint even while serving internet users.

### Configuration commands

```bash
# Outbound: VNet integration with route-all
az webapp vnet-integration add -g $RG -n $APP --vnet vnet-app --subnet snet-app-int
az webapp config set -g $RG -n $APP --vnet-route-all-enabled true

# Inbound: private endpoint and disable public access
az network private-endpoint create -g $RG -n pe-$APP --vnet-name vnet-app --subnet snet-pe \
  --private-connection-resource-id $(az webapp show -g $RG -n $APP --query id -o tsv) \
  --group-id sites --connection-name sites
az webapp update -g $RG -n $APP --set publicNetworkAccess=Disabled

# Or: restrict to your Front Door only
az webapp config access-restriction add -g $RG -n $APP --priority 100 --rule-name frontdoor \
  --service-tag AzureFrontDoor.Backend --http-header x-azure-fdid=<your-front-door-id>

# Hardening
az webapp update -g $RG -n $APP --https-only true
az webapp config set -g $RG -n $APP --min-tls-version 1.2 --ftps-state Disabled
az resource update -g $RG --namespace Microsoft.Web --resource-type basicPublishingCredentialsPolicies \
  --parent sites/$APP -n scm --set properties.allow=false
```

### Easy Auth with Entra ID

```bash
az webapp auth microsoft update -g $RG -n $APP \
  --client-id <app-registration-client-id> \
  --issuer https://login.microsoftonline.com/<tenant-id>/v2.0 \
  --allowed-audiences api://<app-registration-client-id>
az webapp auth update -g $RG -n $APP --enabled true --action LoginWithAzureActiveDirectory
```

The portal's "Add identity provider" flow creates the app registration for you, which is easier for a first attempt.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Private endpoint vs access restrictions | Private endpoint for internal apps and maximum isolation; access restrictions with the Front Door header for public apps behind Front Door (or Front Door Premium Private Link) |
| ASE v3 vs multi-tenant + private networking | Multi-tenant for most; ASE when compliance requires single tenancy or at very large scale |
| Easy Auth vs code-based auth | Easy Auth for sign-in in front of apps and simple APIs; libraries (MSAL, Microsoft.Identity.Web) for complex flows; often both |
| Route all | On when egress must be inspected or a static IP is needed |
| Admin access to SCM | Restrict; deploy through CI with Entra-based publishing |

## Hands-on

1. Using the app from file 01, enable Easy Auth with Microsoft Entra ID through the portal. Browse to the app and sign in, then call `/.auth/me` to see your claims.
2. Add an access restriction allowing only your public IP and confirm others are blocked (use a phone on mobile data).
3. Create a VNet with two subnets (`snet-app-int` /26 delegated, `snet-pe`), add VNet integration, and create a storage account with only a private endpoint. From the Kudu console (or an endpoint in your app), resolve and call the storage account to confirm it uses the private IP.
4. Disable FTP and basic publishing credentials and confirm deployment still works through `az webapp deploy` with your Entra identity.

## Security, cost and reliability notes

- **Security:** both directions matter. Private inbound without outbound integration still sends data to dependencies over public endpoints.
- **Security:** protect the SCM/Kudu endpoint; it allows file access and command execution.
- **Security:** Easy Auth is not authorisation by itself; restrict allowed audiences, client applications and users or groups.
- **Cost:** private endpoints, Front Door Premium, NAT gateway and ASE add cost; ASE in particular is for clear requirements.
- **Reliability:** integration subnets that are too small block scale-out and swaps.

## Common mistakes

- Thinking VNet integration makes the app private (it is outbound only).
- Private endpoint without private DNS, so clients resolve the public IP.
- Allowing the `AzureFrontDoor.Backend` service tag without checking the `X-Azure-FDID` header, letting anyone's Front Door reach your app.
- /28 integration subnets that run out of addresses.
- Easy Auth enabled but allowing any tenant's users.
- Leaving basic auth publishing enabled.

## Check yourself

1. Which features control inbound traffic and which control outbound?
2. What does "route all" change?
3. How do you ensure only your Front Door instance can reach an app?
4. What does a fully private App Service app require?
5. When is App Service Environment v3 justified?
6. What does Easy Auth provide, and what remains the application's job?

<details>
<summary>Answers</summary>

1. Inbound: access restrictions, private endpoints, Front Door Private Link, ASE. Outbound: VNet integration, route all, NAT gateway, Hybrid Connections.
2. All outbound traffic, including internet-bound traffic, goes through the integration subnet so NSGs, UDRs, firewalls and NAT gateways apply.
3. Use an access restriction with the `AzureFrontDoor.Backend` service tag plus the `X-Azure-FDID` header set to your Front Door's ID, or Front Door Premium with Private Link and public access disabled.
4. A private endpoint for inbound with public access disabled, VNet integration for outbound, private endpoints for dependencies, and private DNS.
5. When compliance requires single-tenant isolation, for very large scale or high app density, or when the whole platform must sit inside your VNet.
6. Authentication (sign-in, token validation, token store, claims headers) in front of the app. Fine-grained authorisation and downstream token flows remain the application's responsibility.

</details>

## References

- [App Service networking features](https://learn.microsoft.com/azure/app-service/networking-features)
- [VNet integration](https://learn.microsoft.com/azure/app-service/overview-vnet-integration)
- [Private endpoints for App Service](https://learn.microsoft.com/azure/app-service/overview-private-endpoint)
- [Access restrictions](https://learn.microsoft.com/azure/app-service/overview-access-restrictions)
- [App Service Environment v3](https://learn.microsoft.com/azure/app-service/environment/overview)
- [Authentication and authorization in App Service](https://learn.microsoft.com/azure/app-service/overview-authentication-authorization)
- [Security recommendations for App Service](https://learn.microsoft.com/azure/app-service/security-recommendations)
- [Front Door with Private Link to App Service](https://learn.microsoft.com/azure/frontdoor/standard-premium/how-to-enable-private-link-web-app)
