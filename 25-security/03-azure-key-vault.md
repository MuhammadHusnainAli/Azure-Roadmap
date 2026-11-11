# Azure Key Vault

## Why it matters

Leaked credentials are one of the most common causes of cloud breaches: connection strings in Git, keys in app settings, certificates emailed between teams and never rotated. **Azure Key Vault** centralises secrets, encryption keys and certificates behind Entra authentication, RBAC, network controls, logging and rotation. It's also the key store for customer-managed key (CMK) encryption across Azure services (file 04). For stricter requirements, **Managed HSM** and **Azure Cloud HSM** provide single-tenant, FIPS 140-3 Level 3 HSMs. The best Key Vault strategy, though, is to need it less: managed identities remove most secrets entirely, and Key Vault holds only what remains.

## Key concepts

### Object types

| Object | Examples | Notes |
|---|---|---|
| **Secrets** | Third-party API keys, passwords, connection strings that can't use Entra | Versioned; expiry and activation dates |
| **Keys** | RSA/EC keys for CMK encryption, signing, wrapping | Software-protected or HSM-protected; operations performed inside Key Vault |
| **Certificates** | TLS certificates with private keys | Lifecycle management; auto-renewal with integrated CAs or self-signed |

### Service options

| Option | Tenancy | Protection | Use |
|---|---|---|---|
| **Key Vault Standard** | Multitenant | Software keys (FIPS 140 Level 1 validated modules) | Secrets, certificates, software keys |
| **Key Vault Premium** | Multitenant | HSM-protected keys (FIPS 140-3 Level 3 validated HSMs); RSA-HSM, EC-HSM, OCT-HSM (Preview) | CMK requiring HSM protection |
| **Managed HSM** | **Single-tenant**, fully managed | FIPS 140-3 Level 3; customer-controlled **security domain**; local RBAC | High-assurance key management, regulated CMK, keys only (no secrets/certs) |
| **Azure Cloud HSM** | Single-tenant cluster of three HSMs; customer has full administrative authority | FIPS 140-3 Level 3 | PKCS#11, TLS offload, CA private keys, TDE for IaaS apps needing direct HSM access |
| **Azure Payment HSM** | Dedicated | Payment certifications | PIN processing, EMV, payment keys |

### Access control

| Model | Recommendation |
|---|---|
| **Azure RBAC (data plane)** | Recommended: roles such as Key Vault Secrets User, Key Vault Crypto User, Key Vault Certificates Officer, Key Vault Administrator; assign at vault or object scope |
| **Access policies** (legacy) | Avoid for new vaults; migrate to RBAC |
| **Managed HSM local RBAC** | Roles inside the HSM (Crypto User, Crypto Officer); management plane via Azure RBAC |

Separate management plane (who can create/configure vaults) from data plane (who can read secrets or use keys).

### Protection and recovery

| Feature | Behaviour |
|---|---|
| **Soft delete** | On by default; deleted vaults and objects recoverable for 7–90 days (default 90) |
| **Purge protection** | Prevents permanent deletion during retention; required for CMK scenarios; can't be disabled once on |
| **Backup** | Object-level backup/restore within the same geography |
| **Logging** | Diagnostic settings to Log Analytics (`AzureDiagnostics`/`AKVAuditLogs`); Event Grid events for near-expiry and expiry |

### Rotation

| Object | Mechanism |
|---|---|
| **Keys** | **Key rotation policy**: automatic rotation on schedule, with notification before expiry; CMK-consuming services pick up new versions (with versionless key URIs) |
| **Certificates** | Auto-renewal through integrated CAs (DigiCert, GlobalSign) or your CA; consumers using Key Vault references update automatically |
| **Secrets** | Event Grid `SecretNearExpiry` event → Function or Logic App rotates the credential at its source and stores the new version |

### Network security

Disable public network access and use **private endpoints**; or restrict with firewall rules and "trusted Microsoft services" bypass where needed. Key Vault traffic stays on the Microsoft backbone with private endpoints and private DNS zone `privatelink.vaultcore.azure.net`.

## How it works on Azure

Bicep for a hardened vault with RBAC, purge protection and private access:

```bicep
param location string = resourceGroup().location
param appPrincipalId string

resource kv 'Microsoft.KeyVault/vaults@2023-07-01' = {
  name: 'kv-contoso-app-prod'
  location: location
  properties: {
    tenantId: tenant().tenantId
    sku: { family: 'A', name: 'premium' }
    enableRbacAuthorization: true
    enableSoftDelete: true
    softDeleteRetentionInDays: 90
    enablePurgeProtection: true
    publicNetworkAccess: 'Disabled'
    networkAcls: { defaultAction: 'Deny', bypass: 'AzureServices' }
  }
}

// Key Vault Secrets User (built-in role definition ID)
resource secretsUser 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(kv.id, appPrincipalId, 'secrets-user')
  scope: kv
  properties: {
    roleDefinitionId: subscriptionResourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6')
    principalId: appPrincipalId
    principalType: 'ServicePrincipal'
  }
}
```

