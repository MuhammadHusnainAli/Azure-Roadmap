# AKS Security and Identity

> Topic: 07 · Containers and Kubernetes · Tracks: E ● A ● C ○ · Status checked: 2026-10

## Why it matters

A Kubernetes cluster concentrates risk: one cluster may run dozens of teams' workloads, hold credentials to many Azure services, and expose an API that can do anything to all of them. AKS security is about four things: who can reach and use the API server, what identities pods use to reach Azure, what is allowed to run, and how threats are detected. AI workloads add their own stakes, because agent pods often hold access to data and models.

## Key concepts

### Layers of AKS security

| Layer | Controls |
|---|---|
| **Cluster access** | Entra authentication, Azure RBAC for Kubernetes, local accounts disabled, private API server |
| **Workload identity** | Microsoft Entra Workload ID for pods; no secrets in pods |
| **Admission control** | Azure Policy (Gatekeeper), deployment safeguards, Pod Security Standards, image signature verification |
| **Supply chain** | Trusted registries, scanning, signing, SBOMs |
| **Runtime** | Defender for Containers, network policy, node hardening, sandboxed pods |
| **Secrets and data** | Key Vault CSI driver, encryption of etcd secrets with Key Vault keys (KMS), disk encryption |
| **Nodes** | Automatic OS patching, no SSH, Azure Linux, confidential nodes where needed |

### Cluster access: Entra and Azure RBAC

| Model | Authentication | Authorisation |
|---|---|---|
| Local accounts | Certificate in kubeconfig | Kubernetes RBAC. Disable this |
| Entra + Kubernetes RBAC | Entra users and groups | `RoleBinding`s referencing Entra object IDs |
| **Entra + Azure RBAC** (recommended) | Entra | Azure role assignments at cluster or namespace scope |

Built-in roles: `Azure Kubernetes Service RBAC Reader`, `Writer`, `Admin`, `Cluster Admin`, plus `Azure Kubernetes Service Cluster User Role` to download the kubeconfig. Combine with PIM for just-in-time cluster admin and Conditional Access for the AKS server application.

`kubelogin` handles Entra token acquisition for `kubectl`; in CI use workload identity federation or a managed identity (`kubelogin convert-kubeconfig -l azurecli` or `-l workloadidentity`).

### Microsoft Entra Workload ID

Pods authenticate to Entra as a **user-assigned managed identity** (or app registration) by exchanging a Kubernetes service account token through **federated identity credentials**. No secrets anywhere.

```text
Pod (service account "chat-sa", namespace "ai")
   │ projected service account token (issuer = cluster OIDC issuer)
   ▼
Entra ID: federated credential on managed identity "id-chat"
   subject = system:serviceaccount:ai:chat-sa
   │ issues access token for id-chat
   ▼
Foundry, Key Vault, Storage, Cosmos DB … (RBAC on id-chat)
```

Requirements: the cluster's **OIDC issuer** and **Workload Identity** enabled, a service account annotated with the client ID, the pod labelled `azure.workload.identity/use: "true"`. Azure SDKs with `DefaultAzureCredential` work unchanged. The older **pod-managed identity** (AAD Pod Identity) is retired; migrate to Workload Identity.

### Admission control and policy

| Mechanism | What it enforces |
|---|---|
| **Azure Policy for AKS** | Gatekeeper constraints from Azure Policy definitions and initiatives (for example "no privileged containers", "allowed registries", "required resource limits"); compliance reported in Azure Policy |
| **Deployment safeguards** | Curated best-practice policies (warning or enforcement) applied by AKS, default in AKS Automatic |
| **Pod Security Admission** | Kubernetes Pod Security Standards (`privileged`, `baseline`, `restricted`) per namespace |
| **Image signature verification** | Verify Notation signatures at admission (Ratify with Gatekeeper, or the AKS image integrity feature; check current status) |

### Microsoft Defender for Containers

| Capability | Detail |
|---|---|
| Vulnerability assessment | Registry and running images, with agentless scanning |
| Runtime threat detection | Sensor (eBPF) on nodes detects suspicious processes, crypto-mining, privilege escalation |
| Kubernetes audit analysis | Detection on control plane activity |
| Posture management | Misconfiguration recommendations, attack path analysis in Defender CSPM |
| Gating | Block or warn on vulnerable images at deployment (check current status) |

