# Microsoft Entra Agent ID

> Topic: 02 · Identity and Access · Tracks: E ● A ● C ● · Status checked: 2026-10

## Why it matters

AI agents read data, call APIs and take actions. Each of those needs an identity to be authorised and audited against. Until recently an agent borrowed a service principal or, worse, a user's credentials, which left nobody able to answer "which agents do we have, who owns them, and what can they touch?" Microsoft Entra Agent ID gives agents their own identity type. This area is new and moving quickly; treat specifics as a snapshot and verify against current documentation before a design review.

## Key concepts

### Why existing identity types fall short

| | Human user | Application (service principal) | AI agent |
|---|---|---|---|
| Lifetime | Years | Months to years | Minutes to years |
| Volume | One per person | Tens to hundreds | Potentially thousands, created dynamically |
| Created by | HR and IT | Developers and IT | Anyone, including business users in low-code tools |
| Behaviour | Bounded by a person's actions | Deterministic code | Non-deterministic, goal-seeking |
| Acts for | Itself | Itself or a user | Itself, a user, or another agent |
| Authentication | Passwords, MFA, passkeys | Secrets, certificates, federation | Token-based, no human credentials |

The identity problems agents introduce:

- **Distinguishing** agent activity from human and ordinary workload activity in logs and policies.
- **Right-sizing** access when an agent's behaviour is not fully predictable.
- **Keeping agents away** from the most sensitive roles and systems.
- **Scale and churn:** many agents, created and destroyed quickly, without leaving orphaned credentials.
- **Accountability:** a named human responsible for each agent.

### Object model

| Object | What it is |
|---|---|
| **Agent identity blueprint** | A template that defines a kind of agent: its configuration, the permissions it may request, how its instances authenticate. Comparable to an app registration |
| **Agent identity** | An individual identity created from a blueprint and used by a running agent to obtain tokens. Comparable to a service principal, but a distinct type |
| **Agent's user account** (agent user) | An optional user-like account paired one to one with an agent identity, for systems that only work with user objects (for example a mailbox or a Teams presence) |
| **Sponsor** | The human accountable for the agent's existence and access |
| **Owner** | Whoever administers the blueprint or identity |
| **Agent registry** | The inventory of agents in the organisation, with metadata for discovery and governance |

Blueprints make scale manageable: you grant and govern at the blueprint level and can create or retire many agent identities consistently.

### How agents use their identity

| Mode | What happens | Access is |
|---|---|---|
| **Autonomous** | The agent acts as itself, using rights granted directly to the agent identity: Microsoft Graph permissions, Azure RBAC roles, Entra roles, app roles | Whatever the agent was granted |
| **Delegated (on behalf of)** | The agent acts for a signed-in user, with rights that user delegated | Bounded by the user's own access and by what was consented |
| **Inbound authentication** | Other clients, users and agents call the agent with Entra-issued tokens, so the agent can identify its caller | – |

Agents do not use passwords or MFA. They authenticate with platform-managed credentials through token exchange, in the same spirit as managed identities and federated credentials.

**Prefer delegated access whenever a user is present.** An agent answering a question for Alice should see what Alice can see. Reserve autonomous access for genuinely unattended work, and scope it tightly.

### Where agent identities come from

You usually do not create them by hand. Platforms do:

| Platform | Behaviour |
|---|---|
| **Microsoft Foundry** | Agents built in Foundry Agent Service receive an agent identity; publishing an agent gives it a distinct identity for production use |
| **Copilot Studio** | Each agent created receives an agent identity; the creating user is recorded as its sponsor |
| **Microsoft first-party agents** | For example the Conditional Access optimisation agent, Security Copilot agents |
| **Custom and third-party agents** | Registered through Microsoft Graph APIs and SDK support for agent identities, including agents running outside Azure |

Agent sign-ins and actions are logged in Entra ID as agent activity, separate from users and service principals.

### Governance and protection

The identity platform extends its existing controls to agents:

| Control | Applied to agents |
|---|---|
| **Inventory** | Agent identities and blueprints listed in the Entra admin centre; agent registry |
| **Sign-in and audit logs** | Agent authentications and changes, attributable to the agent |
| **Conditional Access** | Policies targeting agents, for example blocking risky agents or restricting which resources agents can reach |
| **ID Protection** | Risk detections for agent identities, such as anomalous access patterns |
| **Lifecycle governance** | Sponsors, expiry, lifecycle workflows when a sponsor leaves |
| **Access reviews and access packages** | Recertifying what agents can access |
| **Restrictions on privilege** | Agents are kept away from the most powerful directory roles by design |

