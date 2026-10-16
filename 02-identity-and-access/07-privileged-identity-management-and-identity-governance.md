# Privileged Identity Management and Identity Governance

> Topic: 02 · Identity and Access · Tracks: E ○ A ● C ● · Status checked: 2026-10

## Why it matters

Access tends only to grow. People change roles, projects end, contractors leave, and the permissions stay. Identity governance is the discipline of making sure the right identities have the right access for the right time, and can prove it to an auditor. Privileged Identity Management tackles the sharpest end of that problem: standing administrative access.

## Key concepts

### The four questions

Identity governance exists to answer:

1. Who has access to what?
2. Should they?
3. How did they get it?
4. What are they doing with it?

### Privileged Identity Management

PIM makes privileged access **just-in-time, time-bound and approved** instead of permanent.

| Assignment type | Meaning |
|---|---|
| **Eligible** | The user may activate the role when needed |
| **Active** | The user holds the role now, without activating |
| **Permanent** | No end date (eligible or active) |
| **Time-bound** | Has a start and end date |

The target state is: no permanent active assignments for privileged roles, except break-glass accounts.

**What PIM manages**

| Scope | Examples |
|---|---|
| **Microsoft Entra roles** | Global Administrator, Conditional Access Administrator |
| **Azure resource roles** | Owner, Contributor, User Access Administrator at any scope |
| **PIM for Groups** | Just-in-time membership or ownership of a group |

**PIM for Groups** is the versatile one. Make users eligible for membership of a group, and assign that group whatever it needs: several Azure roles, Entra roles, app roles, or access in other systems. One activation grants a coherent bundle.

**Role settings** (per role)

| Setting | Options |
|---|---|
| Maximum activation duration | Typically 1 to 8 hours |
| On activation require | MFA, an authentication context (step-up), justification, ticket number |
| Approval | Named approvers must approve |
| Assignment rules | Whether permanent eligible or active assignment is allowed; maximum duration |
| Notifications | Who is told about assignments and activations |

**Activation flow**

```text
User requests activation ─► checks (MFA / auth context / justification / ticket)
        ─► approval if required ─► role active for N hours ─► automatically removed
```

Everything is recorded in the PIM audit history.

**Alerts and discovery:** PIM flags too many Global Administrators, roles assigned outside PIM, stale eligible assignments and redundant roles. The *Discovery and insights* view helps convert permanent assignments to eligible.

Licensing: PIM needs Entra ID P2 or ID Governance for the users who benefit.

### Emergency access accounts

Two cloud-only accounts with permanent Global Administrator, for when normal access fails.

- Not tied to any individual; not federated; not dependent on one MFA provider or device.
- Credentials: passkeys (FIDO2) or certificates, stored separately and securely. Mandatory MFA applies to them too.
- Excluded from Conditional Access policies that could block them.
- **Alert on every sign-in.**
- Tested on a schedule.

### Access reviews

Periodic recertification of access.

| Review | Reviewers | Typical outcome |
|---|---|---|
| Group membership | Group owners or managers | Remove inactive members |
| Application assignment | App owners | Remove users who no longer need it |
| Entra roles and Azure roles (through PIM) | Selected reviewers or self | Remove unused privilege |
| Guest users | Sponsors | Remove or block guests no longer needed |
| Access package assignments | Managers or sponsors | Expire access |

Useful settings: recurrence, **recommendations** based on sign-in activity, **auto-apply results**, and what to do if reviewers do not respond (the safe default is to remove access).

### Entitlement management

Bundles access and makes requesting it self-service and governed.

| Concept | Meaning |
|---|---|
| **Catalog** | A container of resources (groups, apps, SharePoint sites, Entra roles) and access packages |
| **Access package** | A bundle of resource roles a user can request |
| **Policy** | Who can request, who approves, how long access lasts, review schedule |
| **Connected organisation** | An external organisation whose users may request access |

A new project member requests the *Project Alpha contributor* package, their manager approves, and they receive the Teams site, the dev resource group role and the Foundry project role in one step. After 180 days it expires unless renewed. Nobody files five tickets and nobody forgets to remove access.

It is also the cleanest way to govern **guests**: access packages can create the guest account on approval and remove it when the last assignment expires.

### Lifecycle workflows

Automation for joiner, mover and leaver events driven by HR attributes such as hire date, department and termination date.

| Event | Example tasks |
|---|---|
| **Joiner** | Generate a Temporary Access Pass and send it to the manager, add to groups, assign licences, send welcome email |
| **Mover** | Change group memberships and access packages on a department change |
| **Leaver** | Disable the account, remove from groups, remove licences, revoke sessions, delete after a retention period |

Custom steps run through Logic Apps. Combine with inbound HR provisioning (Workday, SuccessFactors, or API-driven) so the HR system is the source of truth.

### Governing non-human identities

The same questions apply to applications, workloads and agents, and they are easier to forget:

- **App registrations and service principals:** owners, expiring and unused credentials, unused permissions. Entra recommendations flag these.
- **Managed identities:** governed through the Azure resources they belong to and their role assignments.
- **Agent identities:** each has a **sponsor** accountable for it; lifecycle and access reviews apply (file 10).
- **Guests:** sponsors and access reviews.

An orphaned service principal with a valid secret and Contributor on a subscription is as dangerous as a forgotten admin account.

## How it works on Azure

### PIM for Azure resources in practice

A common design for a workload subscription:

| Persona | Standing access | Eligible through PIM |
|---|---|---|
| Developers | Reader on production; Contributor on dev | Contributor on a production resource group, 4 hours, with justification |
| Operations | Reader, Monitoring Contributor | Contributor on production, approval required |
| Platform engineers | Reader at management group | Owner or User Access Administrator, approval and step-up |
| Security | Security Reader | Security Admin |
| Pipelines | A workload identity with exactly the roles needed | – |

