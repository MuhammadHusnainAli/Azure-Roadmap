# GitHub Copilot for Azure Workloads

## Why it matters

Developers spend a surprising share of their time not writing business logic: looking up Azure CLI syntax, writing Bicep, wiring identity, chasing deployment errors, upgrading frameworks, migrating legacy apps. GitHub Copilot has evolved from code completion into an agentic assistant that can plan, change code across a repository, run tools, open pull requests and work in the cloud while you do something else. With **GitHub Copilot for Azure** and the **Azure MCP Server**, it also understands and acts on Azure; with **GitHub Copilot modernization** it upgrades Java, .NET and C++ apps and migrates them to Azure. For architects this changes delivery estimates, platform engineering (instructions, guardrails, approved MCP servers) and security review of agent-written code.

## Key concepts

### Copilot surfaces

| Surface | What it does |
|---|---|
| **Inline completions and chat in the IDE** | Suggestions, explanations, edits |
| **Agent mode (IDE)** | Plans multi-step changes, edits multiple files, runs terminal commands and tools (including MCP), iterates on errors |
| **Copilot cloud agent** (formerly coding agent) | Delegate an issue or task; works in a cloud environment and opens a pull request for review |
| **Copilot in the cloud** (from 28 October 2026, paid plans) | Unified sessions that move between GitHub.com, mobile, the Copilot app, IDEs and Copilot CLI; environments: cloud sandbox, GitHub Actions runner or remote custom environment |
| **Copilot code review** | Reviews pull requests |
| **Copilot CLI** | Agentic assistant in the terminal |
| **GitHub Agentic Workflows** | Automations described in Markdown that run as GitHub Actions workflows using Copilot or another coding agent |

Check your plan and organisation policies: GitHub consolidated policies for the new cloud experience, and cloud and Actions compute draw on the account's monthly allowance separately from AI credits.

### GitHub Copilot for Azure

An extension (built into some IDEs) that adds Azure expertise and tools:

| Scenario | Examples |
|---|---|
| **Learn** | "What Azure services should I use for this app?", grounded in current Microsoft Learn docs |
| **Design and develop** | "Build a RAG app in Python to deploy to Azure" |
| **Deploy** | "Deploy my application to Azure" (generates infrastructure, uses `azd`) |
| **Troubleshoot** | Diagnose resource and deployment problems |

It uses the **Azure MCP Server** for tool calls against Azure and installs **Azure skills** (`azure-prepare`, `azure-validate`, `azure-deploy`) for structured multi-step workflows. Supported clients include VS Code (GA), Visual Studio 2026 (GA), Visual Studio 2022 17.14.30+ (Preview), JetBrains IDEs (GA) and Claude Code (GA via the Azure plugin).

### GitHub Copilot modernization

An agentic, end-to-end solution that analyses and upgrades **Java, .NET and C++** apps and helps migrate supported apps to Azure, with humans reviewing every change:

| Capability | Detail |
|---|---|
| **Assessment and planning** | Analyses code, configuration and dependencies; generates a modernisation plan |
| **Code transformation** | Runtime, framework and toolset upgrades (using tools such as OpenRewrite); predefined tasks for Azure migration (secrets to Key Vault, messaging, managed identity) |
| **Build and test** | Fixes build issues, migrates and runs unit tests |
| **Security** | Scans for CVEs after upgrades and applies fixes |
| **Deploy** | Generates containerisation assets, IaC and CI/CD |
| **Modernization agent (CLI)** | Orchestrates assessment and planning across many applications |

IDE upgrade and migration experiences and the modernization agent CLI are GA.

### Custom instructions and context

| File | Scope |
|---|---|
| `.github/copilot-instructions.md` | Repository-wide instructions (coding standards, architecture, Azure conventions) |
| `.github/instructions/*.instructions.md` with `applyTo` globs | Path-specific instructions (for example Bicep files, tests) |
| `AGENTS.md` | Agent instructions read by Copilot and other coding agents |
| MCP configuration | Approved MCP servers (Azure MCP Server, internal tools) |
| Copilot Memory | Longer-term understanding of repositories and preferences |

Example `.github/copilot-instructions.md`:

```markdown
# Azure conventions for this repository
- Infrastructure is Bicep in /infra using Azure Verified Modules; never write ARM JSON.
- Use managed identity and DefaultAzureCredential; never add keys or connection strings with secrets.
- All resources disable public network access and local auth unless the README says otherwise.
- Use `azd` for provisioning and deployment; environments are dev, test, prod.
- Python 3.12, type hints, pytest; run `pytest` and `az bicep build` before proposing changes.
```

Path-specific example `.github/instructions/bicep.instructions.md`:

```markdown
---
applyTo: "infra/**/*.bicep"
---
Tag every resource with owner, cost-centre and environment. Use the latest GA API versions.
```

