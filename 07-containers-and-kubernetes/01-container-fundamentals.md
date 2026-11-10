# Container Fundamentals

## Why it matters

Almost every modern Azure host runs containers: Container Apps, AKS, App Service, Functions, Container Instances, Azure Machine Learning endpoints and Foundry hosted agents. A container image is the unit you build, scan, sign, store and deploy. Engineers who build good images ship faster and more securely; architects who understand images can reason about cold starts, supply-chain risk and portability.

## Key concepts

### Containers versus virtual machines

| | Virtual machine | Container |
|---|---|---|
| Isolation boundary | Hypervisor, own kernel | Shared host kernel, isolated with namespaces and cgroups |
| Start time | Tens of seconds to minutes | Milliseconds to seconds (plus image pull) |
| Size | Gigabytes | Megabytes to a few gigabytes |
| Packaging | Disk image | Image of layered file systems plus metadata |
| Density | A few per host | Dozens to hundreds per host |
| Security isolation | Strong | Weaker by default; strengthened with sandboxing (Kata, Hyper-V, confidential containers) |

A container is just a process with a restricted view of the system. The image is the file system and configuration that process starts with.

### Images, layers and tags

- An **image** is an ordered stack of read-only **layers** plus a **config** (entrypoint, environment, user, exposed ports) and a **manifest** that lists them.
- Each instruction in a Dockerfile that changes the file system (`RUN`, `COPY`, `ADD`) creates a layer. Layers are content-addressed by SHA-256 digest and shared between images.
- A **tag** (`myapp:1.4.2`) is a mutable pointer. A **digest** (`myapp@sha256:9f2…`) is immutable. Deploy by digest, or by tags you never overwrite, in production.
- A **multi-architecture image** (an image index) holds variants for `linux/amd64`, `linux/arm64` and so on. The runtime picks the right one.

### The Open Container Initiative (OCI)

The OCI defines three specifications: **image**, **runtime** and **distribution**. Any OCI-compliant registry stores any OCI artefact, not only container images. That is why registries now also hold Helm charts, signatures, software bills of materials (SBOMs), vulnerability reports and even AI model weights, linked to an image through the **referrers** API.

### Registries

A registry stores repositories of images. Public registries include Docker Hub, GitHub Container Registry and the Microsoft Artifact Registry (`mcr.microsoft.com`). Private workloads should pull from a private registry you control, such as Azure Container Registry (file 02), and should **cache or import** public base images rather than pulling from the internet at deploy time. Docker Hub rate-limits anonymous pulls, which has broken many unprepared deployments.

### Multi-stage builds

Build tools, compilers and test dependencies do not belong in the runtime image. A multi-stage Dockerfile builds in one stage and copies only the result into a minimal final stage.

```dockerfile
# syntax=docker/dockerfile:1

# ---- build stage ----
FROM python:3.12-slim AS build
WORKDIR /app
RUN pip install --no-cache-dir uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project
COPY src ./src

# ---- runtime stage ----
FROM python:3.12-slim
WORKDIR /app
RUN useradd --uid 10001 --create-home appuser
COPY --from=build /app/.venv /app/.venv
COPY --from=build /app/src ./src
ENV PATH="/app/.venv/bin:$PATH" PYTHONUNBUFFERED=1
USER 10001
EXPOSE 8000
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Base image choices

| Base | Size | Notes |
|---|---|---|
| Full distribution (`ubuntu`, `python:3.12`) | Large | Easy debugging, large attack surface |
| Slim variants (`python:3.12-slim`, `debian:bookworm-slim`) | Medium | Good default |
| Alpine | Small | musl libc can break Python wheels and native libraries |
| **Azure Linux** (`mcr.microsoft.com/azurelinux/base/...`) | Small | Microsoft-maintained, used across Azure services |
| Distroless / chiseled (for example .NET chiseled Ubuntu images) | Very small | No shell or package manager; hardest to attack and to debug |
| GPU bases (`nvidia/cuda`, framework images) | Very large | Pin versions; layers of several gigabytes slow cold start |

### Layer ordering and caching

Builders reuse a cached layer when its instruction and inputs are unchanged. Put things that change rarely (OS packages, dependency manifests) before things that change often (application code). Copying the whole repository before installing dependencies invalidates the dependency layer on every commit.

Add a `.dockerignore` to keep `.git`, virtual environments, test data, secrets and local `.env` files out of the build context.

### Twelve-factor behaviour inside a container

- Configuration from environment variables, never baked in.
- Logs to stdout and stderr, not to files.
- Stateless processes; state goes to a database, cache or storage.
- Fast start and graceful shutdown on `SIGTERM`.
- One main process per container. Sidecars are separate containers.

## How it works on Azure

| Azure host | What it runs |
|---|---|
| Azure Container Instances | Single containers or small groups, no orchestrator (file 03) |
| Azure Container Apps | Serverless containers with scale to zero, revisions, jobs (files 04, 05) |
| Azure Kubernetes Service | Managed Kubernetes (files 06 – 11) |
| App Service / Functions | Web apps and functions from custom containers (topic 10) |
| Azure Machine Learning, Foundry | Model serving and hosted agents from images |
| Azure Batch | Container tasks on pools |

Build options on Azure: local Docker or Podman, **ACR Tasks** (`az acr build`, no local Docker needed), GitHub Actions or Azure Pipelines, and `azd` or Container Apps "deploy from source", which builds with Cloud Native Buildpacks or ACR.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Base image | Slim or Azure Linux for most apps; distroless for hardened production; pin by digest or exact version |
| Image size | Smaller pulls faster and scales out faster. Matters most for scale-to-zero and GPU images |
| Tags | Immutable version tags or commit SHAs. Never deploy `latest` |
| Architecture | Build `amd64`; add `arm64` if you will use Arm (Cobalt) nodes for cost |
| Where to build | In CI or ACR Tasks, not on developer laptops, so builds are reproducible and auditable |
| Model weights in the image | Small models only. Large weights belong in Blob Storage or an OCI artefact pulled at start, or a shared volume |
| Root user | Never in production. Set a numeric `USER` |

## Hands-on

```bash
# 1. Build and inspect a multi-stage image locally
docker build -t demo-api:0.1.0 .
docker image ls demo-api
docker history demo-api:0.1.0            # see each layer and its size
docker run --rm -p 8000:8000 demo-api:0.1.0