Reading a secret with a managed identity:

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

client = SecretClient("https://kv-contoso-app-prod.vault.azure.net", DefaultAzureCredential())
partner_api_key = client.get_secret("partner-api-key").value
```

Better still, use **Key Vault references** in App Service, Functions, Container Apps or App Configuration (`@Microsoft.KeyVault(SecretUri=...)`) so code never handles the secret, and cache values to avoid throttling.

Key rotation policy:

```bash
az keyvault key rotation-policy update --vault-name kv-contoso-app-prod --name cmk-storage \
  --value '{"lifetimeActions":[{"trigger":{"timeAfterCreate":"P90D"},"action":{"type":"Rotate"}},{"trigger":{"timeBeforeExpiry":"P30D"},"action":{"type":"Notify"}}],"attributes":{"expiryTime":"P1Y"}}'
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Vault per app per environment** | Limits blast radius and simplifies RBAC; avoid one shared vault for everything. |
| **Eliminate secrets** | Managed identities and workload identity federation first; Key Vault for the remainder. |
| **Standard vs Premium vs Managed HSM** | Standard for secrets; Premium for HSM-backed CMK; Managed HSM for single-tenant, customer-controlled security domain and regulated workloads; Cloud HSM for direct HSM APIs (PKCS#11). |
| **RBAC vs access policies** | RBAC with PIM for administrators. |
| **Private access** | Private endpoints for production; consider deployment agents' network path. |
| **Throttling** | Cache secrets; use references; don't call Key Vault per request. |

## Hands-on

1. Deploy the Bicep vault; grant your app's managed identity Secrets User; read a secret from code.
2. Replace an app setting connection string with a Key Vault reference.
3. Create a key with a rotation policy and use it as CMK for a storage account with a versionless key URI.
4. Subscribe to `SecretNearExpiry` with Event Grid and trigger a Function.
5. Send diagnostic logs to Log Analytics and query who accessed a secret.

## Security, cost and reliability notes

- **Purge protection** on vaults holding CMKs; losing a key means losing data.
- **Least privilege**: Secrets User for apps; Administrator only via PIM.
- **Monitoring**: alert on unusual access, failed access, and near-expiry; Defender for Key Vault detects anomalous access.
- **Availability**: Key Vault replicates within region and to the paired region for read-only failover; Managed HSM supports multi-region replication.
- **Cost**: per operation (Standard/Premium), per HSM key; Managed HSM and Cloud HSM are hourly and significantly more expensive.

## Common mistakes

| Mistake | Fix |
|---|---|
| Storing secrets that could be managed identities | Use managed identities |
| One vault shared by many apps and environments | Vault per app per environment |
| Access policies granting "all" permissions | RBAC with narrow roles |
| No purge protection for CMK vaults | Enable it |
| Fetching secrets on every request | Cache or use references |

## Check yourself

1. What object types does Key Vault store?
2. Compare Standard, Premium, Managed HSM and Cloud HSM.
3. Why prefer Azure RBAC over access policies?
4. What do soft delete and purge protection do?
5. How are keys, certificates and secrets rotated?
6. How do you keep secrets out of application code entirely?

<details>
<summary>Answers</summary>

1. Secrets, keys and certificates.
2. Standard: multitenant, software keys. Premium: multitenant with FIPS 140-3 Level 3 HSM-protected keys. Managed HSM: single-tenant, customer security domain, keys only. Cloud HSM: single-tenant HSM cluster with full customer administrative control and direct HSM APIs.
3. Fine-grained, consistent Azure permissions, scoping to objects, PIM integration and separation of management and data planes.
4. Soft delete keeps deleted items recoverable for 7–90 days; purge protection prevents permanent deletion during that period.
5. Keys via rotation policies; certificates via auto-renewal with integrated CAs; secrets via near-expiry events and automation that rotates at the source.
6. Managed identities, and Key Vault references in platform settings so the platform resolves secrets.

</details>

## References

- [About Azure Key Vault](https://learn.microsoft.com/azure/key-vault/general/overview)
- [Managed HSM overview](https://learn.microsoft.com/azure/key-vault/managed-hsm/overview)
- [What is Azure Cloud HSM?](https://learn.microsoft.com/azure/cloud-hsm/overview)
- [Azure RBAC for Key Vault](https://learn.microsoft.com/azure/key-vault/general/rbac-guide)
- [Key autorotation](https://learn.microsoft.com/azure/key-vault/keys/how-to-configure-key-rotation)
- [Soft delete overview](https://learn.microsoft.com/azure/key-vault/general/soft-delete-overview)
