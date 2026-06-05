# Kubernetes — Staff Software Engineer Interview Deep Dive

End-to-end prep for a Staff Software Engineer interview on **Kubernetes** — whether the role is at the upstream project (Kubernetes SIGs / CNCF), at a hyperscaler's managed-k8s team (GKE, EKS, AKS), at a k8s-distribution vendor (Red Hat OpenShift, VMware Tanzu, Rancher), or at an end-user with deep k8s infrastructure (Lyft, Spotify, Pinterest).

The depth target is **staff-level**: you should be able to trace a `kubectl apply` end-to-end through every component, name the controller that owns each resource type, cite the scale limit (max 5000 nodes / 150K pods) and the cause, and design a CRD + operator with finalizers, owner references, status sub-resources, and version conversion.

23 numbered files + this README. Each file is self-contained; read out of order, hit your weak spots. Read `02` first if you've never traced a kubectl call through the apiserver.

## How to use this pack

- **6+ weeks out**: read `01` → `06` to internalize the control plane. Stand up a kind / minikube cluster and trace `kubectl create pod` through audit logs.
- **3–4 weeks out**: deep dive on the rest (`07`–`18`). Build a toy CRD + controller. Run chaos tests (kill kube-apiserver, watch what happens).
- **1–2 weeks out**: case studies (`19`), corner cases (`20`), system design (`21`), coding (`22`).
- **Last 48 hours**: `23-staff-topics-and-cheatsheets.md`, the 60-second pitches in this README, STAR stories.

## File map

| # | File | What it covers |
|---|------|----------------|
| 01 | [01-role-and-interview-context.md](01-role-and-interview-context.md) | Where K8s SWEs work; loop format; signals graders look for |
| 02 | [02-architecture-overview.md](02-architecture-overview.md) | Control plane (apiserver/etcd/scheduler/controller-manager) + data plane (kubelet/runtime/CNI), packet+request flow |
| 03 | [03-api-server-deep-dive.md](03-api-server-deep-dive.md) | REST surface, admission chain, validation, conversion, watch streams, aggregation, EncryptionConfiguration |
| 04 | [04-etcd-deep-dive.md](04-etcd-deep-dive.md) | Raft, MVCC, watch, lease, compaction, defrag, sizing, the 8GB DB limit |
| 05 | [05-controllers-and-informers.md](05-controllers-and-informers.md) | Reconcile pattern, work queues, informers, shared cache, leader election |
| 06 | [06-scheduler-deep-dive.md](06-scheduler-deep-dive.md) | Filter/Score plugin framework, PreFilter, Permit, Bind, scheduler extender, Volcano/Yunikorn |
| 07 | [07-kubelet-and-cri.md](07-kubelet-and-cri.md) | kubelet syncloop, PLEG, pod sandbox, CRI, image pull, eviction, NodeReady, cgroup management |
| 08 | [08-networking-cni-and-services.md](08-networking-cni-and-services.md) | CNI spec + plugins (Calico/Cilium/Flannel), Services, EndpointSlices, kube-proxy modes, DNS |
| 09 | [09-network-policies-and-mesh.md](09-network-policies-and-mesh.md) | NetworkPolicy CRDs, Cilium eBPF, service meshes (Istio/Linkerd/Cilium ambient), Gateway API |
| 10 | [10-storage-csi-and-volumes.md](10-storage-csi-and-volumes.md) | PV/PVC, StorageClass, CSI drivers, dynamic provisioning, snapshots, ephemeral volumes |
| 11 | [11-auth-rbac-and-security.md](11-auth-rbac-and-security.md) | TLS chain, ServiceAccount, kubeconfig, RBAC, webhook authz, OPA/Gatekeeper, Pod Security Admission, image signing |
| 12 | [12-workloads-deployments-and-jobs.md](12-workloads-deployments-and-jobs.md) | Deployment / StatefulSet / DaemonSet / Job / CronJob — semantics, edge cases, rollouts |
| 13 | [13-custom-resources-and-operators.md](13-custom-resources-and-operators.md) | CRD, schema validation, conversion webhook, finalizers, owner references, controller-runtime, kubebuilder |
| 14 | [14-scaling-hpa-vpa-and-karpenter.md](14-scaling-hpa-vpa-and-karpenter.md) | HPA (metrics, custom, external), VPA, Cluster Autoscaler, Karpenter, descheduler |
| 15 | [15-observability-events-and-audit.md](15-observability-events-and-audit.md) | metrics-server, kube-state-metrics, audit logs, Events, OpenTelemetry, ContainerLogPath |
| 16 | [16-multi-cluster-and-cluster-api.md](16-multi-cluster-and-cluster-api.md) | Cluster API, Karmada / KubeFed, multi-cluster services, fleet management, hub-and-spoke |
| 17 | [17-upgrades-and-version-policy.md](17-upgrades-and-version-policy.md) | Skew policy (N-2), control-plane upgrade order, node drain, API deprecation lifecycle |
| 18 | [18-performance-and-scale-limits.md](18-performance-and-scale-limits.md) | The famous limits (5000 nodes/150K pods/300K containers), what hits each limit, scale-test methodology |
| 19 | [19-case-studies-and-realistic-scenarios.md](19-case-studies-and-realistic-scenarios.md) | Production stories: Lyft, Spotify, Pinterest, AWS, Google; the famous incidents |
| 20 | [20-corner-cases-and-alternatives.md](20-corner-cases-and-alternatives.md) | "Name 3 ways to solve this" file: terminating pods, watch starvation, etcd OOM, etc. |
| 21 | [21-system-design-questions.md](21-system-design-questions.md) | 12 design prompts: build a scheduler; build a CRD; build a CNI; build kubelet-lite |
| 22 | [22-coding-and-operator-problems.md](22-coding-and-operator-problems.md) | Operator coding problems with controller-runtime; Go-flavored algorithm questions |
| 23 | [23-staff-topics-and-cheatsheets.md](23-staff-topics-and-cheatsheets.md) | Staff scope + last-48-hours flashcards: defaults, limits, RFCs/KEPs, port numbers |