Production changes go through pipelines. Human elevation is the exception, and every use of it is logged with a reason.

### Just-in-time beyond PIM

- **Just-in-time VM access** (Defender for Cloud): opens management ports on request for a limited time.
- **Azure Bastion:** removes the need for public management ports entirely.
- **Customer Lockbox:** requires your approval before Microsoft engineers access your data during a support case.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Activation duration | Long enough to finish a task (2 to 4 hours), short enough to matter |
| Approval | Require for the highest-impact roles. Approval on everything creates rubber-stamping and delays incident response |
| Direct eligibility vs PIM for Groups | Groups when a persona needs a bundle of roles or access in several systems |
| Review frequency | Quarterly for privileged roles and guests; six-monthly or annually for standard access |
| Self-review vs manager review | Manager or owner review is stronger. Self-review is acceptable with activity-based recommendations |
| Auto-apply | Yes, with "remove if no response" for anything sensitive |

**Friction vs security:** if elevation takes thirty minutes during an outage, people will find a way to keep permanent access. Make the normal path quick (self-activation with MFA and justification) and reserve approval for the roles that warrant it.

## Hands-on

Requires P2 and Privileged Role Administrator, or use a test tenant.

1. **Discovery:** *ID Governance → Privileged Identity Management → Microsoft Entra roles → Discovery and insights*. Note how many permanent Global Administrators exist.
2. **Role settings:** open settings for a low-impact role such as *Reports Reader*. Set maximum duration 2 hours, require justification and MFA.
3. **Eligible assignment:** make a test user eligible for the role for 30 days.
4. **Activate:** sign in as the test user, open *My roles*, activate, and observe the prompts. Check the audit history afterwards.
5. **Azure resources:** in *PIM → Azure resources*, select a resource group and make the test user eligible for Contributor.
6. **PIM for Groups:** create a security group, enable it in PIM, and make a user eligible for membership.
7. **Access review:** *ID Governance → Access reviews*: create a one-time review of a test group with recommendations and auto-apply enabled.
8. **Access package:** create a catalog, add a group, create a package with manager approval and 30-day expiry, and request it from `myaccess.microsoft.com`.

Activate an eligible Azure role from the CLI:

```bash
az rest --method put \
  --url "https://management.azure.com/<scope>/providers/Microsoft.Authorization/roleAssignmentScheduleRequests/$(uuidgen)?api-version=2020-10-01" \
  --body '{
    "properties": {
      "principalId": "<your-object-id>",
      "roleDefinitionId": "<scope>/providers/Microsoft.Authorization/roleDefinitions/<role-guid>",
      "requestType": "SelfActivate",
      "justification": "Investigating incident 1234",
      "scheduleInfo": { "expiration": { "type": "AfterDuration", "duration": "PT2H" } }
    }
  }'
```

## Security, cost and reliability notes

- **Security:** require phishing-resistant authentication for activating the most privileged roles by attaching an authentication context.
- **Security:** alert on role assignments made **outside** PIM. They indicate either a process gap or an attacker.
- **Security:** an eligible assignment is still a risk if the account is compromised. Eligibility should also be reviewed and time-bound.
- **Cost:** P2 or ID Governance licences per governed user. Guest governance is billed on monthly active users.
- **Reliability:** make sure on-call staff are eligible and have tested activation before an incident. Keep break-glass accounts outside PIM.

## Common mistakes

- Making everyone *eligible* for Global Administrator and calling it least privilege.
- Requiring approval for every role, so approvers click through without reading.
- Activation durations of eight hours by default.
- Never reviewing eligible assignments.
- Access reviews where non-response keeps access.
- Governing users carefully and ignoring service principals, app permissions and agents.
- Break-glass accounts that depend on the same MFA method as everyone else.
- No alert on break-glass sign-in.

## Check yourself

1. What is the difference between an eligible and an active assignment?
2. Name three things PIM can require at activation.
3. When is PIM for Groups a better choice than direct role eligibility?
4. What should happen to access when a reviewer does not respond?
5. What is an access package?
6. Which lifecycle events do lifecycle workflows automate, and what triggers them?
7. List four properties of a well-managed emergency access account.

<details>
<summary>Answers</summary>

1. Eligible means the user can activate the role when needed; active means they hold it now.
2. MFA or an authentication context, justification, a ticket number, approval.
3. When a persona needs several roles or access across multiple systems granted by one activation.
4. It should be removed.
5. A bundle of resource roles (groups, apps, sites, roles) with a policy governing who can request it, who approves, how long it lasts and how it is reviewed.
6. Joiner, mover and leaver, triggered by user attributes such as hire date, department change and termination date.
7. Cloud-only and not tied to a person; strong phishing-resistant credential stored securely; excluded from lockout-prone Conditional Access policies; every sign-in alerted; regularly tested.

</details>

## References

- [What is Microsoft Entra Privileged Identity Management?](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/pim-configure)
- [Privileged Identity Management for Groups](https://learn.microsoft.com/entra/id-governance/privileged-identity-management/concept-pim-for-groups)
- [What is Microsoft Entra ID Governance?](https://learn.microsoft.com/entra/id-governance/identity-governance-overview)
- [What are access reviews?](https://learn.microsoft.com/entra/id-governance/access-reviews-overview)
- [What is entitlement management?](https://learn.microsoft.com/entra/id-governance/entitlement-management-overview)
- [What are lifecycle workflows?](https://learn.microsoft.com/entra/id-governance/what-are-lifecycle-workflows)
- [Manage emergency access accounts](https://learn.microsoft.com/entra/identity/role-based-access-control/security-emergency-access)
- [Securing privileged access](https://learn.microsoft.com/security/privileged-access-workstations/overview)
