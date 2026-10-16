# Database Security and Compliance

> Topic: 09 · Databases · Tracks: E ○ A ● C ● · Status checked: 2026-10

## Why it matters

Databases hold the data attackers want and regulators care about: customer records, financial transactions, health data, and increasingly the conversation logs and memories of AI systems. Azure databases provide strong controls for identity, network isolation, encryption, auditing, tamper evidence and threat detection, but most are configuration choices. Architects must design them in, engineers must use them, and consultants must be able to map them to compliance requirements.

## Key concepts

### The control layers

| Layer | Controls |
|---|---|
| **Identity** | Microsoft Entra authentication, managed identities, Entra-only mode, least-privilege database roles |
| **Network** | Private endpoints or VNet integration, public access disabled, firewall rules as a fallback |
| **Encryption at rest** | Transparent Data Encryption (TDE) or service-side encryption with service-managed or customer-managed keys |
| **Encryption in transit** | TLS 1.2+ enforced |
| **Encryption in use** | Always Encrypted (with secure enclaves), confidential computing options |
| **Data-level access** | Row-level security, column permissions, dynamic data masking |
| **Auditing and tamper evidence** | Auditing to Log Analytics or storage; ledger tables; Azure confidential ledger |
| **Threat protection** | Microsoft Defender for SQL, open-source relational databases and Cosmos DB |
| **Governance** | Azure Policy, Microsoft Purview (discovery, classification, lineage), Defender CSPM |

### Identity

| Service | Entra support | Disable passwords |
|---|---|---|
| Azure SQL Database / MI | Users, groups, service principals, managed identities | **Microsoft Entra-only authentication** |
| PostgreSQL / MySQL flexible server | Users, groups, managed identities | Disable password auth (PostgreSQL); restrict local users (MySQL) |
| Cosmos DB | Data-plane RBAC with Entra | `disableLocalAuth: true` |
| DocumentDB | Entra (check driver support) | Restrict native users |
| Managed Redis | Entra data access policies | Disable access keys |

Principles: one identity per application; grant the narrowest database roles (`db_datareader`, specific schemas or containers); administer through Entra groups with PIM; no shared admin passwords in configuration.

### Encryption options in Azure SQL

| Feature | Protects | Notes |
|---|---|---|
| **TDE** | Data files, logs, backups at rest | On by default; service-managed or **customer-managed key** (TDE protector in Key Vault or Managed HSM) |
| **Always Encrypted** | Specific columns, from the database and its administrators | Encryption keys held by the client; DBAs see ciphertext |
| **Always Encrypted with secure enclaves** | Same, plus rich queries (comparisons, pattern matching) on encrypted columns inside a trusted enclave | VBS enclaves in Azure SQL Database |
| **Dynamic data masking** | Presentation only: masks values for non-privileged users | Not encryption; does not stop determined users with query access |
| **Row-level security** | Rows filtered per user or tenant | Essential for shared-schema multi-tenant apps |
| TLS | In transit | Minimum TLS version setting |

Customer-managed keys exist across Azure SQL, PostgreSQL, MySQL, Cosmos DB and Managed Redis. Use them where regulation requires key custody, and protect the Key Vault with soft delete and purge protection.

### Auditing

- **Azure SQL auditing** writes events (logins, queries, schema changes) to Log Analytics, Event Hubs or storage. Set at server level for all databases. Use action groups to control volume.
- **PostgreSQL** `pgaudit` extension and server logs; **MySQL** audit log parameters; **Cosmos DB** diagnostic logs (data-plane requests, control-plane operations).
- Retain audit logs according to policy, in a workspace the database administrators cannot alter.

### Tamper evidence: ledger

| Feature | Detail |
|---|---|
| **Ledger tables** (Azure SQL, SQL Server 2022+) | **Updatable** ledger tables keep history; **append-only** ledger tables allow inserts only. A cryptographic hash chain over transactions (blockchain-style) proves no tampering |
| **Database digests** | Periodically stored in immutable Blob Storage or Azure confidential ledger; verify with `sys.sp_verify_database_ledger` |
| **Azure confidential ledger** | Managed, tamper-proof ledger service running in confidential computing enclaves (based on the Confidential Consortium Framework). Use for audit records and digests that even administrators must not alter |

