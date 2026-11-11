# Testing

## Why it matters

Fast delivery is only safe if tests catch problems before customers do. Unit and integration tests verify logic, but many production failures come from things they can't see: a page that breaks in one browser, an API that collapses at 2,000 requests per second, a dependency failure that cascades because retries weren't configured, a zone outage nobody rehearsed. Azure provides managed services for the harder kinds of testing: **Azure App Testing** brings together **Azure Load Testing** (JMeter and Locust at scale) and **Playwright Workspaces** (cloud browsers for end-to-end tests and browser automation), and **Azure Chaos Studio** injects faults to prove resilience. Wiring them into pipelines makes performance and resilience regressions visible like failing unit tests.

## Key concepts

### Test types

| Type | Goal | Where it runs |
|---|---|---|
| Unit | Logic correctness | Every commit |
| Integration / contract | Components and APIs work together | PR, dev environment |
| End-to-end (E2E) | User journeys across UI, APIs, data | Staging, PR preview environments |
| Load / performance | Throughput, latency, capacity limits, regressions | Staging; scheduled and pre-release |
| Chaos / resilience | Behaviour under faults (zone down, dependency failures, latency) | Staging, game days, carefully in production |
| Security (DAST) | Runtime vulnerabilities | Staging |
| AI evaluation | Quality and safety of AI outputs | [Topic 22](../22-evaluation-observability-and-genaiops/README.md) |

### Azure App Testing

| Component | Capabilities |
|---|---|
| **Azure Load Testing** | Fully managed high-scale load generation; URL-based quick tests or **JMeter** and **Locust** scripts; server-side metrics from Azure resources for bottleneck analysis; **fail criteria** (response time, error rate, resource metrics) for CI/CD gates; private endpoint and on-premises testing via VNet injection; test comparison across runs |
| **Playwright Workspaces** | Fully managed cloud browsers for Playwright tests at scale (parallel runs across browsers/OS), browser automation and powering AI agents through browser interactions; session recordings, traces and artefacts; Entra authentication |

### Azure Chaos Studio

Managed chaos engineering and resilience testing. The current resource model, **Chaos Studio Workspaces and Scenarios**, is in **public preview**:

| Concept | Meaning |
|---|---|
| **Workspace** | Connected to a scope (subscription, resource group or service group); discovers resources and recommends scenarios |
| **Scenario** | Preconfigured resilience test, for example **Compute Zone Down**, DNS outage, database failover, cache stampede, messaging disruption; customise with the scenario designer |
| **Faults/actions** | Service-direct (for example shut down VM, AKS Chaos Mesh faults, Cosmos DB failover) and agent-based (CPU pressure, kill process, network latency) |
| **Safeguards** | Scoped permissions, managed identity, explicit targets, stop experiments, observe with Azure Monitor |

Pair Chaos Studio experiments with load tests: resilience problems show up under load.

### Gates

| Gate | Example criteria |
|---|---|
| Load test | p95 < 500 ms, error rate < 1%, CPU < 75% at 2× expected peak |
| E2E | All critical journeys pass in Chromium, Firefox, WebKit |
| Chaos | Availability SLO maintained during zone-down scenario; recovery < 5 minutes |

## How it works on Azure

Load test configuration (`loadtest.yaml`) with fail criteria:

```yaml
version: v0.1
testId: orders-api-peak
displayName: Orders API peak load
testPlan: orders.jmx
engineInstances: 4
failureCriteria:
  - avg(response_time_ms) > 300
  - percentage(error) > 1
  - p95(response_time_ms) > 800
autoStop:
  errorPercentage: 50
  timeWindow: 60
```

GitHub Actions step:

```yaml
- uses: azure/load-testing@<sha>
  with:
    loadTestConfigFile: loadtest.yaml
    loadTestResource: lt-contoso
    resourceGroup: rg-testing
```

Playwright tests against cloud browsers (after creating a Playwright workspace, per the quickstart):

```bash
npm init @azure/playwright@latest                     # adds the service package and playwright.service.config.ts
export PLAYWRIGHT_SERVICE_URL="<region endpoint from the workspace>"
az login                                              # Entra authentication to the workspace
npx playwright test --config=playwright.service.config.ts --workers=20
npx playwright show-report
```

Chaos experiment flow: create a workspace scoped to the staging resource group, select **Compute Zone Down**, run while a load test generates traffic, watch Application Insights availability and failover behaviour, record findings and fixes.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Where to load test** | Production-like staging with realistic data volumes; production only with careful scoping. |
| **Frequency** | Smoke load test per release; full peak test before events; scheduled weekly regressions. |
| **E2E scope** | Critical journeys only; many E2E tests become slow and flaky. |
| **Chaos maturity** | Start with game days in staging; automate experiments later; production chaos only with strong observability and blast-radius control. |
| **Test data** | Synthetic or masked data; never copy production PII to test environments without controls. |
| **Ownership** | Teams own their tests; platform provides shared resources and templates. |

## Hands-on

1. Create a URL-based load test, then convert it to JMeter with fail criteria and run it in CI.
2. Compare two load test runs and identify a regression using server-side metrics.
3. Run a Playwright suite on Playwright Workspaces across three browsers with 20 workers.
4. Create a Chaos Studio workspace and run a Compute Zone Down scenario against a zone-redundant app under load.
5. Document results in a resilience review and fix one finding.

## Security, cost and reliability notes

- **Load tests can look like attacks**: coordinate with WAF/DDoS settings and owners; never load test third-party systems without permission.
- **Chaos permissions**: experiments use managed identities with explicit roles on targets; restrict who can start them.
- **Cost**: load testing virtual user hours, Playwright test minutes, extra environment capacity.
- **Preview**: Chaos Studio Workspaces/Scenarios are Preview; validate before relying on them for compliance evidence.

## Common mistakes

| Mistake | Fix |
|---|---|
| Load testing only before launch | Regular regression tests in CI |
| Load tests without server-side metrics | Connect Azure resources to find bottlenecks |
| Huge flaky E2E suites | Focus on critical journeys |
| Chaos without hypotheses or observability | Define steady state and success criteria |
| Testing in an unrealistic environment | Production-like staging |

## Check yourself

1. What does Azure App Testing include?
2. Which frameworks does Azure Load Testing support, and how do fail criteria work?
3. What do Playwright Workspaces provide?
4. What are Chaos Studio workspaces and scenarios?
5. Why combine chaos and load tests?
6. Give three example quality gates.

<details>
<summary>Answers</summary>

1. Azure Load Testing and Playwright Workspaces.
2. JMeter and Locust (plus URL-based tests); fail criteria on response time, error rate or resource metrics fail the test run and the pipeline.
3. Managed cloud browsers for scalable Playwright testing and browser automation, with recordings, traces and artefacts.
4. A workspace connects to a scope, discovers resources and recommends scenarios; scenarios are preconfigured resilience tests (Preview).
5. Many resilience weaknesses only appear under realistic load.
6. p95 latency and error rate thresholds in load tests; all critical journeys passing in E2E; SLO maintained and recovery time met in chaos scenarios.

</details>

## References

- [What is Azure Load Testing?](https://learn.microsoft.com/azure/app-testing/load-testing/overview-what-is-azure-load-testing)
- [Azure App Testing documentation](https://learn.microsoft.com/azure/app-testing/)
- [What is Playwright Workspaces?](https://learn.microsoft.com/azure/app-testing/playwright-workspaces/overview-what-is-microsoft-playwright-workspaces)
- [What is Azure Chaos Studio?](https://learn.microsoft.com/azure/chaos-studio/chaos-studio-overview)
