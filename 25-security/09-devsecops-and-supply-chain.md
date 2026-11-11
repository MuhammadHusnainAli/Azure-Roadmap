# DevSecOps and Supply Chain

## Why it matters

Modern applications are assembled more than written: open-source packages, base container images, GitHub Actions, Terraform modules, AI models and MCP servers all flow into production through pipelines that hold powerful credentials. Attacks such as SolarWinds, the `xz` backdoor, compromised npm packages and poisoned CI actions show that the build system is a target in its own right. **DevSecOps** moves security into every stage of delivery (code, dependencies, secrets, IaC, build, artefacts, deployment) and **supply chain security** ensures you can prove what you shipped, where it came from and that it wasn't tampered with. Azure and GitHub provide the tools: GitHub Advanced Security, Defender for Cloud DevOps security, Artifact Signing, Notary Project signing for containers, SBOMs and build attestations.

## Key concepts

### Controls across the pipeline

| Stage | Control | Tools |
|---|---|---|
| **Code** | Static analysis (SAST), secure code review, branch protection | GitHub code scanning (CodeQL), Copilot Autofix, required reviews |
| **Secrets** | Secret scanning and push protection | GitHub Secret Protection; Advanced Security for Azure DevOps |
| **Dependencies** | Software composition analysis (SCA), vulnerable and malicious package alerts, dependency review on PRs | Dependabot, dependency review, Advanced Security dependency scanning (with malware findings) |
| **Infrastructure as Code** | Misconfiguration scanning before deployment | Defender for Cloud DevOps security (IaC scanning), template analysers, Checkov-style scanners |
| **Build** | Hardened, ephemeral runners; least-privilege tokens; pinned actions; OIDC to cloud | GitHub Actions/Azure Pipelines settings, workload identity federation |
| **Artefacts** | SBOM generation, provenance attestations, code and container signing | SBOM tools, GitHub artifact attestations, **Artifact Signing**, **Notary Project (Notation)** for ACR |
| **Deploy** | Verify signatures and policies before running | AKS image integrity / admission policies, deployment gates, environments with approvals |
| **Runtime** | Vulnerability management and threat detection | Defender for Containers, Defender for Servers, code-to-cloud mapping |

### GitHub security products

| Product | Includes |
|---|---|
| **GitHub Secret Protection** | Secret scanning, push protection, custom patterns |
| **GitHub Code Security** | Code scanning (CodeQL and third-party SARIF), premium Dependabot features, dependency review, security campaigns |
| **Advanced Security for Azure DevOps** | Code, dependency and secret scanning for Azure Repos |

Available for GitHub Team and Enterprise Cloud; some features free for public repositories.

### Defender for Cloud DevOps security

Connects GitHub, Azure DevOps and GitLab to Defender for Cloud: unified view of code, secret, dependency and IaC findings; pull-request annotations; **code-to-cloud mapping** (which repository and pipeline produced a vulnerable container running in production); prioritisation through the security graph (file 05).

### Signing and provenance

| Capability | Detail |
|---|---|
| **Artifact Signing** (formerly Trusted Signing) | Microsoft-managed code signing with zero-touch certificate lifecycle in FIPS 140-3 Level 3 HSMs; Public Trust, Private Trust, VBS enclave and code integrity policy signing; digest signing so files never leave your endpoint |
| **Notary Project / Notation for ACR** | Sign and verify container images and OCI artefacts with keys in Key Vault; replaces Docker Content Trust |
| **Docker Content Trust (DCT) in ACR** | **Deprecated; removed 31 March 2028** – migrate to Notary Project |
| **Build provenance attestations** | GitHub artifact attestations (SLSA provenance, Sigstore-based) prove which workflow built an artefact |
| **SBOM** | Software bill of materials (SPDX or CycloneDX) per build; required by some regulations and customers |

### Frameworks

| Framework | Purpose |
|---|---|
| **SLSA** (Supply-chain Levels for Software Artifacts) | Maturity levels for build integrity and provenance |
| **NIST SSDF** (SP 800-218) | Secure software development practices |
| **OpenSSF Scorecard** | Assess open-source dependency hygiene |
| **Well-Architected secure development lifecycle** | Azure guidance for threat modelling, secure coding and pipeline security |

## How it works on Azure

Secure GitHub Actions workflow building, attesting, signing and pushing an image to ACR without secrets:

```yaml
name: build-and-sign
on: { push: { branches: [main] } }
permissions:
  contents: read
  id-token: write          # OIDC to Azure and for attestations
  attestations: write
  packages: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<full-commit-sha>             # pin actions to SHAs
      - uses: azure/login@<full-commit-sha>
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - name: Build and push
        run: |
          az acr login -n acrcontoso
          docker build -t acrcontoso.azurecr.io/orders:${{ github.sha }} .
          docker push acrcontoso.azurecr.io/orders:${{ github.sha }}
          echo "DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' acrcontoso.azurecr.io/orders:${{ github.sha }} | cut -d@ -f2)" >> $GITHUB_ENV
      - uses: actions/attest-build-provenance@<full-commit-sha>
        with:
          subject-name: acrcontoso.azurecr.io/orders
          subject-digest: ${{ env.DIGEST }}
          push-to-registry: true
      - name: Sign with Notation (key in Key Vault)
        run: |
          notation sign --signature-format cose --id "${{ vars.KV_SIGNING_KEY_ID }}" --plugin azure-kv \
            acrcontoso.azurecr.io/orders@${{ env.DIGEST }}
```

