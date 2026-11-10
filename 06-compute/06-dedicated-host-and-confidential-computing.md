# Dedicated Host and Confidential Computing

## Why it matters

Standard Azure VMs share physical servers with other customers, isolated by the hypervisor, and that is the right choice for nearly everything. Two requirements go further. Some organisations need the **physical server to themselves**, for licensing or compliance. Others need assurance that **nobody outside the workload, including the cloud operator, can read data while it is being processed**. The second is confidential computing, and it increasingly comes up in AI conversations about sensitive prompts, regulated data and multi-party analytics.

## Key concepts

### The isolation spectrum

| Level | What you get | Shared with |
|---|---|---|
| **Standard VM** | Hypervisor isolation on multi-tenant hosts | Other customers on the same physical server |
| **Isolated VM sizes** (`i` suffix, and certain large sizes) | The VM occupies the whole physical host | Nobody, but you run one VM per host and do not control the host |
| **Azure Dedicated Host** | A physical server allocated to your subscription, on which you place your own VMs | Nobody; you control placement and maintenance timing |
| **Confidential VM** | Memory encrypted with keys the host and hypervisor cannot access | Hardware is shared, but the operator and hypervisor are outside the trust boundary |

Dedicated Host addresses *who else is on the hardware*. Confidential computing addresses *who can see inside the VM*. They are different problems and can be combined.

## Part 1 · Azure Dedicated Host

### Key concepts

A dedicated host is a physical server, of a specific hardware type, assigned to you. You deploy VMs onto it.

| Concept | Detail |
|---|---|
| **Host** | One physical server of a given **host SKU**, tied to a VM family and hardware generation (for example a Dasv5-type host supports Dasv5-series VMs) |
| **Host group** | A collection of hosts in a region and, optionally, a zone and fault-domain count |
| **Placement** | **Manual** (name the host) or **automatic** (Azure places the VM on any host in the group with capacity) |
| **Capacity** | Fixed vCPUs and memory per host; you pack VMs up to that capacity |
| **Billing** | Per host, per hour, **regardless of how many VMs run on it**. VMs on the host have no separate compute charge. Reservations and Azure Hybrid Benefit apply |
| **Maintenance control** | Schedule platform updates within a rolling window instead of accepting Azure's timing |
| **Availability** | Spread hosts across zones or fault domains. A single host is a single point of failure |
| **Scale sets** | Flexible and Uniform scale sets can target a host group |

### When it is justified

| Driver | Explanation |
|---|---|
| **Licensing** | Software licensed per physical core or socket (some Windows Server, SQL Server and Oracle terms). With Azure Hybrid Benefit on a dedicated host you can license the host and run unlimited virtual machines within product terms, which can be much cheaper for dense workloads |
| **Compliance or contract** | A requirement for physical single tenancy |
| **Maintenance control** | Workloads extremely sensitive to even brief platform pauses |
| **Host-level visibility** | Needing to know and control which VMs share hardware |

### When it is not

- General workloads. Hypervisor isolation is strong and satisfies the major compliance standards.
- As a performance optimisation. The hardware is the same.
- For small footprints. You pay for the whole host, and resilience needs at least two.
- For data-in-use protection. That is confidential computing.

Operational costs: capacity planning at host level, tracking host SKU retirements, limited VM size choice per host type, and no autoscaling beyond the hosts you have bought.

## Part 2 · Confidential computing

### Key concepts

Data has three states:

| State | Conventional protection |
|---|---|
| At rest | Storage and disk encryption |
| In transit | TLS |
| **In use** (in memory, being processed) | Historically none: data is in clear text in RAM |

**Confidential computing** protects data in use by running the computation inside a hardware-based **trusted execution environment (TEE)**: a region of memory the CPU encrypts and isolates so that code outside it, including the hypervisor, the host OS and cloud administrators, cannot read or tamper with it.

**Attestation** is what makes it trustworthy: the hardware produces a signed report of what is running in the TEE, a verifier checks it against a policy, and only then are secrets (such as decryption keys) released. Without attestation you are trusting a claim; with it you have evidence.

The **threat model** matters. Confidential computing removes the *cloud operator and platform software* from the trusted computing base. It does not protect against vulnerabilities in your own code, a compromised guest administrator, or data you choose to send elsewhere.

### Technologies

