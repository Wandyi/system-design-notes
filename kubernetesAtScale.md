# Staff-Level Kubernetes: Scenarios & Best Practices at Scale

Hard-won patterns for running distributed systems on Kubernetes with focus on **scalability**, **reliability**, and **availability**. Organized by scenario first, then cross-cutting practices.

---

## 1. Multi-Tenant Platform Cluster (Internal PaaS)

### Scenario
One cluster shared by 50+ teams, 500+ services. A noisy neighbor can't take down the platform; a bad deploy can't consume all capacity; teams can self-serve without SRE babysitting.

### Design
- **Namespace-per-team** with `ResourceQuota` + `LimitRange`. Without these, one team's leak OOMs nodes for everyone.
  - `ResourceQuota`: caps total CPU/memory/PVC count per ns.
  - `LimitRange`: default + max per pod, so pods without requests still get a sane floor.
- **NetworkPolicies default-deny**: traffic between namespaces is blocked unless explicitly allowed. A compromised pod in `team-a` can't scan `team-b`'s internal services.
- **Pod Security Standards** (restricted) enforced via Pod Security Admission. No privileged containers, no hostPath, read-only root FS.
- **Gatekeeper / Kyverno** for policy-as-code: "images must come from our registry", "every Deployment must have PDB", "prod namespaces require 2 replicas min".

### Scalability
- **Cluster autoscaler vs Karpenter**: Karpenter wins for heterogeneous workloads — it picks cheapest instance fitting pending pods instead of scaling a fixed nodegroup.
- **Etcd is the ceiling**. One cluster ≠ infinite scale. Practical limits: ~5000 nodes, ~150K pods, ~300K total objects. Beyond that, split clusters.
- **Control-plane separation**: run platform addons (monitoring, logging, ingress) in dedicated node pools with taints so user workloads can't evict them.

### Traps
- **Default `ServiceAccount` token auto-mounted** → every pod has API creds. Set `automountServiceAccountToken: false` at namespace default.
- **Events flood etcd**. A crashloop pod emits events every few seconds; 1000 crashlooping pods = etcd pressure. Event TTL is 1h by default — keep it there, don't bump it.

---

## 2. Stateful Service (Database / Kafka / Elasticsearch)

### Scenario
Running Postgres or Kafka on Kubernetes. One node failure must not lose data or cause extended unavailability.

### Design
- **StatefulSets + stable network identity**: `pod-0`, `pod-1`, `pod-2`. Combined with a headless Service, gives DNS names that outlive pod recreations.
- **PersistentVolumes with `volumeBindingMode: WaitForFirstConsumer`**: ensures the PV is provisioned in the AZ the pod lands in. Without this, pod gets scheduled in AZ-a but its EBS volume is in AZ-b → pending forever.
- **Pod topology spread constraints**: 3 replicas must be in 3 different AZs. Not just anti-affinity — explicit `maxSkew: 1` across `topology.kubernetes.io/zone`.
- **PodDisruptionBudget** of `maxUnavailable: 1` (or `minAvailable: N-1`). Prevents a node drain from taking down quorum.

### Reliability — the non-obvious parts
- **Use an operator** (Strimzi for Kafka, CloudNativePG for Postgres, ECK for Elasticsearch). Raw StatefulSets don't understand leader election, failover, or backup semantics.
- **Don't share etcd with stateful workloads** — heavy clients (watch-heavy controllers) can starve etcd. 
  Stateful platforms usually ship their own consensus (Raft in etcd-for-Kafka, Zookeeper→KRaft, Patroni for PG).
- **Backups live outside the cluster** — Velero for resources, native tools (pg_basebackup, Kafka MirrorMaker2) for data. A cluster losing etcd can rebuild apps but not data.

### Traps
- **`kubectl delete pvc` cascades silently**. `reclaimPolicy: Retain` on production StorageClass so a fat-fingered delete doesn't vaporize data.
- **Rolling update of a stateful leader**: operator-managed graceful leader handoff is required; naive StatefulSet rollout ping-pongs leadership.
- **Node upgrades evict the leader mid-transaction**. Use `terminationGracePeriodSeconds` long enough for graceful shutdown (often 60–300s for DBs) and PDB.