### Secrets

- **Key Vault Secrets Store CSI driver** mounts secrets, keys and certificates as files (and optionally syncs to Kubernetes secrets), using Workload Identity to read Key Vault. Supports rotation.
- **KMS etcd encryption** encrypts Kubernetes `Secret` objects with a key in Key Vault (customer-managed), including private Key Vaults.
- Avoid storing long-lived credentials at all: prefer Workload Identity to managed services over secrets.

### Node and runtime hardening

| Control | Detail |
|---|---|
| Node OS auto-upgrade | `SecurityPatch` or `NodeImage` channel |
| Azure Linux | Smaller attack surface |
| Disable SSH | AKS supports disabling SSH access to nodes |
| **Pod Sandboxing** | Kata Containers-based VM isolation per pod for untrusted workloads (multi-tenant, model-generated code) |
| **Confidential nodes** | AMD SEV-SNP or Intel TDX VMs for node pools; confidential containers for pod-level TEE (check current status) |
| Host-based encryption | Encrypts temp and cache disks at the host |
| Image Cleaner | Removes unused vulnerable images from nodes |

## How it works on Azure

### Workload Identity end to end

```bash
RG=rg-aks-sec; AKS=aks-sec; LOC=swedencentral
az aks update -g $RG -n $AKS --enable-oidc-issuer --enable-workload-identity
ISSUER=$(az aks show -g $RG -n $AKS --query oidcIssuerProfile.issuerUrl -o tsv)

az identity create -g $RG -n id-chat
CLIENT_ID=$(az identity show -g $RG -n id-chat --query clientId -o tsv)
PRINCIPAL_ID=$(az identity show -g $RG -n id-chat --query principalId -o tsv)

az identity federated-credential create -g $RG --identity-name id-chat -n aks-ai-chat \
  --issuer $ISSUER --subject system:serviceaccount:ai:chat-sa --audiences api://AzureADTokenExchange

# Grant data access (example: read blobs)
az role assignment create --assignee-object-id $PRINCIPAL_ID --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Reader" --scope <storage-account-id>
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: chat-sa
  namespace: ai
  annotations:
    azure.workload.identity/client-id: "<CLIENT_ID>"
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: chat-api, namespace: ai }
spec:
  replicas: 2
  selector: { matchLabels: { app: chat-api } }
  template:
    metadata:
      labels: { app: chat-api, azure.workload.identity/use: "true" }
    spec:
      serviceAccountName: chat-sa
      securityContext: { runAsNonRoot: true, seccompProfile: { type: RuntimeDefault } }
      containers:
      - name: api
        image: myregistry.azurecr.io/chat-api:1.4.2
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities: { drop: [ "ALL" ] }
        resources: { requests: { cpu: 250m, memory: 256Mi }, limits: { memory: 512Mi } }
```

### Granting a team namespace access with Azure RBAC

```bash
AKS_ID=$(az aks show -g $RG -n $AKS --query id -o tsv)
az role assignment create --assignee <entra-group-object-id> \
  --role "Azure Kubernetes Service RBAC Writer" --scope "$AKS_ID/namespaces/ai"
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Authorisation model | Entra with Azure RBAC; local accounts disabled; PIM for admin roles |
| Identity per workload | One managed identity per application (not per cluster), with least-privilege RBAC |
| Namespace isolation | Soft multi-tenancy for teams in one organisation; separate clusters (or Pod Sandboxing plus strict policy) for hostile tenants |
| Policy rollout | Audit first, then deny; start with the AKS baseline initiative |
| Secrets | Workload Identity to services first; Key Vault CSI for unavoidable secrets; KMS for etcd |
| Untrusted code | Pod Sandboxing node pools, or Container Apps dynamic sessions |

## Hands-on

1. Create a cluster with `--enable-aad --enable-azure-rbac --disable-local-accounts --enable-oidc-issuer --enable-workload-identity`.
2. Configure the Workload Identity example above with a storage account, then run a pod with the Azure CLI image and `az login --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)" --service-principal -u $AZURE_CLIENT_ID -t $AZURE_TENANT_ID` followed by `az storage blob list --auth-mode login`.
3. Assign the **Kubernetes cluster pod security restricted standards for Linux-based workloads** Azure Policy initiative in audit mode, wait for evaluation, and review compliance with `kubectl get constrainttemplates` and in the portal.
4. Try deploying a privileged pod and observe the policy result.
5. Enable Defender for Containers on the subscription and review recommendations for the cluster.