| Technology | Isolation boundary | Application changes |
|---|---|---|
| **AMD SEV-SNP** | The whole VM | None (lift and shift) |
| **Intel TDX** | The whole VM | None |
| **Intel SGX** | An application **enclave**: a small, protected part of a process | Significant: partition the app or use a library OS |
| **Confidential GPUs** | VM plus an attested GPU, with encrypted CPU-to-GPU communication | Few |

Trend: **VM-level** confidentiality (SEV-SNP, TDX) is the mainstream because it needs no code changes. SGX offers the smallest trust boundary for specialised cases.

### Azure offerings

| Offering | Description |
|---|---|
| **Confidential VMs** | `DC` and `EC` series with an `a` (AMD SEV-SNP) or `e` (Intel TDX) designation. Full VM memory encryption, optional **confidential OS disk encryption** with keys bound to the VM's vTPM and released only after attestation |
| **Confidential VMs with GPUs** | NVIDIA H100-based confidential sizes for protected AI inference and fine-tuning |
| **Application enclaves** | `DC`-series with Intel SGX |
| **Confidential containers on Azure Container Instances** | Container groups in a TEE with a verifiable execution policy |
| **Confidential containers / confidential node pools on AKS** | Pods or nodes backed by confidential VMs |
| **Azure Attestation** | Managed attestation service; also the open-source guest attestation tooling |
| **Azure Key Vault Managed HSM** with **secure key release** | Releases a key only to a TEE that passes attestation |
| **Azure confidential ledger** | Tamper-evident, append-only ledger running in TEEs |
| **Always Encrypted with secure enclaves** | Azure SQL queries over encrypted columns inside an enclave |
| **Azure Confidential Clean Rooms** | Multi-party collaboration on combined data without exposing raw data to each other |
| **Confidential inferencing** | Model inference services running in TEEs, where offered |

### Trusted Launch is not confidential computing

| | Trusted Launch | Confidential VM |
|---|---|---|
| Protects | Boot integrity: Secure Boot, vTPM, measured boot | All of that **plus** memory encryption from the host |
| Threat addressed | Boot kits, rootkits, tampered OS | A privileged platform insider or compromised hypervisor reading memory |
| Cost and availability | Free, default on most sizes | Specific sizes, fewer regions, some feature limits |

Trusted Launch is a baseline for every VM. Confidential VMs are for a specific, stronger requirement.

### Confidential VM specifics

- **Generation 2** images with confidential VM support.
- **OS disk encryption options:** confidential disk encryption (keys released after attestation; platform- or customer-managed) or without.
- **Limitations** compared with standard VMs: smaller size range, fewer regions, restrictions on some features such as certain live migration, backup, replication, resize and accelerated networking scenarios. They are being reduced over time; check the list.
- **Performance:** a modest overhead from memory encryption, larger for I/O-heavy workloads.
- **Attestation in practice:** boot attestation happens through the platform; for a stronger posture, your application or a secure key release policy verifies the attestation report before using secrets.

### Confidential computing and AI

| Scenario | How confidential computing helps |
|---|---|
| **Sensitive prompts and data** in self-hosted inference | The model runs in a confidential GPU VM or container; prompts and outputs are not visible to the platform operator |
| **Protecting model weights** | Proprietary or fine-tuned weights are decrypted only inside an attested TEE |
| **Multi-party training and analytics** | Clean rooms: several organisations contribute data to a joint computation without revealing raw data |
| **Regulated processing** | Evidence, through attestation, that data was processed only by approved code in a protected environment |
| **Sovereignty requirements** | A technical control against operator access, complementing contractual ones (topic 03, file 10) |

Honest limits:

- For **managed model APIs** (Foundry Models), you rely on Microsoft's contractual and operational controls, not a TEE you attest, unless you use a specifically confidential inferencing offering.
- Confidential GPU capacity is scarce, regionally limited and expensive.
- It does not address **prompt injection, data leakage through outputs, or over-privileged agents**. Those are application-layer problems.
- Most enterprise requirements are met by private networking, customer-managed keys, Customer Lockbox and strong access control. Confidential computing is for when the requirement is explicitly "the operator must not be able to see the data in use".

### Related controls often asked about together

| Control | Addresses |
|---|---|
| **Customer-managed keys** | Who controls encryption keys at rest |
| **Customer Lockbox** | Approval of Microsoft support access to data |
| **Encryption at host** | Host-level caches and temporary disks |
| **Azure Dedicated Host** | Physical tenancy |
| **Confidential computing** | Data in use |

A client asking for "maximum protection" should be walked through which of these map to an actual risk they have.

