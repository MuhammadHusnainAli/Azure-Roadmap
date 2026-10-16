# Conditional Access and Identity Protection

> Topic: 02 · Identity and Access · Tracks: E ○ A ● C ● · Status checked: 2026-10

## Why it matters

Conditional Access is the Zero Trust policy engine for Microsoft's cloud. It turns "verify explicitly" from a slogan into a rule: this user, on this device, from this place, with this risk level, gets this access under these conditions. Architects design the policy set; consultants are asked about it in every security review; and engineers meet it whenever a deployment script or an agent is blocked.

## Key concepts

### The model

```text
        SIGNALS                    DECISION                 ENFORCEMENT
 who · what · where · how   ──►  policies evaluated  ──►  block / grant with controls / limit session
```

A policy is an if-then statement: **if** these assignments match, **then** apply these access controls.

All matching policies apply together. If any one blocks, access is blocked. Otherwise every grant requirement from every matching policy must be satisfied.

### Assignments (the "if")

| Assignment | Options |
|---|---|
| **Users and identities** | Users, groups, directory roles, guests by type; **workload identities**; **agents** |
| **Target resources** | Cloud apps (all or selected), user actions (register security info, register device), authentication context, Global Secure Access traffic |
| **Network** | Named locations (IP ranges, countries), compliant network |
| **Conditions** | User risk, sign-in risk, insider risk, device platform, client apps (browser, modern clients, legacy auth), device filters, authentication flows (device code, authentication transfer) |

### Access controls (the "then")

**Grant**

| Control | Meaning |
|---|---|
| Block access | Deny |
| Require multifactor authentication | Any MFA |
| Require authentication strength | Specific methods, such as phishing-resistant |
| Require device to be marked compliant | Intune-compliant |
| Require Microsoft Entra hybrid joined device | Domain-joined and registered |
| Require approved client app / app protection policy | Mobile app management |
| Require risk remediation / password change | Used with user risk |
| Require terms of use | Acceptance recorded |

Multiple controls can be combined with *require all* or *require one*.

**Session**

| Control | Meaning |
|---|---|
| Sign-in frequency | How often to reauthenticate |
| Persistent browser session | Whether "stay signed in" is allowed |
| App-enforced restrictions | Limited web access in SharePoint and Exchange |
| Conditional Access App Control | Route the session through Defender for Cloud Apps for real-time monitoring |
| Continuous access evaluation | Customise or strictly enforce location |
| Token protection | Bind tokens to the device |

### Report-only and What If

- **Report-only mode:** the policy is evaluated and logged but not enforced. Run every new policy this way first and review the results in sign-in logs and the *Conditional Access insights and reporting* workbook.
- **What If tool:** simulate a sign-in (user, app, location, device, risk) and see which policies would apply. The first tool to open when someone says "I can't sign in."

### Named locations

Define trusted IP ranges or countries. Use them carefully: "inside the office network" is a weak signal by Zero Trust standards. Locations are better used to **block** (countries you never operate in) than to **exempt** from MFA.

### Authentication context

A label (for example *Sensitive data*, *Privileged action*) that a resource can demand. A Conditional Access policy targets the label and applies stronger controls. Used for step-up authentication on specific SharePoint sites, PIM role activation, and protected actions in Entra such as changing Conditional Access policies.

### Microsoft Entra ID Protection

Machine-learning risk detection, requiring P2.

| Risk type | Asks | Example detections |
|---|---|---|
| **Sign-in risk** | Is this particular sign-in suspicious? | Anonymous IP, atypical travel, unfamiliar sign-in properties, malicious IP, token anomalies, password spray |
| **User risk** | Is this account probably compromised? | Leaked credentials, threat intelligence, anomalous user activity, attacker-in-the-middle |

Levels are low, medium, high. Risk is consumed by Conditional Access:

| Policy | Typical setting |
|---|---|
| Sign-in risk medium or above | Require MFA (which self-remediates the risk) |
| User risk high | Require secure password change with MFA, or block |