## The 60-second elevator pitch — Kubernetes architecture

> "Kubernetes is a declarative orchestrator. The cluster state is stored in **etcd** (Raft-replicated, MVCC). The **kube-apiserver** is the only thing that talks to etcd; it exposes a versioned REST API, runs requests through an **admission chain** (authentication → authorization → mutating webhooks → validation → validating webhooks → resource quota), then writes to etcd and broadcasts via watch streams. **Controllers** (in the controller-manager binary, plus custom operators) watch for resource changes via informers (a local cache + reconciliation loop) and reconcile desired vs actual state — they own the 'eventually consistent' nature of the cluster. The **scheduler** watches for unscheduled pods and binds them to nodes via the Filter→Score plugin framework. On each node, the **kubelet** watches the apiserver for pods bound to itself; it talks to a **container runtime** (containerd / CRI-O via CRI gRPC), which uses **runc** + **OCI** to start containers and a **CNI plugin** to attach network. Pod-to-pod traffic uses the CNI's chosen dataplane (Calico-BGP, Cilium-eBPF, flannel-vxlan); **kube-proxy** (or eBPF replacement) implements Service VIPs via iptables/IPVS/eBPF. The whole design is **eventually consistent, retry-friendly, and watch-driven** — every control loop is idempotent reconciliation against the current state, not procedural orchestration."

## The 60-second pitch — the reconcile loop

> "The fundamental K8s pattern is: a controller watches a resource type via an informer (which keeps a local cache and emits add/update/delete events), pushes the changed object's key into a work queue, and a reconcile worker pops the key, fetches the current state from the cache, computes the desired state, and makes apiserver calls to converge. Reconciles are **idempotent** — running twice for the same object should produce the same outcome. The work queue **deduplicates** keys, so a flood of events results in one reconcile. **Rate limiting** + **exponential backoff** on failures prevents hot loops. **Leader election** (via a Lease) ensures only one replica reconciles at a time. The controller never trusts its cache for writes — it always issues an apiserver PATCH/UPDATE that the apiserver validates against the current resource version (optimistic concurrency)."

## The 60-second pitch — etcd's role

