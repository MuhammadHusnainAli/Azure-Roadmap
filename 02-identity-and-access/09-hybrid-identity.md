# Hybrid Identity

## Why it matters

Most established organisations still run Active Directory on-premises. Their users, groups and many applications live there. Hybrid identity connects that estate to Microsoft Entra ID so people have one identity everywhere. Engineers rarely build this, but architects and consultants must understand it: it determines how users sign in, which groups exist in the cloud to assign roles to, and what happens to cloud access when a domain controller is offline.

## Key concepts

### The goal

One identity per person, usable on-premises and in the cloud, with a single place where it is created and disabled.

```text
On-premises AD DS  ──── synchronisation ────►  Microsoft Entra ID
 (source of authority                           (cloud identity,
  for synced objects)                            authentication, policy)
```

For synchronised objects, on-premises AD is the **source of authority**: you change them there, and the change flows to the cloud.

### Synchronisation tools

| | **Microsoft Entra Connect Sync** | **Microsoft Entra Cloud Sync** |
|---|---|---|
| Architecture | A server you run on-premises, with a SQL database | Lightweight agents on-premises; configuration and engine in the cloud |
| High availability | A second server in staging mode, switched manually | Multiple active agents |
| Disconnected multi-forest | Limited | Supported |
| Device sync, hybrid join support | Yes | Not for device objects |
| Group writeback and advanced rules | Broad | Narrower but growing |
| Large directories and very large groups | Yes | With limits |
| Direction of travel | Maintenance | Where new investment goes |

**Guidance:** use Cloud Sync for new deployments unless you need something only Connect Sync offers. Existing Connect Sync deployments should check whether Cloud Sync now meets their needs and plan to move.

What gets synchronised: users, groups, contacts and (with Connect Sync) devices, filtered by OU, domain or attribute. Synchronise only what needs cloud access.

### Sign-in methods

| Method | How authentication happens | Dependencies | Recommendation |
|---|---|---|---|
| **Password hash synchronisation (PHS)** | A hash of the password hash is synced; Entra ID validates sign-ins itself | None at sign-in time | **Default choice** |
| **Pass-through authentication (PTA)** | Entra ID hands the validation to an on-premises agent, which checks against AD | Agents and domain controllers must be reachable | Only where policy forbids any password-derived data in the cloud |
| **Federation** (AD FS or third-party IdP) | Entra ID redirects to your federation service | A highly available federation farm and proxies | Only for requirements Entra ID cannot meet; otherwise migrate away |

Why PHS is preferred:

- **Resilience:** cloud sign-in keeps working if on-premises is down, including during a ransomware incident.
- **Security:** enables leaked-credential detection in ID Protection.
- **Simplicity:** nothing extra to operate.

Enable PHS **even when using PTA or federation**, as a fallback you can switch to in an emergency.

**AD FS** is a legacy component with a large attack surface. Microsoft's guidance is to migrate applications to Entra ID and retire it. *Staged rollout* lets you move groups of users from federation to cloud authentication gradually.

### Seamless single sign-on

For domain-joined devices on the corporate network, Seamless SSO signs users in without a prompt using Kerberos. On Entra-joined and hybrid-joined Windows devices, the Primary Refresh Token already provides SSO, which is the better mechanism.

### Device identities

| State | Device belongs to | Managed by | Use |
|---|---|---|---|
| **Entra registered** | The user (personal device) | Optionally Intune | BYOD |
| **Entra joined** | The organisation, cloud only | Intune | New corporate Windows devices. **Target state** |
| **Entra hybrid joined** | The organisation, joined to AD and registered in Entra ID | Group Policy and/or Intune | Existing domain-joined estates in transition |

Device state feeds Conditional Access. Aim for Entra join on new devices and treat hybrid join as a transitional state.

**Cloud Kerberos trust** lets Entra-joined devices using Windows Hello for Business reach on-premises resources (file shares, apps using Kerberos) without the old certificate or key trust complexity.

### Writeback

Changes that flow from the cloud to on-premises:

| Writeback | Purpose |
|---|---|
| **Password writeback** | Self-service password reset in the cloud updates the on-premises password |
| **Group writeback / group provisioning to AD** | Cloud-managed groups appear in AD to control on-premises resources |
| **Device writeback** | Supports some on-premises device-based access scenarios |

### Microsoft Entra Domain Services

A managed AD DS domain in Azure: domain join, Group Policy, LDAP, Kerberos and NTLM, without domain controllers to patch.

