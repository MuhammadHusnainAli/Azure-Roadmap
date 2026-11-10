# Microsoft Entra ID Fundamentals

## Why it matters

Microsoft Entra ID (formerly Azure Active Directory) is the identity provider behind Azure, Microsoft 365, Dynamics 365 and thousands of third-party applications. Nothing in Azure works without it. Understanding its objects, and above all the fact that it has a role system separate from Azure's, prevents the most common access confusion you will meet.

## Key concepts

### What Entra ID is, and is not

Entra ID is a cloud identity and access management service. It authenticates identities and issues tokens using modern protocols: OAuth 2.0, OpenID Connect and SAML.

It is **not** Active Directory Domain Services in the cloud:

| | Active Directory Domain Services | Microsoft Entra ID |
|---|---|---|
| Protocols | Kerberos, NTLM, LDAP | OAuth 2.0, OpenID Connect, SAML |
| Structure | Forests, domains, OUs | Flat tenant; administrative units for delegation |
| Device management | Group Policy | Intune and Conditional Access |
| Reach | Corporate network | Internet |
| Typical use | Domain-joined servers and PCs | Cloud and SaaS applications |

When a legacy application genuinely needs Kerberos or LDAP in Azure, use Microsoft Entra Domain Services or domain controllers on VMs (see [Hybrid Identity](09-hybrid-identity.md)).

### The Microsoft Entra family

"Entra" is a product family. Entra ID is its core.

| Product | Purpose |
|---|---|
| **Microsoft Entra ID** | Identity provider and access management |
| **Entra ID Governance** | Access reviews, entitlement management, lifecycle workflows |
| **Entra ID Protection** | Risk detection for users and sign-ins |
| **Entra External ID** | Customer and partner identities |
| **Entra Domain Services** | Managed AD DS |
| **Entra Workload ID** | Protection and governance for application identities |
| **Entra Agent ID** | Identities for AI agents |
| **Entra Verified ID** | Verifiable credentials |
| **Entra Private Access / Internet Access** | Identity-centric network access (Global Secure Access) |

### Tenant

A tenant is one dedicated instance of Entra ID, created when an organisation first signs up for a Microsoft cloud service.

- It has a globally unique **tenant ID** (a GUID) and an initial domain `<name>.onmicrosoft.com`.
- You add and verify **custom domains** (`contoso.com`) so users sign in with familiar addresses.
- Each Azure subscription trusts exactly one tenant.
- A tenant is a hard identity and security boundary. Most organisations should have exactly one production tenant.

### Identity types

| Type | Represents | Notes |
|---|---|---|
| **User (member)** | An employee | Created in the cloud or synchronised from on-premises AD |
| **User (guest)** | An external collaborator | Invited through B2B; authenticates in their home tenant |
| **Group** | A set of principals | Security groups and Microsoft 365 groups |
| **Service principal** | An application's identity in a tenant | Used by apps, pipelines and automation |
| **Managed identity** | A service principal whose credentials Azure manages | The default for Azure-hosted workloads |
| **Agent identity** | An AI agent | A distinct identity type with its own policies |
| **Device** | A registered or joined device | Used in Conditional Access decisions |

Anything that can be granted access is called a **security principal**. Every principal has an immutable **object ID**; applications additionally have an **application (client) ID**. You will be asked for one or the other constantly, and mixing them up is a classic failure.

### Groups

| Aspect | Options |
|---|---|
| Type | **Security** (access control) or **Microsoft 365** (collaboration: mailbox, Teams, SharePoint) |
| Membership | **Assigned** (manual), **Dynamic user**, **Dynamic device** (rule-based; needs P1) |
| Role-assignable | A security group flagged at creation so it can hold Entra roles; membership is protected |
| Nesting | Supported for security groups, but not honoured everywhere (for example some app assignments and licensing) |

Assign access to groups, not individuals. It is the single most effective habit in identity design.

### Two role systems

This is the concept to get right.

| | **Microsoft Entra roles** | **Azure roles (Azure RBAC)** |
|---|---|---|
| Controls | Entra ID itself: users, groups, apps, policies, licences | Azure resources: subscriptions, VMs, storage, Foundry |
| Scope | Tenant, administrative unit, or a single object | Management group, subscription, resource group, resource |
| Examples | Global Administrator, User Administrator, Application Administrator, Security Reader | Owner, Contributor, Reader, Storage Blob Data Reader |
| Managed in | Entra admin centre | Access control (IAM) on any Azure scope |