> "etcd is the single source of truth: Raft for consensus (typically 3 or 5 voters), MVCC-style storage (every write gets a new revision; the watch stream is a revision range subscription), lease-based ephemeral keys (for node heartbeats, leader election), and a hard limit of 8GB on the underlying boltdb DB size (configurable but expensive past ~16GB). It's exposed only to the apiserver — every other component reads/writes through apiserver as a proxy. The two most-common failure modes: **disk slowness** (>50ms write latency causes apiserver lag → cluster appears frozen) and **revision compaction lag** (uncompacted history fills the DB; defragmentation is needed periodically). At scale, etcd is replaced by sharded apiserver flavors (event tabs to separate etcd cluster) or by entirely different stores (kine = etcd-on-postgres for k3s)."

## High-frequency topic clusters (what they actually probe)

| Cluster | File | Probability |
|---------|------|-------------|
| Architecture + packet flow | 02 | **Very high** |
| Reconcile pattern + informers | 05 | **Very high** |
| API server admission + watch | 03 | **Very high** |
| Scheduler plugins + extenders | 06 | High |
| etcd Raft + scaling + 8GB | 04 | **Very high** |
| CNI + Service VIPs + kube-proxy modes | 08 | **Very high** |
| CRD + operator design | 13 | **Very high** |
| RBAC + admission + OPA | 11 | High |
| HPA / Cluster Autoscaler / Karpenter | 14 | High |
| Scale limits + their causes | 18 | **Very high** (staff signal) |
| Production incidents + war stories | 19 | High |
| Corner cases + alternatives | 20 | **Very high** |

## What signals interviewers grade on

Past k8s staff interviewers cite four discriminators:

1. **Vocabulary precision.** "Pod" vs "container." "Service" vs "Endpoint" vs "EndpointSlice." "Controller" vs "operator" vs "manager." A staff candidate uses these correctly without thinking.
2. **End-to-end traceability.** Asked "what happens when I run `kubectl apply -f deployment.yaml`?" — staff can trace through API server → admission → etcd → deployment-controller → replicaset-controller → scheduler → kubelet → CRI → runc → CNI in 5 minutes flat.
3. **Failure-mode storytelling.** "What if etcd is slow?" → "Watch stream lags → controllers reconcile against stale cache → apiserver returns 504 → operators retry with backoff → if it persists, schedulers can't bind pods, kubelet heartbeats fail, the cluster appears frozen even though nodes are healthy."
4. **Limits and how to push them.** Quote the actual numbers (5000 nodes, 150K pods, 300K containers, 8GB etcd) and the techniques to push past (events to a separate etcd, apiserver hash sharding, watch cache tuning, namespace federation).

## A note on currency

K8s moves fast. Recent (2024–2026) developments worth knowing:

- **In-place pod resize** (alpha → beta in 1.27+, GA pending) — modify Resources without recreate.
- **Sidecar containers as a first-class concept** (1.28+ KEP-753) — lifecycle now correctly orchestrated.
- **Image volume sources** (1.31+) — mount an OCI image as a volume.
- **Kubelet credential providers** moving to plugin model (`KubeletCredentialProvider`).
- **Gateway API** stable; replacing Ingress for advanced routing.
- **structured authn** config (1.30+) — replacing webhook authenticator.
- **CRD/conversion webhook** still the only schema-evolution story.
- **Karpenter** widely adopted; replacing Cluster Autoscaler at major shops.
- **Cilium 1.16+** dominant in new clusters; ambient service mesh + kube-proxy replacement standard.
- **OCI artifacts** for non-image content (Helm charts, configs).
- **CSI snapshot + group snapshot** GA.
- **etcd's MVCC** progress watch (efficient initial sync).

Don't bluff currency. "I've followed mainline through about 1.29" is staff-honest.

## Sources used to build this pack

- Kubernetes docs (kubernetes.io)
- KEP (Kubernetes Enhancement Proposals) tree — github.com/kubernetes/enhancements
- *Kubernetes in Action, 2nd ed* (Lukša)
- *Programming Kubernetes* (Hausenblas / Schimanski) — controller-runtime deep dive
- *Production Kubernetes* (Dotson et al.) — operational patterns
- SIG meeting notes (sig-scalability, sig-api-machinery, sig-scheduling)
- KubeCon talks (search "k8s scale" / "operator" / "scheduler")
- Lyft, Spotify, Pinterest engineering blogs
- AWS EKS, Google GKE, Azure AKS architecture blogs
- Cilium docs, Karpenter docs, Crossplane docs
- The big incident postmortems (k8s GitHub issues, cncf.io postmortems)
