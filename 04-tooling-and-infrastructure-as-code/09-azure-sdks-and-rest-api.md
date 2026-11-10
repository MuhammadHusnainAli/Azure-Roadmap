# Azure SDKs and REST API

## Why it matters

The CLI and IaC create resources. Your **application** uses them through the SDKs: uploading a blob, querying a search index, calling a model, running an agent. The Azure SDKs share one design across services and languages, so learning the patterns once (credentials, retries, paging, long-running operations, errors, tracing) pays off in every client you use afterwards. Underneath them all is the REST API, which you need to understand when an SDK lags a feature or something goes wrong.

## Key concepts

### Two kinds of library

| | **Management (control plane)** | **Client (data plane)** |
|---|---|---|
| Purpose | Create and configure resources | Use a resource |
| Endpoint | `management.azure.com` | The service's own endpoint |
| Python package pattern | `azure-mgmt-<service>` | `azure-<service>-<feature>` |
| .NET pattern | `Azure.ResourceManager.<Service>` | `Azure.<Service>.<Feature>` |
| Examples | `azure-mgmt-storage`, `azure-mgmt-cognitiveservices` | `azure-storage-blob`, `azure-search-documents`, `azure-ai-projects` |
| Typical user | Tooling, platform automation | Applications |

Applications mostly need data-plane clients. If your application code is creating infrastructure with management libraries, ask whether that belongs in IaC.

### Languages and support

First-class SDKs exist for **.NET, Python, JavaScript/TypeScript, Java and Go**, with C++, and newer SDKs such as Rust at varying maturity. All follow the **Azure SDK design guidelines**, which is why a `BlobServiceClient` looks alike in every language.

Packages are **GA**, **beta/preview** (`b1`, `-beta.1`) or deprecated. Preview packages change between releases; pin them exactly.

### Core packages

| Package (Python name) | Role |
|---|---|
| `azure-identity` | Credentials for Entra ID authentication |
| `azure-core` | Shared HTTP pipeline: retries, logging, tracing, paging, polling, exceptions |
| `azure-core-tracing-opentelemetry` | Emits OpenTelemetry spans for SDK calls |
| `azure-monitor-opentelemetry` | Sends telemetry to Application Insights |

### SDKs you will use for AI work

| Purpose | Python | Notes |
|---|---|---|
| Foundry projects: agents, connections, evaluations, tracing | `azure-ai-projects` | `AIProjectClient`; the entry point for a Foundry project |
| Call OpenAI-compatible models | `openai` | The standard OpenAI library, pointed at your Azure endpoint with an Entra token provider |
| Call other catalogue models | `azure-ai-inference`, or the OpenAI-compatible endpoint | Unified chat and embeddings API |
| Agents and multi-agent workflows | `agent-framework` (Microsoft Agent Framework) | Topic 19 |
| Search | `azure-search-documents` | Query, index and manage |
| Evaluation | `azure-ai-evaluation` | Topic 22 |
| Content Safety | `azure-ai-contentsafety` | |
| Document Intelligence | `azure-ai-documentintelligence` | |
| Speech | `azure-cognitiveservices-speech` | |
| Storage | `azure-storage-blob` | |
| Cosmos DB | `azure-cosmos` | |
| Key Vault | `azure-keyvault-secrets` | |
| Messaging | `azure-servicebus`, `azure-eventhub`, `azure-eventgrid` | |

AI package names and their scope have shifted repeatedly. Check the current Foundry SDK overview before starting a project, and pin versions.

### Authentication

Covered in topic 02, file 04. In code:

```python
from azure.identity import DefaultAzureCredential, ManagedIdentityCredential

credential = DefaultAzureCredential()                       # laptop and Azure, unchanged
# credential = ManagedIdentityCredential(client_id=...)     # explicit in production

from azure.storage.blob import BlobServiceClient
blobs = BlobServiceClient("https://<account>.blob.core.windows.net", credential=credential)
```

- Create **one credential** and reuse it across clients. It caches tokens.
- Pass the **endpoint and a credential**, never a connection string or key.
- For libraries that want a token callback (such as `openai`), use `get_bearer_token_provider(credential, scope)`.

