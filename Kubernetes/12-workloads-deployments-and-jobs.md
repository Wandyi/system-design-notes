# 12 · Workloads — Deployments, StatefulSets, DaemonSets, Jobs

The user-facing resources that own Pods. Each has different semantics and corner cases.

## 12.1 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: web}
spec:
  replicas: 3
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      containers: [{name: nginx, image: nginx:1.27}]
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  revisionHistoryLimit: 10
  progressDeadlineSeconds: 600
```

### The ownership chain

```
   Deployment
       │ owns (via pod-template-hash label)
       ▼
   ReplicaSet (one per PodTemplate revision)
       │ owns
       ▼
   Pods
```

The Deployment controller computes a hash of `spec.template` → creates a new ReplicaSet with that hash → scales up new, scales down old per `strategy`.

Old ReplicaSets are retained (count `revisionHistoryLimit`) for rollback:

```bash
kubectl rollout history deployment/web
kubectl rollout undo deployment/web --to-revision=2
```

### Update strategies

| Strategy | Behavior |
|----------|----------|
| **RollingUpdate** | Gradual; bounded by maxSurge + maxUnavailable |
| **Recreate** | Kill all then start; downtime; needed for ROX/RWX volumes with single-writer apps |

### MaxSurge / MaxUnavailable

```
replicas: 10
maxSurge: 25% = 2.5 → 3 extra pods during update
maxUnavailable: 25% = 2.5 → 3 pods may be unavailable

So during update: between 8 (10 - 2) and 13 (10 + 3) pods running.
```

Common tuning:
- `maxSurge: 0`: never run more than `replicas`. Slower but constant footprint.
- `maxUnavailable: 0`: never have less than `replicas` ready. Requires extra capacity.
- `25% / 25%`: balanced default.

### progressDeadlineSeconds

If rollout doesn't make progress (no new pods become ready) for this duration, deployment is marked Failed:
```
status:
  conditions:
  - type: Progressing
    status: "False"
    reason: ProgressDeadlineExceeded
```

Default 600s.

## 12.2 StatefulSet

For workloads that need:
- **Stable, unique pod names** (`web-0`, `web-1`, `web-2`).
- **Stable DNS** via headless Service (`web-0.svc.namespace.svc.cluster.local`).
- **Ordered, controlled rollout** (one pod at a time by default).
- **Persistent volumes per pod** (via volumeClaimTemplates).

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata: {name: web}
spec:
  serviceName: web   # headless Service for DNS
  replicas: 3
  selector: {matchLabels: {app: web}}
  template:
    metadata: {labels: {app: web}}
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts: [{name: data, mountPath: /data}]
  volumeClaimTemplates:
  - metadata: {name: data}
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: gp3
      resources: {requests: {storage: 10Gi}}
  podManagementPolicy: OrderedReady   # OrderedReady | Parallel
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
      maxUnavailable: 1   # 1.27+
```

### Pod identity

Pod 0 starts first; once ready, pod 1 starts; etc. Deletion in reverse order. PVCs persist when pod is deleted (so when re-created, pod-N gets the same data).

### Partition

`partition: N` updates only pods with index ≥ N. Used for canary:
```
partition: 2 → pod 2 (latest) updated; 0 and 1 stay on old.
```

### MaxUnavailable (1.27+ GA)

Allows parallelism in update: `maxUnavailable: 2` → 2 pods can update at once.

### Headless Service requirement

`spec.serviceName` references a Service with `clusterIP: None`. CoreDNS resolves each pod by name. Without this, StatefulSet still works but pods aren't DNS-discoverable individually.

### When NOT to use StatefulSet

- App doesn't need stable identity — use Deployment.
- App needs RWX shared storage (use Deployment with shared PVC).
- Pure batch — use Job.

### PVC retention policy (1.27+)

```yaml
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain   # or Delete
    whenScaled: Retain    # or Delete
```