ID Protection also covers **workload identities** (risky service principals) and **agents** (risky agent identities), with their own detections such as anomalous resource access.

### Conditional Access for workload identities

Service principals do not do MFA, so policies for them use different levers: block by location, and block when risk is detected. This applies to single-tenant service principals and requires Workload ID Premium licences. Managed identities are not covered.

### Conditional Access for agents

Agent identities can be targeted by policy, for example to block risky agents or restrict which resources agents may reach. These capabilities are tied to Microsoft Agent 365 licensing. See [Microsoft Entra Agent ID](10-microsoft-entra-agent-id.md).

## How it works on Azure

### A baseline policy set

Microsoft publishes policy templates. A solid starting set:

| # | Policy | Targets | Control |
|---|---|---|---|
| 1 | Block legacy authentication | All users | Block |
| 2 | Require MFA for all users | All users, all resources | MFA |
| 3 | Require phishing-resistant MFA for administrators | Privileged directory roles | Authentication strength |
| 4 | Require MFA for Azure management | All users → *Windows Azure Service Management API* | MFA |
| 5 | Require compliant or hybrid-joined device | All users, selected or all apps | Compliant device |
| 6 | Sign-in risk policy | All users, risk medium+ | MFA |
| 7 | User risk policy | All users, risk high | Password change |
| 8 | Block device code flow | All users except documented exceptions | Block |
| 9 | Block access from unexpected countries | All users | Block |
| 10 | Secure security info registration | All users → user action | MFA or trusted location, or Temporary Access Pass |
| 11 | Require MFA for guests | Guest and external users | MFA |
| 12 | Block risky workload identities | Selected service principals | Block |

Policy 4 targets the app that covers the Azure portal, Azure Resource Manager, the CLI and PowerShell. It is what stands between a stolen password and your subscriptions.

### Exclusions and break-glass

- Exclude the two **emergency access accounts** from policies that could lock everyone out, and alert on every sign-in by those accounts.
- Exclude **service accounts** only as a temporary measure while migrating them to workload identities.
- Keep exclusions in a **group**, reviewed regularly. Exclusions are where policy sets rot.

**Microsoft-managed policies** may also appear in your tenant: Microsoft creates them in report-only mode, and they switch on after a notice period unless you act. Review them rather than deleting them reflexively.

### Naming and operations

Name policies so their intent is readable in a log entry, for example `CA004-Admins-AzureMgmt-RequirePhishResistantMFA`. Manage them as code (Microsoft Graph, Terraform, or Microsoft365DSC) with review and a rollback path. Entra can also **restore** recently deleted policies.

### Protecting AI applications

- Your custom AI app is an enterprise application. Target it with Conditional Access like any other: MFA, compliant device, and assignment required.
- Use **authentication context** for step-up on sensitive agent actions.
- The Foundry portal and Azure management are covered by the Azure management policy.
- With **Global Secure Access**, policies can also apply to traffic heading to third-party AI services (file 11).

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Security defaults vs Conditional Access | Conditional Access once you have P1; they cannot be combined |
| Target "all resources" vs selected | All resources with explicit exclusions is safer than a growing include list |
| Device compliance | Strong control, but needs Intune maturity. Phase it in: admins first, then sensitive apps |
| Location as trust | Use to block, not to waive MFA |
| Session lifetime | Leave defaults for compliant devices; tighten for unmanaged devices and admin portals |
| Many small policies vs few broad | Fewer, clearly named, persona-based policies are easier to reason about and stay within limits |

**Persona-based design:** group policies by persona (internals, admins, guests, service accounts, agents) and number them in ranges. It scales better than one policy per application.

## Hands-on

Requires P1 and the Conditional Access Administrator role. Use a test tenant if you can.

