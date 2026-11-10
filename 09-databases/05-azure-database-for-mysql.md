# Azure Database for MySQL

## Why it matters

MySQL runs a large share of the web: WordPress, Drupal, Magento, countless PHP and LAMP applications, and many SaaS products. When those workloads move to Azure, Azure Database for MySQL flexible server is the natural managed target. Architects and consultants meet it mainly in migrations; engineers meet it when an application or framework expects MySQL.

## Key concepts

### Deployment option

| Option | Status |
|---|---|
| **Flexible server** | The service |
| Single server | **Retired** (September 2024); migrate any remaining servers |

### Compute tiers

| Tier | Fit |
|---|---|
| **Burstable** (B-series) | Dev, test, small sites |
| **General Purpose** (D-series) | Most production workloads |
| **Business Critical** (E-series, formerly Memory Optimized) | High-performance and memory-intensive workloads, with faster I/O options |

Storage: Premium SSD with **autoscale IOPS** (pay for IOPS used) or **pre-provisioned IOPS**; storage autogrow. **Accelerated logs** (Business Critical) reduce commit latency for write-heavy workloads.

### Versions

MySQL 8.0 and 8.4 (LTS) are the main supported versions; MySQL 5.7 has reached end of community support and Azure offers **extended support** (charged) for a period. Plan major version upgrades; in-place major version upgrade is supported.

### High availability and replicas

| Feature | Detail |
|---|---|
| **Zone-redundant HA** | Standby in another zone, synchronous replication, automatic failover |
| Same-zone HA | Standby in the same zone |
| **Read replicas** | Asynchronous, same or cross-region, up to several per source; promote for DR |
| **Geo-redundant backup** | Restore in the paired region if the primary region is unavailable |
| Backups | Automatic, point-in-time restore 1–35 days; long-term retention options (check current status) |

### Other features

- **Data-in replication** from an external MySQL (on-premises or another cloud) for migration with minimal downtime.
- Server parameters for tuning; **slow query logs** and **audit logs** to Log Analytics.
- **Entra authentication**, VNet integration (private access) or private endpoints.
- **Stop and start** to save cost on non-production.
- **Customer-managed keys** for encryption at rest.
- Fabric mirroring and Azure Data Factory connectors for analytics (check mirroring status).

## How it works on Azure

### Migration paths

| Source | Method | Downtime |
|---|---|---|
| Single Server | Built-in migration tooling or Azure Database Migration Service (online) | Minimal |
| On-premises or other cloud MySQL, small | `mysqldump` / `mydumper` and restore | Offline |
| Larger, minimal downtime | **Azure Database Migration Service** (online), or dump and restore plus **data-in replication** for catch-up | Minimal |
| Import from an existing server snapshot | MySQL import (`az mysql flexible-server import create`) using a physical backup in Blob | Short |

### WordPress on Azure, typical pattern

```text
Front Door (CDN, WAF)
  → App Service (Linux, PHP) or Container Apps
      → Azure Database for MySQL flexible server (General Purpose, zone-redundant HA, private access)
      → Azure Files or Blob for media uploads
  App Service offers a "WordPress on App Service" template wiring these together.
```

### Connecting with Entra

The pattern matches PostgreSQL: acquire a token for `https://ossrdbms-aad.database.windows.net/.default`, use it as the password with the cleartext plugin over TLS, and refresh it before expiry. Create users for managed identities with `CREATE AADUSER 'app-identity' IDENTIFIED BY '<client-id>';` (syntax per current docs).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| MySQL or PostgreSQL for new apps | PostgreSQL for new apps unless the framework or team strongly prefers MySQL; MySQL for existing MySQL workloads |
| Tier | Burstable for dev; General Purpose for most production; Business Critical for write-heavy or memory-heavy workloads |
| IOPS | Autoscale IOPS for variable workloads; pre-provisioned for steady high I/O |
| HA | Zone-redundant HA for production |
| DR | Cross-region read replica (promotable) for low RPO; geo-redundant backup for lower cost and longer RTO |
| Version | MySQL 8.4 LTS for new deployments; plan to leave 5.7 |

