# CI/CD for AI Applications and Agents

## Why it matters

An AI application changes in more ways than code: prompts, agent instructions, tool definitions, model versions, retrieval indexes, guardrail configurations and evaluation datasets all alter behaviour, often without a single line of application code changing. If those changes bypass the pipeline, you lose reproducibility, review and rollback; if the pipeline treats them like ordinary code with unit tests only, regressions in quality and safety reach production. CI/CD for AI extends the patterns from files 02–05 with **versioned AI artefacts**, **evaluation gates**, **safety scans** and **controlled promotion** of agents across environments. [Topic 22](../22-evaluation-observability-and-genaiops/README.md) covers evaluation in depth; this file shows how it fits the delivery pipeline.

## Key concepts

### Artefacts to version and promote

| Artefact | Where it lives | Deployed by |
|---|---|---|
| Application code | Git | App pipeline |
| Prompts, agent instructions, skills | Git (`prompts/`, `agents/`), deployed as Foundry agent versions | Agent pipeline (SDK or `azd ai agent`) |
| Tool definitions, MCP server code | Git; MCP servers as apps | App pipeline + agent pipeline |
| Model deployments (model, version, SKU, capacity, upgrade policy) | IaC (Bicep/Terraform) | Infra pipeline |
| Guardrails (RAI policies) | IaC | Infra pipeline |
| Search indexes, chunking/enrichment config | IaC + index definition JSON | Data/index pipeline (with alias swap) |
| Evaluation datasets and evaluator configs | Git / Foundry datasets | Evaluation stage |
| Infrastructure (Foundry account, projects, networking) | IaC | Platform pipeline |

### Pipeline stages for an agent

| Stage | Checks |
|---|---|
| **PR** | Lint prompts/agent YAML, unit tests for tools, **smoke evaluation** comparing candidate agent version vs production baseline, safety evaluators |
| **Merge to main** | Build MCP/tool services, deploy infra changes, create a new **agent version** in dev |
| **Test/staging** | Full evaluation suite, **AI red teaming** scan, load test for latency and tokens, cost estimate |
| **Approval** | Human sign-off for high-risk agents; review evaluation report |
| **Production** | Promote the same agent definition to the production project; canary traffic via gateway or versioned endpoints; continuous evaluation enabled |
| **Rollback** | Point consumers back to the previous agent version/deployment; previous index alias |

### Environment promotion

Use separate Foundry **projects** (or resources) per environment with their own connections, model deployments and Application Insights. Promote **definitions** (agent YAML, prompts, index schema) through Git and pipelines, not by editing in the production portal. Parameterise environment-specific values (model deployment names, connection IDs, endpoints).

### Model version changes

Treat model upgrades as releases: pin versions, set upgrade policy to avoid automatic upgrades in production, evaluate the new version in staging, roll out gradually, and monitor ([Topic 15 file 14](../15-foundry-models-and-azure-openai/14-model-lifecycle.md)).

### Index changes

Rebuild indexes in parallel (`orders-v2`) and switch an **alias** or configuration once evaluation passes; keep the old index for rollback.

## How it works on Azure

Repository layout:

```text
agents/support-agent/agent.yaml        # model, instructions file, tools, guardrail reference
agents/support-agent/instructions.md
tools/order-mcp/                       # MCP server (Container Apps)
infra/                                 # Foundry, model deployments, raiPolicies, Search, APIM
eval/datasets/support-smoke.json       # PR gate
eval/datasets/support-full.jsonl       # release gate
.github/workflows/agent.yml
```

GitHub Actions agent workflow (abridged):

```yaml
name: support-agent
on:
  pull_request: { paths: [ "agents/support-agent/**", "tools/**", "eval/**" ] }
  push: { branches: [ main ], paths: [ "agents/support-agent/**", "tools/**" ] }
permissions: { id-token: write, contents: read }
jobs:
  pr-eval:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    environment: ai-dev
    steps:
      - uses: actions/checkout@<sha>
      - uses: azure/login@<sha>
        with: { client-id: "${{ vars.AZURE_CLIENT_ID }}", tenant-id: "${{ vars.AZURE_TENANT_ID }}", subscription-id: "${{ vars.AZURE_SUBSCRIPTION_ID }}" }
      - run: python scripts/deploy_agent.py --project ${{ vars.FOUNDRY_DEV_ENDPOINT }} --agent agents/support-agent --tag pr-${{ github.event.number }}
      - uses: microsoft/ai-agent-evals@v3-beta
        with:
          azure-ai-project-endpoint: ${{ vars.FOUNDRY_DEV_ENDPOINT }}
          deployment-name: gpt-5-mini
          agent-ids: "support-agent:${{ vars.PROD_BASELINE_VERSION }},support-agent:pr-${{ github.event.number }}"
          data-path: ${{ github.workspace }}/eval/datasets/support-smoke.json
  release:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: ai-production        # required reviewers
    steps:
      - uses: actions/checkout@<sha>
      - uses: azure/login@<sha>
        with: { client-id: "${{ vars.AZURE_CLIENT_ID }}", tenant-id: "${{ vars.AZURE_TENANT_ID }}", subscription-id: "${{ vars.AZURE_SUBSCRIPTION_ID }}" }
      - run: python scripts/deploy_agent.py --project ${{ vars.FOUNDRY_PROD_ENDPOINT }} --agent agents/support-agent
```

