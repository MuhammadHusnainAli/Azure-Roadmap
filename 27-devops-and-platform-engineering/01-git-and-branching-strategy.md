# Git and Branching Strategy

## Why it matters

Every pipeline, environment promotion and audit trail starts with how code changes enter the main branch. A branching model that creates long-lived divergent branches leads to painful merges, slow releases and "it worked on my branch" incidents; a model with no controls lets untested or unreviewed code (or an agent's pull request) reach production. Modern teams converge on **trunk-based development** with short-lived branches, **pull requests** as the unit of review, and **branch protection rules** that enforce review, tests and security checks automatically. The same model applies to application code, infrastructure as code, prompts and agent definitions.

## Key concepts

### Branching models

| Model | How it works | Fits |
|---|---|---|
| **Trunk-based development** | Everyone integrates small changes into `main` frequently (at least daily) via short-lived branches; incomplete features hidden behind feature flags; release from `main` | Most teams practising CI/CD; recommended |
| **GitHub flow** | Branch from `main`, open PR, review, merge, deploy | Simple web apps; essentially trunk-based with PRs |
| **Release branches** | Cut `release/x.y` from `main` for stabilisation and hotfixes | Products with supported versions or slower release trains |
| **GitFlow** | Long-lived `develop`, `release`, `hotfix`, `feature` branches | Legacy; adds overhead and merge risk; avoid for continuous delivery |

### Pull requests

A PR packages a change for review and automated validation: description and linked work item, small diff, CI status checks, code owners' review, security scans, and deployment previews where useful. Keep PRs small (a few hundred lines) and short-lived (hours to a couple of days).

### Protection rules

| Control | GitHub | Azure Repos |
|---|---|---|
| Require PR before merge | Rulesets / branch protection | Branch policies |
| Required reviewers / code owners | `CODEOWNERS`, required approvals | Required reviewers, automatically included reviewers by path |
| Required status checks | Required checks (CI, CodeQL, IaC scans) | Build validation policies |
| Linear history / merge strategy | Squash or rebase merges | Limit merge types |
| Signed commits | Require signed commits | Not native (use policies) |
| Block force push and deletion | Rulesets | Branch security permissions |
| Secret push protection | Secret scanning push protection | Advanced Security for Azure DevOps |
| Bypass | Limited roles, audited | Limited, audited |

GitHub **rulesets** apply consistently across many repositories at organisation level, which is better than per-repo settings for platform teams.

### Commit and versioning conventions

| Practice | Benefit |
|---|---|
| **Conventional Commits** (`feat:`, `fix:`, `chore:`) | Automated changelogs and semantic versioning |
| **Semantic versioning** for libraries, templates, modules | Clear compatibility contracts |
| **Tags and releases** | Immutable references for deployments |
| **Linking work items** | Traceability for audit |

### Repository strategy

| Option | Trade-off |
|---|---|
| **Monorepo** | Atomic cross-component changes, shared tooling; needs path-based CI and ownership |
| **Polyrepo** | Clear ownership and independent pipelines; harder cross-repo changes |
| **Infrastructure repos** | Separate platform/landing-zone repos from application repos; app repos hold their own workload IaC |

## How it works on Azure

`CODEOWNERS` routing reviews:

```text
# .github/CODEOWNERS
/infra/                @contoso/platform-team
/src/payments/         @contoso/payments-team
/prompts/ /agents/     @contoso/ai-team @contoso/responsible-ai
*.bicep                @contoso/platform-team
```

GitHub ruleset intent for `main` across all repositories in the organisation:

```text
Target: default branch, all repositories with topic "production"
Rules: require pull request (1 approval, code owner review, dismiss stale approvals),
       require status checks: build, unit-tests, codeql, iac-scan,
       require linear history, block force pushes and deletions,
       require signed commits
Bypass: release-managers team (audited)
```

Azure Repos branch policy with the CLI:

```bash
az repos policy approver-count create --branch main --repository-id <repo-id> \
  --minimum-approver-count 1 --creator-vote-counts false --allow-downvotes false \
  --reset-on-source-push true --blocking true --enabled true
az repos policy build create --branch main --repository-id <repo-id> \
  --build-definition-id <pipeline-id> --display-name "CI validation" \
  --queue-on-source-update-only true --valid-duration 720 --blocking true --enabled true
```

Feature flags (Azure App Configuration, file 07) let incomplete work merge to `main` safely, which is what makes trunk-based development practical.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Model** | Trunk-based with short-lived branches; release branches only for supported versions. |
| **Merge strategy** | Squash merges for clean history on `main`; rebase if teams want granular commits. |
| **Review depth** | Code owners for sensitive paths (infra, auth, prompts); one approval elsewhere. |
| **Agent-authored PRs** | Same rules as humans; require human approval ([Topic 24 file 07](../24-copilot-and-low-code-agents/07-github-copilot-for-azure-workloads.md)). |
| **Org-wide policy** | Rulesets/policies managed as code by the platform team. |
| **Hotfixes** | Fix on `main` and deploy forward; cherry-pick to release branches if needed. |

## Hands-on

1. Create a repository with `main` protected by a ruleset requiring PRs, a CI check and code owner review.
2. Add a `CODEOWNERS` file and observe automatic reviewer assignment.
3. Open a PR that fails CI and confirm merge is blocked.
4. Enable secret scanning push protection and try to push a test secret.
5. Adopt Conventional Commits and generate a changelog with a release tool.

## Security, cost and reliability notes

- **Branch protection is a security control**: unreviewed changes to IaC or pipelines can compromise the estate.
- **Protect pipeline definitions** (`.github/workflows`, `azure-pipelines.yml`) with code owners.
- **Bypass permissions** should be rare, audited and time-bound.
- **Long-lived branches** increase integration risk and delay security fixes.

## Common mistakes

| Mistake | Fix |
|---|---|
| GitFlow for continuous delivery | Trunk-based |
| Huge PRs open for weeks | Small, frequent PRs; feature flags |
| Admins bypassing protections routinely | Audited, exceptional bypass |
| Per-repo settings drifting | Organisation rulesets as code |
| Workflows editable without review | Code owners on CI files |

## Check yourself

1. Describe trunk-based development and why it suits CI/CD.
2. When are release branches justified?
3. What controls belong in branch protection?
4. What does `CODEOWNERS` do?
5. How do feature flags support trunk-based development?
6. Why protect pipeline definition files?

<details>
<summary>Answers</summary>

1. Small changes integrated into `main` frequently via short-lived branches, releasing from `main`; it minimises merge pain and keeps `main` always releasable.
2. When you must support multiple released versions or stabilise a release train.
3. Required PRs, approvals and code owner review, required status checks, linear history, block force push/deletion, signed commits, push protection, controlled bypass.
4. Automatically requests reviews from owners of matching paths and can be required for approval.
5. Incomplete work can merge to `main` but stay hidden in production until enabled.
6. Changing them can alter what gets deployed or exfiltrate secrets, so they need strict review.

</details>

## References

- [Trunk-based development (Azure DevOps guidance)](https://learn.microsoft.com/devops/develop/how-microsoft-develops-devops)
- [About rulesets (GitHub)](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)
- [About code owners (GitHub)](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
- [Azure Repos branch policies](https://learn.microsoft.com/azure/devops/repos/git/branch-policies)