### Licensing

- **Microsoft Entra Agent ID** as a platform for creating and managing agent identities is available to all Microsoft Entra customers.
- Extending Entra **security features** to agents (the Conditional Access, protection and governance capabilities above) requires **Microsoft Agent 365**.
- Agent 365 is included in Microsoft 365 E7 and available as an add-on to E5 and certain other suites, licensed per user.

Licensing in this area has changed more than once. Confirm it before quoting.

### How it fits with Agent 365 and Foundry Control Plane

Three layers are easy to confuse:

| Layer | Role | Audience |
|---|---|---|
| **Entra Agent ID** | The identity itself: authentication, authorisation, identity governance | Identity and security teams |
| **Microsoft Agent 365** | Organisation-wide control plane for agents: registry, access control, visualisation, interoperability with Microsoft 365, and security through Defender and Purview | IT administrators |
| **Foundry Control Plane** | Fleet operations for agents you build: observability, evaluation, guardrails, cost, policy | Developers and platform teams |

Agent ID is the foundation the other two rely on. Topic 23 covers agent governance as a whole.

## How it works on Azure

### Authorising an agent on Azure resources

An agent identity is a security principal, so Azure RBAC applies as usual:

```text
Agent identity ──role assignment──► scope
   e.g.  Search Index Data Reader on one search service
         Storage Blob Data Reader on one container
         a custom role allowing exactly one API operation
```

Design rules:

1. **One identity per agent**, not one shared identity for a fleet with different jobs.
2. **Narrowest scope.** Resource or sub-resource, never subscription.
3. **Data roles, not management roles.** Agents rarely need control-plane rights.
4. **Separate identities for development and production.** Foundry's distinction between a project's shared identity during development and a published agent's own identity supports this.
5. **Never Owner, Contributor or User Access Administrator.**

### Tool access is where risk concentrates

An agent's permissions are effectively the union of what its tools can do. For each tool decide:

| Question | Safer answer |
|---|---|
| Whose identity does this tool call use? | The user's (delegated) when a user is present |
| Read or write? | Read-only unless a write is the purpose |
| Can it reach beyond the task's data? | Scope to specific resources |
| Is the action reversible? | Require human approval for irreversible or high-impact actions |
| Can it be triggered by untrusted content? | Assume yes (prompt injection), and limit accordingly |

A prompt injection cannot make an agent exceed its identity's permissions. That is the point of least privilege here: it bounds the damage of a failure you should expect to happen eventually.

### MCP and agent-to-agent calls

- A remote **MCP server** should authenticate callers with Entra ID. With agent identities, the server can tell which agent is calling and, in delegated flows, for which user.
- In **agent-to-agent** calls, identity should propagate so the final resource can authorise the original user or the calling agent, not an anonymous hop.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Autonomous vs delegated | Delegated by default for interactive agents. Autonomous for scheduled or event-driven agents, with narrow scope |
| Agent identity vs managed identity vs service principal | Use the agent identity the platform provides for the agent. Use managed identities for the surrounding infrastructure (the app hosting it, search, storage) |
| Agent user account | Only when a target system cannot work with a non-user identity |
| Who can create agents | Decide deliberately. Open creation in low-code tools produces sprawl; a governed intake with sponsors keeps the inventory meaningful |
| Sponsor model | A named individual plus a backup or team, with a lifecycle rule for when the sponsor leaves |
| Approval gates | Human-in-the-loop for writes, external communication, financial or privileged actions |

**A practical checklist for an agent design review**

- Does every agent have a unique identity and a named sponsor?
- Is each permission justified by a specific tool or task?
- Which actions run as the user and which as the agent?
- What is the worst outcome if the agent is manipulated, and is that acceptable?
- How is the agent disabled quickly, and who can do it?
- Where are its sign-ins and actions logged, and who looks?
- When is its access reviewed, and when does it expire?

## Hands-on