Then enforce at deploy time: AKS image integrity / admission policy that only allows images signed by your key from your registry, and a deployment environment requiring approval for production.

Pipeline hardening checklist:

- OIDC/workload identity federation; no long-lived cloud secrets in CI.
- `GITHUB_TOKEN` default permissions read-only; elevate per job.
- Pin third-party actions to commit SHAs; allow-list actions at organisation level.
- Ephemeral runners; self-hosted runners isolated and not shared across trust levels.
- Protected branches and environments; required reviews and status checks.
- Private package feeds/proxies with upstream allow-lists to reduce dependency confusion.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Block vs warn** | Block on secrets (push protection) and critical vulnerabilities in new code; warn on legacy debt and track. |
| **Where to scan** | PR-time for fast feedback; scheduled scans for newly disclosed CVEs; runtime for what's actually deployed. |
| **Signing scope** | Sign release artefacts and container images; verify at deployment. |
| **SBOM** | Generate for every release; store with artefacts; use for vulnerability response. |
| **Ownership** | Platform team owns pipeline templates; app teams own fixing findings with SLAs. |
| **AI components** | Treat models, prompts, MCP servers and agent tools as supply chain: provenance, review, pinning ([Topic 23 file 02](../23-responsible-ai-security-and-governance/02-ai-threat-landscape.md)). |

## Hands-on

1. Enable secret scanning with push protection and try pushing a test secret.
2. Enable CodeQL code scanning and Dependabot; fix one alert using Copilot Autofix.
3. Connect the repository to Defender for Cloud DevOps security; review IaC findings on a Bicep file.
4. Build the workflow above: OIDC, attestation and Notation signing with a Key Vault key.
5. Configure AKS to allow only signed images; deploy unsigned and signed images to compare.
6. Generate an SBOM for a release and store it as a release asset.

## Security, cost and reliability notes

- **CI credentials are crown jewels**: compromise of a pipeline can deploy anything; least privilege and environment approvals.
- **Fork PRs**: never expose secrets or OIDC to untrusted workflows (`pull_request_target` caution).
- **Alert fatigue**: prioritise exploitable and reachable vulnerabilities; use security campaigns for backlog.
- **Cost**: GitHub Code Security and Secret Protection licensing per active committer; Defender DevOps features within Defender CSPM.
- **Migration deadline**: move ACR signing from DCT to Notary Project before 31 March 2028.

## Common mistakes

| Mistake | Fix |
|---|---|
| Cloud secrets stored in CI | OIDC federation |
| Actions referenced by mutable tags | Pin to commit SHAs |
| Scanning without fixing ownership | SLAs and owners |
| Signing images but not verifying at deploy | Admission/image integrity policies |
| Ignoring transitive dependencies | SCA and SBOM |

## Check yourself

1. List pipeline stages and one security control for each.
2. What do GitHub Secret Protection and GitHub Code Security include?
3. What does Defender for Cloud DevOps security add?
4. What is Artifact Signing, and what was it previously called?
5. What replaces Docker Content Trust in ACR, and by when must you migrate?
6. What are SBOMs and provenance attestations for?

<details>
<summary>Answers</summary>

1. Code: SAST; secrets: push protection; dependencies: SCA/Dependabot; IaC: misconfiguration scanning; build: OIDC and pinned actions; artefacts: signing and SBOM; deploy: signature verification; runtime: Defender vulnerability and threat detection.
2. Secret Protection: secret scanning, push protection, custom patterns. Code Security: code scanning, premium Dependabot, dependency review, security campaigns.
3. Unified DevOps findings across GitHub, Azure DevOps and GitLab, IaC scanning, PR annotations, code-to-cloud mapping and graph-based prioritisation.
4. Microsoft's managed code-signing service with HSM-protected, zero-touch certificate management; formerly Trusted Signing.
5. Notary Project (Notation) signing; DCT is removed on 31 March 2028.
6. SBOMs list components for vulnerability and licence management; provenance attestations prove which workflow and source built an artefact.

</details>

## References

- [What is Artifact Signing?](https://learn.microsoft.com/azure/trusted-signing/overview)
- [About GitHub Advanced Security](https://docs.github.com/en/get-started/learning-about-github/about-github-advanced-security)
- [Defender for Cloud DevOps security](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-devops-introduction)
- [Sign and verify artefacts in ACR](https://learn.microsoft.com/azure/container-registry/overview-sign-verify-artifacts)
- [Docker Content Trust in ACR (deprecated)](https://learn.microsoft.com/azure/container-registry/container-registry-content-trust)
- [Artifact attestations for build provenance](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations/using-artifact-attestations-to-establish-provenance-for-builds)
- [Secure development lifecycle (Well-Architected)](https://learn.microsoft.com/azure/well-architected/security/secure-development-lifecycle)
- [SLSA](https://slsa.dev/)