Use ledger for financial records, audit trails, supply chain records, and AI decision logs where you must prove integrity.

### Threat protection

**Microsoft Defender for Cloud** database plans:

| Plan | Covers |
|---|---|
| Defender for Azure SQL / SQL servers on machines | Vulnerability assessment, SQL injection detection, anomalous access, brute force |
| Defender for open-source relational databases | PostgreSQL, MySQL (and MariaDB where applicable) |
| Defender for Azure Cosmos DB | Anomalous access, potential SQL injection, data exfiltration patterns |

Pair with **Microsoft Purview** to discover and classify sensitive data (personal data, financial data) and track lineage, and with Defender CSPM for attack path analysis.

### Compliance mapping

| Requirement | Azure capability |
|---|---|
| Encrypt personal data at rest with customer-controlled keys | TDE or service encryption with CMK |
| Limit administrator access to sensitive columns | Always Encrypted |
| Prove records were not altered | Ledger tables, confidential ledger |
| Audit all access | Auditing to Log Analytics with retention |
| Data residency | Region choice, geo-replication only to approved regions, policy restricting locations |
| Right to erasure | Delete paths for all copies including replicas, caches, search indexes, backups (backups expire by retention) |
| Least privilege | Entra groups, PIM, database roles, RLS |

Azure's compliance offerings (ISO 27001, SOC, PCI DSS, HIPAA, regional frameworks) cover the platform; you remain responsible for configuration (shared responsibility, topic 01).

## How it works on Azure

### Azure SQL hardening checklist

```bash
# Entra-only authentication
az sql server ad-only-auth enable -g $RG -n $SRV

# Disable public network access (after private endpoint is in place)
az sql server update -g $RG -n $SRV --set publicNetworkAccess=Disabled
az sql server update -g $RG -n $SRV --minimal-tls-version 1.2

# Server-level auditing to Log Analytics
az sql server audit-policy update -g $RG -n $SRV --state Enabled --lat-enabled true \
  --lat-workspace-resource-id $WORKSPACE_ID

# Defender for SQL on the server
az sql server advanced-threat-protection-setting update -g $RG -n $SRV --state Enabled
```

### Row-level security for multi-tenant data

```sql
CREATE SCHEMA sec;
GO
CREATE FUNCTION sec.fn_tenant(@TenantId INT)
RETURNS TABLE WITH SCHEMABINDING
AS RETURN SELECT 1 AS ok WHERE @TenantId = CAST(SESSION_CONTEXT(N'TenantId') AS INT);
GO
CREATE SECURITY POLICY sec.TenantFilter
ADD FILTER PREDICATE sec.fn_tenant(TenantId) ON dbo.Orders,
ADD BLOCK PREDICATE sec.fn_tenant(TenantId) ON dbo.Orders
WITH (STATE = ON);
-- The app sets: EXEC sp_set_session_context @key = N'TenantId', @value = 42;
```

### An append-only ledger table