They are independent. A **Global Administrator has no access to Azure resources by default**, and a subscription **Owner cannot create users**. The one bridge: a Global Administrator can **elevate access** to become User Access Administrator at the root scope. That action is logged and should be rare and temporary.

Key Entra roles to recognise:

| Role | Can |
|---|---|
| Global Administrator | Everything in the tenant. Keep to fewer than five people |
| Privileged Role Administrator | Manage role assignments and PIM |
| User Administrator | Manage users and groups |
| Application Administrator / Cloud Application Administrator | Manage app registrations and enterprise apps |
| Conditional Access Administrator | Manage Conditional Access policies |
| Security Administrator / Security Reader | Manage or read security features |
| Global Reader | Read everything, change nothing |
| AI Administrator | Manage Microsoft 365 Copilot and AI-related enterprise services |
| Agent ID Administrator / Agent ID Developer | Manage agent identities and blueprints |

### Administrative units

A container for users, groups and devices that lets you scope an Entra role to a subset of the directory. Use them to let a regional helpdesk reset passwords only for its own region. **Restricted management** administrative units protect sensitive objects from tenant-wide admins.

### Licensing

Features depend on the licence tier.

| Tier | Adds |
|---|---|
| **Free** | Users, groups, SSO, basic MFA through security defaults, B2B |
| **P1** | Conditional Access, dynamic groups, self-service password reset with writeback, application proxy, hybrid features |
| **P2** | ID Protection (risk-based policies), Privileged Identity Management, basic access reviews |
| **Entra ID Governance** (add-on) | Full entitlement management, lifecycle workflows, advanced access reviews |
| **Entra Suite** | Bundles Private Access, Internet Access, ID Governance, ID Protection, Verified ID premium |

P1 is included in Microsoft 365 E3 and Business Premium; P2 in Microsoft 365 E5. Workload identities and agents have their own licensing for premium protection features.

### Security defaults

A free, all-or-nothing baseline for tenants without Conditional Access: MFA registration for all users, MFA for administrators, MFA when needed for users, and legacy authentication blocked. It is on by default for new tenants. Once you have P1, turn it off and replace it with Conditional Access policies, which you can tailor.

Independently of your configuration, Microsoft now **enforces MFA for Azure management**: sign-ins to the Azure portal, Entra admin centre and Intune admin centre, and create, update or delete operations through the CLI, PowerShell, IaC tools and REST API. Workload identities are exempt, which is one more reason never to automate with a user account.

## How it works on Azure

### Sign-in at a glance

```text
1. User opens an application
2. Application redirects to login.microsoftonline.com/{tenant}
3. Entra ID authenticates the user (password + MFA, passkey, ...)
4. Conditional Access evaluates policies
5. Entra ID issues tokens (ID token, access token, refresh token)
6. Application presents the access token to the API
7. API validates the token and authorises the request
```

For Azure resources, step 7 is Azure Resource Manager (control plane) or the service itself (data plane) checking Azure RBAC.

### Logs you need to know

| Log | Shows | Default retention |
|---|---|---|
| **Sign-in logs** | Every authentication, with Conditional Access result | 7 days (Free), 30 days (P1/P2) |
| **Audit logs** | Changes to directory objects | 7 or 30 days |
| **Provisioning logs** | User provisioning to and from apps | 30 days |

Send them to a Log Analytics workspace through diagnostic settings for longer retention and alerting. Sign-in logs are split into interactive, non-interactive, service principal and managed identity sign-ins.

## Design decisions and trade-offs

- **One tenant.** Extra tenants multiply administration and break collaboration. Use administrative units, subscriptions and management groups for separation.
- **Cloud-only vs synchronised.** New organisations can be cloud-only. Those with on-premises AD synchronise identities (file 09).
- **Fewest possible permanent admins.** Standing Global Administrator access is the highest-value target in the tenant. Use PIM (file 07).
- **Groups everywhere.** Role-assignable groups for Entra roles, security groups for Azure RBAC and app access.
- **Separate admin accounts.** Administrators use a dedicated cloud-only account for privileged work, not their day-to-day mailbox account.

## Hands-on

