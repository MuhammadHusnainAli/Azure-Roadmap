# Security Trimming and Permissions

## Why it matters

The fastest way to turn a RAG pilot into a security incident is to index the HR SharePoint site and let everyone ask the chatbot about salaries. Language models can't be trusted to hide content they've been given: anything in the prompt can be revealed. **Security trimming** means each user only retrieves documents they are allowed to see, enforced **in the retrieval engine** before anything reaches the model. Azure AI Search now supports native document-level permissions using Microsoft Entra tokens, ACLs, SharePoint permissions and Purview sensitivity labels, alongside the classic security-filter pattern. Architects own this design; consultants must raise it in every RAG engagement.

## Key concepts

### Two separate layers of access

| Layer | Question | Mechanism |
|---|---|---|
| **Service access** | Can this app call the index at all? | Entra RBAC: Search Index Data Reader / Contributor |
| **Document access** | Which documents may *this user* see in results? | Security filters, native ACL/RBAC scopes, SharePoint ACLs, Purview labels |

An app with Search Index Data Reader can read everything unless document-level trimming is applied.

### Four approaches in Azure AI Search

| Approach | How it works | Status | Best for |
|---|---|---|---|
| **Security filters** | App stores user/group IDs in a filterable string field and adds `$filter` with the caller's identities | **GA**, any API version | Custom identity systems, non-Microsoft security, push indexes |
| **POSIX-like ACL / RBAC scopes** | Index stores Entra permission metadata; the query carries the user's token in `x-ms-query-source-authorization`; the engine compares claims to metadata | Preview | ADLS Gen2 (ACLs and RBAC scopes) and Blob (RBAC scopes) |
| **SharePoint in Microsoft 365 ACLs** | SharePoint indexer ingests permission metadata (including site groups); token-based enforcement at query time | Preview | Indexed SharePoint content |
| **Microsoft Purview sensitivity labels** | Indexer extracts labels from Blob, ADLS Gen2, SharePoint, OneLake; query-time evaluation of token, label and Purview policies | Preview | Content governed by Purview information protection |

### Token-based query enforcement

For the native approaches, the app sends the **user's** Entra token in the `x-ms-query-source-authorization` header. Azure AI Search:

1. Checks the app's own RBAC (Search Index Data Reader).
2. Extracts user, group and scope claims from the user token.
3. Compares them to the permission metadata stored with each document.
4. Returns only authorised documents.

Permission changes in the source (group membership, ACLs, labels) take effect **only after the metadata is re-synchronised** to the index (indexer run, push update, Purview refresh). Design for that lag.

### Remote sources

Remote SharePoint knowledge sources (agentic retrieval) enforce permissions directly through the Copilot Retrieval API, including ACLs and sensitivity labels, with no indexing. Work IQ is permission-trimmed by Microsoft 365 (file 09).

### Chunks inherit permissions

When a skillset chunks documents, permission fields must be projected onto **every chunk** via index projections. Otherwise chunks have no ACLs and are either unfiltered (leak) or unreturned.

## How it works on Azure

### Security filter pattern (GA)

```text
Index field:   group_ids  Collection(Edm.String), filterable, not retrievable
At query:      resolve caller's groups (token claims or Microsoft Graph)
Filter:        group_ids/any(g: search.in(g, 'a1b2,c3d4,e5f6'))
```

Use `search.in` for efficient matching against many IDs. Filters run during retrieval and, with `preFilter` mode, inside vector search (file 05).

Watch out for the **groups overage** problem: tokens include at most a limited number of group IDs; users in many groups get an overage claim instead, so resolve groups via Microsoft Graph (`transitiveMemberOf`) in that case.

### Native ACL pattern (Preview)

1. Index schema created with a preview API, with permission filter fields (for example `userIds`, `groupIds`, `rbacScope`) and `permissionFilterOption` enabled.
2. Ingest with the ADLS Gen2 indexer (or Blob knowledge source) using a preview API, or push documents with ACL metadata.
3. Query with the user's token in `x-ms-query-source-authorization`.

### Sensitivity labels (Preview)

- Requires the search service's **system-assigned** managed identity with elevated Purview permissions (granted by a Global or Privileged Role Administrator).
- Single-tenant only; RBAC authentication required; REST and SDKs only during preview; autocomplete and suggest are unavailable on Purview-enabled indexes.
- In agentic retrieval, set `ingestionPermissionOptions` to include `sensitivityLabel`; responses include per-reference `sensitivityLabelInfo` you can use for banners.

### In agents

For a Foundry agent using Foundry IQ, pass the user token via structured inputs into the MCP tool's `x-ms-query-source-authorization` header ([file 08](08-foundry-iq.md)). The agent itself should run with the user's delegated identity where possible ([Topic 18 file 13](../18-agents-with-foundry-agent-service/13-agent-identity-and-authorisation.md)).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Filters vs native ACLs** | Security filters are GA and flexible but you own group resolution and sync. Native ACLs reuse Entra, SharePoint and Purview but are Preview and source-specific. |
| **Users vs groups** | Store groups, not individual users, wherever possible; fewer IDs and fewer reindexes. |
| **Sync frequency** | Match permission change sensitivity. Revoked access persists in search until the next sync; for high-risk content, sync frequently or add a post-retrieval check. |
| **Separate indexes instead of filters** | For a few large, hard boundaries (for example per business unit or per tenant), separate indexes are simpler and safer than filters (file 13). |
| **Indexed vs remote SharePoint** | Remote enforces live permissions with no copy; indexed gives control and hybrid ranking but introduces sync lag. |
| **Label-aware responses** | Surface sensitivity labels in the UI and prevent agents from sending labelled content to external tools. |