Default both Retain — manual cleanup. `Delete` makes scale-down also drop PVC (and underlying PV per reclaim policy).

## 12.3 DaemonSet

Runs one pod per matching node:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata: {name: fluentd}
spec:
  selector: {matchLabels: {app: fluentd}}
  template:
    metadata: {labels: {app: fluentd}}
    spec:
      nodeSelector: {kubernetes.io/os: linux}
      tolerations:
      - operator: Exists   # tolerate everything; useful for system DaemonSets
      containers:
      - name: fluentd
        image: fluentd:v1.16
        resources: {requests: {cpu: 100m, memory: 200Mi}}
```

Use cases:
- Log collectors (fluentd, fluent-bit).
- Node agents (Node exporter, kube-proxy, CSI drivers).
- Network plugins (Cilium, Calico).
- Storage daemons (Ceph OSDs in rook-ceph).

### Tolerations

DaemonSets typically tolerate everything (so they run on all nodes including tainted ones).

### Update strategy

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1   # only 1 pod at a time
      maxSurge: 0          # 1.25+; can use extra capacity briefly
```

OnDelete strategy: pods only updated when manually deleted (rare; for highly-controlled rollouts).

### DaemonSet pods are scheduled by the default scheduler

(Since KEP-2535, GA 1.17.) DaemonSet controller creates pods with `spec.nodeName` set via affinity; scheduler binds. Same scheduler features apply (tolerations, affinity).

## 12.4 Job

Runs pods until N completions:

```yaml
apiVersion: batch/v1
kind: Job
metadata: {name: import-data}
spec:
  parallelism: 5      # at most 5 concurrent
  completions: 10     # need 10 successful completions
  backoffLimit: 6     # max 6 failures across all retries
  activeDeadlineSeconds: 600
  ttlSecondsAfterFinished: 100  # auto-cleanup
  template:
    spec:
      restartPolicy: OnFailure   # or Never
      containers: [{name: worker, image: my-job}]
```

### completionMode

- **NonIndexed** (default): any pod can complete the next; useful for embarrassingly-parallel work.
- **Indexed**: each completion has an index (0..completions-1) via `JOB_COMPLETION_INDEX` env var; useful for sharded work (e.g., process partition N).

### Pod failure policy (1.26+ GA)

```yaml
spec:
  podFailurePolicy:
    rules:
    - action: FailJob
      onExitCodes: {operator: In, values: [42]}   # exit 42 = unrecoverable; fail
    - action: Ignore
      onExitCodes: {operator: In, values: [137]}   # SIGKILL = retry doesn't help
    - action: Count
      onPodConditions: [{type: DisruptionTarget}]   # only count evictions toward backoff
```

Replaces the binary "any failure counts" model.

### CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata: {name: backup}
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid   # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 200
  jobTemplate:
    spec:
      template:
        spec:
          containers: [...]
```

The CronJob controller creates one Job per scheduled time. `concurrencyPolicy`:
- **Allow**: jobs can overlap.
- **Forbid**: skip new if previous still running.
- **Replace**: kill running, start new.

`startingDeadlineSeconds`: if controller is restarted, skip jobs that should have started more than N seconds ago.

## 12.5 ReplicaSet — usually you don't create directly

The Deployment controller manages ReplicaSets for you. Direct creation is rare:
- For something with no rollout (a one-off set of N pods).
- For low-level testing.

## 12.6 The init container pattern

```yaml
spec:
  initContainers:
  - name: setup
    image: alpine
    command: ["sh", "-c", "echo ready > /shared/status"]
    volumeMounts: [{name: shared, mountPath: /shared}]
  containers:
  - name: app
    image: my-app
    volumeMounts: [{name: shared, mountPath: /shared}]
  volumes: [{name: shared, emptyDir: {}}]