```bash
az aks enable-addons -g $RG -n $AKS --addons azure-policy
kubectl get pods -n gatekeeper-system
kubectl run priv --image=nginx --privileged    # audited or denied depending on policy effect
```

## Security, cost and reliability notes

- **Security:** treat `cluster-admin` like a subscription owner. Grant just-in-time through PIM.
- **Security:** one shared identity for all pods is a lateral-movement path. Federate per service account.
- **Security:** restrict allowed registries by policy so only your ACR can be used.
- **Cost:** Defender for Containers is billed per vCPU core of nodes; ACNS and confidential nodes add cost. Weigh against risk.
- **Reliability:** enforce policies first in audit mode; a sudden deny can block system upgrades or critical deployments. Exclude system namespaces correctly.

## Common mistakes

- Leaving local accounts enabled, bypassing Entra and audit.
- Client secrets in Kubernetes secrets to call Azure services, instead of Workload Identity.
- Federated credential subject typo (namespace or service account name), producing `AADSTS70021` errors.
- Policies switched straight to deny, breaking deployments.
- Pods running as root with writable root file systems.
- Treating namespaces as a hard security boundary for untrusted tenants.
- Still using retired pod-managed identity.

## Check yourself

1. What are the three authorisation models for AKS, and which is recommended?
2. Explain how a pod obtains an Entra token with Workload Identity.
3. Which three settings must match between the federated credential and the pod?
4. How does Azure Policy enforce rules inside AKS?
5. What does Defender for Containers add beyond image scanning?
6. When would you use Pod Sandboxing?
7. What does KMS etcd encryption protect?

<details>
<summary>Answers</summary>

1. Local accounts with Kubernetes RBAC, Entra with Kubernetes RBAC, Entra with Azure RBAC. Entra with Azure RBAC, local accounts disabled.
2. Kubernetes projects a signed service account token into the pod; the Azure SDK exchanges it with Entra, which trusts the cluster's OIDC issuer through a federated credential on the managed identity, and returns an access token for that identity.
3. The issuer (cluster OIDC issuer URL), the subject (`system:serviceaccount:<namespace>:<name>`) and the audience (`api://AzureADTokenExchange`).
4. The add-on runs Gatekeeper as an admission controller, with constraints generated from Azure Policy assignments; results are reported back to Azure Policy.
5. Runtime threat detection on nodes, Kubernetes audit log analysis, posture recommendations and attack path analysis.
6. For untrusted or multi-tenant workloads, such as running customer or model-generated code, where container isolation is not enough.
7. Kubernetes `Secret` objects at rest in etcd, encrypted with a customer-managed key in Key Vault.

</details>

## References

- [AKS security concepts](https://learn.microsoft.com/azure/aks/concepts-security)
- [Entra integration with Azure RBAC for Kubernetes](https://learn.microsoft.com/azure/aks/manage-azure-rbac)
- [Workload Identity on AKS](https://learn.microsoft.com/azure/aks/workload-identity-overview)
- [Azure Policy for Kubernetes](https://learn.microsoft.com/azure/governance/policy/concepts/policy-for-kubernetes)
- [Deployment safeguards](https://learn.microsoft.com/azure/aks/deployment-safeguards)
- [Defender for Containers](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-containers-introduction)
- [Key Vault Secrets Store CSI driver](https://learn.microsoft.com/azure/aks/csi-secrets-store-driver)
- [KMS etcd encryption](https://learn.microsoft.com/azure/aks/use-kms-etcd-encryption)
- [Pod Sandboxing](https://learn.microsoft.com/azure/aks/use-pod-sandboxing)
- [AKS security baseline](https://learn.microsoft.com/security/benchmark/azure/baselines/azure-kubernetes-service-aks-security-baseline)