### Client lifetime

Clients are thread-safe and hold connection pools. **Create them once and reuse them** for the life of the process (a module-level singleton, or dependency injection). Creating a client per request wastes connections and re-acquires tokens.

### The HTTP pipeline

Every request passes through policies in `azure-core`:

```text
your call → [ custom policies → retry → authentication → logging → tracing ] → HTTP transport → service
```

You configure it through client options:

```python
from azure.core.pipeline.policies import RetryPolicy
from azure.search.documents import SearchClient

client = SearchClient(
    endpoint, index_name, credential,
    retry_total=5,                 # attempts
    retry_backoff_factor=0.8,      # exponential backoff base
    retry_backoff_max=60,
    connection_timeout=10,
    read_timeout=60,
    logging_enable=False,
)
```

### Retries

Default behaviour retries transient failures (connection errors, 408, 429, 500, 502, 503, 504) with **exponential backoff and jitter**, and honours the `Retry-After` header.

Guidance:

- Keep SDK retries on. Do **not** wrap them in another naive retry loop; nested retries multiply load during an outage.
- Only retry operations that are safe to repeat. A write that timed out may have succeeded.
- Set an overall **timeout** so a user-facing request does not hang through many retries.
- For model calls, 429 means you are over your tokens-per-minute or requests-per-minute rate. Backoff helps short bursts; sustained 429s need more quota, a gateway that load-balances, or provisioned capacity (topics 11, 15).
- The `openai` library has its own `max_retries` and `timeout` settings; configure them deliberately.

### Paging

List operations return a pager that fetches pages lazily.

```python
for blob in container.list_blobs(name_starts_with="docs/"):      # iterates across pages
    print(blob.name)

pages = container.list_blobs(results_per_page=100).by_page()     # explicit pages
first_page = list(next(pages))
```

Do not call `list()` on a pager for a large collection unless you mean to load everything into memory.

### Long-running operations

Operations that take a while return a **poller**; method names start with `begin_`.

```python
from azure.ai.documentintelligence import DocumentIntelligenceClient

poller = di_client.begin_analyze_document("prebuilt-layout", body=f)   # returns immediately
result = poller.result()               # blocks until done (optionally with timeout=)

poller.status()                        # "InProgress", "Succeeded", "Failed"
token = poller.continuation_token()    # persist to resume polling from another process
```

### Errors

```python
from azure.core.exceptions import (
    HttpResponseError, ResourceNotFoundError, ResourceExistsError,
    ClientAuthenticationError, ServiceRequestError,
)

try:
    blob_client.download_blob().readall()
except ResourceNotFoundError:
    ...                                    # 404: expected, handle it
except ClientAuthenticationError as e:
    ...                                    # credential problem: token acquisition failed
except HttpResponseError as e:
    log.error("status=%s code=%s request_id=%s", e.status_code, e.error.code if e.error else None,
              e.response.headers.get("x-ms-request-id"))
    raise
```

| Status | Usual meaning |
|---|---|
| 400 | Bad request: malformed input, unsupported API version, content filter triggered (model APIs) |
| 401 | Not authenticated: no token, wrong audience, expired |
| 403 | Authenticated but not authorised: missing RBAC role, or blocked by network rules |
| 404 | Resource, deployment or index not found; also wrong endpoint |
| 409 | Conflict: already exists, or an operation is in progress |
| 412 | Precondition failed: ETag mismatch |
| 429 | Throttled |
| 5xx | Service error; retryable |

Always log the **request ID** (`x-ms-request-id`, `apim-request-id` for AI services, or `x-ms-client-request-id` that you supply). Support cannot trace an issue without it.

A 403 with RBAC apparently correct is very often a **network** block: public access disabled and the caller is not on the private network.

### Async

Each client has an async twin in an `.aio` namespace (Python), or `Async` methods (.NET, Java).

```python
from azure.identity.aio import DefaultAzureCredential
from azure.storage.blob.aio import BlobServiceClient

async with DefaultAzureCredential() as cred, BlobServiceClient(url, credential=cred) as svc:
    async for c in svc.list_containers():
        print(c.name)
```