```sql
CREATE TABLE dbo.AgentDecisions (
    DecisionId BIGINT IDENTITY PRIMARY KEY,
    AgentId NVARCHAR(100), Action NVARCHAR(200), Approver NVARCHAR(200), DecidedAt DATETIME2
) WITH (LEDGER = ON (APPEND_ONLY = ON));

EXECUTE sys.sp_generate_database_ledger_digest;
-- Later: EXECUTE sys.sp_verify_database_ledger N'<digest json>';
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| CMK or service-managed keys | Service-managed by default; CMK when regulation or contracts require key custody, accepting the operational risk |
| Always Encrypted | For a few highly sensitive columns that DBAs must not see; plan for query limitations (enclaves help) |
| Masking vs RLS vs encryption | Masking for casual exposure in tools; RLS for tenant or user isolation; encryption for protection from privileged users |
| Ledger | For records where integrity proof has business or regulatory value |
| Audit volume | Audit what you need; use action groups and filters to control cost |
| Network | Private endpoints and public access disabled for production; controlled admin access paths |

## Hands-on

1. On the Azure SQL database from file 02, enable Entra-only authentication and confirm SQL logins fail.
2. Create the RLS policy above on a sample `Orders` table and test with two `SESSION_CONTEXT` values.
3. Enable auditing to a Log Analytics workspace, run a few queries, and query the audit log:

```kusto
AzureDiagnostics
| where Category == "SQLSecurityAuditEvents"
| project TimeGenerated, server_principal_name_s, action_name_s, statement_s
| order by TimeGenerated desc
| take 20
```

4. Create the ledger table, insert rows, generate a digest, then verify it.
5. Enable Defender for SQL on the server and review the vulnerability assessment findings.

## Security, cost and reliability notes

- **Security:** the most common database breaches come from exposed endpoints and leaked credentials. Private networking and Entra-only auth close both.
- **Security:** remember all copies: replicas, backups, exports, caches, search indexes, Fabric mirrors and analytics copies inherit the obligation to protect.
- **Cost:** auditing and Defender plans have ongoing costs; scope audit events and choose plans per risk.
- **Reliability:** CMK misconfiguration (deleted key, lost access) makes databases inaccessible; monitor key health and enable Key Vault protections.

## Common mistakes

- Shared SQL admin passwords in application settings.
- "Allow Azure services" firewall exceptions treated as private networking.
- Relying on dynamic data masking as a security control against users who can run queries.
- Multi-tenant apps filtering by tenant only in application code, without RLS.
- Audit logs stored where database admins can delete them.
- Forgetting to secure Fabric mirrors, exports and AI search indexes built from the database.

## Check yourself

1. What does Microsoft Entra-only authentication do?
2. Compare TDE, Always Encrypted and dynamic data masking.
3. What do secure enclaves add to Always Encrypted?
4. How do ledger tables provide tamper evidence?
5. What is Azure confidential ledger used for?
6. How does row-level security help multi-tenant applications?
7. Which Defender plans cover Azure databases?

<details>
<summary>Answers</summary>

1. It disables SQL authentication on the server so only Entra identities can connect.
2. TDE encrypts data at rest transparently (files, logs, backups); Always Encrypted encrypts specific columns with client-held keys so the database and its admins see only ciphertext; dynamic data masking only obscures values in query results for non-privileged users and is not encryption.
3. The ability to run richer operations (comparisons, range queries, pattern matching) on encrypted data inside a trusted enclave.
4. Transactions are hashed into a cryptographic chain; database digests stored in immutable storage or confidential ledger let you verify that no rows or history were altered.
5. Storing tamper-proof records, such as audit logs and database digests, in a managed ledger running in confidential computing enclaves.
6. The database enforces that each session only sees and modifies rows for its tenant, independent of application bugs.
7. Defender for Azure SQL (and SQL servers on machines), Defender for open-source relational databases, and Defender for Azure Cosmos DB.

</details>

## References

- [Azure SQL security overview](https://learn.microsoft.com/azure/azure-sql/database/security-overview)
- [Microsoft Entra-only authentication](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-only-authentication)
- [TDE with customer-managed keys](https://learn.microsoft.com/azure/azure-sql/database/transparent-data-encryption-byok-overview)
- [Always Encrypted with secure enclaves](https://learn.microsoft.com/sql/relational-databases/security/encryption/always-encrypted-enclaves)
- [Row-level security](https://learn.microsoft.com/sql/relational-databases/security/row-level-security)
- [Ledger overview](https://learn.microsoft.com/sql/relational-databases/security/ledger/ledger-overview)
- [Azure confidential ledger](https://learn.microsoft.com/azure/confidential-ledger/overview)
- [Defender for databases](https://learn.microsoft.com/azure/defender-for-cloud/tutorial-enable-databases-plan)
- [Cosmos DB security baseline](https://learn.microsoft.com/security/benchmark/azure/baselines/azure-cosmos-db-security-baseline)
