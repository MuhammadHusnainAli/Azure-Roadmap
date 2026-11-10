# Azure Container Storage

## Why it matters

Containers are ephemeral, but many workloads are not: databases, message brokers, vector stores, model caches and training checkpoints all need storage that survives a pod restart or reschedule. Kubernetes abstracts storage through volumes and CSI drivers, and Azure offers several backing services with very different performance, cost and availability. Choosing wrongly gives you slow model loading, lost data on zone failure, or a large storage bill.

## Key concepts

### Kubernetes storage objects

| Object | Purpose |
|---|---|
| **Volume** (`emptyDir`, `configMap`, `secret`, `hostPath`) | Pod-lifetime or node-local storage |
| **PersistentVolume (PV)** | A piece of storage in the cluster |
| **PersistentVolumeClaim (PVC)** | A pod's request for storage of a size and access mode |
| **StorageClass** | Template for dynamically creating PVs: provisioner, SKU, reclaim policy, binding mode |
| **CSI driver** | Plug-in connecting Kubernetes to a storage system |
| **VolumeSnapshot** | Point-in-time copy of a PV |

Access modes: `ReadWriteOnce` (one node), `ReadOnlyMany`, `ReadWriteMany` (many nodes), `ReadWriteOncePod`.

### Storage options for AKS

| Option | Type | Access | Latency / throughput | Fit |
|---|---|---|---|---|
| **Azure Disk CSI** (Premium SSD, Premium SSD v2, Ultra Disk) | Block | RWO | Low latency; scales with disk tier | Databases, stateful sets |
| **Azure Files CSI** (SMB, NFS; Premium or Standard) | File | RWX | Moderate | Shared config, content, legacy apps needing shared files |
| **Azure Blob CSI** (BlobFuse2 or NFS 3.0) | Object exposed as files | RWX | High throughput for large sequential reads | Datasets, model weights, logs |
| **Azure NetApp Files** (Trident CSI) | File (NFS, SMB) | RWX | Very low latency, high throughput | Demanding enterprise and HPC workloads |
| **Azure Managed Lustre** (CSI) | Parallel file system | RWX | Extreme throughput | Large training data, checkpoints |
| **Azure Container Storage** | Container-native volumes | RWO | Very low latency with local NVMe | High-performance ephemeral data, caches, databases with replication |
| `emptyDir` on OS or temp disk | Ephemeral | Pod | Fast, limited size | Scratch space |

### Azure Container Storage

Azure Container Storage is a **container-native volume management service** for AKS. Rather than one Azure disk per PVC, it manages storage pools and provisions volumes from them, with fast attach and Kubernetes-native operations.

Its focus has narrowed over time:

| Capability | Detail |
|---|---|
| **Local NVMe (ephemeral disk)** | Exposes the local NVMe drives on storage-optimised and GPU VM sizes as persistent volumes for pods, with very low latency and high IOPS. Data is lost if the node is lost, so use replication at the application layer |
| Earlier backing options | Earlier versions also pooled Azure Disks and Elastic SAN. Check the current version's supported backends before designing; the newer version concentrates on local NVMe and is installed as an AKS extension |
| Integration | Installed as an AKS extension; StorageClasses created for you; works with KAITO for model caching on GPU nodes (check current status) |

Use cases:

- **Model weight caching** on GPU nodes: load weights once per node from Blob or ACR into local NVMe, start pods faster.
- **High-performance databases** with built-in replication (for example PostgreSQL operators, Cassandra, vector databases) where the database replicates across nodes and zones.
- **Scratch for AI pipelines**: embedding batches, intermediate data, checkpoints before upload.

### Reclaim policies and binding

- `reclaimPolicy: Delete` removes the disk when the PVC is deleted; `Retain` keeps it. Use `Retain` for anything precious.
- `volumeBindingMode: WaitForFirstConsumer` provisions the disk in the zone where the pod is scheduled; essential with zonal disks.
- Zonal disks (LRS) pin a pod to one zone. **ZRS disks** (Premium SSD and Standard SSD ZRS) can attach in any zone of the region, enabling pod failover across zones at some latency cost.

## How it works on Azure

### Built-in StorageClasses on AKS

| StorageClass | Backing |
|---|---|
| `managed-csi` | Standard SSD LRS |
| `managed-csi-premium` | Premium SSD LRS |
| `azurefile-csi` / `azurefile-csi-premium` | Azure Files Standard / Premium (SMB) |
| `azureblob-fuse-premium`, `azureblob-nfs-premium` | Blob via BlobFuse2 or NFS (after enabling the Blob CSI driver) |

Create your own classes for Premium SSD v2, ZRS disks, NFS file shares, or encryption with customer-managed keys.

### Example: a ZRS Premium disk class and a stateful set

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: { name: premium-zrs-retain }
provisioner: disk.csi.azure.com
parameters: { skuName: Premium_ZRS, cachingMode: ReadOnly }
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: vectordb }
spec:
  serviceName: vectordb
  replicas: 3
  selector: { matchLabels: { app: vectordb } }
  template:
    metadata: { labels: { app: vectordb } }
    spec:
      topologySpreadConstraints:
      - { maxSkew: 1, topologyKey: topology.kubernetes.io/zone, whenUnsatisfiable: DoNotSchedule,
          labelSelector: { matchLabels: { app: vectordb } } }
      containers:
      - name: db
        image: qdrant/qdrant:v1.12.0
        volumeMounts: [ { name: data, mountPath: /qdrant/storage } ]
  volumeClaimTemplates:
  - metadata: { name: data }
    spec: { accessModes: [ ReadWriteOnce ], storageClassName: premium-zrs-retain, resources: { requests: { storage: 64Gi } } }