# 2. Check that it does not run as root
docker run --rm demo-api:0.1.0 id

# 3. Get the immutable digest
docker inspect --format='{{index .RepoDigests 0}}' demo-api:0.1.0 2>/dev/null || \
  docker inspect --format='{{.Id}}' demo-api:0.1.0

# 4. Scan for vulnerabilities (Trivy is one free option)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image demo-api:0.1.0

# 5. Generate an SBOM
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock anchore/syft demo-api:0.1.0 -o spdx-json > sbom.json
```

Then change one line of application code and rebuild. Only the last layers should rebuild. If dependencies reinstall, fix the layer order.

## Security, cost and reliability notes

- **Security:** every package in the image is attack surface. Minimal bases and multi-stage builds cut vulnerabilities more than any scanner can.
- **Security:** never put secrets in an image, including in an intermediate layer that a later `RUN rm` "deletes". Layers are permanent; anyone who can pull can read them. Use managed identities and Key Vault at run time, and BuildKit secret mounts at build time.
- **Security:** sign images and verify signatures at deploy time (files 02 and 08). Produce an SBOM for each release.
- **Cost:** large images cost registry storage and egress and slow every scale-out event.
- **Reliability:** handle `SIGTERM` and finish in-flight requests. Orchestrators send it before killing the container.
- **Reliability:** pin base images. A floating `python:3` tag can change behaviour under you on a rebuild.

## Common mistakes

- Running as root.
- `COPY . .` before installing dependencies, destroying the cache.
- No `.dockerignore`, so `.env` files or `.git` end up in the image.
- Deploying `latest`, so nobody knows what is running and rollbacks are guesses.
- Pulling base images from Docker Hub in production and hitting rate limits.
- Baking environment-specific configuration into the image.
- Writing logs to a file inside the container.
- Multi-gigabyte images for a small API, often because a GPU base was used where none was needed.
- Alpine plus Python packages with native extensions, leading to slow builds or runtime failures.

## Check yourself

1. What is the difference between an image tag and a digest, and which should a production deployment reference?
2. Why does layer order matter in a Dockerfile?
3. What problem does a multi-stage build solve?
4. Why can't you remove a secret from an image with a later `RUN rm`?
5. What is the OCI referrers API used for?
6. Name three base image choices and when each fits.
7. Why should production pull base images from a private registry?

<details>
<summary>Answers</summary>

1. A tag is a mutable name that can be moved to another image; a digest is the immutable SHA-256 of the manifest. Production should reference a digest, or an immutable version tag that is never overwritten.
2. Each layer is cached and reused only if it and every layer before it are unchanged. Putting rarely changing steps first keeps rebuilds fast.
3. It keeps build tools, compilers and dev dependencies out of the runtime image, making it smaller and safer.
4. Layers are additive. The secret remains in the earlier layer and can be extracted by anyone who pulls the image.
5. Attaching related artefacts, such as signatures, SBOMs and scan results, to an image in the registry.
6. Slim distributions as a general default; distroless or chiseled for hardened production; Azure Linux for Microsoft-maintained minimal images; full images only for debugging or build stages; CUDA bases only for GPU work.
7. To avoid public registry rate limits and outages, to control and scan what is used, and to keep the deployment path private.

</details>

## References

- [Containers on Azure: overview](https://learn.microsoft.com/azure/architecture/guide/choose-azure-container-service)
- [Dockerfile best practices](https://docs.docker.com/build/building/best-practices/)
- [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Open Container Initiative specifications](https://opencontainers.org/)
- [Azure Linux container images](https://learn.microsoft.com/azure/azure-linux/intro-azure-linux)
- [Microsoft Artifact Registry](https://mcr.microsoft.com/)
- [The Twelve-Factor App](https://12factor.net/)