```

Init containers run sequentially before app containers. Each must succeed (exit 0) before next.

Common uses:
- Wait for dependency (db reachable).
- Initialize config.
- Pull secrets.

Failure of init container → pod fails (and is retried per restartPolicy).

## 12.7 Sidecar containers (KEP-753, GA 1.29)

Pre-1.29: sidecars were just "app containers"; lifecycle issues (sidecar starts after app; sidecar doesn't terminate before app; etc.).

Post-1.29: native sidecar containers (init containers with `restartPolicy: Always`):

```yaml
spec:
  initContainers:
  - name: logging-agent
    restartPolicy: Always   # makes this a sidecar
    image: fluent-bit
  containers:
  - name: app
    image: my-app
```

Semantics:
- Sidecar starts before app containers.
- App can rely on sidecar being ready.
- On pod termination, app stops first; sidecars after.

Used by service meshes (Istio's holdApplicationUntilProxyStarts), log shippers.

## 12.8 Pod disruption budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: {name: my-app-pdb}
spec:
  minAvailable: 80%   # OR maxUnavailable: 20%
  selector: {matchLabels: {app: my-app}}
```

Constrains **voluntary** evictions:
- `kubectl drain` for node maintenance.
- Cluster Autoscaler / Karpenter scale-down.
- Descheduler.

Eviction API checks PDB; if would violate, returns 429 + Retry-After.

NOT respected by: kubelet evictions (OOM, disk pressure), preemption (well, preemption tries to respect but can override for high-priority pods), node failure.

## 12.9 Pod priority and preemption

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: {name: high-priority}
value: 1000000
preemptionPolicy: PreemptLowerPriority   # or Never
```

Pods with priorityClassName get scheduled before lower-priority pods. If no node has room, high-priority pod can preempt lower-priority pods (subject to PDB).

Built-in: `system-cluster-critical` (2000000000), `system-node-critical` (2000001000).

## 12.10 Resource requests and limits

```yaml
spec:
  containers:
  - resources:
      requests:
        cpu: "500m"
        memory: "256Mi"
      limits:
        cpu: "1000m"
        memory: "512Mi"
```

### Requests
- Used by scheduler to find a node with enough capacity.
- Translated to cgroup `cpu.shares` (CPU) and `memory.min` (memory, partial).
- Pod's QoS class derives from request/limit combination.

### Limits
- Cgroup max: `memory.max` (memory; OOMKill if exceeded) and CFS throttling (CPU; soft).
- Not enforced for ephemeral-storage by default (unless feature gate enabled).

### QoS

| Class | When | Eviction priority |
|-------|------|-------------------|
| **Guaranteed** | requests == limits for cpu+memory of every container | Last evicted |
| **Burstable** | At least one container has requests set; not Guaranteed | Middle |
| **BestEffort** | No requests/limits | First evicted |

### Memory limit gotcha

Container that exceeds memory limit gets OOMKilled (signal SIGKILL). Pod's restartPolicy decides what's next (restart container; Job: counts toward backoff).

### CPU limit gotcha

CFS throttling causes microsecond-scale pauses. For latency-sensitive apps (HTTP server hitting p99 SLO), set requests but **not** limits → app can burst when needed; node-level cgroup keeps overall consumption bounded.

## 12.11 In-place pod resize (KEP-1287, beta 1.27+)

```yaml
# 1.27+ alpha; 1.32 (estimated) GA
spec:
  containers:
  - resources:
      requests: {cpu: "1", memory: "1Gi"}
      limits:   {cpu: "1", memory: "1Gi"}
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired   # change without restart
    - resourceName: memory
      restartPolicy: RestartContainer
```

Then:
```bash
kubectl patch pod my-pod --patch '{"spec":{"containers":[{"name":"app","resources":{"limits":{"cpu":"2"}}}]}}'
```

Without pod recreation. Game-changer for stateful workloads.

## 12.12 Pod lifecycle phases

```
   Pending  → ContainerCreating → Running → Succeeded
                                          → Failed
                                          → Unknown