| | Entra Domain Services | AD DS on Azure VMs |
|---|---|---|
| You manage | Nothing at the DC level | Everything |
| Schema extensions | Limited | Yes |
| Domain/Enterprise Admin rights | No | Yes |
| Trusts | One-way outbound forest trusts (and more with Enterprise tier options) | Any |
| Relationship to on-premises AD | Separate domain populated from Entra ID | Can be an extension of your existing forest |
| Good for | Lifting legacy apps that need LDAP or Kerberos | Full AD functionality, replicating your existing domain into Azure |

Entra Domain Services needs password hashes in a legacy-compatible format, so password hash sync must be enabled for hybrid users.

### Reaching on-premises applications

| Option | What it does |
|---|---|
| **Entra application proxy** | Publishes on-premises web apps externally through an outbound-only connector, with Entra pre-authentication and Conditional Access |
| **Entra Private Access** | Identity-based access to any private application or network resource, replacing VPN (file 11) |
| **Kerberos constrained delegation with application proxy** | SSO to legacy Windows-integrated apps |

### Identity for Azure Files and other services

Azure Files supports identity-based SMB access using on-premises AD DS, Entra Domain Services, or **Microsoft Entra Kerberos** for hybrid and, increasingly, cloud-only identities. This is how lift-and-shift file shares keep their NTFS permissions, and how FSLogix profile containers work for Azure Virtual Desktop.

## How it works on Azure

### Reference topology

```text
On-premises                          Microsoft cloud
┌──────────────────┐                 ┌────────────────────────────┐
│ AD DS forest     │                 │ Microsoft Entra ID tenant  │
│  users, groups   │── Cloud Sync ──►│  synced users and groups   │
│                  │   agents        │  cloud-only admin accounts │
│ HR system ───────┼─ provisioning ─►│  Conditional Access, PIM   │
└──────────────────┘                 └──────────┬─────────────────┘
        ▲                                       │ tokens
        │ ExpressRoute / VPN                    ▼
┌───────┴──────────┐                 ┌────────────────────────────┐
│ Legacy apps      │◄── Private ─────│ Azure subscriptions        │
│ (Kerberos, LDAP) │    Access       │  RBAC assigned to groups   │
└──────────────────┘                 └────────────────────────────┘
```

Key practices:

- **Privileged cloud accounts are cloud-only.** Never synchronise on-premises admin accounts into privileged Entra roles. A compromise of AD must not become a compromise of the cloud.
- **Separate tiers.** Domain admin accounts should not sign in to cloud services, and cloud admin accounts should not exist on-premises.
- **Sync servers are tier-zero assets.** Protect Connect Sync servers and provisioning agents like domain controllers.

### Domain controllers in Azure

When Azure workloads need AD DS, deploy domain controllers on VMs in the identity subscription of the landing zone (topic 03): at least two, across availability zones, as an additional AD site, with DNS configured so Azure VNets resolve the domain. Do not rely on a VPN back to on-premises for every authentication.

### Groups for Azure RBAC and AI access

Synchronised security groups can be used directly in Azure role assignments and app assignments. Decide where each group should be mastered:

| Group purpose | Master in |
|---|---|
| Maps to an on-premises team or existing AD group | AD, synchronised |
| Exists only for cloud access (Azure roles, Foundry projects, apps) | Entra ID, cloud-only |
| Privileged access | Entra ID, role-assignable, managed with PIM |

For permission-aware RAG over file shares or SharePoint, the group memberships used for security trimming come from here. If on-premises group membership is stale, the AI application's answers will reflect that.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Sync tool | Cloud Sync unless a specific Connect Sync capability is required |
| Sign-in method | PHS. Add PTA only for a firm policy requirement. Retire federation |
| Device strategy | Entra join for new devices; hybrid join only as a bridge |
| Legacy app needs a domain | Modernise to OIDC or SAML if possible; otherwise Entra Domain Services; AD DS on VMs for full control |
| What to synchronise | The minimum. Exclude service accounts and admin accounts that have no cloud purpose |
| Source of authority | Move toward HR-driven provisioning directly into Entra ID, with AD as a downstream target. Entra supports converting the source of authority of synced groups and users to the cloud as part of this |

**The long-term direction** is cloud-first identity: Entra ID as the primary directory, AD DS shrinking to what legacy systems need.

## Hands-on

Building a full hybrid lab needs a domain controller, so this is a review exercise for most learners.