`deploy_agent.py` reads `agent.yaml` and instructions and calls `project.agents.create_version(...)` with environment-specific model deployment and tool connection IDs ([Topic 18](../18-agents-with-foundry-agent-service/README.md)); hosted agents use `azd ai agent` deployments. Before the production stage, a staging job runs the full evaluation and a red teaming scan ([Topic 22 files 05 and 09](../22-evaluation-observability-and-genaiops/05-evaluation-in-ci-cd.md)).

Canary for agents: route a percentage of traffic in API Management or the application to the new agent version, compare continuous evaluation scores and latency, then promote.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Portal edits** | Allowed in dev for experimentation; production changes only through pipelines. |
| **Gate strictness** | Smoke gates on PRs (minutes); full evaluation and red teaming before production. |
| **Statistical comparison** | Compare against the production baseline in the same run, not absolute numbers alone. |
| **Model and prompt coupling** | Version them together; a prompt tuned for one model may regress on another. |
| **Cost of evaluation** | Budget judge tokens; keep PR datasets small. |
| **Separation of duties** | AI team owns prompts/agents; platform owns infra; responsible AI reviewers own sign-off for high-risk agents. |

## Hands-on

1. Restructure an agent project into the repository layout above.
2. Write `deploy_agent.py` that creates an agent version from YAML in a given project.
3. Add the PR evaluation job comparing the candidate with the production baseline.
4. Add a staging stage with full evaluation and a red teaming scan; add production approval.
5. Rebuild a search index as `v2`, evaluate, and switch the alias; roll back.

## Security, cost and reliability notes

- **Least privilege**: pipeline identities scoped per project; evaluation identity separate from deployment identity.
- **Data**: evaluation datasets may contain sensitive examples; store and access accordingly.
- **Reliability**: keep previous agent versions and index aliases for instant rollback.
- **Cost**: evaluation and red teaming tokens per run; schedule heavy suites.

## Common mistakes

| Mistake | Fix |
|---|---|
| Editing production agents in the portal | Pipeline-only promotion |
| Unit tests only | Evaluation gates and safety scans |
| Auto-upgrading production models | Pinned versions and staged upgrades |
| Rebuilding indexes in place | Parallel index and alias switch |
| One project for all environments | Separate projects per environment |

## Check yourself

1. Which AI artefacts must be versioned and promoted?
2. What checks belong at PR, staging and production stages?
3. How should agents be promoted between environments?
4. How do you upgrade a model version safely?
5. How do you change a search index without downtime?
6. How do you canary a new agent version?

<details>
<summary>Answers</summary>

1. Code, prompts/instructions/skills, tool definitions and MCP servers, model deployments, guardrails, indexes and their config, evaluation datasets and configs, infrastructure.
2. PR: lint, unit tests, smoke evaluation vs baseline, safety evaluators. Staging: full evaluation, red teaming, load/cost tests. Production: approval, canary, continuous evaluation, rollback readiness.
3. By deploying the same definitions from Git to separate Foundry projects via pipelines with environment parameters.
4. Pin versions, disable auto-upgrade, evaluate in staging, roll out gradually, monitor.
5. Build a new index in parallel, evaluate, switch an alias/config, keep the old one for rollback.
6. Route a percentage of traffic to the new version via the gateway or app, compare evaluations and latency, then promote.

</details>

## References

- [Run evaluations with GitHub Actions](https://learn.microsoft.com/azure/foundry/how-to/evaluation-github-action)
- [MLOps and GenAIOps for AI workloads (Well-Architected)](https://learn.microsoft.com/azure/well-architected/ai/mlops-genaiops)
- [GenAIOps for organisations with MLOps investments](https://learn.microsoft.com/azure/architecture/ai-ml/guide/genaiops-for-mlops)
- [Run AI Red Teaming Agent in the cloud](https://learn.microsoft.com/azure/foundry/how-to/develop/run-ai-red-teaming-cloud)
