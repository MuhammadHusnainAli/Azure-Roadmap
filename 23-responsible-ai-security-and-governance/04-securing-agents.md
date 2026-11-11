# Securing Agents

## Why it matters

A chatbot that says something wrong is embarrassing. An agent that does something wrong (refunds the wrong customer, emails a confidential file, deletes a production table) is an incident. Agents combine an LLM that can be manipulated (file 02) with tools that act on real systems under delegated authority. You can't make the model un-manipulable, so security comes from the architecture around it: **least agency** and least privilege, a distinct identity per agent, human approval for consequential actions, sandboxed execution, controlled network egress, guardrails at tool boundaries, monitoring and a way to stop the agent. This file turns those principles into concrete Azure controls.

## Key concepts

### Defence in depth for agents

| Layer | Control | Stops or limits |
|---|---|---|
| **Scope** | Narrow purpose, few tools, separate read and write agents | Excessive agency, goal hijack blast radius |
| **Identity** | One Entra Agent ID per agent; on-behalf-of where results must respect user rights | Confused deputy, privilege abuse |
| **Authorisation** | Least-privilege RBAC at smallest scope; tool allow-lists; parameter constraints in the tool, not the prompt | Tool misuse |
| **Approval** | Human-in-the-loop for high-risk or irreversible actions | Irreversible damage from hijack or error |
| **Guardrails** | Prompt Shields; tool call and tool response scanning; PII; task adherence (file 03) | Injection, data leakage |
| **Execution isolation** | Sandboxed code execution; hosted agents in isolated containers; no secrets in the sandbox | Unexpected code execution |
| **Network** | Private endpoints; egress controls for hosted agents; firewall allow-lists | Exfiltration, callbacks to attacker infrastructure |
| **Data** | Retrieval security trimming; sensitivity labels and DLP (file 05); memory scoping | Sensitive data disclosure, memory poisoning |
| **Detection** | Tracing, Defender for Cloud AI threat protection, Sentinel | Undetected abuse |
| **Response** | Disable/stop agent, revoke identity, rotate secrets, roll back version | Rogue agents, cascading failures |

### Action risk tiers

Use the same taxonomy as the AI Red Teaming Agent's prohibited actions (Topic 22 file 09):

| Tier | Examples | Rule |
|---|---|---|
| **Prohibited** | Social scoring, emotion inference for decisions, actions outside policy | Never: don't give the agent the tool |
| **High-risk** | Payments, account changes, medical or legal decisions, external email | Human-in-the-loop confirmation |
| **Irreversible** | Deletes, resets, production deployments | Disclosure + explicit confirmation; prefer soft-delete |
| **Low-risk** | Read-only lookups, drafts | Autonomous, logged |

### Enforce in code, not in the prompt

"Never refund more than £200" in the system prompt is guidance. A refund tool that rejects amounts over £200 unless an approval token is present is a control. Validate every tool argument server-side, check the caller's identity and authorisation in the tool, and return minimal data.

### Agent control specifications

Emerging work standardises runtime enforcement: OWASP's **Agent Control Standard** (v0.1) defines policy enforcement points and an **Agent Bill of Materials (AgBOM)** listing an agent's model, tools, data sources and permissions. Even before adopting a standard, keep an agent specification per agent: purpose, owner, model version, instructions version, tools and scopes, data sources, guardrail, approval rules, egress allow-list, and kill-switch procedure. It becomes the security review artefact and the registry entry (file 09).

## How it works on Azure

### Identity and access

- Foundry agents get an **Entra Agent ID** agent identity with no permissions by default; grant the minimum roles ([Topic 18 file 13](../18-agents-with-foundry-agent-service/13-agent-identity-and-authorisation.md), [Topic 02 file 10](../02-identity-and-access/10-microsoft-entra-agent-id.md)).
- Use on-behalf-of for SharePoint, mail and line-of-business data so the agent sees only what the user can.
- Give callers agent-scoped **Foundry Agent Consumer**, not project-wide roles.
- Apply Conditional Access to agent identities where supported.

### Tool approval

MCP tools in Foundry support approval policies; require approval for write tools:

```python
from azure.ai.projects.models import MCPTool

crm_tool = MCPTool(
    server_label="crm",
    server_url="https://apim-contoso.azure-api.net/crm-mcp",
    require_approval={"always": {"tool_names": ["update_customer", "issue_refund"]},
                      "never":  {"tool_names": ["get_customer", "list_orders"]}},
    allowed_tools=["get_customer", "list_orders", "update_customer", "issue_refund"],
)
```

The run returns an `mcp_approval_request` item; your app shows the proposed call to the user or an operator and continues by sending an `mcp_approval_response` only on approval. Log who approved what.