1. In the Entra admin centre, open **Agent ID** (under *Identity* or *Entra ID*, depending on the current navigation). Review **Agent identities** and **Agent identity blueprints**. Note any created by Microsoft services or Copilot Studio.
2. Open one agent identity. Find its blueprint, sponsor, owners and granted permissions.
3. Open **Sign-in logs** and filter to agent sign-ins. Compare the fields with a user sign-in.
4. If you have a Foundry project, create a simple agent in the Foundry portal, then return to Entra and locate the identity associated with the project or agent.
5. Grant that identity a narrow role and verify it:

   ```bash
   AGENT_OBJECT_ID=<object id from the Entra portal>
   SEARCH_ID=$(az search service show -n <search> -g <rg> --query id -o tsv)

   az role assignment create \
     --assignee-object-id $AGENT_OBJECT_ID --assignee-principal-type ServicePrincipal \
     --role "Search Index Data Reader" --scope $SEARCH_ID

   az role assignment list --assignee $AGENT_OBJECT_ID --all \
     --query "[].{Role:roleDefinitionName, Scope:scope}" -o table
   ```

6. Write down, for one agent you know, the answers to the design review checklist above.

Query agent identities through Microsoft Graph (the API surface is evolving; check the current reference):

```bash
az rest --method get \
  --url "https://graph.microsoft.com/beta/servicePrincipals?\$filter=servicePrincipalType eq 'Agent'&\$select=displayName,id,appId" \
  --query "value[]" -o table
```

## Security, cost and reliability notes

- **Security:** treat agent sprawl as a first-order risk. An unknown agent with data access is shadow IT that can act.
- **Security:** the people who can edit an agent's instructions or tools effectively control its identity. Protect authoring access.
- **Security:** watch for agents granted application-level Graph permissions (for example read all mail or all sites). Those are tenant-wide.
- **Security:** build a kill switch into operations: disabling the agent identity stops token issuance.
- **Cost:** the identity platform is included; protection and governance features depend on Agent 365 licensing.
- **Reliability:** an agent whose identity expires or is disabled stops working. Tie lifecycle rules to monitoring so that this is deliberate, not a surprise.

## Common mistakes

- Running agents under a developer's personal credentials or a shared service account.
- One over-privileged identity for every agent in a project.
- Granting application permissions when delegated access would do.
- No sponsor, or a sponsor who has left.
- Assuming content filters make broad permissions safe.
- Reusing the development identity in production.
- Forgetting that agents created in Copilot Studio by business users are also identities in your tenant.
- Describing preview capabilities to a client as generally available.

## Check yourself

1. Name three ways agents differ from ordinary applications from an identity perspective.
2. What is the relationship between an agent identity blueprint and an agent identity?
3. Explain autonomous vs delegated access.
4. What is a sponsor?
5. Why does least privilege matter especially for agents exposed to untrusted content?
6. How do Entra Agent ID, Agent 365 and Foundry Control Plane relate?
7. An interactive HR agent looks up a user's leave balance. Whose identity should the lookup use, and why?

<details>
<summary>Answers</summary>

1. Any three of: created dynamically and in large numbers; short or unpredictable lifetimes; non-deterministic behaviour; can act for users or other agents; often created by non-IT users.
2. The blueprint is the template defining a kind of agent and what it may request. Agent identities are the individual identities instantiated from it.
3. Autonomous: the agent acts as itself using rights granted to its identity. Delegated: it acts on behalf of a user, limited to what the user can do and has consented to.
4. The human accountable for an agent's existence, purpose and access.
5. Prompt injection can redirect the agent's behaviour but cannot exceed the identity's permissions, so narrow permissions bound the damage.
6. Agent ID provides the identity and identity governance. Agent 365 is the organisation-wide control plane for managing and securing agents. Foundry Control Plane handles operational governance and observability for agents built on Foundry.
7. The user's identity through delegated access, so the HR system's own authorisation ensures they can see only their own balance.

</details>

## References

- [What are agent identities?](https://learn.microsoft.com/entra/agent-id/what-are-agent-identities)
- [Microsoft Entra Agent ID documentation](https://learn.microsoft.com/entra/agent-id/)
- [Agent identity concepts in Microsoft Foundry](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-identity)
- [Microsoft Agent 365 overview](https://learn.microsoft.com/microsoft-agent-365/overview)
- [Microsoft Agent 365 integration with Foundry](https://learn.microsoft.com/azure/foundry/agents/concepts/agent-365-integration)
- [Conditional Access for workload identities](https://learn.microsoft.com/entra/identity/conditional-access/workload-identity)
- [AI agents adoption guidance (Cloud Adoption Framework)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/)
