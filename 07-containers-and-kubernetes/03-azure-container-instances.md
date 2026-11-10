# Azure Container Instances

## Why it matters

Azure Container Instances (ACI) runs a container in seconds with no cluster, no platform and no scaling logic. It is the lowest-level container service in Azure, billed per second, and it underpins other services such as AKS virtual nodes. Knowing what ACI does well, and where it stops, keeps teams from building an orchestrator on top of it, or from standing up AKS for a nightly script.

## Key concepts

### Container groups

The unit of deployment is a **container group**: one or more containers scheduled on the same host, sharing a lifecycle, a network namespace (and so an IP address and port space), and optionally volumes. It is the ACI equivalent of a Kubernetes pod.

```text
Container group  (10.0.2.4, or a public IP + DNS label)
 ├─ app        :8080
 ├─ sidecar    :9090  (log shipper, proxy)
 └─ volumes: Azure Files share, emptyDir, secret
```

Multi-container groups are Linux only. Windows groups hold a single container.

### Key properties

| Property | Detail |
|---|---|
| Sizing | You request vCPU and memory per container; billing is per second for the group's total |
| Restart policy | `Always`, `OnFailure`, `Never`. Use `Never` or `OnFailure` for run-to-completion tasks |
| Networking | Public IP with DNS label, or deployment into a delegated VNet subnet (private) |
| Storage | Azure Files mounts, `emptyDir`, secret volumes. No persistent local disk |
| Identity | System or user-assigned managed identity |
| Images | Pull from ACR with managed identity, or any registry |
| Scaling | None. One group is one instance. You create more groups to get more instances |
| Availability | Can be pinned to an availability zone; no automatic multi-zone spread |

### Specialised capabilities

| Capability | What it gives you |
|---|---|
| **Confidential containers** | Container groups run in a hardware trusted execution environment (AMD SEV-SNP), with a confidential computing enforcement (CCE) policy that attests exactly which containers and settings may run. Useful for sensitive data processing without rewriting the app |
| **Spot containers** | Discounted, evictable capacity for interruptible batch work. Check region and status |
| **Standby pools** | A pool of pre-created, pre-warmed container groups that you can claim in seconds, avoiding image pull and start latency. Useful for bursty, latency-sensitive workloads |
| **NGroups** | Manage many identical container groups as a set, with rolling updates and zone spreading. Check current status before relying on it |
| **Virtual nodes** | Lets AKS schedule pods onto ACI for burst capacity without adding VMs |

GPU support in ACI has been retired. For serverless GPUs use Azure Container Apps (file 05).

### Where ACI fits

| Good fit | Poor fit |
|---|---|
| Short scripts, scheduled jobs, build agents | Long-running web apps that need scaling, revisions, TLS ingress |
| Test environments created and destroyed per pipeline run | Microservices that call each other with service discovery |
| Burst capacity for AKS (virtual nodes) | Anything needing autoscale or scale to zero on HTTP |
| Confidential processing of a sensitive dataset | Large fleets (use Container Apps or AKS) |
| Isolated sandboxes per tenant or job | GPU workloads |

For most new run-to-completion workloads, **Container Apps jobs** (file 04) now offer a better experience: scheduling, event triggers, retries and shared environments. ACI remains the choice for the specialised capabilities above and for direct, one-off container runs.

## How it works on Azure

| Item | Detail |
|---|---|
| Deployment | CLI, YAML (`az container create --file`), ARM/Bicep, or the AKS virtual nodes connector |
| Private networking | Subnet delegated to `Microsoft.ContainerInstance/containerGroups`. Outbound via NAT gateway or firewall |
| Logs | `az container logs`, or send to Log Analytics |
| Pricing | Per second for vCPU and memory, with surcharges for Windows and confidential SKUs |
| Limits | Per-group vCPU and memory maximums and regional quotas apply; check current numbers |

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| ACI or Container Apps job | Container Apps jobs for scheduled or event-driven work in an app environment; ACI for direct runs, confidential containers or standby pools |
| ACI or AKS virtual nodes | Virtual nodes add burst capacity to an existing cluster; otherwise use ACI directly |
| Public or private | Private (VNet) for anything touching internal data |
| Long-running services | Prefer Container Apps; ACI has no built-in scaling, rolling updates or managed TLS |

## Hands-on