## How it works on Azure

### Choosing

```text
Is the requirement about software licensing tied to physical hardware,
or contractual single tenancy of the server?
   └─ Yes → Azure Dedicated Host (≥ 2 hosts across zones for availability)

Is the requirement that the cloud operator must not be able to read data during processing?
   ├─ Whole VM, no code changes            → Confidential VM (SEV-SNP or TDX)
   ├─ Containers                           → Confidential containers on ACI or AKS
   ├─ GPU inference or fine-tuning         → Confidential GPU VM
   ├─ Smallest possible trust boundary     → SGX enclave
   ├─ Multi-party data collaboration       → Confidential Clean Rooms
   └─ Database queries over encrypted data → Always Encrypted with secure enclaves

Is it general defence in depth?
   └─ Trusted Launch + encryption at host + private networking + CMK where required
```

### Secure key release pattern

```text
1. Confidential VM boots; hardware generates an attestation report
2. Workload sends the report to Azure Attestation → receives a signed token
3. Workload presents the token to Managed HSM / Key Vault
4. The key's release policy checks the token claims (TEE type, measurements, configuration)
5. Only if they match is the key released, wrapped for that TEE
6. Data is decrypted inside the TEE only
```

This is how you make "data can only be processed in an environment I have verified" enforceable, not just documented.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Dedicated Host | Only for licensing economics, a contractual tenancy requirement, or strict maintenance control |
| Confidential VMs | When the threat model explicitly includes the platform operator, or a regulator or customer requires data-in-use protection |
| SEV-SNP, TDX or SGX | VM-level (SEV-SNP or TDX) by default. SGX for specialised, enclave-designed applications |
| Customer-managed keys with secure key release | When you need cryptographic enforcement of where data can be decrypted |
| Confidential GPUs | For self-hosted models over highly sensitive data, with capacity and cost accepted |
| Scope | Apply to the specific sensitive workload, not the whole estate |

**Cost and constraint.** Dedicated hosts bill for the whole server. Confidential sizes cost somewhat more, are available in fewer regions and support fewer features. Both narrow your options for scaling and disaster recovery. Make sure the requirement is real and specific before accepting those limits.

## Hands-on

Mostly exploration; confidential and dedicated capacity may not be available in your subscription or region.

```bash
LOC=swedencentral

# 1. Which confidential VM sizes exist here?
az vm list-skus -l $LOC --resource-type virtualMachines \
  --query "[?starts_with(name,'Standard_DC') || starts_with(name,'Standard_EC')].{Size:name, Zones:join(',',locationInfo[0].zones), ConfType:capabilities[?name=='ConfidentialComputingType'].value|[0], Restricted:restrictions[0].reasonCode}" -o table

# 2. Dedicated host SKUs here
az vm list-skus -l $LOC --resource-type hostGroups/hosts --query "[].name" -o tsv | sort -u | head -20

# 3. Images that support confidential VMs
az vm image list --publisher Canonical --offer ubuntu-24_04-lts --all -l $LOC \
  --query "[?contains(sku,'cvm')].urn" -o tsv | tail -3
```

If you have quota, deploy a confidential VM and confirm its protection from inside:

```bash
RG=rg-cvm-lab; az group create -n $RG -l $LOC -o none
az vm create -g $RG -n vm-cvm --size Standard_DC2as_v5 \
  --image Canonical:ubuntu-24_04-lts:cvm:latest \
  --security-type ConfidentialVM --os-disk-security-encryption-type DiskWithVMGuestState \
  --enable-secure-boot true --enable-vtpm true \
  --public-ip-address "" --admin-username azureuser --generate-ssh-keys -o none

az vm show -g $RG -n vm-cvm --query "securityProfile"
az vm run-command invoke -g $RG -n vm-cvm --command-id RunShellScript \
  --scripts "dmesg | grep -i -E 'sev|snp|tdx|memory encryption' | head" --query "value[0].message" -o tsv

az group delete -n $RG --yes --no-wait
```

Adjust the size and image SKU to what step 1 and 3 show. The kernel log should report memory encryption as active.

Paper exercises:

1. A bank wants an internal assistant over customer records and asks for "confidential computing". Write five questions that establish the actual threat model, and say which controls you would propose if the answers show the concern is (a) network exposure, (b) key control, (c) Microsoft support access, (d) operator access to memory.
2. A company runs 60 SQL Server Enterprise VMs of 8 vCPUs each. Outline how you would evaluate Dedicated Hosts with Azure Hybrid Benefit against ordinary VMs.