---

## 3. High-RPS Stateless Web Tier

### Scenario
Edge-facing API serving 50K RPS, P99 SLO 100 ms. Zero-downtime deploys; graceful handling of traffic spikes.

### Design
- **HPA with custom metrics**, not just CPU. RPS or queue-depth is what actually correlates to load. Prometheus Adapter or KEDA.
- **Pre-warmed pools**: HPA `behavior` with `scaleUp.stabilizationWindowSeconds: 0` + `scaleDown` window of 300s — scale up aggressively, scale down slowly to avoid flap.
- **Readiness probe != liveness probe**.
  - Readiness: remove from Service endpoints when unhealthy (don't kill).
  - Liveness: kill and restart (only for stuck processes — overly aggressive liveness is a common incident root cause).
- **Startup probe** for slow-booting apps (JVM). Prevents liveness from killing during warmup.

### Zero-downtime deploy
- **Rolling update strategy** with `maxSurge: 25%, maxUnavailable: 0`.
- **`preStop` hook**: `sleep 15` before SIGTERM. Reason: Service endpoints update is eventually consistent — kube-proxy might still be routing to the pod for a few seconds after it's marked Terminating. Without preStop sleep, in-flight requests 502.
- **App handles SIGTERM**: stop accepting new connections, finish in-flight ones, then exit before `terminationGracePeriodSeconds`.

### Traffic spikes (flash sale, viral event)
- **HPA + Karpenter stack**: HPA adds pods → pending → Karpenter adds nodes in ~60s. For spikes faster than 60s you need **over-provisioning pods** (low-priority pause pods that get evicted to make room for real workload) or **warm pool** nodes.
- **Rate limiting upstream** (ingress / API gateway), not just autoscaling. Autoscale has a lag; rate limit has none.

### Traps
- **CPU throttling**: setting `limits.cpu` below what the app actually needs causes CFS throttling at the 100ms quota boundary — latency spikes invisible to CPU% metrics. 
  - Either remove CPU limits on latency-sensitive services or set them well above P99 usage. (Memory limits, by contrast, always set — OOMKill is cleaner than swap/leak.)
- **SO_REUSEPORT across pods isn't a thing**. Two pods don't share a port; Service does the load balancing — understand your kube-proxy mode (iptables vs IPVS vs eBPF/Cilium).

---

## 4. Batch / ML Training Workloads

### Scenario
Nightly training jobs, 1000-GPU cluster, jobs take 4–24 hours. Cost and throughput > latency.

### Design
- **Jobs / CronJobs**, not Deployments.
- **Kueue / Volcano / Argo Workflows** for queueing, gang-scheduling, and fair-share across teams. Raw Jobs don't queue — 1000 pending pods hammer the scheduler.
- **Priority classes**: training jobs are `batch` (preemptible), inference is `critical`.
- **Spot instances with checkpointing**: save progress every N minutes to S3; on interruption, restart from last checkpoint. 70% cost savings.
- **GPU time-slicing / MIG** for small models sharing a GPU.

### Reliability
- **`backoffLimit` + `activeDeadlineSeconds`** on Jobs. Infinite retries from a poisoned pill is a common cost incident.
- **Idempotent workers**: re-running a failed task must not corrupt state.

### Traps
- **Pod eviction during training**: `tolerations` for spot-termination taint, `priorityClass` high enough to avoid eviction. But **don't** set to `system-cluster-critical` — that'll starve real system pods.

---

## 5. Multi-Cluster / Multi-Region

### Scenario
99.99% availability, survive a region failure, data residency in EU.

### Design
- **One cluster per region** — never span clusters across regions. Latency, etcd replication, and blast radius all say no.
- **Global load balancing**: Route53 latency routing, Cloudflare, or AWS Global Accelerator. Health checks pull the region out of rotation on failure.
- **Cluster API / Crossplane** for uniform cluster provisioning. Hand-crafted clusters drift.
- **GitOps (ArgoCD / Flux) with ApplicationSet**: single manifest renders across N clusters. Rollouts are staged (canary region first).
- **Service mesh (Istio multi-cluster / Linkerd multi-cluster / Cilium Cluster Mesh)** for cross-cluster service discovery and mTLS. Don't roll your own.

### Reliability
- **Blast radius containment**: a bad config change deployed globally in minutes is the textbook multi-region outage. Always stage: 1 cluster → 10% → 50% → 100%, with bake time + automated rollback on SLO burn.
- **Data tier separate from compute tier**: DBs and queues are often global services (Aurora Global, Kafka MirrorMaker) — don't try to run them via k8s federation.

### Traps
- **"Federation v2" / KubeFed is dead**. Use mesh + GitOps. Don't try to have "one control plane to rule them all" — the failure mode is all clusters at once.
- **DNS-based failover has TTL latency**. Clients cache. Budget minutes for DNS-only failover; seconds only with Global Accelerator or Anycast.

---

## 6. Upgrade Strategy (The Forgotten Reliability Risk)

### Scenario
Kubernetes ships a minor version every ~4 months; each version is supported ~1 year. You're always upgrading.

### Design
- **N-1 rule**: stay one minor behind latest. Ahead = bleeding; farther behind = migration cliff.
- **Upgrade cadence**: quarterly minor, monthly patches. Make it routine, not a project.
- **Staged rollout**: dev → stg → canary prod cluster → rest of prod. Soak 1 week between stages.
- **API deprecation tooling**: `kubectl convert`, `pluto`, `kube-no-trouble` to find v1beta1 resources before they break on upgrade.

### Reliability
- **Surge upgrades** on managed node pools (EKS, GKE): spin up new nodes, drain old, never below capacity.
- **Control plane first, then workers**. Skew of ≤2 minors is supported; never upgrade workers ahead of control plane.

### Traps
- **Stuck PDBs block drains**. A PDB requiring `minAvailable: 100%` with no surge blocks eviction forever. Audit PDBs before upgrade.
- **Finalizers on deleted namespaces**. A namespace stuck Terminating holds PVCs and Services. Investigate the finalizer owner rather than force-removing (you can easily orphan cloud resources that keep billing).

---

## 7. Observability & Debugging at Scale

### Design
- **Metrics**: Prometheus federation or Thanos/Mimir for multi-cluster, long-term retention. Don't try to keep 90d of raw metrics in a single Prom.
- **Logs**: Loki or ELK with hot/warm/cold tiers. Never ingest DEBUG logs in prod.
- **Traces**: OpenTelemetry collector as DaemonSet, head-based sampling (1–5%) + tail-based for errors.
- **USE + RED** dashboards per service, standardized. If every team rolls its own, you can't compare.
- **SLOs + error budgets** (Sloth, OpenSLO) — the single most important signal for on-call. Raw latency graphs don't tell you when to page.

### Debugging anti-patterns
- **`kubectl exec` on prod pods** to debug is a crutch. Prefer ephemeral debug containers (`kubectl debug`) with proper tooling images. Treat pods as cattle.
- **Live-editing resources** (`kubectl edit`) in prod bypasses GitOps — drift + audit loss. Use `kubectl diff` against git state as a routine.

---

## 8. Security (Non-Negotiable at Scale)

- **RBAC** with least-privilege ServiceAccounts per workload. No `cluster-admin` for apps, ever.
- **IRSA / Workload Identity** (AWS/GCP): pod → cloud IAM without static keys. Rotate-by-design.
- **Secrets**:
  - Not in plain Secrets (base64 ≠ encryption). Use external secrets operator backed by Vault / AWS Secrets Manager.
  - Enable etcd encryption at rest — unencrypted-at-rest etcd is a compliance red flag.
- **Image scanning + admission control**: Trivy / Snyk in CI, Gatekeeper policy rejecting `latest` tag and CVEs above threshold.
- **Runtime security**: Falco / Tetragon for syscall-level detection (crypto miner, reverse shell).
- **Audit logging** piped to a SIEM. `kube-apiserver --audit-policy` at `Metadata` level minimum.

---

## 9. Cross-Cutting Best Practices

### Workload hygiene
- Always set **requests**; set **memory limits** always, **CPU limits** rarely (see trap in §3).
- **3 replicas minimum** for prod; 1 replica = you have no HA.
- **Topology spread** across AZs for everything stateful or user-facing.
- **PDB on every Deployment/StatefulSet** — `minAvailable: N-1` typically.
- **Liveness probes only for stuck states**; readiness for everything.
- **Pod anti-affinity** (soft) to discourage co-location on the same node.

### Traffic management
- **Ingress** + cert-manager for TLS. One controller per cluster (NGINX, Envoy-based like Contour or Istio Gateway).
- **Gateway API** over Ingress for new work — richer semantics, multi-tenant safer.
- **Service mesh** is worth it at ~50+ services (mTLS, retries, traffic splitting); below that, overhead outweighs benefit.

### Deployment
- **GitOps (ArgoCD / Flux)** over imperative `kubectl apply`. Cluster state == git state, always.
- **Helm or Kustomize**, not both for the same thing. Kustomize is simpler; Helm is necessary when you need parametric templates or ecosystem charts.
- **Progressive delivery**: Argo Rollouts or Flagger for canary/blue-green with metric-based promotion.
- **Pin image digests** (`@sha256:...`), not tags, in prod. Tag mutability is a supply chain vector.

### Capacity & cost
- **Karpenter + Spot** for stateless, on-demand for stateful.
- **VPA in `recommendation` mode** to right-size `requests` (don't let VPA and HPA both auto-act on CPU — they fight).
- **Cluster consolidation**: 3 clusters of 100 nodes beats 30 clusters of 10 — amortizes control plane cost and operational overhead.
- **Descheduler** periodically rebalances to drain underused nodes.

---

## 10. The 10 Things Staff Engineers Insist On

1. **PDBs on everything.** Drains without PDBs are incidents waiting to happen.
2. **Requests set accurately, CPU limits off for latency services.** CFS throttling is silent murder.
3. **Readiness ≠ liveness.** Misusing liveness causes more outages than it prevents.
4. **GitOps, not kubectl.** If it's not in git, it doesn't exist.
5. **One cluster per region, mesh for cross-region.** No federated kube-apiserver dreams.
6. **Operators for stateful workloads.** Don't DIY DB HA on StatefulSets.
7. **Namespace-level quotas + NetworkPolicies** from day one, retrofitting is painful.
8. **Upgrade constantly, never let yourself get 3 versions behind.** Migration cliffs hurt.
9. **SLOs drive paging, not raw metrics.** Error budget or bust.
10. **Graceful shutdown is a feature.** preStop + SIGTERM handling + terminationGracePeriod tuned per app.

---

## Quick Decision Reference

| Problem | Reach for |
|---|---|
| Stateful workload | Operator + StatefulSet + topology spread + Retain PV |
| Traffic spike | HPA custom metrics + Karpenter + rate limit + over-provisioning |
| Zero-downtime deploy | Rolling update + preStop sleep + SIGTERM handler + readiness |
| Multi-region HA | Cluster per region + mesh + global LB + GitOps staged rollout |
| Cost pressure | Karpenter + Spot + VPA recommend + bin-packing |
| Noisy neighbor | ResourceQuota + LimitRange + PriorityClass + NetworkPolicy |
| Batch/ML | Kueue/Volcano + Spot + checkpointing + activeDeadline |
| Secrets | External Secrets Operator + IRSA/Workload Identity + etcd encryption |
| Debugging | kubectl debug + OpenTelemetry + SLO dashboards, not kubectl exec |
| Upgrade | N-1 version + surge nodes + staged rollout + pluto/kubent |