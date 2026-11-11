# 27 · DevOps and Platform Engineering

> Tracks: E ● A ○ C ○ · Part E · Secure, Operate, Migrate and Extend

Delivering on Azure safely and quickly depends on how code, infrastructure and AI artefacts move from a developer's branch to production. This topic covers trunk-based development and protection rules; GitHub Actions with OIDC, environments and reusable workflows; Azure DevOps including Managed DevOps Pools; CI/CD for Bicep and Terraform with what-if/plan gates, deployment stacks and drift detection; CI/CD for App Service, Container Apps and AKS; CI/CD for AI applications and agents with evaluation gates; release strategies and feature flags; testing with Azure App Testing (Load Testing and Playwright Workspaces) and Chaos Studio; developer platforms in light of the Dev Box and Deployment Environments retirements; and platform engineering with golden paths and DORA metrics.

## What you will be able to do

- Run trunk-based development with enforced reviews and checks.
- Deploy to Azure from GitHub Actions and Azure Pipelines without secrets.
- Build IaC pipelines with previews, approvals and drift detection.
- Deliver applications with build-once artefacts and safe rollouts on App Service, Container Apps and AKS.
- Promote AI agents, prompts, models and indexes with evaluation and safety gates.
- Use canary, blue-green and feature flags to limit release risk.
- Add load, browser and chaos testing to delivery pipelines.
- Plan developer workstations and self-service environments around current Azure lifecycles.
- Design an internal developer platform and measure developer experience.

## Prerequisites

[Topic 04](../04-tooling-and-infrastructure-as-code/README.md) (IaC and CLI), [Topic 10](../10-application-platform/README.md) and [Topic 07](../07-containers-and-kubernetes/README.md) (hosting platforms), [Topic 22](../22-evaluation-observability-and-genaiops/README.md) for AI evaluation. Git basics.

## Reading order

| # | File | Time | Hands-on |
|---|---|---|---|
| 01 | [Git and Branching Strategy](01-git-and-branching-strategy.md) | 30 min | Rulesets, CODEOWNERS |
| 02 | [GitHub Actions for Azure](02-github-actions-for-azure.md) | 50 min | OIDC, environments, reusable workflow |
| 03 | [Azure DevOps](03-azure-devops.md) | 45 min | YAML pipeline, checks, Managed DevOps Pool |
| 04 | [CI/CD for Infrastructure](04-ci-cd-for-infrastructure.md) | 50 min | what-if, stacks, Terraform plan, drift |
| 05 | [CI/CD for Applications](05-ci-cd-for-applications.md) | 50 min | Container Apps canary, slot swap |
| 06 | [CI/CD for AI Applications and Agents](06-ci-cd-for-ai-applications-and-agents.md) | 45 min | Agent pipeline with evaluation gate |
| 07 | [Release Strategies](07-release-strategies.md) | 35 min | Feature flags, automated canary |
| 08 | [Testing](08-testing.md) | 45 min | Load test gate, Playwright, chaos scenario |
| 09 | [Developer Platforms](09-developer-platforms.md) | 35 min | Dev container, sandbox workflow |
| 10 | [Platform Engineering](10-platform-engineering.md) | 35 min | Golden path, DORA metrics |

**AI engineers:** all of it, especially 02, 05, 06 and 08. **Architects:** 01, 04, 07, 09 and 10. **Consultants:** 07, 09 and 10.

## Completion checklist

- [ ] I have protected `main` with rulesets, code owners and required checks.
- [ ] I deploy to Azure with OIDC and protected environments.
- [ ] I have a YAML Azure Pipeline with workload identity federation and environment checks.
- [ ] I preview IaC changes, deploy with approval and detect drift.
- [ ] I build artefacts once and deploy with slots, revisions or GitOps.
- [ ] I gate agent changes on evaluation against a production baseline.
- [ ] I have released a feature with flags and an automated canary.
- [ ] I run load tests with fail criteria and a chaos scenario under load.
- [ ] I know the Dev Box and Deployment Environments retirement dates and alternatives.
- [ ] I have defined a golden path and collected DORA metrics.

## Maps to

- **AZ-400**: DevOps engineering (source control, pipelines, release, security, instrumentation).
- **AI-300**: CI/CD and GenAIOps for AI workloads.
- **AZ-104/AZ-305** (adjacent): IaC deployment practices.

## Next

28 · [Resiliency, Backup and Disaster Recovery](../28-resiliency-backup-and-disaster-recovery/README.md)
