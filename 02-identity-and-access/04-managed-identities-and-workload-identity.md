# Managed Identities and Workload Identity

## Why it matters

An AI application talks to a model endpoint, a search service, a storage account, a database and a key vault. Done the old way, that is five secrets to store, rotate and eventually leak. Done properly, it is zero: each service proves who it is with an identity Azure manages. "Keyless" is the expected standard for anything you build on Azure now, and this file is how you get there.

## Key concepts

### Workload identities

A **workload identity** is an identity for software rather than a person. In Entra ID there are three kinds:

| Kind | Credentials | Use when |
|---|---|---|
| **Managed identity** | None you can see. Azure issues and rotates them | The workload runs on Azure (or on Arc-enabled infrastructure) |
| **Service principal with federated credential** | None stored. Trusts tokens from an external issuer | The workload runs outside Azure but has its own identity provider: GitHub Actions, Kubernetes, another cloud |
| **Service principal with secret or certificate** | You store and rotate them | Nothing else is possible |

Work down that list and stop at the first one that fits.

### Managed identities

A managed identity is a service principal whose lifecycle and credentials are handled by Azure. Your code asks a local endpoint for a token; there is nothing to store.

| | **System-assigned** | **User-assigned** |
|---|---|---|
| Created | As a property of one resource | As a standalone Azure resource |
| Lifecycle | Deleted with the resource | Independent |
| Shared across resources | No, one to one | Yes, one to many |
| Role assignments | Recreated whenever the resource is recreated | Survive resource recreation |
| Best for | A single resource with its own unique access | Several resources sharing access; pre-provisioned access; blue-green deployments; scale-out |

**Default to user-assigned for applications.** You can create the identity and its role assignments before the compute exists, replace the compute without redoing RBAC, and avoid the propagation delay that hits a freshly created system-assigned identity. Use system-assigned when the identity really is specific to one resource, such as a search service reading from storage.

Don't over-share. One user-assigned identity used by ten unrelated applications gives each of them the union of all permissions.

### How a token is obtained

```text
Your code ──► Azure Identity library ──► local identity endpoint ──► Entra ID
                                          (IMDS 169.254.169.254
                                           or IDENTITY_ENDPOINT)
          ◄── access token for the requested resource ◄──────────────
```

- On **VMs and scale sets** the endpoint is the Instance Metadata Service at `169.254.169.254`.
- On **App Service, Functions and Container Apps** it is given by the `IDENTITY_ENDPOINT` and `IDENTITY_HEADER` environment variables.
- On **AKS** it works through workload identity federation (below).

Tokens are cached by the library and typically valid for up to 24 hours. That has a consequence: **adding or removing a role or group membership may not take effect until the token refreshes.** Role assignments made directly on the identity are evaluated by the resource at request time and apply within minutes; changes that depend on token claims, such as group membership, can lag for hours.

### The Azure Identity library

Every Azure SDK accepts a credential object from the Azure Identity library.

| Credential | Use |
|---|---|
| `DefaultAzureCredential` | Tries a chain of sources; works on a laptop and in Azure unchanged |
| `ManagedIdentityCredential` | Managed identity only; explicit and fast |
| `WorkloadIdentityCredential` | AKS and other federated workloads |
| `AzureCliCredential` / `AzureDeveloperCliCredential` | Local development using your signed-in tool |
| `ClientSecretCredential` / `ClientCertificateCredential` | Service principal with a secret or certificate |
| `ChainedTokenCredential` | Your own explicit chain |

`DefaultAzureCredential` tries, in order: environment variables, workload identity, managed identity, then developer tools (Visual Studio Code, Azure CLI, Azure PowerShell, Azure Developer CLI).

It is convenient but has costs in production: a slower first token while it probes, and the risk of silently picking up an unintended credential. Common practice:

```python
import os
from azure.identity import DefaultAzureCredential, ManagedIdentityCredential

if os.getenv("AZURE_CLIENT_ID") and os.getenv("RUNNING_IN_AZURE"):
    credential = ManagedIdentityCredential(client_id=os.environ["AZURE_CLIENT_ID"])
else:
    credential = DefaultAzureCredential()
```

When a resource has more than one user-assigned identity, you **must** specify which one by client ID, or the request is ambiguous and fails.

### Resource scopes for tokens

A token is requested for a specific resource. The common audiences:

| Service | Scope |
|---|---|
| Azure Resource Manager | `https://management.azure.com/.default` |
| Foundry Models / Azure OpenAI / Foundry Tools | `https://cognitiveservices.azure.com/.default` |
| Foundry project (agents) | `https://ai.azure.com/.default` |
| Azure AI Search | `https://search.azure.com/.default` |
| Storage | `https://storage.azure.com/.default` |
| Key Vault | `https://vault.azure.net/.default` |
| Azure SQL | `https://database.windows.net/.default` |
| Azure Database for PostgreSQL / MySQL | `https://ossrdbms-aad.database.windows.net/.default` |
| Cosmos DB | `https://cosmos.azure.com/.default` |
| Microsoft Graph | `https://graph.microsoft.com/.default` |