## How it works on Azure

| Task | How Copilot helps |
|---|---|
| Scaffold an app | Agent mode + GitHub Copilot for Azure creates code, Bicep and `azure.yaml` |
| Deploy | `azure-deploy` skill / `azd up` with validation |
| Diagnose | Azure MCP Server queries resources, logs and metrics |
| Modernise | Copilot modernization upgrades and migrates; generates IaC for App Service, Container Apps or AKS |
| Automate backlog | Assign issues to the cloud agent; review its pull requests; CI runs tests and security scans |

Platform-engineering setup for safe use:

- Organisation policies: allowed models, MCP servers, cloud agent availability, content exclusions.
- Firewall allow-lists for cloud environments; runners with least-privilege Azure access (OIDC to a dev subscription only).
- Branch protection: agent pull requests require human review and passing checks (GitHub Advanced Security, tests, IaC scanning).
- Shared instruction files and templates in an internal "golden path" repository.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **What to delegate to the cloud agent** | Well-specified, testable tasks (tests, small features, refactors, dependency upgrades); not ambiguous architecture changes. |
| **Azure access for agents** | Read-only or dev-only; never production credentials in agent environments. |
| **MCP servers** | Approve a small set (Azure MCP Server, internal docs); review third-party servers as supply chain. |
| **Modernisation** | Use the CLI for portfolio assessment, IDE for per-app upgrades; budget human review time. |
| **Measuring value** | Track cycle time, PR throughput, change failure rate, not lines of code. |
| **Data and IP** | Configure content exclusions for sensitive paths; review enterprise data terms. |

## Hands-on

1. Install GitHub Copilot for Azure in VS Code; ask it to explain your subscription's resources and deploy a sample app with `azd`.
2. Add `.github/copilot-instructions.md` and a Bicep path-specific instruction file; ask agent mode to add a storage account and check it follows the rules.
3. Assign a small issue to the Copilot cloud agent; review the pull request and its session log.
4. Run GitHub Copilot modernization on a sample Java or .NET app; review the plan and changes.
5. Configure branch protection so agent PRs require review and passing security scans.

## Security, cost and reliability notes

- **Agent-written code is untrusted until reviewed**: same review, tests and scanning as human code.
- **Prompt injection via repository content and issues** can steer agents; restrict tools and permissions.
- **Secrets**: never in instructions or agent environments; use OIDC and Key Vault.
- **Cost**: Copilot plans, AI credits and compute minutes; monitor usage.
- **Determinism**: agent results vary; rely on tests and CI as the contract.

## Common mistakes

| Mistake | Fix |
|---|---|
| Giving agents production Azure credentials | Dev-only, least-privilege OIDC |
| No custom instructions, inconsistent output | Repository and path-specific instructions |
| Merging agent PRs without review | Branch protection |
| Unvetted MCP servers | Approved list |
| Measuring success by code volume | Delivery and quality metrics |

## Check yourself

1. Name four GitHub Copilot surfaces and what they do.
2. What does GitHub Copilot for Azure add, and which server does it use?
3. What can GitHub Copilot modernization do?
4. Where do repository and path-specific instructions live?
5. Which environments can Copilot in the cloud use?
6. Name three platform controls for safe agent use.

<details>
<summary>Answers</summary>

1. Any four: completions/chat, agent mode in the IDE, Copilot cloud agent, Copilot in the cloud, code review, Copilot CLI, Agentic Workflows.
2. Azure knowledge, deployment, resource information and troubleshooting via natural language, using the Azure MCP Server and Azure skills (prepare, validate, deploy).
3. Assess and plan; upgrade runtimes, frameworks and toolsets for Java, .NET and C++; migrate to Azure with predefined tasks; fix builds and tests; scan and fix CVEs; generate containerisation, IaC and CI/CD.
4. `.github/copilot-instructions.md`, `.github/instructions/*.instructions.md` with `applyTo`, and `AGENTS.md`.
5. Cloud sandbox, GitHub Actions runner, or a remote custom environment.
6. Any three: organisation policies (models, MCP, features), firewall allow-lists, least-privilege OIDC, branch protection with review and scans, content exclusions.

</details>

## References

- [What is GitHub Copilot for Azure?](https://learn.microsoft.com/azure/developer/github-copilot-azure/introduction)
- [GitHub Copilot modernization](https://learn.microsoft.com/azure/developer/github-copilot-app-modernization/overview)
- [Azure MCP Server](https://learn.microsoft.com/azure/developer/azure-mcp-server/overview)
- [GitHub Copilot on GitHub.com](https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent)
- [Add repository custom instructions](https://docs.github.com/en/copilot/how-tos/configure-custom-instructions/add-repository-instructions)