```bash
RG=rg-aci-lab; LOC=swedencentral
az group create -n $RG -l $LOC -o none

# 1. A public hello-world container
az container create -g $RG -n hello --image mcr.microsoft.com/azuredocs/aci-helloworld \
  --cpu 1 --memory 1 --ports 80 --ip-address Public \
  --dns-name-label hello-$RANDOM --os-type Linux -o none
az container show -g $RG -n hello --query "{fqdn:ipAddress.fqdn, state:instanceView.state}" -o table

# 2. A run-to-completion task
az container create -g $RG -n task --image mcr.microsoft.com/azurelinux/base/core:3.0 \
  --restart-policy Never --os-type Linux \
  --command-line "/bin/sh -c 'echo processing; sleep 10; echo done'" -o none
az container logs -g $RG -n task
az container show -g $RG -n task --query "instanceView.state" -o tsv     # Succeeded

# 3. A two-container group defined in YAML
cat > group.yaml <<'EOF'
apiVersion: 2023-05-01
location: swedencentral
name: multi
properties:
  osType: Linux
  restartPolicy: Always
  ipAddress: { type: Public, ports: [ { protocol: tcp, port: 80 } ] }
  containers:
  - name: web
    properties:
      image: mcr.microsoft.com/azuredocs/aci-helloworld
      resources: { requests: { cpu: 1, memoryInGb: 1 } }
      ports: [ { port: 80 } ]
  - name: sidecar
    properties:
      image: mcr.microsoft.com/azurelinux/base/core:3.0
      command: [ "/bin/sh", "-c", "while true; do curl -s localhost:80 >/dev/null && echo ok; sleep 15; done" ]
      resources: { requests: { cpu: 0.5, memoryInGb: 0.5 } }
type: Microsoft.ContainerInstance/containerGroups
EOF
az container create -g $RG --file group.yaml -o none
az container logs -g $RG -n multi --container-name sidecar

az group delete -n $RG --yes --no-wait
```

The sidecar reaches the web container on `localhost`, showing the shared network namespace.

## Security, cost and reliability notes

- **Security:** public IP groups are directly reachable. Use VNet deployment for internal workloads and put a gateway in front of anything public.
- **Security:** pass secrets as secure environment variables or secret volumes, or better, fetch them with a managed identity from Key Vault.
- **Security:** for regulated data, confidential containers give hardware isolation and attestation without code changes.
- **Cost:** per-second billing is cheap for short tasks. A group left running with `Always` costs the same as a small VM, without the VM discounts.
- **Reliability:** no self-healing beyond the restart policy, and no multi-instance spread. Build redundancy yourself or choose Container Apps.

## Common mistakes

- Building a home-made orchestrator of ACI groups instead of using Container Apps or AKS.
- Forgetting `--restart-policy Never` on a task, so it restarts forever.
- Long-running groups that nobody remembers to delete.
- Expecting GPU support, which has been retired.
- Public IPs on groups processing internal data.

## Check yourself

1. What is a container group, and what do its containers share?
2. Which restart policy suits a run-to-completion job?
3. What do confidential containers on ACI provide?
4. What problem do standby pools solve?
5. When would you use Container Apps jobs instead of ACI?
6. Why can't ACI host a GPU inference workload today?

<details>
<summary>Answers</summary>

1. A set of containers scheduled together on one host, sharing lifecycle, network namespace (IP and ports) and optional volumes.
2. `Never`, or `OnFailure` if you want retries on failure.
3. Execution inside a hardware TEE with a policy that attests which containers and configuration may run, protecting data in use.
4. Start-up latency: pre-created container groups can be claimed in seconds without waiting for provisioning and image pulls.
5. For scheduled, event-driven or manually triggered jobs that benefit from retries, parallelism and a shared environment with other apps.
6. GPU support in ACI was retired. Use Container Apps serverless GPUs, AKS or managed endpoints.

</details>

## References

- [Azure Container Instances documentation](https://learn.microsoft.com/azure/container-instances/)
- [Container groups](https://learn.microsoft.com/azure/container-instances/container-instances-container-groups)
- [Deploy into a virtual network](https://learn.microsoft.com/azure/container-instances/container-instances-vnet)
- [Confidential containers on ACI](https://learn.microsoft.com/azure/container-instances/container-instances-confidential-overview)
- [Standby pools for ACI](https://learn.microsoft.com/azure/container-instances/container-instances-standby-pool-overview)
- [Virtual nodes on AKS](https://learn.microsoft.com/azure/aks/virtual-nodes)
- [Choose an Azure container service](https://learn.microsoft.com/azure/architecture/guide/choose-azure-container-service)