<details>
<summary>Notes on exercise 1</summary>

(a) Private endpoints, no public access, controlled egress. (b) Customer-managed keys in Key Vault or Managed HSM. (c) Customer Lockbox, plus logging and access controls. (d) Confidential VMs or containers with attestation and secure key release; for managed model APIs, a confidential inferencing offering or self-hosting on confidential GPUs.

</details>

## Security, cost and reliability notes

- **Security:** confidential computing changes who you must trust. It does not make an insecure application secure.
- **Security:** attestation is the point. A confidential VM whose attestation nobody verifies provides weaker assurance than the same VM with secure key release.
- **Security:** key management becomes critical. Use Managed HSM, purge protection and tested recovery.
- **Cost:** dedicated hosts are paid in full whether packed or empty. Model utilisation honestly.
- **Reliability:** both options restrict size, region and feature availability, including some backup and disaster recovery features. Design availability at the application layer and check feature support before committing.
- **Reliability:** a host or host SKU retirement requires migration planning.

## Common mistakes

- Buying dedicated hosts for "security" when the requirement was data-in-use protection, or the reverse.
- A single dedicated host for a production workload.
- Assuming confidential VMs protect against application vulnerabilities or a malicious guest administrator.
- Deploying confidential VMs with no attestation-based policy, then describing the result as verified.
- Confusing Trusted Launch with confidential computing.
- Applying confidential computing estate-wide without a threat model.
- Discovering feature limitations (backup, resize, region) after the design is fixed.
- Promising a client that a managed model API runs in a TEE without confirming the specific offering.

## Check yourself

1. What problem does Azure Dedicated Host solve, and what does confidential computing solve?
2. How is a dedicated host billed?
3. What are the three states of data, and which does confidential computing address?
4. What is a trusted execution environment, and why is attestation essential?
5. Compare SEV-SNP or TDX confidential VMs with SGX enclaves.
6. How does Trusted Launch differ from a confidential VM?
7. Give two AI scenarios where confidential computing is appropriate and one risk it does not address.
8. What is secure key release?

<details>
<summary>Answers</summary>

1. Dedicated Host gives physical single tenancy and host-level control, mainly for licensing and compliance. Confidential computing protects data while in use from the platform operator and hypervisor.
2. Per host per hour, regardless of how many VMs run on it; the VMs themselves have no separate compute charge.
3. At rest, in transit, in use. Confidential computing addresses data in use.
4. A hardware-isolated, memory-encrypted environment that code outside it cannot read or tamper with. Attestation provides verifiable evidence of what is running inside, so secrets are released only to an environment you have checked.
5. SEV-SNP and TDX protect the whole VM with no application changes. SGX protects a small application enclave, minimising the trust boundary but requiring the application to be designed for it.
6. Trusted Launch protects boot integrity (Secure Boot, vTPM). A confidential VM additionally encrypts memory so the host cannot read it.
7. Appropriate: self-hosted inference over highly sensitive prompts, protecting proprietary model weights, multi-party analytics in clean rooms. Not addressed: prompt injection, leakage through outputs, over-privileged agents.
8. A Key Vault or Managed HSM capability that releases a key only to a TEE presenting an attestation token satisfying the key's release policy.

</details>

## References

- [Azure Dedicated Hosts](https://learn.microsoft.com/azure/virtual-machines/dedicated-hosts)
- [Azure confidential computing](https://learn.microsoft.com/azure/confidential-computing/overview)
- [About Azure confidential VMs](https://learn.microsoft.com/azure/confidential-computing/confidential-vm-overview)
- [Trusted execution environments](https://learn.microsoft.com/azure/confidential-computing/trusted-execution-environment)
- [Microsoft Azure Attestation](https://learn.microsoft.com/azure/attestation/overview)
- [Secure key release with confidential VMs](https://learn.microsoft.com/azure/confidential-computing/concept-skr-attestation)
- [Confidential containers on Azure](https://learn.microsoft.com/azure/confidential-computing/confidential-containers)
- [Azure confidential GPU options](https://learn.microsoft.com/azure/confidential-computing/gpu-options)
- [Azure Confidential Clean Rooms](https://learn.microsoft.com/azure/confidential-computing/confidential-clean-rooms)
- [Virtual machine isolation in Azure](https://learn.microsoft.com/azure/virtual-machines/isolation)