1. In the Entra admin centre, open **Identity → Hybrid management → Microsoft Entra Connect**. Note whether sync is enabled, which tool, the last sync time and the sign-in method.
2. Open **Users** and add the *On-premises sync enabled* column. Identify which accounts are synchronised and which are cloud-only.
3. Check whether any privileged role holders are synchronised accounts:

   ```bash
   az rest --method get \
     --url "https://graph.microsoft.com/v1.0/directoryRoles?\$expand=members" \
     --query "value[].{Role:displayName, Members:members[].{UPN:userPrincipalName, Synced:onPremisesSyncEnabled}}" -o json
   ```

4. Open **Devices → All devices** and review the *Join type* column.
5. Read the **Microsoft Entra Connect Health** or sync health blade for errors.

Optional full lab: deploy a Windows Server VM, promote it to a domain controller, create a few users, install the Cloud Sync provisioning agent, configure sync with PHS, and sign in to `myapps.microsoft.com` as a synchronised user.

## Security, cost and reliability notes

- **Security:** on-premises AD is the usual starting point for attacks that pivot to the cloud. Isolate cloud admins, protect sync infrastructure, and monitor for changes to sync and federation configuration.
- **Security:** federation trust is a high-value target (token-signing certificate theft). Another reason to retire AD FS.
- **Reliability:** PTA and federation make cloud sign-in depend on on-premises availability. PHS does not.
- **Reliability:** run at least three PTA agents or multiple Cloud Sync agents if you use them.
- **Cost:** sync tools are free. Entra Domain Services is billed hourly by tier. Domain controllers on VMs cost compute, storage and operations effort.

## Common mistakes

- Synchronising Domain Admins and assigning them Global Administrator.
- Choosing PTA or federation without a concrete requirement.
- Not enabling PHS as a fallback.
- Treating the Connect Sync server as an ordinary member server.
- Assuming Entra Domain Services is an extension of the on-premises domain.
- Keeping hybrid join as the permanent device strategy.
- Synchronising every OU, including disabled and service accounts.
- Routing every Azure workload's authentication over a VPN to on-premises DCs.

## Check yourself

1. Why is password hash synchronisation the recommended sign-in method?
2. Name two differences between Entra Connect Sync and Cloud Sync.
3. What is the difference between Entra joined and Entra hybrid joined?
4. When would you choose Entra Domain Services over AD DS on VMs?
5. Why must privileged cloud accounts be cloud-only?
6. What does password writeback enable?
7. How can users reach an on-premises web app with Entra pre-authentication and no inbound firewall ports?

<details>
<summary>Answers</summary>

1. Sign-in does not depend on on-premises infrastructure, it enables leaked-credential detection, and it is the simplest to operate.
2. Connect Sync is a server you run with staging-mode failover; Cloud Sync uses lightweight agents with cloud-managed configuration and active-active agents. Cloud Sync supports disconnected forests; Connect Sync supports device sync and more advanced rules.
3. Entra joined devices are joined only to Entra ID. Hybrid joined devices are joined to on-premises AD and also registered in Entra ID.
4. When a lifted application needs LDAP, Kerberos or domain join and you do not need schema extensions, admin rights on the domain, or an extension of your existing forest.
5. So that compromise of on-premises AD cannot be used to take over the cloud tenant.
6. Self-service password reset and password changes in the cloud are written back to on-premises AD.
7. Entra application proxy (or Entra Private Access), using an outbound-only connector.

</details>

## References

- [What is hybrid identity with Microsoft Entra ID?](https://learn.microsoft.com/entra/identity/hybrid/whatis-hybrid-identity)
- [What is Microsoft Entra Cloud Sync?](https://learn.microsoft.com/entra/identity/hybrid/cloud-sync/what-is-cloud-sync)
- [Choose the right authentication method](https://learn.microsoft.com/entra/identity/hybrid/connect/choose-ad-authn)
- [Migrate from federation to cloud authentication](https://learn.microsoft.com/entra/identity/hybrid/connect/migrate-from-federation-to-cloud-authentication)
- [What is a device identity?](https://learn.microsoft.com/entra/identity/devices/overview)
- [What is Microsoft Entra Domain Services?](https://learn.microsoft.com/entra/identity/domain-services/overview)
- [Microsoft Entra application proxy](https://learn.microsoft.com/entra/identity/app-proxy/overview-what-is-app-proxy)
- [Protecting Microsoft 365 from on-premises attacks](https://learn.microsoft.com/entra/architecture/protect-m365-from-on-premises-attacks)
- [Deploy AD DS in an Azure virtual network](https://learn.microsoft.com/azure/architecture/example-scenario/identity/adds-extend-domain)
