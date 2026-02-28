# Kubernetes Day 2 Operations
### The Rack2Cloud Diagnostic Method & Failure Signature Library

![Status](https://img.shields.io/badge/status-architecture--pattern-orange)

> **Architecture Principle:** Kubernetes is an eventual-consistency engine, not a hierarchy of virtual machines. Day 2 outages are rarely single-component failures; they are the result of intersecting control loops (Identity, Compute, Network, Storage) grinding against each other.

---

## 📚 Canonical Architecture Reference  
This repository contains the diagnostic frameworks, failure signatures, and CLI triage protocols for debugging complex Kubernetes outages across AWS, Azure, and GCP.

**Want the full diagnostic theory and Azure-native checklists?** 👉 [Download the formatted PDF Playbook here](https://www.rack2cloud.com/architecture-failure-playbooks/)

**The continuously maintained master index lives here:** 👉 [https://www.rack2cloud.com/kubernetes-day-2-operations-guide/](https://www.rack2cloud.com/kubernetes-day-2-operations-guide/)

---

## Problem Statement

Most engineering teams treat Kubernetes incidents as isolated bugs and debug the symptom (e.g., staring at a 502 error or a Pending pod) rather than the system. 

When a storage placement decision creates a cross-zone network bottleneck, teams waste hours debugging the CNI or Ingress controller when the root cause was a CSI binding mode. True Day 2 reliability requires diagnosing the intersections of the cluster's control loops.

---

## System Model

![Intersecting Control Loops](https://www.rack2cloud.com/wp-content/uploads/2026/02/diagram-k8s-control-loops.jpg)

**The 4 Intersecting Loops:**
1. **Identity Loop:** Authenticates the request (ServiceAccount → Cloud IAM/Entra ID).
2. **Compute Loop:** Places the workload (Scheduler → Kubelet & Budgets).
3. **Network Loop:** Routes the packet (CNI → IP Tables → Ingress).
4. **Storage Loop:** Provisions the physics (CSI → EBS/Azure Disk/PD).

---

## Failure Signature & Mitigation Model

| Symptom / Error | Failing Loop | Root Cause Physics | Mitigation Strategy |
| :--- | :--- | :--- | :--- |
| **`ImagePullBackOff`** | **Identity** | The registry is up, but the Node's STS/OIDC token expired or clock drifted. | Ephemeral workload identity; remove static secrets; monitor IMDS reachability. |
| **Pending Pods (<50% CPU)** | **Compute** | Scheduler bin-packing fragmentation, or `maxUnavailable: 0` PDB deadlocks. | Soften affinity rules (`ScheduleAnyway`); audit strict Requests/Limits. |
| **`502/504 Gateway Timeout`** | **Network** | MTU packet fragmentation drops, or `NodePort` health check mismatches. | Lower CNI MTU to account for overlay headers; spoof Host headers to test paths. |
| **AZ Node Affinity Conflict** | **Storage** | Disk was created in Zone A, but Scheduler wants to place Pod in Zone B. | Enforce `volumeBindingMode: WaitForFirstConsumer` on all StatefulSets. |

---

## Zero-Trust Day 2 Requirements

To guarantee cluster stability under load, the architecture must enforce these loop rules:
1. **Identity is Ephemeral:** Cloud credentials must never be hardcoded as Kubernetes Secrets. Map IAM roles directly to ServiceAccounts.
2. **Compute is Budgeted:** A pod without CPU/Memory requests is a rogue process. Unbudgeted pods will be evicted first during node pressure.
3. **Data Has Gravity:** Storage provisioning must be deferred until the Compute scheduler has definitively selected an Availability Zone. 
4. **Loop-to-Loop Observability:** Logs are useless without context. Every structured log line must contain `trace_id`, `namespace`, `pod`, `node`, and `zone`.

---

## Non-Goals

- Day 0 Cluster Installation guides
- CI/CD pipeline tutorials
- Application code debugging

*This is a systems engineering and infrastructure diagnostic framework.*

---

## Support

If this framework helped you survive a Day 2 outage, please star the repository. 

Architectural frameworks maintained by **[Rack2Cloud](https://www.rack2cloud.com)**.