```bash
# Tenant and signed-in user
az account show --query "{Tenant:tenantId, User:user.name}" -o table
az ad signed-in-user show --query "{Name:displayName, UPN:userPrincipalName, ObjectId:id}" -o table

# Create a security group and add yourself
az ad group create --display-name grp-roadmap-lab --mail-nickname grp-roadmap-lab
ME=$(az ad signed-in-user show --query id -o tsv)
az ad group member add --group grp-roadmap-lab --member-id $ME
az ad group member list --group grp-roadmap-lab --query "[].displayName" -o tsv

# Which Entra roles do I hold? (Microsoft Graph)
az rest --method get \
  --url "https://graph.microsoft.com/v1.0/me/transitiveMemberOf/microsoft.graph.directoryRole" \
  --query "value[].displayName"

# Which Azure roles do I hold on the current subscription?
az role assignment list --assignee $ME --all --query "[].{Role:roleDefinitionName, Scope:scope}" -o table

# Clean up
az ad group delete --group grp-roadmap-lab
```

Compare the last two outputs. They come from different role systems.

In the Entra admin centre, also look at **Identity → Overview → Properties** (tenant ID, licence), **Monitoring → Sign-in logs** (find your own sign-in and open the Conditional Access tab), and **Roles and admins**.

## Security, cost and reliability notes

- **Security:** keep two cloud-only **emergency access (break-glass)** accounts with Global Administrator, protected by passkeys or certificates, excluded from most Conditional Access policies, and monitored with an alert on any sign-in.
- **Security:** restrict who can register applications, invite guests and consent to app permissions. All three are open to every user by default.
- **Cost:** licences are per user. Guest users are billed on a monthly-active-user model.
- **Reliability:** Entra ID is a global, non-regional service with a 99.99% SLA at P1/P2. Authentication is a dependency of everything, so do not add your own single points of failure (such as a federation server) in front of it without need.

## Common mistakes

- Expecting a Global Administrator to see Azure subscriptions.
- Using a personal user account for automation, which now fails under mandatory MFA.
- Confusing an application's client ID with its service principal object ID when assigning roles.
- Leaving user consent and app registration open to all users.
- Creating a new tenant for a dev environment.
- Relying on security defaults in a tenant that has P1 licences and could use Conditional Access.
- Having no break-glass account, or one that nobody has tested.

## Check yourself

1. Why can a Global Administrator not delete a storage account by default?
2. What is the difference between a member and a guest user?
3. Name three differences between Entra ID and AD DS.
4. What does a P1 licence add that matters most for security?
5. What is a role-assignable group and why does it exist?
6. Which Azure operations now require MFA regardless of tenant configuration, and which identities are exempt?

<details>
<summary>Answers</summary>

1. Entra roles and Azure roles are separate systems. Global Administrator is an Entra role with no Azure RBAC permissions unless access is elevated.
2. A member belongs to the tenant. A guest is an external identity invited through B2B who authenticates in their home directory and has restricted default permissions.
3. Protocols (OAuth/OIDC/SAML vs Kerberos/NTLM/LDAP), structure (flat tenant vs forests and OUs), management (Conditional Access and Intune vs Group Policy), reach (internet vs corporate network).
4. Conditional Access.
5. A security group marked at creation as able to hold Entra roles. Its membership is protected so that lower-privileged admins cannot add themselves to a privileged group.
6. Portal sign-ins and create, update, delete operations against Azure Resource Manager from the CLI, PowerShell, IaC, SDKs and REST. Managed identities and service principals are exempt.

</details>

## References

- [What is Microsoft Entra ID?](https://learn.microsoft.com/entra/fundamentals/whatis)
- [Compare Active Directory to Microsoft Entra ID](https://learn.microsoft.com/entra/fundamentals/compare)
- [Azure roles, Microsoft Entra roles, and classic subscription administrator roles](https://learn.microsoft.com/azure/role-based-access-control/rbac-and-directory-admin-roles)
- [Microsoft Entra built-in roles](https://learn.microsoft.com/entra/identity/role-based-access-control/permissions-reference)
- [Microsoft Entra licensing](https://learn.microsoft.com/entra/fundamentals/licensing)
- [Security defaults in Microsoft Entra ID](https://learn.microsoft.com/entra/fundamentals/security-defaults)
- [Plan for mandatory multifactor authentication for Azure](https://learn.microsoft.com/entra/identity/authentication/concept-mandatory-multifactor-authentication)
- [Manage emergency access accounts](https://learn.microsoft.com/entra/identity/role-based-access-control/security-emergency-access)