Use async for I/O-bound services, which is nearly every AI application: an API waiting on model and search calls handles far more concurrent requests asynchronously. Do not mix sync clients into async code; they block the event loop. Close async clients and credentials.

### Concurrency and conditional requests

Many services return an **ETag**. Passing it back (`if-match`) makes an update succeed only if nobody else changed the resource, giving optimistic concurrency without locks.

### Observability

The SDKs emit **OpenTelemetry** spans for each call when tracing is enabled, and log through the language's standard logging.

```python
from azure.monitor.opentelemetry import configure_azure_monitor
configure_azure_monitor(connection_string=APPINSIGHTS_CONNECTION_STRING)   # traces, metrics, logs

import logging
logging.getLogger("azure.core.pipeline.policies.http_logging_policy").setLevel(logging.WARNING)
```

HTTP logging redacts headers and query parameters by default. Enabling full body logging can capture prompts, documents and personal data; decide deliberately. GenAI tracing (model calls, tool calls, agent steps) follows OpenTelemetry's generative AI semantic conventions and is covered in topic 22.

## How it works on Azure

### The REST API underneath

**Control plane**

```http
PUT https://management.azure.com/subscriptions/{sub}/resourceGroups/{rg}
    /providers/Microsoft.Storage/storageAccounts/{name}?api-version=2023-05-01
Authorization: Bearer <token for https://management.azure.com/>
Content-Type: application/json

{ "location": "swedencentral", "sku": { "name": "Standard_LRS" }, "kind": "StorageV2" }
```

| Pattern | Detail |
|---|---|
| **`api-version`** | Required on every call. Fixes the request and response shape |
| **Verbs** | `PUT` create or replace (idempotent), `PATCH` partial update, `GET` read, `DELETE`, `POST` for actions (`/listKeys`, `/restart`) |
| **Async operations** | `201`/`202` with an `Azure-AsyncOperation` or `Location` header to poll, and `Retry-After` |
| **Paging** | Response contains `value` and `nextLink`; follow `nextLink` until absent |
| **Filtering** | OData `$filter`, `$top`, `$expand`, `$select` where supported |
| **Errors** | `{ "error": { "code": "...", "message": "...", "details": [...] } }` |
| **Throttling** | `429` with `Retry-After`; remaining budget in `x-ms-ratelimit-remaining-*` headers |

**Data plane** follows each service's own conventions but the same broad rules: an `api-version`, a bearer token with the service's audience, JSON errors, and request IDs.

Calling a model directly:

```bash
ENDPOINT="https://<resource>.openai.azure.com"
TOKEN=$(az account get-access-token --resource https://cognitiveservices.azure.com --query accessToken -o tsv)

curl -sS "$ENDPOINT/openai/v1/chat/completions" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{ "model": "<deployment-name>", "messages": [ { "role": "user", "content": "Reply with one word." } ] }'
```

The versionless `/openai/v1/` route is the current OpenAI-compatible surface; older code uses `/openai/deployments/{deployment}/chat/completions?api-version=...`. Both exist in the wild. Topic 15 covers the API surfaces.

### When to drop to REST

| Situation | Approach |
|---|---|
| A feature exists only in a new or preview API version | `az rest`, or the SDK's low-level `send_request`, or `requests`/`httpx` with a token |
| No SDK for your language | REST with an identity library for tokens |
| Debugging what the SDK sends | Enable HTTP logging, or reproduce with `curl` |
| A lightweight script | `az rest` |

Each resource type's REST reference on Microsoft Learn includes a *Try it* button that runs the call with your credentials. Service APIs are described in OpenAPI or TypeSpec in the `Azure/azure-rest-api-specs` repository.

### A complete keyless RAG call