## Hands-on

Implement GA security filters for a push index.

```python
from azure.identity import DefaultAzureCredential
from azure.search.documents import SearchClient
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import SearchIndex, SimpleField, SearchableField, SearchFieldDataType

endpoint, cred = "https://<search>.search.windows.net", DefaultAzureCredential()
SearchIndexClient(endpoint, cred).create_or_update_index(SearchIndex(name="hr-secure", fields=[
    SimpleField(name="id", type=SearchFieldDataType.String, key=True),
    SearchableField(name="content", type=SearchFieldDataType.String),
    SimpleField(name="group_ids", type=SearchFieldDataType.Collection(SearchFieldDataType.String),
                filterable=True, retrievable=False),
]))

client = SearchClient(endpoint, "hr-secure", cred)
client.upload_documents([
    {"id": "1", "content": "Salary bands for 2026 ...", "group_ids": ["grp-hr-managers"]},
    {"id": "2", "content": "Annual leave is 25 days ...", "group_ids": ["grp-all-staff"]},
])

def search_as(user_groups: list[str], q: str):
    flt = "group_ids/any(g: search.in(g, '{}'))".format(",".join(user_groups))
    return [d["id"] for d in client.search(search_text=q, filter=flt, select=["id"])]

print(search_as(["grp-all-staff"], "salary"))                       # [] or only doc 2
print(search_as(["grp-all-staff", "grp-hr-managers"], "salary"))    # includes doc 1
```

In a real app, take `user_groups` from the validated user token (`groups` claim) or Microsoft Graph, never from the request body. Then try the preview native pattern with the [push API quickstart](https://github.com/Azure-Samples/azure-search-python-samples) for document permissions and compare effort.

## Security, cost and reliability notes

- **Never trust client-supplied group lists.** Derive identities server-side from validated tokens.
- **Mark ACL fields non-retrievable** so permission data isn't returned to clients.
- **Test with negative cases**: a regression test asserting user X cannot retrieve document Y should run in CI.
- **Logs and caches**: retrieved content in traces, caches or conversation memory must also respect permissions; a shared cache keyed only by question can leak answers between users.
- **Agent memory**: don't store retrieved restricted content in shared or long-term memory.
- **Performance**: filtering inside the engine is faster than post-query trimming in your app, especially at high QPS.

## Common mistakes

| Mistake | Fix |
|---|---|
| Relying on the system prompt to hide content | Trim at retrieval |
| App identity with full read access and no document filters | Add filters or token-based enforcement |
| ACLs on parent documents but not on chunks | Project ACL fields to every chunk |
| Assuming permission changes apply instantly | Plan for sync lag; resync frequently |
| Ignoring group overage in tokens | Resolve via Microsoft Graph when overage claims appear |
| Shared semantic cache across users | Scope caches by permission set or disable for restricted content |

## Check yourself

1. Why is RBAC on the search service not enough for RAG over sensitive content?
2. Name the four document-level access approaches and their status.
3. What does the `x-ms-query-source-authorization` header carry and how is it used?
4. When do permission changes in the source take effect in search results?
5. Write a filter that matches documents whose `group_ids` contains any of three groups.
6. What must you do with ACL fields when documents are chunked?
7. What are three preview limitations of Purview sensitivity label enforcement?
8. Give two places outside the index where restricted content can leak.

<details>
<summary>Answers</summary>

1. RBAC controls whether an app can query the index, not which documents each end user may see.
2. Security filters (GA), POSIX-like ACL/RBAC scopes (Preview), SharePoint ACLs (Preview), Purview sensitivity labels (Preview).
3. The end user's Entra token; the service compares its user, group and scope claims with each document's permission metadata and returns only authorised documents.
4. After the permission metadata is re-synchronised to the index (indexer run, push update or Purview refresh).
5. `group_ids/any(g: search.in(g, 'g1,g2,g3'))`.
6. Project them onto every chunk via index projections.
7. Any three: single-tenant only; requires RBAC authentication; REST/SDK only; no autocomplete or suggest; requires the system-assigned identity with elevated Purview permissions.
8. Traces and logs, caches, conversation or long-term agent memory, tool calls to external services.

</details>

## References

- [Document-level access control in Azure AI Search](https://learn.microsoft.com/azure/search/search-document-level-access-overview)
- [Security filters for trimming results](https://learn.microsoft.com/azure/search/search-security-trimming-for-azure-search)
- [Query-time ACL and RBAC enforcement](https://learn.microsoft.com/azure/search/search-query-access-control-rbac-enforcement)
- [Ingest Purview sensitivity labels](https://learn.microsoft.com/azure/search/search-indexer-sensitivity-labels)
- [Connect using role-based access control](https://learn.microsoft.com/azure/search/search-security-rbac)
- [azure-search-python-samples](https://github.com/Azure-Samples/azure-search-python-samples)