### Sandboxing and egress

| Need | Control |
|---|---|
| Run generated code | Code interpreter in a Microsoft-managed sandbox, or your own container with no credentials and no network |
| Custom agent code | Hosted agents (isolated containers) with their own identity |
| Restrict outbound calls | Guardrail **network egress controls** (Preview) for hosted agents; Azure Firewall FQDN allow-lists for VNet-integrated runtimes |
| Private data paths | Private endpoints for Foundry, Search, Storage; network-secured Foundry agent setup ([Topic 14 file 04](../14-microsoft-foundry-platform/04-foundry-networking.md)) |

### Gateways

Put MCP servers and APIs behind **API Management**: authenticate with Entra, validate tokens and scopes, rate-limit, log, and expose only approved operations; register them in API Center as the approved tool catalogue ([Topic 11 file 07](../11-integration-messaging-and-apis/07-api-management-as-ai-gateway.md)).

### Detect and respond

- Tracing with tool call arguments and results (governed content recording) for investigations.
- Defender for Cloud AI threat protection alerts (file 06) into Defender XDR and Sentinel.
- Foundry Control Plane and Agent 365 to find and stop or block agents centrally (file 10).
- Runbook: disable agent version or endpoint, disable the agent identity, revoke connections, preserve traces, notify owners.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Autonomy level** | Start with human approval for all writes; relax per tool with evidence from evaluation and monitoring. |
| **One agent or several** | Split untrusted-content readers from privileged actors; pass structured, validated data between them. |
| **Approval UX** | Show the exact action and parameters in plain language; avoid approval fatigue by approving only meaningful actions. |
| **Third-party MCP servers** | Treat as supply chain: review, pin versions, host privately or via gateway, restrict tools. |
| **Memory** | Scope by user; don't store instructions from untrusted content; review retention. |
| **Cost controls** | Max turns, max tool calls and token limits per run to stop loops. |

## Hands-on

1. Write an agent specification (AgBOM-style) for a customer-service agent.
2. Give the agent its own identity with read-only access to one Search index and on-behalf-of access to SharePoint.
3. Add an MCP tool with `require_approval` for write operations; implement the approval step in your client.
4. Configure an agent guardrail with indirect attack scanning on tool responses and PII on tool calls.
5. Simulate a compromised document instructing a refund; confirm the approval step and guardrail prevent it; review the trace.
6. Write and test the kill-switch runbook.

## Security, cost and reliability notes

- **Assume compromise**: design so a fully hijacked agent can do only low-impact things without a human.
- **Secrets**: never in prompts, instructions, memory or code-interpreter files; use managed identity and Key Vault-backed connections.
- **Audit**: log approvals, tool calls and identity used; retain per policy.
- **Reliability**: approvals and guardrails add latency; design asynchronous flows for long approvals.

## Common mistakes

| Mistake | Fix |
|---|---|
| Rules in the system prompt as the only control | Enforce in tools and identity |
| Shared service principal for all agents | Per-agent identity |
| Agent reads email and can send email without approval | Approval or separate agents |
| Code execution with network and credentials | Isolated sandbox |
| No way to stop an agent quickly | Kill switch via control plane and identity disablement |

## Check yourself

1. List five layers of defence for agents.
2. How should high-risk and irreversible actions be handled?
3. Why enforce limits in tools rather than prompts?
4. What is an Agent Bill of Materials?
5. How do MCP approval policies work in Foundry?
6. What belongs in an agent kill-switch runbook?

<details>
<summary>Answers</summary>

1. Any five: scope, identity, authorisation, approval, guardrails, execution isolation, network, data, detection, response.
2. High-risk: human-in-the-loop confirmation; irreversible: disclosure plus explicit confirmation, preferably with soft-delete or rollback.
3. Prompts can be overridden by injection; server-side validation and authorisation can't be talked out of their checks.
4. An inventory of an agent's model, tools, data sources, permissions and dependencies used for review and runtime governance.
5. `require_approval` marks tools that need approval; the run pauses with an approval request and resumes only after the client approves.
6. Disable the agent version/endpoint, disable its identity, revoke connections and secrets, preserve traces, notify owners, and roll back.

</details>

## References

- [Govern and secure AI agents (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents/governance-security-across-organization)
- [Guardrails and controls overview](https://learn.microsoft.com/azure/foundry/guardrails/guardrails-overview)
- [What is Microsoft Entra Agent ID?](https://learn.microsoft.com/entra/agent-id/identity-platform/what-is-agent-id)
- [Secure AI (CAF)](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/ai/secure)
- [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)