```

- **Pending**: pod accepted but not yet scheduled, or images pulling.
- **Running**: at least one container started.
- **Succeeded**: all containers exit 0 (terminal).
- **Failed**: at least one container exited non-zero or was killed by system (terminal).
- **Unknown**: kubelet unreachable.

Conditions (multiple, on Pod.Status.Conditions):
- **PodScheduled** — scheduler bound.
- **Initialized** — init containers complete.
- **ContainersReady** — all (non-init) containers ready.
- **Ready** — pod ready (per readiness probe).
- **DisruptionTarget** — pod about to be deleted by a disruption (eviction, preemption).

## 12.13 Common interview probes

- **"Walk me through a Deployment rollout."** Deployment ctrl computes new RS hash; creates new RS; scales up new + down old per maxSurge/maxUnavailable; tracks revisions in RS history.
- **"Why use StatefulSet over Deployment?"** Stable identity + DNS + PVCs per pod + ordered rollout. Required for distributed databases, etc.
- **"What's a DaemonSet good for?"** Per-node agents: log collector, network plugin, monitoring exporter.
- **"How would you handle a CronJob that's running too long?"** activeDeadlineSeconds; ttlSecondsAfterFinished; concurrencyPolicy: Forbid; alert on duration.
- **"How does Init Container differ from sidecar?"** Init runs sequentially before app; sidecar (1.29+) runs alongside app for its lifetime.
- **"What's a PDB and what does it block?"** Voluntary eviction (drain, scale-down). Not OOM, not kubelet eviction.

## 12.14 Corner cases

- **Two Deployments with overlapping selectors** — they fight over pods; non-determinism. Don't.
- **StatefulSet rollback** — manually decrement `partition`, or apply old spec, or `kubectl rollout undo`.
- **Pod stuck in Terminating** — finalizer not removed; or kubelet unreachable. `kubectl delete --force --grace-period=0` for emergencies.
- **DaemonSet bypassing maintenance** — tolerations make it run everywhere; drain doesn't evict DaemonSets (their pods restart).
- **CronJob misses runs** — startingDeadlineSeconds; controller down too long. Use Job alternatives (Argo Workflows) for missed-run guarantees.
- **Job retries infinitely** — backoffLimit reached but new Pod created? Check controller logs; might be a stuck condition.
- **Pod gets evicted by disk pressure** — kubelet evicts BestEffort first; check `kubectl describe node` conditions.

## 12.15 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Stateless web service | Deployment | rollouts via Argo Rollouts | manual ReplicaSet (rare) |
| Stateful database | StatefulSet + operator | DBaaS (RDS) outside k8s | DaemonSet (rare) |
| Per-node agent | DaemonSet | static pod | systemd unit (outside k8s) |
| Periodic task | CronJob | Argo Workflows scheduled | external scheduler + Job |
| One-off task | Job | bare pod | kubectl run |
| Canary rollout | Deployment + Service (DNS RR) | Argo Rollouts (canary) | service mesh weighted routing |
| Blue-green | two Deployments + Service switch | Argo Rollouts (blueGreen) | manual |
| Pod identity | StatefulSet | manual SA + pod-name | operator-managed |

## Must-internalize

- Deployment → ReplicaSet (one per template hash) → Pods.
- RollingUpdate: maxSurge + maxUnavailable; progressDeadlineSeconds.
- StatefulSet: stable identity, headless Service, volumeClaimTemplates, OrderedReady (default).
- DaemonSet: per-node; tolerates everything for system agents.
- Job: parallelism + completions + backoffLimit + ttlSecondsAfterFinished.
- CronJob: schedule + concurrencyPolicy (Forbid/Allow/Replace).
- Init container = sequential pre-start; Sidecar (1.29+ KEP-753) = parallel lifecycle.
- QoS: Guaranteed > Burstable > BestEffort (eviction order).
- CPU limits cause CFS throttling (tail latency); often skipped for latency-sensitive.
- PDB blocks voluntary evictions; not kubelet OOM.
- In-place resize (1.27+ beta) avoids pod recreate.
