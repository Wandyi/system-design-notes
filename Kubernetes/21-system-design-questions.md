# 21 · System Design Questions

12 prompts that have appeared in staff-level Kubernetes design rounds, with answer sketches. Each is a 45-60 minute round. The signal isn't the "right answer" — it's the *structured walk* through scope, components, scale, failure modes, trade-offs.

## 21.1 Design a custom scheduler for batch workloads

**Prompt**: support gang scheduling (all pods of a job land together or none) at 5000-node scale.

**Walk**:
1. **Scope**: gang = pod group; either all schedulable or none. Group sizes 10s-1000s.
2. **Path choice**: scheduling framework plugin OR separate scheduler. Separate scheduler is cleaner here (different model).
3. **Architecture**:
   - PodGroup CRD (or use Volcano's): groups pods by label + count.
   - Scheduler watches Pods with `schedulerName: gang-sched`.
   - Permit plugin "wait" until all members of the group reservable.
   - When ready, release all via Permit "Allow".
   - On failure, requeue the whole group.
4. **Cache**: assignmentCache tracks tentative reservations.
5. **Preemption**: low-priority gang jobs can be preempted entire-group.
6. **Failure modes**:
   - Partial schedule (some bound before group complete) → rollback via Unbind.
   - Leader-replica failover loses cache → re-init from apiserver state.
7. **Scale**: scheduling cycle latency must stay <1s per group.

**References**: Volcano (CNCF), Yunikorn, kube-batch.

## 21.2 Design a CRD for a multi-region database

**Prompt**: model a "DistributedDB" CR that runs replicas across 3 regions, with leader election, backup, and tunable replication lag SLO.

**Walk**:
1. **Spec**: regions, replicaCount per region, instanceType, replicationLagSLO, backupSchedule, version.
2. **Status**: per-region replica states, current leader, replication lag, snapshot info, conditions.
3. **Subresources**: `status`; maybe `scale` (target replicaCount).
4. **Validation (OpenAPI + CEL)**:
   - regions non-empty, distinct.
   - replicaCount >= 1 per region.
   - replicationLagSLO between 0 and reasonable max.
   - storageClass exists.
5. **Conversion webhook**: only if multi-version; v1 sufficient.
6. **Finalizer**: clean up cloud resources (snapshots, VPC peering) on delete.
7. **Reconcile**:
   - Per-region StatefulSet + PVC.
   - Leader pod: a Service + annotation + status.
   - Cross-region replication: peering, cert exchange, replication conn strings.
   - Backup CronJob.
8. **Multi-cluster**: hub-and-spoke; Karmada or custom; one cluster per region.
9. **Failure**:
   - Region down → leader election promotes.
   - PVC deleted → operator recreates StatefulSet pod, app re-replicates from peers.
10. **Observability**: per-region replicas as Pod conditions; replicate lag as Prometheus metric.

## 21.3 Design a Pod auto-restart-on-secret-rotation system

**Prompt**: when a Secret used by Pods is updated, restart those Pods.

**Walk**:
1. **Why**: env-var-injected secrets don't auto-refresh; cert rotation needs pod restart.
2. **Approach**:
   - Annotation on Deployment lists Secrets to watch.
   - Reloader operator (existing OSS) watches ConfigMap/Secret changes.
   - On change, patch Deployment's pod template annotation (`reloader.stakater.com/last-reloaded`) to trigger rollout.
3. **Spec**:
   ```yaml
   metadata:
     annotations:
       reloader.stakater.com/auto: "true"
   ```
4. **Reconcile**:
   - Watch Secret events.
   - Find Deployments with annotation pointing to changed Secret.
   - Patch pod template annotation → rolling restart.
5. **Failure modes**:
   - Mass rotation → simultaneous rollouts → cluster overload. Stagger.
   - Bad Secret → all pods crash. PDB protection + dry-run validation.
6. **Alternative**: External Secrets Operator with `refreshInterval` syncs Secret; combined with Reloader.

## 21.4 Design a multi-tenant cluster

**Prompt**: 100 tenant teams sharing one cluster. Hard isolation, fair share, cost attribution.

**Walk**:
1. **Namespace per tenant**.
2. **RBAC**: each tenant gets a Role + RoleBinding to their namespace.
3. **NetworkPolicy**: deny cross-namespace; allow only same ns + kube-system DNS.
4. **ResourceQuota** per ns: requests cap.
5. **LimitRange**: per-pod defaults + max.
6. **PSA**: `restricted` on every tenant ns.
7. **PriorityClass**: tiers (gold, silver, bronze) — gold preempts silver.
8. **Cost attribution**: OpenCost / kubecost; per-ns reports.
9. **Hardening**:
   - No hostPath; no hostNetwork; no NET_ADMIN cap.
   - gVisor or Kata runtime for sensitive tenants (RuntimeClass).
10. **Failure modes**:
   - Tenant breaks own ns → only their ns affected.
   - Noisy neighbor across nodes → Pod Anti-affinity, dedicated nodes via taints.
11. **Limits**: ~1000 namespaces practical; beyond, multi-cluster.

## 21.5 Design a Pod startup acceleration system

**Prompt**: improve cold-start time from 60s to <5s for a fleet of ephemeral CI runners.

**Walk**:
1. **Bottleneck analysis**: image pull (40s), kubelet processing (5s), init container (10s), app start (5s).
2. **Image optimizations**:
   - Smaller image (distroless, alpine). Multi-stage build.
   - Pre-pull common images on every node via DaemonSet.
   - Image lazy loading (stargz / eStargz / nydus).
   - Image registry mirror in cluster.
3. **Kubelet**:
   - Tune `--registry-qps`, `--registry-burst`.
   - Faster CRI (containerd 2.x).
4. **CNI**:
   - Pre-allocate IPs (Cilium auto-direct).
5. **Init container removal**: bake init steps into image.
6. **Sidecar containers (KEP-753)**: proper ordering avoids retries.
7. **In-place pod resize (KEP-1287)**: avoid recreate when scaling.
8. **Hot pool**: pre-warmed Pods waiting in "available" state; KEDA scales from there.
9. **Numbers**: with stargz lazy-load + pre-warmed pool → <2s cold start.

## 21.6 Design a cluster-wide observability stack

**Prompt**: collect metrics, logs, traces from 1000-pod cluster; alert on SLOs; cost-aware.

**Walk**:
1. **Metrics**:
   - Prometheus Operator + ServiceMonitor.
   - Thanos sidecar; long-term in S3.
   - Recording rules to pre-aggregate.
   - Cardinality budget: <1M active series per Prometheus.
2. **Logs**:
   - Fluent-bit DaemonSet → Loki.
   - Structured logging (slog).
   - Retention: 30d hot, 90d cold.
3. **Traces**:
   - OpenTelemetry Collector DaemonSet.
   - Tempo backend.
   - Tail sampling: keep errors + slow requests.
4. **Events**:
   - Audit log → fluentd → SIEM.
   - Cluster events → metrics (kube-eventer).
5. **Alerts**:
   - SLO-based: error rate, latency p99, availability.
   - Burn-rate alerts (multi-window).
   - PagerDuty for critical; Slack for warning.
6. **Cost**:
   - Loki cheap (S3); Prometheus needs Thanos compaction.
   - Cardinality the main cost lever.
7. **Dashboards**: Grafana with curated set; per-team dashboards via Argo CD.
8. **Self-healing**: AlertManager → automated remediation (e.g., restart kubelet).

## 21.7 Design Kubernetes-native blue-green deployments

**Prompt**: zero-downtime production releases; instant rollback; pre-prod traffic.

**Walk**:
1. **Two Deployments**: `app-blue`, `app-green`. Same labels except for color.
2. **Two Services**: `app-svc` (active) + `app-preview` (preview, points to inactive).
3. **Cutover**:
   - Deploy green.
   - Run smoke tests against preview Service.
   - Patch active Service's selector to point at green.
   - DNS / connection drain — wait connect drain time.
   - Old blue stays as fallback.
4. **Rollback**: re-point active Service to blue.
5. **Rolling cleanup**: after N min validation, scale down old.
6. **Controller**: Argo Rollouts handles all this declaratively.
7. **Failure modes**: traffic gets stuck on stale Endpoints during selector flip; use sticky sessions or graceful drain.

## 21.8 Design a network policy enforcement system

**Prompt**: enforce "every Service can only talk to declared dependencies"; allow developers to declare; reject undeclared.

**Walk**:
1. **CRD**: `ServiceDependency`:
   ```yaml
   spec:
     source: my-service
     targets: [auth-svc, payment-svc]
   ```
2. **Operator** reconciles to k8s NetworkPolicy.
3. **Admission webhook** on Service create: warn if no dependency CR.
4. **Cluster default-deny** NetworkPolicy in every namespace.
5. **Observability**: Hubble (Cilium) flow events → analytics; "shadow" allow-but-log mode for migration.
6. **Iteration**: start in "audit" mode for 2 weeks; identify undeclared deps; flip to enforce.

## 21.9 Design a cost optimization platform

**Prompt**: reduce cluster cost by 40% without breaking SLO.

**Walk**:
1. **Measure**: OpenCost / kubecost — current per-ns spend.
2. **Right-size**:
   - VPA recommender (Off mode); compute "right" requests.
   - Find over-provisioned: cpu_idle / cpu_requested > 50% → reduce.
3. **Bin-pack**: NodeResourcesFit `mostAllocated` strategy; or Karpenter consolidation.
4. **Spot**:
   - Karpenter NodePool with mixed Spot + On-Demand.
   - Tolerate Spot interrupts.
   - Don't put control plane on Spot.
5. **Reserved Instances / Savings Plans** for baseline.
6. **Cluster Autoscaler / Karpenter** to scale down aggressively at off-hours.
7. **Per-team chargeback**: reports per ns.
8. **Hygiene**:
   - Orphan PVCs (PVC with no pod for 30d → delete).
   - Old Deployments (rollouts paused; never re-enabled).
   - Inflated requests (no actual usage).
9. **Validate SLO impact**: monitor latency p99 + error rate during; rollback if degraded.

## 21.10 Design a workload identity system

**Prompt**: each pod gets a unique cloud identity (IAM role) without manual config.

**Walk**:
1. **Cloud-side**: IAM role per ServiceAccount.
2. **k8s-side**: SA annotation `eks.amazonaws.com/role-arn` (IRSA pattern).
3. **Token**: projected SA token with audience matching IAM.
4. **AWS SDK in pod**: detects via env vars / file path; calls STS AssumeRoleWithWebIdentity; gets temp creds.
5. **Multi-cloud**: same pattern for GCP Workload Identity, Azure Workload Identity. SPIFFE/SPIRE for non-cloud.
6. **Policy**:
   - Each SA only what the pod needs (least privilege).
   - Webhook validates SA annotations match a registered role.
7. **Observability**: CloudTrail audit per role; metric of unused roles.

## 21.11 Design a chaos engineering platform

**Prompt**: scheduled fault injection in production; validate resilience.

**Walk**:
1. **CRD**: `ChaosExperiment` (pod-kill, network-delay, CPU-stress, etc.).
2. **Controller** (chaos-mesh, litmus): inject via privileged DaemonSet or sidecar.
3. **Scoping**: namespace + label selector to limit blast radius.
4. **Schedule**: nightly chaos; weekly chaos; off-hours.
5. **Safety guard**:
   - Approval workflow for production.
   - Hard stop on SLO breach.
   - Audit log per experiment.
6. **Verification**: pre/post Prometheus query — error rate stays below threshold.
7. **Tools**: Chaos Mesh (CNCF), Litmus, AWS FIS.

## 21.12 Design a kubelet replacement for edge

**Prompt**: run small-footprint k8s on edge devices (e.g., gateway nodes with 1GB RAM).

**Walk**:
1. **Use k3s or microk8s**: bundled kubelet + containerd + low memory.
2. **kubelet light-mode**: disable PLEG (use evented), reduce sync periods, skip metrics-server.
3. **CRI**: containerd is fine; CRI-O lighter.
4. **CNI**: flannel-vxlan (small), no IPVS, simple network.
5. **No etcd locally**: use kine + SQLite (k3s default) or remote etcd.
6. **Control plane**: at central data center; edge devices register via Cluster API.
7. **Disconnect tolerance**:
   - Local Pod cache; runs without apiserver for some time.
   - Resync on reconnect.
8. **Updates**: rolling, OTA, can rollback on health-check failure.
9. **Numbers**: k3s can run on 512MB Raspberry Pi; production edge typically 2GB+.

## 21.13 Common pitfalls in the design round

- **No scale numbers.** Always state assumptions: cluster size, pod count, qps.
- **No failure modes.** At least 3.
- **Designed around what k8s already has.** Yes, use Deployments, etc. — the staff signal is integrating + extending, not reinventing.
- **No observability.** Logs, metrics, alerts for the new design.
- **No on-call.** What pages? What's the runbook?
- **No security.** RBAC, NetworkPolicy, PSA, secrets handling.
- **Single point of failure.** Operator pod down → system down? Add leader election + replicas.

## 21.14 The 60-second pitch on a k8s system design

> "Start with scope (cluster size, workload type, scale). Then identify if it's a control-plane extension (CRD + operator), a data-plane extension (CNI plugin, CSI driver), an admission rule (webhook or VAP), or a workload pattern (Deployment + Service + autoscaling). For CRDs: walk through schema, validation, finalizer, status, RBAC. For operators: informer pattern, reconcile, multi-cluster strategy. For admission: failure policy, scope, performance budget. Name 3 failure modes (e.g., webhook down, finalizer stuck, race conditions) and how each is detected + recovered. End with observability (metrics, logs, audit) and cost. The grader is watching for *whether you understand the k8s primitives well enough to build on them* rather than reinventing them."

## Must-internalize

- The 12 prompts; have a 5-minute outline of each.
- Always: scope numbers, components, scale per box, 3 failure modes, observability, security.
- For CRD designs: schema + validation + finalizer + status + RBAC.
- For operators: informer + reconcile + leader-election + sharding.
- For admission: failure policy + scope + perf budget.
- For multi-cluster: hub-and-spoke; per-cluster standalone; federation as value-add.
- For cost optimization: measure first, right-size, bin-pack, Spot, consolidation.
