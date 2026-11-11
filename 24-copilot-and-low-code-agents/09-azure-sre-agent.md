# Azure SRE Agent

## Why it matters

When an alert fires at 3 a.m., the on-call engineer's first 30 minutes are usually spent gathering context: which metrics moved, what was deployed, which logs matter, whether this happened before. **Azure SRE Agent** automates that investigation. It connects to Azure resources, Azure Monitor and Application Insights, incident platforms (PagerDuty, ServiceNow), source control (GitHub, Azure DevOps) and partner observability tools, then correlates signals, identifies probable root causes, proposes or (when configured) executes mitigations, and writes up the incident. It also runs scheduled health and compliance checks and answers natural-language operational questions. For architects, it's a concrete example of an autonomous operations agent that must be governed carefully: identity, tool permissions, run modes, network isolation and cost.

## Key concepts

### What it does

| Pattern | Example |
|---|---|
| **Automate incidents** | On alert: query monitoring, correlate signals across systems, identify probable root cause, propose mitigations, prefill tickets |
| **Automate scheduled workflows** | Proactive health checks, compliance sweeps, routine operational tasks with results to the incident platform or channel |
| **Investigate and advise** | "What changed in the last hour?", "Why is this service degraded?" with grounded, source-cited answers |

Example from Microsoft: in a memory incident, SRE Agent finds a memory trend in Application Insights starting 40 minutes before the alert, links it to a GitHub deployment two hours earlier, identifies the commit, proposes restarting the pod or adjusting the HPA threshold, and creates a prefilled ServiceNow or PagerDuty ticket.

### Extension points

| Extension | Use |
|---|---|
| **Skills** | Marketplace runbooks and Azure CLI scripts, without custom code |
| **Custom agents** | Specialist agents for operational domains (several ship ready to use; build your own in the agent builder) |
| **Python tools** | Custom logic, transformations and API integrations |
| **MCP servers** | Partner connectors (Datadog, Splunk, New Relic, Dynatrace, Elasticsearch) or any custom MCP tool |
| **Agent hooks** | Event-triggered automations at lifecycle points (after a tool runs, when the agent stops) for policy, telemetry or external approvals |

### Integrations

| Category | Systems |
|---|---|
| Monitoring | Azure Monitor (metrics, logs, alerts, workbooks), Application Insights, Log Analytics |
| Incidents | Azure Monitor alerts, PagerDuty, ServiceNow |
| Source and CI/CD | GitHub (repos, issues), Azure DevOps (repos, work items) |
| Data | Azure Data Explorer, MCP servers |
| Communication | Microsoft Teams, Outlook |
| Managed connectors | SaaS knowledge (SharePoint, Google Drive, Notion, Confluence) depending on configuration |

### Governance controls

| Control | Detail |
|---|---|
| **Identity and RBAC** | Managed identity scoped by Azure RBAC; GitHub Enterprise via a governed "bring your own GitHub App" identity |
| **Run modes** | **Review**: an SRE Agent Administrator approves write actions requiring approval. **Autonomous**: the agent applies them without waiting. Set per response plan or scheduled task |
| **Tool-level access** | Each tool set to **allow**, **ask** or **deny**; admins set global guardrails, team leads per custom agent, users approve in conversation |
| **Network isolation** | VNet integration so agent traffic uses your VNet, NSGs and private DNS to reach private endpoints |
| **Infrastructure as Code** | Deploy agent, network, identity and tool policies with Bicep and Azure CLI in CI/CD |
| **Governed skills** | Private Plugins Marketplace in a private GitHub repository for approved skills |

### Operational memory

The agent retains context from prior investigations (root causes, resolution steps, topology, escalation preferences) and uses it in later incidents, helping new on-call engineers and reducing dependence on individual knowledge.

### Pricing

Usage-based, metered in **Azure Agent Units (AAUs)**; plan capacity with Microsoft's billing guidance.

## How it works on Azure

Rollout pattern:

1. **Scope**: start with one application's resource groups; assign the agent's managed identity **Reader** plus Monitoring Reader; add narrow write roles (for example restart App Service) only for approved mitigations.
2. **Connect**: Azure Monitor alerts, Application Insights, the app's GitHub repo, the incident platform and Teams channel.
3. **Response plans**: define which alerts trigger investigations; start in **Review** mode.
4. **Tools policy**: deny destructive tools; set write tools to "ask".
5. **Skills**: add approved runbooks (restart, scale out, failover) from the private marketplace.
6. **Hooks**: send every executed action to an audit log and change-management system.
7. **Measure**: mean time to detect/investigate/resolve, percentage of correct root-cause hypotheses, approved vs rejected mitigations.
8. **Graduate**: move specific, low-risk mitigations to Autonomous mode once trust is earned.

Deploy with Bicep through the same pipeline as other resources so configuration is versioned and reviewed.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Review vs Autonomous** | Review by default; autonomous only for reversible, well-tested mitigations with clear blast radius. |
| **Permissions** | Least privilege per scope; separate agents for separate apps or teams. |
| **Data access** | Logs and repos can contain sensitive data; restrict scopes and connectors. |
| **Partner observability** | Use MCP connectors where telemetry lives outside Azure Monitor. |
| **Relationship to Azure Copilot** | Azure Copilot for interactive portal work; SRE Agent for continuous, app-focused operations. |
| **Cost** | AAU consumption grows with alert volume and scheduled tasks; tune alerts first. |

## Hands-on

1. Create an SRE Agent for a sample App Service app with Reader access; connect Application Insights and the GitHub repo.
2. Trigger an alert (for example inject errors) and review the investigation thread and proposed mitigation in Review mode.
3. Add a skill that restarts the app and set it to "ask".
4. Configure a scheduled health check that posts results to Teams.
5. Add a hook that writes executed actions to a Log Analytics table; query it.
6. Deploy the same configuration with Bicep.

## Security, cost and reliability notes

- **Autonomous write actions are production changes**: integrate with change management and keep an audit trail.
- **Prompt injection via logs and tickets**: untrusted text in logs or issues can try to steer the agent; tool policies and approvals limit impact.
- **Identity**: managed identity only; no stored secrets; review role assignments regularly.
- **Network**: use VNet integration for private resources.
- **Reliability**: the agent supplements, not replaces, on-call humans and runbooks; keep manual procedures working.

## Common mistakes

| Mistake | Fix |
|---|---|
| Granting Contributor on the subscription | Scoped, least-privilege roles |
| Autonomous mode from day one | Start in Review; graduate selectively |
| Noisy alerts driving cost | Tune alerting before connecting |
| Configuration only in the portal | Bicep in CI/CD |
| No measurement | Track MTTR and hypothesis accuracy |

## Check yourself

1. What three work patterns does SRE Agent support?
2. Name the five extension points.
3. Compare Review and Autonomous run modes.
4. How are tool permissions controlled?
5. What network and identity controls are available?
6. How is SRE Agent billed?

<details>
<summary>Answers</summary>

1. Automating incidents, automating scheduled workflows, investigating and advising.
2. Skills, custom agents, Python tools, MCP servers, agent hooks.
3. Review: an SRE Agent Administrator approves write actions requiring approval; Autonomous: the agent applies them without waiting.
4. Each tool is set to allow, ask or deny, with global admin guardrails, per-custom-agent settings and in-conversation user approval.
5. VNet integration with NSGs and private DNS; managed identity with Azure RBAC; governed GitHub App identity for GitHub Enterprise.
6. Usage-based, metered in Azure Agent Units (AAUs).

</details>

## References

- [Overview of Azure SRE Agent](https://learn.microsoft.com/azure/sre-agent/overview)
- [Run modes](https://learn.microsoft.com/azure/sre-agent/run-modes)
- [Agent hooks](https://learn.microsoft.com/azure/sre-agent/agent-hooks)
- [Billing for Azure SRE Agent](https://learn.microsoft.com/azure/sre-agent/billing)
