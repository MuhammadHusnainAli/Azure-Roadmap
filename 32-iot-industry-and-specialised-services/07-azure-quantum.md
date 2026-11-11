# Azure Quantum

## Why it matters

Quantum computing promises speed-ups for specific problems (simulating molecules and materials, certain optimisation and cryptography problems) that classical computers struggle with. Today's hardware is still limited, but organisations in chemistry, pharmaceuticals, finance and materials science are experimenting now, and every security architect must plan for **post-quantum cryptography** because future quantum computers could break today's public-key encryption. **Azure Quantum** provides cloud access to quantum hardware from partners (IonQ, Quantinuum, Pasqal, Rigetti), the free open-source **Microsoft Quantum Development Kit (QDK)** with Q#, Python and other languages, the **Resource Estimator** for future fault-tolerant machines, and hybrid quantum-classical workflows. This file is awareness level.

## Key concepts

### Building blocks

| Component | Purpose |
|---|---|
| **Azure Quantum workspace** | Azure resource to submit jobs to quantum hardware and simulators, manage providers and track jobs/costs |
| **Providers** | Quantum hardware partners, including **IonQ** and **Quantinuum** (trapped ions), **Pasqal** (neutral atoms) and **Rigetti** (superconducting); availability and targets change, so check the target list |
| **Microsoft Quantum Development Kit (QDK)** | Free, open-source SDK: VS Code extension and Python package; write programs in **Q#**, OpenQASM, Qiskit or Cirq; local simulation, debugging, circuit visualisation |
| **Q#** | Microsoft's high-level quantum programming language expressing programs as classical control with quantum operations |
| **Resource Estimator** | Estimates qubits and runtime needed to run an algorithm on future fault-tolerant hardware under different qubit and error-correction assumptions |
| **Hybrid quantum computing** | Classical and quantum processors working together (from batch submission to interactive sessions and integrated hybrid programs) |
| **Microsoft quantum hardware research** | Topological qubit research (Majorana-based approach); not a production service |

### Realistic use today

| Activity | Value now |
|---|---|
| **Learning and experimentation** | Build skills and intuition with simulators and small hardware runs |
| **Algorithm research** | Chemistry, optimisation, finance experiments |
| **Resource estimation** | Understand when a problem might become feasible |
| **Post-quantum readiness** | Inventory cryptography; plan migration to quantum-safe algorithms (independent of Azure Quantum) |

Most business problems are better served today by classical HPC, optimisation solvers and AI; quantum is a research investment.

## How it works on Azure

Hello world with the QDK in Python (local simulation):

```python
# pip install qsharp
import qsharp

qsharp.eval("""
operation BellPair() : (Result, Result) {
    use (q1, q2) = (Qubit(), Qubit());
    H(q1);
    CNOT(q1, q2);
    let results = (M(q1), M(q2));
    ResetAll([q1, q2]);
    return results;
}
""")
print(qsharp.run("BellPair()", shots=10))
```

Submitting to hardware: create an Azure Quantum workspace with selected providers, connect from the QDK (`azure-quantum` Python package), choose a target (simulator or QPU), submit the job, and review results and costs. Provider credits for experimentation may be available through Azure Quantum programmes.

Resource estimation: run the Resource Estimator on an algorithm (for example factoring or chemistry simulation) to see physical qubits and runtime under different assumptions.

Post-quantum planning (security track): inventory where RSA/ECC are used (TLS, code signing, certificates, VPNs, data encrypted for long-term storage), follow NIST post-quantum standards adoption in platforms and libraries, and prioritise data with long confidentiality lifetimes ("harvest now, decrypt later").

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Invest or watch** | Industries with chemistry/materials/optimisation problems may invest in skills and pilots; most should watch and prepare crypto. |
| **Language** | Q# for Microsoft tooling and resource estimation; Qiskit/Cirq/OpenQASM where teams already use them (QDK supports them). |
| **Hardware choice** | Different modalities suit different experiments; use simulators first. |
| **Cost control** | Hardware jobs are billed per provider pricing; set budgets and use simulators for development. |
| **Security** | Post-quantum cryptography migration is a near-term action regardless of quantum computing adoption. |

## Hands-on

1. Install the QDK VS Code extension and run the Bell pair example locally.
2. Create an Azure Quantum workspace and run a small job on a provider simulator.
3. Use the Resource Estimator on a sample algorithm and interpret the results.
4. Draft a cryptographic inventory for one application as a first post-quantum readiness step.

## Security, cost and reliability notes

- **Workspace access** via Entra RBAC; job data may be sent to third-party providers under their terms.
- **Cost**: provider-specific pricing per shot/job/time; budgets and alerts.
- **Maturity**: results on current hardware are noisy; treat as research.

## Common mistakes

| Mistake | Fix |
|---|---|
| Promising business speed-ups today | Treat as research; compare to classical |
| Running experiments directly on hardware | Simulate first |
| Ignoring post-quantum cryptography | Start inventory and planning |
| Uncontrolled hardware spending | Budgets |

## Check yourself

1. What does Azure Quantum provide?
2. Name the hardware providers and their qubit modalities.
3. What is the QDK, and which languages does it support?
4. What does the Resource Estimator do?
5. What is hybrid quantum computing?
6. Why should organisations act on post-quantum cryptography now?

<details>
<summary>Answers</summary>

1. Cloud access to quantum hardware and simulators via workspaces, the QDK, resource estimation and hybrid workflows.
2. IonQ and Quantinuum (trapped ions), Pasqal (neutral atoms), Rigetti (superconducting); check current availability.
3. Microsoft's free, open-source quantum SDK supporting Q#, OpenQASM, Qiskit and Cirq with simulation and debugging.
4. Estimates qubits and runtime needed on future fault-tolerant hardware under different assumptions.
5. Classical and quantum processors working together on one problem.
6. Data encrypted today could be harvested and decrypted later; migration takes years.

</details>

## References

- [What is Azure Quantum?](https://learn.microsoft.com/azure/quantum/overview-azure-quantum)
- [Quantum Development Kit](https://learn.microsoft.com/azure/quantum/qdk-main-overview)
- [Quantum computing targets](https://learn.microsoft.com/azure/quantum/qc-target-list)
- [Resource Estimator](https://learn.microsoft.com/azure/quantum/intro-to-resource-estimation)
- [Hybrid quantum computing](https://learn.microsoft.com/azure/quantum/hybrid-computing-overview)