1. **Templates:** *Protection → Conditional Access → Create new policy from templates*. Read the template for *Require multifactor authentication for Azure management*.
2. **Report-only:** create it in report-only mode, scoped to a test group.
3. **What If:** simulate yourself signing in to *Windows Azure Service Management API* from another country on an unmanaged device. Read which policies apply and why.
4. **Sign-in logs:** open a recent sign-in, then the *Conditional Access* and *Report-only* tabs. For each policy note *Success*, *Failure* or *Not applied*, and which condition decided it.
5. **Workbook:** open *Insights and reporting* and view the impact of the report-only policy.
6. **Authentication context:** create one called *Privileged action* and note where it can be attached.
7. **ID Protection (P2):** review *Risky users*, *Risky sign-ins* and *Risk detections*.

List policies through Graph:

```bash
az rest --method get \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --query "value[].{Name:displayName, State:state}" -o table
```

## Security, cost and reliability notes

- **Security:** policies apply after first-factor authentication. They do not replace strong credentials.
- **Security:** review exclusions quarterly. An excluded account with a password is your weakest point.
- **Security:** protect Conditional Access itself: restrict who can edit policies, require step-up for changes with protected actions, and alert on policy modifications.
- **Cost:** Conditional Access needs P1; risk-based policies need P2; workload identity and agent policies have their own licences.
- **Reliability:** a bad policy can lock out the whole tenant. Always test in report-only, always exclude break-glass, never roll out to all users in one step.

## Common mistakes

- Enabling a block policy for all users with no break-glass exclusion.
- Skipping report-only mode.
- Trusting the corporate network to skip MFA.
- Forgetting non-interactive and service principal sign-ins when analysing impact.
- Letting exclusion groups grow unreviewed.
- Blocking device code flow without checking which tools legitimately use it.
- Assuming Conditional Access covers managed identities.
- Applying strict device compliance to guests who cannot possibly meet it, without cross-tenant trust configured.

## Check yourself

1. What happens when two policies match and one requires MFA while the other requires a compliant device?
2. What is report-only mode for?
3. Distinguish sign-in risk from user risk and give a typical response to each.
4. Which target resource covers the Azure portal, CLI and ARM?
5. Why should break-glass accounts be excluded, and what compensating control is needed?
6. How does Conditional Access differ for service principals?
7. What is an authentication context?

<details>
<summary>Answers</summary>

1. Both must be satisfied: MFA and a compliant device.
2. Evaluating and logging a policy's effect without enforcing it, so you can measure impact before turning it on.
3. Sign-in risk is the probability that a specific sign-in is not the legitimate user; respond with MFA. User risk is the probability the account is compromised; respond with a secure password change or block.
4. Windows Azure Service Management API (Azure management).
5. So a misconfigured policy or MFA outage cannot lock out every administrator. Compensate with strong credentials, minimal use and an alert on every sign-in.
6. No MFA or device controls. Policies can block by location or by detected risk, apply to single-tenant service principals, and need Workload ID Premium.
7. A label that resources or actions can require, which a policy targets to enforce step-up controls.

</details>

## References

- [What is Conditional Access?](https://learn.microsoft.com/entra/identity/conditional-access/overview)
- [Building a Conditional Access policy](https://learn.microsoft.com/entra/identity/conditional-access/concept-conditional-access-policies)
- [Conditional Access templates](https://learn.microsoft.com/entra/identity/conditional-access/concept-conditional-access-policy-common)
- [Plan a Conditional Access deployment](https://learn.microsoft.com/entra/identity/conditional-access/plan-conditional-access)
- [Conditional Access framework and policies (Zero Trust)](https://learn.microsoft.com/azure/architecture/guide/security/conditional-access-framework)
- [What is Microsoft Entra ID Protection?](https://learn.microsoft.com/entra/id-protection/overview-identity-protection)
- [Conditional Access for workload identities](https://learn.microsoft.com/entra/identity/conditional-access/workload-identity)
- [Microsoft-managed Conditional Access policies](https://learn.microsoft.com/entra/identity/conditional-access/managed-policies)