The SDKs set these for you. You need them when calling REST directly or using a third-party client.

### Workload identity federation

Federation lets an external token stand in for a secret. You configure a **federated identity credential** on an app registration or user-assigned managed identity that says: *trust tokens from this issuer, with this subject, for this audience.* The external workload presents its own token to Entra ID and receives an Entra access token in exchange. Nothing is stored anywhere.

| Scenario | Issuer | Subject example |
|---|---|---|
| **GitHub Actions** | `https://token.actions.githubusercontent.com` | `repo:contoso/app:environment:production` |
| **Azure DevOps** | Service connection with workload identity federation | Managed by the service connection |
| **AKS workload identity** | The cluster's OIDC issuer URL | `system:serviceaccount:<namespace>:<serviceaccount>` |
| **Other clouds** | AWS, GCP identity providers | Role or service account identifier |
| **Terraform Cloud, GitLab, others** | Their OIDC issuers | Workspace or project identifier |
| **Managed identity as a credential** | Entra ID | Lets a multitenant app use a managed identity instead of a secret |

The subject claim must match exactly. A GitHub workflow that runs on a branch when the credential expects an environment will fail with `AADSTS70021`.

Limit: 20 federated credentials per identity. **Flexible federated identity credentials** allow pattern matching on the subject to reduce the number needed.

### AKS workload identity

The pattern for pods:

1. Enable the OIDC issuer and workload identity add-on on the cluster.
2. Create a user-assigned managed identity and grant it roles.
3. Create a Kubernetes service account annotated with the identity's client ID.
4. Add a federated credential on the identity for that service account.
5. Label the pod `azure.workload.identity/use: "true"`.

The older *pod-managed identity* approach is retired. Covered further in topic 07.

## How it works on Azure

### Where managed identities are available

Practically every compute and many platform services: VMs and scale sets, App Service, Functions, Container Apps, Container Instances, AKS, Logic Apps, API Management, Data Factory, Azure AI Search, Foundry resources and projects, Azure Machine Learning, Automation, Azure SQL, Event Grid, and Arc-enabled servers.

Platform services use their own identity too. Common service-to-service links in AI solutions:

| Source identity | Target | Why |
|---|---|---|
| Azure AI Search | Storage | Indexers read documents |
| Azure AI Search | Foundry | Integrated vectorisation and skills |
| Foundry project | Azure AI Search, Storage, Cosmos DB | Agent tools, file storage, thread storage |
| API Management | Foundry | Gateway calls the model with its own identity |
| Container Apps / Functions | Container Registry | Pull images with `AcrPull` |
| Any service | Key Vault | Read the few secrets that still exist |

### Databases

Managed identities work for databases, but the database needs to know the identity:

- **Azure SQL:** `CREATE USER [my-identity] FROM EXTERNAL PROVIDER;` then grant database roles.
- **PostgreSQL / MySQL flexible server:** enable Entra authentication and create a role mapped to the identity; the access token is used as the password.
- **Cosmos DB:** assign a data-plane role with `az cosmosdb sql role assignment create`.

### A complete keyless call

```python
# pip install azure-identity openai
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import AzureOpenAI

token_provider = get_bearer_token_provider(
    DefaultAzureCredential(),
    "https://cognitiveservices.azure.com/.default",
)

client = AzureOpenAI(
    azure_endpoint="https://<your-resource>.openai.azure.com",
    azure_ad_token_provider=token_provider,
    api_version="2024-10-21",
)

resp = client.chat.completions.create(
    model="<your-deployment-name>",
    messages=[{"role": "user", "content": "Say hello in five words."}],
)
print(resp.choices[0].message.content)
```

No key appears anywhere. Locally this uses your `az login` session (you need *Cognitive Services OpenAI User* on the resource). Deployed to Azure it uses the managed identity (which needs the same role). The code does not change.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| System vs user-assigned | User-assigned for app compute; system-assigned for platform services with a unique relationship |
| One identity per what? | Per application per environment. Never share production and non-production |
| Federation vs secret for CI/CD | Federation. There is no good reason left to store a client secret in a pipeline |
| Where secrets remain | Third-party APIs that only accept keys. Store in Key Vault, read with a managed identity |
| Identity in IaC | Create the identity and role assignments in the same template as the workload so they are reviewed together |

**The developer gap:** local development uses the developer's own identity, which needs the same data roles as the app in the dev environment. Grant these to a developer group at resource-group scope in dev only. Do not solve it by handing out keys.

## Hands-on

Deploy a container app that calls Storage with a user-assigned identity.

