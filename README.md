# Kubernetes Day 2 Operations
### The Rack2Cloud Diagnostic Method & Failure Signature Library

![Status](https://img.shields.io/badge/status-architecture--pattern-orange)

> **Architecture Principle:** Kubernetes is an eventual-consistency engine, not a hierarchy of virtual machines. Day 2 outages are rarely single-component failures; they are the result of intersecting control loops (Identity, Compute, Network, Storage) grinding against each other.

---

## About This Repository

This repository consolidates Rack2Cloud research on Kubernetes day-2 operations into a structured reference for platform engineers, SREs, and infrastructure architects responsible for Kubernetes clusters in production.

Day-1 Kubernetes is well-documented. Day-2 is where production clusters break. This repository covers the operational and governance failure modes that appear after initial deployment: autoscaler misconfigurations, ingress migration failures, etcd degradation, storage zone lock-in, policy drift under GitOps, and cluster configuration that nobody owns.

The intended audience is engineers operating Kubernetes clusters in production, not engineers setting them up for the first time.

---

## Problem Statement

Most engineering teams treat Kubernetes incidents as isolated bugs and debug the symptom (e.g., staring at a 502 error or a Pending pod) rather than the system. 

When a storage placement decision creates a cross-zone network bottleneck, teams waste hours debugging the CNI or Ingress controller when the root cause was a CSI binding mode. True Day 2 reliability requires diagnosing the intersections of the cluster's control loops.

---

## System Model

![Intersecting Control Loops](https://www.rack2cloud.com/wp-content/uploads/2026/02/kubernetes_intersecting-control-loop.jpg)

**The 4 Intersecting Loops:**
1. **Identity Loop:** Authenticates the request (ServiceAccount → Cloud IAM/Entra ID).
2. **Compute Loop:** Places the workload (Scheduler → Kubelet & Budgets).
3. **Network Loop:** Routes the packet (CNI → IP Tables → Ingress).
4. **Storage Loop:** Provisions the physics (CSI → EBS/Azure Disk/PD).

---

## Failure Signature & Mitigation Model

| Symptom / Error | Failing Loop | Root Cause Physics | Key Metric | Mitigation Strategy |
| :--- | :--- | :--- | :--- | :--- |
| **`ImagePullBackOff` / `CrashLoopBackOff`** | **Identity** | The registry is up, but the Node's STS/OIDC token expired or clock drifted. Auth fails silently — container crashes before writing useful logs. | `kube_pod_container_status_restarts_total` + exit code from `kubectl describe pod` | Ephemeral workload identity; remove static secrets; monitor IMDS reachability. |
| **Pending Pods (<50% CPU)** | **Compute** | Scheduler bin-packing fragmentation — allocatable headroom exhausted even when utilisation looks low. `maxUnavailable: 0` PDB deadlocks compound this. | `kube_node_status_allocatable` vs `kube_pod_resource_requests_cpu_cores` sum by node | Soften affinity rules (`ScheduleAnyway`); audit strict Requests/Limits ratio. |
| **`502/504 Gateway Timeout`** | **Network** | MTU mismatch between overlay (VXLAN/Geneve) and underlying NIC. Large packets silently dropped — no ICMP response in cloud provider networks. Presents as DNS until proven otherwise. | `node_network_receive_drop_total` + `node_network_transmit_drop_total` per overlay interface | Lower CNI MTU to account for overlay headers; spoof Host headers to test paths. |
| **PVC Stuck / Pod in `ContainerCreating`** | **Storage** | Block storage is zonal. Pod rescheduled across AZ after node failure — volume can't follow. Default storage classes don't use `WaitForFirstConsumer`. | `kube_persistentvolumeclaim_status_phase{phase="Pending"}` correlated with volume attach events | Enforce `volumeBindingMode: WaitForFirstConsumer` on all StatefulSets. |
| **API Server Timeouts / Controller Flapping** | **Control Plane** | etcd I/O-bound — write-ahead log hits disk IOPS ceiling under burst. etcd heartbeat latency increases, API server queues requests, controllers lose sync. Workload layer looks healthy while control plane drowns. | `etcd_disk_wal_fsync_duration_seconds_bucket` p99 > 10ms = warning, > 100ms = critical. `apiserver_request_duration_seconds_bucket` p99 climbing with no traffic increase = control-plane problem. | Dedicated low-latency disk for etcd (gp3 minimum, io2 preferred); separate etcd nodes from workload nodes; monitor WAL fsync p99 continuously. On managed K8s (EKS/AKS/GKE): monitor API server latency as the proxy metric. |
---

## Zero-Trust Day 2 Requirements

To guarantee cluster stability under load, the architecture must enforce these loop rules:
1. **Identity is Ephemeral:** Cloud credentials must never be hardcoded as Kubernetes Secrets. Map IAM roles directly to ServiceAccounts.
2. **Compute is Budgeted:** A pod without CPU/Memory requests is a rogue process. Unbudgeted pods will be evicted first during node pressure.
3. **Data Has Gravity:** Storage provisioning must be deferred until the Compute scheduler has definitively selected an Availability Zone. 
4. **Loop-to-Loop Observability:** Logs are useless without context. Every structured log line must contain `trace_id`, `namespace`, `pod`, `node`, and `zone`.

---

## Framework Structure

### Day-2 Operations Framework

The strategic and methodological foundation for Kubernetes day-2 operations.

- [The Rack2Cloud Method: A Strategic Guide to Kubernetes Day 2 Operations](https://www.rack2cloud.com/kubernetes-day-2-operations-guide-rack2cloud-method/) — The foundational operational framework for Kubernetes day-2.
- [Kubernetes Day-2 Incidents: 5 Real-World Failures and the One Metric That Predicts Them](https://www.rack2cloud.com/kubernetes-day-2-failures/) — Incident taxonomy and leading indicators for day-2 failure.

---

### Autoscaling Architecture

Autoscaling failure modes and the authority model that governs them.

**Autoscaling as Authority**

- [Autoscaling Is an Authority System, Not a Capacity System](https://www.rack2cloud.com/autoscaling-authority-system/) — Autoscaling reframed as a system that grants authority over cluster resources — not one that reacts to capacity signals. *(Added 2026-06-30)*
- [VPA vs HPA: Why Most Teams Choose the Wrong Autoscaler](https://www.rack2cloud.com/vpa-vs-hpa-kubernetes/) — Decision framework for vertical vs. horizontal autoscaling.
- [Vertical Pod Autoscaler in Production: In-Place Resize Works — Until It Doesn't](https://www.rack2cloud.com/vertical-pod-autoscaler-in-place-resize-production/) — VPA production failure modes and in-place resize constraints.
- [Kubernetes 1.35 Removes the Restart Tax — Why Stateful Workloads Just Became Easier to Operate](https://www.rack2cloud.com/kubernetes-1-35-in-place-pod-resize-production/) — In-place pod resize in Kubernetes 1.35 and its day-2 implications.

**Resource Scheduling**

- [Kubernetes Requests vs Limits: The Scheduler Guarantees One Thing. The Kernel Enforces Another.](https://www.rack2cloud.com/kubernetes-resource-requests-vs-limits/) — Resource request and limit semantics as a production failure mode.
- [Your Kubernetes Cluster Isn't Out of CPU — The Scheduler Is Stuck](https://www.rack2cloud.com/kubernetes-scheduler-stuck-cpu-fragmentation/) — CPU fragmentation as a scheduler failure mode.
- [Google Just Moved the Control Plane Boundary](https://www.rack2cloud.com/control-plane-boundary-kubernetes-scale/) — Control plane boundary as a scaling constraint.

---

### Ingress and Gateway API

Migration from Ingress to Gateway API and the failure modes at each stage.

**Migration Architecture**

- [Kubernetes Is Moving Past Ingress. Most Clusters Aren't.](https://www.rack2cloud.com/kubernetes-ingress-gateway-api-migration/) — Gateway API as the direction; why most clusters haven't followed.
- [Kubernetes Ingress to Gateway API Migration: How to Move Without Breaking Production](https://www.rack2cloud.com/migrate-ingress-to-gateway-api-production/) — Migration execution framework for production ingress.
- [Operating Gateway API in Production: What the Migration Guides Don't Cover](https://www.rack2cloud.com/operating-gateway-api-production/) — Post-migration Gateway API operational failure modes.

**Controller Selection**

- [Gateway API Is the Direction. Your Controller Choice Is the Risk.](https://www.rack2cloud.com/gateway-api-kubernetes-controller-decision/) — Controller selection as the primary Gateway API risk decision.
- [Ingress-NGINX Deprecation: What to Do Next (Four Paths, Four Failure Modes)](https://www.rack2cloud.com/ingress-nginx-deprecation-what-to-do/) — Ingress-NGINX deprecation response options and their failure modes.

**Production Debugging**

- [It's Not DNS (It's MTU): Debugging Kubernetes Ingress](https://www.rack2cloud.com/kubernetes-ingress-502-debug-mtu-dns/) — MTU as the non-obvious ingress failure cause.

---

### Networking Architecture

Cluster networking failure modes and architecture decisions.

**Service Mesh and eBPF**

- [Service Mesh vs eBPF in Kubernetes: Cilium vs Calico Networking Explained](https://www.rack2cloud.com/service-mesh-vs-ebpf-kubernetes-cilium-vs-calico/) — Service mesh vs. eBPF architecture decision and failure mode comparison.

**Container Runtime**

- [containerd in Production: 5 Day-2 Failure Patterns at High Pod Density](https://www.rack2cloud.com/containerd-in-production-day2-failure-patterns/) — containerd failure modes at production pod density.
- [containerd vs CRI-O: Memory Overhead at Scale (Real Node Density Limits)](https://www.rack2cloud.com/containerd-vs-cri-o-memory-overhead-scale/) — Runtime memory overhead as a node density constraint.
- [The Container Runtime Benchmark 2026: containerd vs CRI-O vs crun for High-Density Nodes](https://www.rack2cloud.com/container-runtime-optimization-density-guide/) — Runtime performance comparison at high pod density.

**IP and Network Capacity**

- [GKE IP Exhaustion 2026: The /24 Trap & Autopilot's Hidden Cost](https://www.rack2cloud.com/gke-pod-ip-exhaustion-vpc-triage/) — IP exhaustion as a cluster capacity failure mode.
- [Client's GKE Cluster Ate Their Entire VPC: The IP Math I Uncovered During Triage](https://www.rack2cloud.com/gke-pod-ip-exhaustion-triage-part-1/) — IP exhaustion field incident analysis.
- [Client's GKE Cluster Ate Their Entire VPC: The Class E Rescue (Part 2)](https://www.rack2cloud.com/gke-ip-exhaustion-fix-part-2/) — IP exhaustion remediation.

---

### Storage and Persistent Volumes

Storage failure modes for stateful workloads in Kubernetes.

- [PersistentVolumes vs StorageClasses: When You Actually Need Each](https://www.rack2cloud.com/persistentvolume-vs-storageclass-kubernetes/) — PV and StorageClass semantics as a production configuration failure mode.
- [Storage Has Gravity: Debugging PVCs & AZ Lock-in](https://www.rack2cloud.com/kubernetes-pvc-stuck-volume-node-affinity/) — PVC zone lock-in as a stateful workload failure mode.

---

### Control Plane and etcd

Control plane architecture and etcd as a production failure domain.

- [etcd Is Your Kubernetes Database: What It Does, What Breaks, and What to Watch](https://www.rack2cloud.com/etcd-kubernetes-database/) — etcd as the cluster database — failure modes and observability requirements.
- [Resource Pooling Physics: Mastering CPU Wait Time and Memory Ballooning in High-Density Clusters](https://www.rack2cloud.com/resource-pooling-physics-cpu-wait-memory-ballooning/) — Control plane resource contention at high cluster density.

---

### Cluster Governance and Policy

Configuration ownership, drift detection, and policy enforcement in production clusters.

**Configuration Ownership**

- [Configuration Drift Is the Symptom. Ownership Is the Problem.](https://www.rack2cloud.com/configuration-drift-ownership/) — Configuration ownership gaps as the root cause of cluster configuration drift. *(Added 2026-06-30)*
- [The Infrastructure Team Is the Real Single Point of Failure](https://www.rack2cloud.com/infrastructure-bus-factor/) — Knowledge concentration as a cluster operational risk. *(Added 2026-06-30)*

**GitOps and Policy Governance**

- [Policy Drift Is the Real Day-2 Failure in GitOps](https://www.rack2cloud.com/gitops-policy-drift/) — Policy drift as the primary GitOps day-2 failure mode in cluster environments. *(Added 2026-06-30)*
- [The Console Is the Shadow Control Plane](https://www.rack2cloud.com/shadow-control-plane/) — Console access as the mechanism of cluster configuration corruption. *(Added 2026-06-30)*
- [Your CI-CD Pipeline Is Your Real Infrastructure Control Plane](https://www.rack2cloud.com/ci-cd-control-plane-infrastructure/) — CI/CD as the authoritative cluster control plane. *(Added 2026-06-30)*

**Auditability**

- [Infrastructure Needs Auditability, Not Just Idempotency](https://www.rack2cloud.com/infrastructure-auditability/) — Auditability as a cluster governance requirement distinct from idempotency. *(Added 2026-06-30)*

**GitOps Tooling**

- [GitOps Boundary Mapper](https://www.rack2cloud.com/gitops-boundary-mapper/) — Tool for defining and enforcing GitOps boundaries in cluster environments. *(Added 2026-06-30)*

---

### Security and Multi-Cluster Architecture

Security failure modes and multi-cluster architectural constraints.

**Security Boundaries**

- [Kubernetes Is Not an LLM Security Boundary](https://www.rack2cloud.com/kubernetes-llm-security-boundary/) — Kubernetes security model failure under LLM workloads.
- [Seccomp vs AppArmor: Which Actually Stops Container Breakouts?](https://www.rack2cloud.com/seccomp-vs-apparmor-container-breakout/) — Container security policy evaluation for day-2 enforcement.

**Multi-Cluster Failure Modes**

- [Multi-Cloud Failover Is Mostly Theater](https://www.rack2cloud.com/multi-cloud-failover-theater/) — Multi-cluster failover assumptions that fail under real conditions. *(Added 2026-06-30)*
- [Your Cloud Provider Is Not Your HA Strategy](https://www.rack2cloud.com/multi-region-cloud-architecture-ha-strategy/) — Provider availability as an insufficient HA design basis.

---

### Production Debugging Reference

Field-level debugging patterns for common Kubernetes production failures.

- [Kubernetes ImagePullBackOff: It's Not the Registry (It's IAM)](https://www.rack2cloud.com/kubernetes-image-pull-back-off-iam-guide/) — ImagePullBackOff root cause analysis.
- [It's Not DNS (It's MTU): Debugging Kubernetes Ingress](https://www.rack2cloud.com/kubernetes-ingress-502-debug-mtu-dns/) — MTU as a non-obvious ingress failure cause.
- [Your Kubernetes Cluster Isn't Out of CPU — The Scheduler Is Stuck](https://www.rack2cloud.com/kubernetes-scheduler-stuck-cpu-fragmentation/) — CPU fragmentation debugging.
- [Storage Has Gravity: Debugging PVCs & AZ Lock-in](https://www.rack2cloud.com/kubernetes-pvc-stuck-volume-node-affinity/) — PVC zone lock-in debugging.

---

## Assessment Tools

| Tool | Purpose |
|------|---------|
| [GitOps Boundary Mapper](https://www.rack2cloud.com/gitops-boundary-mapper/) | Define and enforce GitOps boundaries in Kubernetes cluster environments |

---

## Canonical Architecture Learning Path

The [Cloud Architecture Path](https://www.rack2cloud.com/cloud-learning-path/) and [Modern Infrastructure & IaC Path](https://www.rack2cloud.com/modern-infrastructure-iac-learning-path/) provide the structured learning context for this repository's content.

Primary page: [The Rack2Cloud Method: A Strategic Guide to Kubernetes Day 2 Operations](https://www.rack2cloud.com/kubernetes-day-2-operations-guide-rack2cloud-method/)

---

## Non-Goals

- Day 0 Cluster Installation guides
- CI/CD pipeline tutorials
- Application code debugging

*This is a systems engineering and infrastructure diagnostic framework.*

---

## Maintenance Notes

This repository is maintained against the Rack2Cloud [Canonical Architecture Specifications](https://www.rack2cloud.com/canonical-architecture-specifications/) governance system.

---

## Support

If this framework helped you survive a Day 2 outage, please star the repository. 

Architectural frameworks maintained by **[Rack2Cloud](https://www.rack2cloud.com)**.