```python
# pip install azure-identity azure-search-documents openai
import os
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from azure.search.documents import SearchClient
from openai import AzureOpenAI

credential = DefaultAzureCredential()

search = SearchClient(
    endpoint=os.environ["AZURE_SEARCH_ENDPOINT"],
    index_name=os.environ["AZURE_SEARCH_INDEX"],
    credential=credential,
)

llm = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    azure_ad_token_provider=get_bearer_token_provider(credential, "https://cognitiveservices.azure.com/.default"),
    api_version="2024-10-21",
    max_retries=3,
    timeout=60,
)

def answer(question: str) -> str:
    hits = search.search(search_text=question, top=5, select=["title", "content"])
    context = "\n\n".join(f"[{h['title']}]\n{h['content']}" for h in hits)
    resp = llm.chat.completions.create(
        model=os.environ["AZURE_OPENAI_DEPLOYMENT"],
        messages=[
            {"role": "system", "content": "Answer only from the sources. Cite titles in brackets. If the sources do not contain the answer, say so."},
            {"role": "user", "content": f"Sources:\n{context}\n\nQuestion: {question}"},
        ],
    )
    return resp.choices[0].message.content

print(answer("How many days of annual leave do new employees get?"))
```

Clients are created once at module level. The caller needs *Search Index Data Reader* and *Cognitive Services OpenAI User*. No key appears anywhere. Later topics improve every part of this: hybrid and vector retrieval, security trimming, streaming, tracing and evaluation.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| SDK or REST | SDK by default. REST for gaps |
| `openai` library or Azure-specific SDK for models | The `openai` library is the common choice for OpenAI-compatible endpoints and keeps code portable. Use the Foundry project SDK when you need project features: agents, connections, tracing, evaluations |
| Sync or async | Async for services handling concurrent requests; sync for scripts and batch jobs |
| Preview SDKs | Accept for features you need, pin exactly, isolate behind your own interface, and budget for breaking changes |
| Framework or SDK | An agent or orchestration framework saves work but adds abstraction. Know the underlying SDK calls so you can debug |
| Where clients are constructed | Once, at start-up, injected where needed |
| Configuration | Endpoints and deployment names from environment or App Configuration; no secrets |

## Hands-on

```bash
python -m venv .venv && source .venv/bin/activate
pip install azure-identity azure-mgmt-resource azure-storage-blob

RG=rg-sdk-lab; LOC=swedencentral; ST=stsdklab$RANDOM
az group create -n $RG -l $LOC -o none
az storage account create -n $ST -g $RG -l $LOC --sku Standard_LRS --allow-shared-key-access false -o none
az role assignment create --assignee-object-id $(az ad signed-in-user show --query id -o tsv) \
  --assignee-principal-type User --role "Storage Blob Data Contributor" \
  --scope $(az storage account show -n $ST -g $RG --query id -o tsv) -o none
export ST SUB=$(az account show --query id -o tsv)
```

```python
# lab.py
import os, logging
from azure.identity import DefaultAzureCredential
from azure.mgmt.resource import ResourceManagementClient
from azure.storage.blob import BlobServiceClient
from azure.core.exceptions import ResourceExistsError, ResourceNotFoundError, HttpResponseError

cred = DefaultAzureCredential()

# 1. Control plane with paging
arm = ResourceManagementClient(cred, os.environ["SUB"])
for i, rg in enumerate(arm.resource_groups.list()):
    print(rg.name, rg.location)
    if i == 9: break

# 2. Data plane, keyless
svc = BlobServiceClient(f"https://{os.environ['ST']}.blob.core.windows.net", credential=cred, retry_total=5)
container = svc.get_container_client("docs")
try:
    container.create_container()
except ResourceExistsError:
    pass

for n in range(5):
    container.upload_blob(f"notes/{n}.txt", f"note {n}", overwrite=True)

# 3. Paging explicitly
for page in container.list_blobs(name_starts_with="notes/", results_per_page=2).by_page():
    print([b.name for b in page])

# 4. Error handling with request ID
try:
    container.get_blob_client("missing.txt").download_blob()
except ResourceNotFoundError as e:
    print("404", e.error_code, e.response.headers.get("x-ms-request-id"))

# 5. Long-running operation (control plane)
poller = arm.resource_groups.begin_delete("rg-does-not-exist-xyz")
try:
    poller.result()
except HttpResponseError as e:
    print(e.status_code, e.error.code)

# 6. See the HTTP traffic
logging.basicConfig(level=logging.INFO)
BlobServiceClient(f"https://{os.environ['ST']}.blob.core.windows.net", credential=cred, logging_enable=True) \
    .get_service_properties()
```