```

### Example: shared model weights from Blob (read-only, many pods)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: model-weights }
spec:
  accessModes: [ ReadOnlyMany ]
  storageClassName: azureblob-fuse-premium
  resources: { requests: { storage: 200Gi } }
```

Combine with a local cache (BlobFuse2 caching or Azure Container Storage on local NVMe) for repeated reads.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Should the state live in the cluster at all | Prefer managed databases and storage services (topics 08 and 09) for important data; keep in-cluster state for caches, scratch, and databases you have a reason to self-run |
| Disk vs Files vs Blob | Disk for single-writer low-latency; Files for shared POSIX or SMB access; Blob for large datasets and model weights |
| Zonal resilience | ZRS disks or application-level replication across zones |
| Local NVMe | Best performance, but ephemeral: only with replication or re-creatable data |
| AI training data | Blob for capacity, Managed Lustre for throughput at scale, local NVMe for hot caches |
| Backup | Azure Backup for AKS (topic 28) or Velero; snapshots for disks |

## Hands-on

```bash
RG=rg-aks-stor; AKS=aks-stor; LOC=swedencentral
az group create -n $RG -l $LOC -o none
az aks create -g $RG -n $AKS --node-count 3 --zones 1 2 3 --enable-blob-driver --generate-ssh-keys -o none
az aks get-credentials -g $RG -n $AKS --overwrite-existing

kubectl get storageclass

# 1. Dynamic disk with a pod; write data, delete the pod, confirm data survives
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: data }
spec: { accessModes: [ ReadWriteOnce ], storageClassName: managed-csi-premium, resources: { requests: { storage: 8Gi } } }
---
apiVersion: v1
kind: Pod
metadata: { name: writer }
spec:
  containers:
  - name: c
    image: busybox
    command: [ "sh", "-c", "echo hello-$(date +%s) >> /data/log.txt; cat /data/log.txt; sleep 3600" ]
    volumeMounts: [ { name: d, mountPath: /data } ]
  volumes: [ { name: d, persistentVolumeClaim: { claimName: data } } ]
EOF
kubectl logs writer
kubectl delete pod writer && kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: { name: writer }
spec:
  containers:
  - name: c
    image: busybox
    command: [ "sh", "-c", "echo hello-$(date +%s) >> /data/log.txt; cat /data/log.txt; sleep 3600" ]
    volumeMounts: [ { name: d, mountPath: /data } ]
  volumes: [ { name: d, persistentVolumeClaim: { claimName: data } } ]
EOF
kubectl logs writer          # two lines: the data persisted

# 2. See the Azure disk behind the PV, and its zone
kubectl get pv -o custom-columns=NAME:.metadata.name,HANDLE:.spec.csi.volumeHandle,ZONE:.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0]

az group delete -n $RG --yes --no-wait
```

## Security, cost and reliability notes

- **Security:** disks are encrypted at rest by default; use disk encryption sets for customer-managed keys. Use private endpoints for Files and Blob used by the cluster.
- **Security:** Blob and Files CSI can use Workload Identity or managed identity instead of account keys; prefer that.
- **Cost:** PVCs with `Retain` outlive clusters and keep billing. Audit orphaned disks regularly.
- **Cost:** Premium disks are billed by provisioned size, not usage. Right-size and use volume expansion when needed.
- **Reliability:** LRS disks tie pods to a zone; plan replicas per zone or use ZRS. Local NVMe data disappears with the node.

## Common mistakes

- Running a production database in-cluster without backups or replication.
- `Immediate` binding with zonal disks, producing pods that cannot schedule because the disk is in another zone.
- Using Azure Files for latency-sensitive databases.
- Baking huge model weights into images instead of using shared or cached volumes.
- Forgetting orphaned retained disks after deleting clusters.
- Storing important data on local NVMe without replication.

## Check yourself

1. What is the difference between a PV, a PVC and a StorageClass?
2. Which Azure storage options provide `ReadWriteMany`?
3. Why use `WaitForFirstConsumer` with Azure Disks?
4. How do ZRS disks change failover behaviour?
5. What is Azure Container Storage's main use of local NVMe, and what is the catch?
6. Where should large model weights live for a fleet of inference pods?

<details>
<summary>Answers</summary>

1. A PV is actual storage; a PVC is a request for storage by a workload; a StorageClass is the template used to provision PVs dynamically.
2. Azure Files, Azure Blob (BlobFuse2 or NFS), Azure NetApp Files and Azure Managed Lustre.
3. So the disk is created in the same zone as the node where the pod is scheduled.
4. A ZRS disk can be attached in any zone in the region, so a pod can be rescheduled to another zone and keep its disk.
5. Fast persistent volumes on the VM's local NVMe for caches, scratch and replicated databases. The data is lost if the node is lost.
6. In Blob Storage (or an OCI artefact in ACR), mounted read-only or downloaded at start, with node-local caching; Managed Lustre for very high throughput needs.

</details>

## References

- [Storage options for applications in AKS](https://learn.microsoft.com/azure/aks/concepts-storage)
- [Azure Container Storage overview](https://learn.microsoft.com/azure/storage/container-storage/container-storage-introduction)
- [Azure Disk CSI driver](https://learn.microsoft.com/azure/aks/azure-disk-csi)
- [Azure Files CSI driver](https://learn.microsoft.com/azure/aks/azure-files-csi)
- [Azure Blob CSI driver](https://learn.microsoft.com/azure/aks/azure-blob-csi)
- [Azure NetApp Files with AKS](https://learn.microsoft.com/azure/aks/azure-netapp-files)
- [Azure Managed Lustre CSI driver](https://learn.microsoft.com/azure/azure-managed-lustre/use-csi-driver-kubernetes)
- [Best practices for storage and backups in AKS](https://learn.microsoft.com/azure/aks/operator-best-practices-storage)