## Hands-on

```bash
RG=rg-mysql-lab; LOC=swedencentral; MY=mysql-lab-$RANDOM
az group create -n $RG -l $LOC -o none

az mysql flexible-server create -g $RG -n $MY -l $LOC --tier Burstable --sku-name Standard_B1ms \
  --version 8.4 --storage-size 32 --admin-user labadmin --admin-password "$(openssl rand -base64 18)Aa1!" \
  --public-access $(curl -s ifconfig.me) -o none

# Inspect configuration and key parameters
az mysql flexible-server show -g $RG -n $MY --query "{version:version, tier:sku.tier, ha:highAvailability.mode, backup:backup}" -o json
az mysql flexible-server parameter show -g $RG -s $MY -n slow_query_log -o table

# Turn on slow query logging
az mysql flexible-server parameter set -g $RG -s $MY -n slow_query_log --value ON -o none

# Stop the server when not in use
az mysql flexible-server stop -g $RG -n $MY

az group delete -n $RG --yes --no-wait
```

Extension: configure a Microsoft Entra admin, connect with a token, and then disable password authentication.

## Security, cost and reliability notes

- **Security:** private access (VNet integration) or private endpoints; TLS 1.2 or later enforced; Entra auth for apps and admins.
- **Security:** enable audit logs for sensitive databases and send them to Log Analytics.
- **Cost:** stop dev servers; reserved capacity for production; autoscale IOPS avoids paying for unused provisioned IOPS.
- **Cost:** MySQL 5.7 extended support is a recurring extra cost; upgrade.
- **Reliability:** zone-redundant HA plus cross-region replica or geo-redundant backups; test failover and promotion.

## Common mistakes

- Remaining on retired Single Server or unsupported MySQL 5.7 without a plan.
- Admin credentials embedded in WordPress or app configuration with public access enabled.
- Using Burstable tier for busy production sites, exhausting CPU credits.
- Ignoring slow query logs when performance degrades.
- Assuming HA protects against regional outages.

## Check yourself

1. What are the three compute tiers of MySQL flexible server and their uses?
2. What is the difference between autoscale IOPS and pre-provisioned IOPS?
3. How do you achieve minimal-downtime migration from an external MySQL server?
4. What options exist for regional disaster recovery?
5. What happens to MySQL 5.7 servers in Azure?

<details>
<summary>Answers</summary>

1. Burstable for dev and light workloads; General Purpose for most production; Business Critical for high-performance, memory- or write-intensive workloads.
2. Autoscale IOPS scales and bills for IOPS actually used; pre-provisioned IOPS reserves a fixed level you pay for regardless of use.
3. Azure Database Migration Service in online mode, or dump and restore followed by data-in replication to catch up, then cutover.
4. Cross-region read replicas that can be promoted, or geo-redundant backups restored in the paired region.
5. They are on extended support, which is charged, for a limited period; they should be upgraded to 8.0 or 8.4.

</details>

## References

- [Azure Database for MySQL flexible server overview](https://learn.microsoft.com/azure/mysql/flexible-server/overview)
- [Compute and storage options](https://learn.microsoft.com/azure/mysql/flexible-server/concepts-service-tiers-storage)
- [High availability](https://learn.microsoft.com/azure/mysql/flexible-server/concepts-high-availability)
- [Read replicas](https://learn.microsoft.com/azure/mysql/flexible-server/concepts-read-replicas)
- [Migrate MySQL to Azure with DMS](https://learn.microsoft.com/azure/dms/tutorial-mysql-azure-external-online)
- [Extended support for MySQL](https://learn.microsoft.com/azure/mysql/concepts-extended-support)
- [Microsoft Entra authentication for MySQL](https://learn.microsoft.com/azure/mysql/flexible-server/concepts-azure-ad-authentication)