```bash
python lab.py

# The same data-plane call with raw REST
TOKEN=$(az account get-access-token --resource https://storage.azure.com --query accessToken -o tsv)
curl -s "https://$ST.blob.core.windows.net/docs?restype=container&comp=list" \
  -H "Authorization: Bearer $TOKEN" -H "x-ms-version: 2023-11-03" | head -c 600

az group delete -n $RG --yes --no-wait
```

If you have a Foundry model deployment and a search index, run the RAG example above as well. Extension: convert `lab.py` to async with the `.aio` clients.

## Security, cost and reliability notes

- **Security:** no keys or connection strings in code or configuration. Endpoints plus `azure-identity`.
- **Security:** be careful with SDK and HTTP logging levels in production; bodies can contain prompts and personal data.
- **Security:** keep dependencies patched. Use lock files and automated update pull requests.
- **Cost:** retries on model calls consume tokens each time. Uncontrolled retry loops and unbounded concurrency are a common cause of surprise bills and self-inflicted throttling.
- **Reliability:** set timeouts everywhere. A missing timeout turns a slow dependency into a hung application.
- **Reliability:** reuse clients; handle 429 with backoff; make writes idempotent where a retry could duplicate.

## Common mistakes

- Creating a new client and credential for every request.
- Connection strings and API keys because the first sample used them.
- Wrapping SDK calls in extra retry loops.
- No timeouts.
- Loading an entire paged collection into memory.
- Sync clients inside async request handlers.
- Catching every exception and discarding the request ID.
- Treating 403 as purely an RBAC issue and never checking network rules.
- Unpinned preview packages.
- Using management libraries in application code to build infrastructure.

## Check yourself

1. What is the difference between `azure-mgmt-storage` and `azure-storage-blob`?
2. Why should clients and credentials be created once and reused?
3. What does the SDK's default retry policy do on a 429?
4. How do you recognise and use a long-running operation in the SDK?
5. A call returns 403 although the identity has the right role. What else should you check?
6. Why log `x-ms-request-id`?
7. When is async the better choice?
8. Name three REST conventions shared across Azure Resource Manager APIs.

<details>
<summary>Answers</summary>

1. The first is a management (control-plane) library for creating and configuring storage accounts; the second is a data-plane client for working with blobs.
2. They hold connection pools and cached tokens and are thread-safe; recreating them wastes connections and token requests.
3. Waits according to `Retry-After` or exponential backoff with jitter, then retries up to the configured limit.
4. The method starts with `begin_` and returns a poller; call `result()` to wait, or check `status()` and persist the continuation token.
5. Network access: whether public access is disabled and the caller is outside the allowed network or private endpoint path. Also role propagation delay and correct scope.
6. It uniquely identifies the request so the service team and support can trace it.
7. For I/O-bound services handling many concurrent requests, such as an API that waits on model and search calls.
8. Any three of: mandatory `api-version`, idempotent `PUT`, asynchronous operations with a polling header, `value`/`nextLink` paging, a standard error envelope, 429 with `Retry-After`.

</details>

## References

- [Azure SDK overview and releases](https://azure.github.io/azure-sdk/)
- [Azure SDK design guidelines](https://azure.github.io/azure-sdk/general_introduction.html)
- [Use the Azure libraries (SDK) for Python](https://learn.microsoft.com/azure/developer/python/sdk/azure-sdk-overview)
- [Azure Identity client library for Python](https://learn.microsoft.com/python/api/overview/azure/identity-readme)
- [Azure Core shared client library for Python](https://learn.microsoft.com/python/api/overview/azure/core-readme)
- [Retry guidance for Azure services](https://learn.microsoft.com/azure/architecture/best-practices/retry-service-specific)
- [Azure REST API reference](https://learn.microsoft.com/rest/api/azure/)
- [Microsoft Foundry SDKs and endpoints](https://learn.microsoft.com/azure/foundry/how-to/develop/sdk-overview)
- [Azure Monitor OpenTelemetry Distro](https://learn.microsoft.com/azure/azure-monitor/app/opentelemetry-enable)
- [Azure REST API specifications](https://github.com/Azure/azure-rest-api-specs)