```bash
RG=rg-mi-lab; LOC=swedencentral
az group create -n $RG -l $LOC

# Identity
az identity create -n id-mi-lab -g $RG
MI_ID=$(az identity show -n id-mi-lab -g $RG --query id -o tsv)
MI_CLIENT=$(az identity show -n id-mi-lab -g $RG --query clientId -o tsv)
MI_PRINCIPAL=$(az identity show -n id-mi-lab -g $RG --query principalId -o tsv)

# Storage with keys disabled
ST=stmilab$RANDOM
az storage account create -n $ST -g $RG -l $LOC --sku Standard_LRS --allow-shared-key-access false
ST_ID=$(az storage account show -n $ST -g $RG --query id -o tsv)

# Grant the identity a data role BEFORE the compute exists
az role assignment create --assignee-object-id $MI_PRINCIPAL --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Contributor" --scope $ST_ID

# Compute with the identity attached
az containerapp env create -n cae-mi-lab -g $RG -l $LOC
az containerapp create -n ca-mi-lab -g $RG --environment cae-mi-lab \
  --image mcr.microsoft.com/azure-cli:latest \
  --user-assigned $MI_ID \
  --env-vars AZURE_CLIENT_ID=$MI_CLIENT ST=$ST \
  --command "/bin/sh" --args "-c" "sleep infinity" --min-replicas 1

# Prove it from inside the container
az containerapp exec -n ca-mi-lab -g $RG --command "/bin/sh"
#   az login --identity --client-id $AZURE_CLIENT_ID
#   az storage container create --account-name $ST -n fromapp --auth-mode login
#   env | grep IDENTITY        # see the endpoint the library calls
#   exit

az group delete -n $RG --yes --no-wait
```

Then set up GitHub Actions federation for a repository you own:

```bash
az ad app create --display-name gh-roadmap-lab
APP_ID=$(az ad app list --display-name gh-roadmap-lab --query "[0].appId" -o tsv)
az ad sp create --id $APP_ID
az ad app federated-credential create --id $APP_ID --parameters '{
  "name": "main-branch",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<owner>/<repo>:ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"]
}'
```

In the workflow, use `azure/login` with `client-id`, `tenant-id` and `subscription-id` and `permissions: id-token: write`. No secret is stored in GitHub.

## Security, cost and reliability notes

- **Security:** anyone who can run code on a resource can use its managed identity. Controlling who can deploy to the resource is part of protecting the identity.
- **Security:** monitor managed identity and service principal sign-in logs. Workload identities are a growing attack target and usually have no Conditional Access on them unless you license and configure it.
- **Security:** scope federated credential subjects tightly (specific repo and environment), never a wildcard across an organisation.
- **Cost:** managed identities are free.
- **Reliability:** handle transient token acquisition failures with the SDK's built-in retry. Pre-create identities and role assignments to avoid first-deploy races.

## Common mistakes

- Using a system-assigned identity for an app, then losing all role assignments when the app is recreated.
- Forgetting to set the client ID when more than one user-assigned identity is attached.
- Assigning a role and testing immediately, before propagation.
- Adding the identity to a group for access and waiting hours for the cached token to reflect it.
- Storing a service principal secret in a pipeline when federation is available.
- Sharing one identity across every app in a subscription.
- Giving developers keys for local work instead of data roles.
- Leaving key authentication enabled "just in case", which means the keys are still a risk.

## Check yourself

1. Give two reasons to prefer a user-assigned identity for an application.
2. What does `DefaultAzureCredential` do, and why might you not use it as-is in production?
3. How does a GitHub Actions workflow authenticate to Azure without a stored secret?
4. An app has two user-assigned identities and token requests fail. What is missing?
5. Which identity needs access to storage for an Azure AI Search indexer to work?
6. Why might a group-based permission change take hours to apply to a managed identity?
7. What scope do you request to call a Foundry model with a token?

<details>
<summary>Answers</summary>

1. Role assignments can be created before the compute and survive its recreation; the identity can be shared across instances or slots of the same app.
2. It tries a chain of credential sources so the same code works locally and in Azure. In production the probing adds latency and may pick an unintended credential, so many teams use `ManagedIdentityCredential` explicitly.
3. Workload identity federation: GitHub issues an OIDC token, and Entra ID exchanges it for an access token based on a federated credential matching the issuer and subject.
4. The client ID of the identity to use.
5. The search service's managed identity.
6. Managed identity tokens are cached for up to 24 hours and group membership is carried in the token.
7. `https://cognitiveservices.azure.com/.default`.

</details>

## References

- [What are managed identities for Azure resources?](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview)
- [Managed identity best practice recommendations](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/managed-identity-best-practice-recommendations)
- [Workload identity federation](https://learn.microsoft.com/entra/workload-id/workload-identity-federation)
- [Azure Identity client library for Python](https://learn.microsoft.com/python/api/overview/azure/identity-readme)
- [Credential chains in the Azure Identity library](https://learn.microsoft.com/azure/developer/python/sdk/authentication/credential-chains)
- [Use Microsoft Entra Workload ID with AKS](https://learn.microsoft.com/azure/aks/workload-identity-overview)
- [Use GitHub Actions to connect to Azure](https://learn.microsoft.com/azure/developer/github/connect-from-azure)
- [Configure keyless authentication with Microsoft Entra ID for Foundry Models](https://learn.microsoft.com/azure/foundry/foundry-models/how-to/configure-entra-id)
