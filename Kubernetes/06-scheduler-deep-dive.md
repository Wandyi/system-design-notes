# 06 · Kubernetes Scheduler Deep Dive

`kube-scheduler` is the controller that watches Pods with `spec.nodeName == ""` and assigns them to nodes. The most algorithmically interesting component of the control plane.

## 6.1 What the scheduler does, in 30 seconds

```
   Pod created (nodeName=="")
                       ↓
   Scheduler informer sees it → push to PriorityQueue
                       ↓
   Scheduling cycle:
       PreFilter → Filter → PostFilter → PreScore → Score → Reserve → Permit → Bind
                       ↓
   Binding cycle (asynchronous):
       PreBind → Bind → PostBind
                       ↓
   apiserver writes pod.spec.nodeName via /binding subresource
```

## 6.2 The scheduling framework

K8s 1.19+ unified the old "predicates + priorities" model into a **plugin framework**. Each extension point is a list of plugins; plugins implement one or more.

| Extension point | When | Purpose |
|-----------------|------|---------|
| **QueueSort** | Pod added to queue | Compare pods for queue priority |
| **PreFilter** | Before filtering | Compute pod-wide state (e.g., resource requests sum) |
| **Filter** | Each node | Is this node feasible? (Hard predicates) |
| **PostFilter** | After filter, if no feasible | Try preemption to make room |
| **PreScore** | Before scoring | Compute pod-wide score state |
| **Score** | Each feasible node | Score the node (0..100) |
| **NormalizeScore** | After all scores | Normalize per-plugin |
| **Reserve** | After picking | Reserve resources optimistically |
| **Permit** | Before bind | Approve or wait/reject (used by gang scheduling) |
| **PreBind** | Before bind | E.g., wait for PVC volume attach |
| **Bind** | Write binding | The actual API call |
| **PostBind** | After bind | Cleanup / metrics |

## 6.3 Built-in plugins (the important ones)

### Filter plugins (check feasibility)

- **NodeName**: pod's `spec.nodeName` matches.
- **NodeAffinity**: node selector / affinity.
- **NodePorts**: node has free host port.
- **NodeUnschedulable**: skip nodes with `spec.unschedulable=true` (cordoned).
- **NodeResourcesFit**: requested CPU/memory/storage fits on node.
- **PodTopologySpread**: TopologySpreadConstraints satisfied.
- **VolumeBinding**: PVCs can bind to a PV on this node (or dynamic provisioner can).
- **VolumeZone**, **VolumeRestrictions**: zone affinity for volumes.
- **TaintToleration**: pod tolerates node's taints.
- **PodAffinity**: affinity/anti-affinity rules to other pods.

### Score plugins (rank feasible nodes)

- **NodeResourcesFit** (Score): leastAllocated / mostAllocated / balancedAllocation strategy.
- **NodeAffinity** (Score): preferred affinities boost score.
- **InterPodAffinity** (Score): preferred pod affinity/anti-affinity.
- **TaintToleration** (Score): preferred toleration matching.
- **PodTopologySpread** (Score): even spread across zones/nodes.
- **ImageLocality**: nodes that already have the image score higher.
- **NodeResourcesBalancedAllocation**: prefer nodes with balanced CPU/mem fill.

## 6.4 Scheduling configuration (KubeSchedulerConfiguration)

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  plugins:
    score:
      disabled:
      - name: ImageLocality
      enabled:
      - name: NodeResourcesFit
        weight: 5
      - name: NodeResourcesBalancedAllocation
        weight: 1
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: LeastAllocated
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1
```

Multiple profiles allow different scheduler behaviors selectable via `pod.spec.schedulerName`.

## 6.5 The priority queue

Scheduler maintains a **priority queue** of pending pods:

- **activeQ**: ready-to-schedule pods.
- **podBackoffQ**: pods that failed to schedule recently; back off before retry.
- **unschedulablePods**: pods deemed unschedulable; revisited on cluster events.

Cluster events (new node added, taint removed, etc.) trigger pods to move from `unschedulablePods` back to `activeQ`.

QueueSort plugin compares pods (default: by PriorityClass, then queueing timestamp).

## 6.6 Preemption — when to evict to make room

If no feasible node exists, **PostFilter** runs preemption. The default `DefaultPreemption` plugin:
1. Considers each node where pod could fit *if* lower-priority pods were evicted.
2. Picks the node with the fewest "victims" needing eviction.
3. Respects PodDisruptionBudgets.
4. Sets `pod.status.nominatedNodeName` to the chosen node.
5. Triggers victim pods' deletion (graceful, with terminationGracePeriod).
6. After victims terminate, scheduler retries to bind.

Preemption only works between pods with **different PriorityClass**:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata: {name: high-priority}
value: 1000000
globalDefault: false
description: "Production workloads"
```

`system-cluster-critical` (2000000000) and `system-node-critical` (2000001000) are built-in.

## 6.7 Pod topology spread

The modern way to balance across failure domains:

```yaml
apiVersion: v1
kind: Pod
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway   # or DoNotSchedule
    labelSelector:
      matchLabels:
        app: my-app
```

The scheduler tries to distribute pods so no zone has more than `maxSkew` more pods than the least-populated zone.

This replaces older `podAntiAffinity` patterns (which were O(N²) and slow).

## 6.8 Affinity and anti-affinity

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values: [linux]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: node-type
            operator: In
            values: [gpu]
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels: {app: cache}
        topologyKey: kubernetes.io/hostname
```

**required**: hard filter (won't schedule if not met).
**preferred**: soft score (preferred but not blocking).

`IgnoredDuringExecution`: the policy applies at scheduling time only; once scheduled, pod is not evicted if rules become violated.

## 6.9 Taints and tolerations

The opposite direction: nodes repel pods that don't tolerate them.

```yaml
# Node:
apiVersion: v1
kind: Node
spec:
  taints:
  - key: dedicated
    value: gpu
    effect: NoSchedule

# Pod that tolerates:
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: gpu
    effect: NoSchedule
```

Effects:
- **NoSchedule**: don't schedule new pods without toleration.
- **PreferNoSchedule**: prefer not to (soft).
- **NoExecute**: also evict existing pods without toleration.

Built-in taints applied by node-lifecycle controller:
- `node.kubernetes.io/not-ready`
- `node.kubernetes.io/unreachable`
- `node.kubernetes.io/memory-pressure`
- `node.kubernetes.io/disk-pressure`
- `node.kubernetes.io/network-unavailable`
- `node.kubernetes.io/unschedulable`

Pods auto-tolerate `not-ready` / `unreachable` with `tolerationSeconds: 300` (5 min) by default → pod evicted after 5 minutes of node being unreachable.

## 6.10 PodDisruptionBudget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: my-app
```

PDB constrains **voluntary** evictions (drain, descheduler, autoscaler). Not eviction by kubelet (NotReady, oom). Respected by:
- `kubectl drain`
- Cluster Autoscaler / Karpenter
- Descheduler
- Scheduler preemption

If draining would violate PDB, the eviction is rejected (Eviction API returns 429).

## 6.11 Bind subresource

The final step:

```
PUT /api/v1/namespaces/default/pods/foo/binding
{
  "apiVersion": "v1",
  "kind": "Binding",
  "metadata": {"name": "foo"},
  "target": {
    "apiVersion": "v1",
    "kind": "Node",
    "name": "node-3"
  }
}
```

This sets `pod.spec.nodeName=node-3`. The kubelet on node-3 sees the pod via its watch and starts it.

Note: the scheduler's "Bind" is just an apiserver API call. Anyone with permission can bind any pod to any node — that's why kube-scheduler is privileged.

## 6.12 Custom schedulers

Two paths:

### Path A: scheduler plugin in the existing framework

Implement the framework interface in Go, add to the scheduler binary, register via KubeSchedulerConfiguration:

```go
type MyPlugin struct{}

func (p *MyPlugin) Name() string { return "MyPlugin" }
func (p *MyPlugin) Filter(ctx context.Context, state *framework.CycleState,
    pod *v1.Pod, nodeInfo *framework.NodeInfo) *framework.Status {
    // your logic
    return framework.NewStatus(framework.Success)
}
```

Build a custom scheduler binary that registers this plugin and starts the framework.

### Path B: separate scheduler binary

Run a completely separate scheduler. Watch pods with `spec.schedulerName == "my-sched"`:

```go
for pod := range podWatcher {
    if pod.Spec.SchedulerName != "my-sched" || pod.Spec.NodeName != "" {
        continue
    }
    node := pickNode(pod)
    bind(pod, node)
}
```

This is how Volcano (batch), Yunikorn (multi-tenant queueing), Coscheduler (gang scheduling) work.

Multiple schedulers coexist (each handles pods with matching `schedulerName`).

## 6.13 Scheduler extender (legacy)

Pre-framework mechanism: the scheduler calls an external HTTP service to extend Filter/Score/Bind. Slow (network hop per pod) and limited. 
Mostly superseded by framework plugins.

## 6.14 Scheduling latency at scale

At 5000 nodes and high churn, scheduling latency matters:
- **Scheduler throughput**: ~100 pods/sec on a tuned system; bottleneck is apiserver writes.
- **Per-pod scheduling time**: dominated by Filter (linear in nodes) and PVC binding.
- **Cache sync**: scheduler relies on node informer cache; stale info → suboptimal placement.

Tunings:
- `--percentage-of-nodes-to-score`: only score top N% of feasible nodes (default 5–50%). Bigger clusters = smaller %. Trades optimality for speed.
- `--kube-api-qps`, `--kube-api-burst`: scheduler's apiserver rate.
- Multiple scheduler replicas with separate `schedulerName` profiles for tenant isolation.

## 6.15 Scheduling-related KEPs to know

- **KEP-249**: Pod topology spread (GA 1.19).
- **KEP-624**: in-place pod resize (beta 1.27+) — affects scheduling assumptions.
- **KEP-753**: sidecar containers (GA 1.29) — separate init/sidecar lifecycle.
- **KEP-3094**: pod scheduling readiness — pods can defer being scheduled until `schedulingGates` are removed.
- **KEP-3243**: pod scheduling profiles with scheduling readiness.
- **KEP-3691**: stable PV node affinity (mainstreaming for cross-zone PVCs).

### SchedulingGates (KEP-3521)

```yaml
spec:
  schedulingGates:
  - name: example.com/gate1
```

Pod is "Pending" until all gates are removed. An external controller (e.g., a custom queue admitter) can hold pods until quota allows. Useful for prioritization beyond what the scheduler natively supports.

## 6.16 Common interview probes

- **"Walk me through how a pod gets scheduled."** Section 6.1 + 6.2.
- **"What's the difference between affinity and tolerations?"** Affinity = pod *wants* nodes; tolerations = pod *can stand* nodes that repel.
- **"How does scheduling interact with PVC?"** Volume Binding plugin: filters out nodes that can't bind the PVC; "WaitForFirstConsumer" mode delays binding until scheduling.
- **"How would you implement gang scheduling?"** Use Permit plugin to "wait" until all pods in the gang are reservable; release all at once. Volcano implements this.
- **"What happens if no node fits a pod?"** PostFilter → preemption → if it succeeds, victims evicted, retry; else pod stays Pending with `PodScheduled=False`.
- **"How do you schedule pods across zones?"** PodTopologySpread with `topologyKey: topology.kubernetes.io/zone`.

## 6.17 Corner cases

- **Scheduler restart loses queue state** — pods in unschedulableQ rescan all events; can take seconds.
- **Watch lag** — scheduler sees stale node info → binds to a now-overcommitted node → kubelet rejects → re-queue.
- **PVC binding race** — two pods need same single-attach PVC. Scheduler binds; kubelet attaches; only one wins.
- **DaemonSet pods bypass scheduler** historically; modern DaemonSet uses scheduler with NodeAffinity + tolerations (KEP-2535).
- **Pods with `spec.nodeName` set explicitly** skip scheduler entirely. Used for static pods, debugging.
- **Pod stuck in Pending forever** — often an unsatisfiable affinity, missing PriorityClass, PVC not bindable, or quota.

## 6.18 Alternatives — when to replace the default scheduler

| Need | Solution |
|------|----------|
| Default + custom tweaks | Plugin in default scheduler |
| Batch / gang | Volcano (separate scheduler) |
| Multi-tenant queueing with quotas | Yunikorn |
| Right-sized binpack for cost | Karpenter (just-in-time provisioning) |
| Capacity-aware (HPC) | Slurm-on-k8s, KubeFlow |
| Topology-aware (NUMA, GPU) | TopologyManager + scheduler hints |

## Must-internalize

- The framework: Filter → Score → Bind, plus extension points.
- Plugin architecture; built-in plugins; how to add a custom one.
- Affinity/anti-affinity, taints/tolerations, topology spread.
- Preemption uses PriorityClass and respects PDB.
- PVC binding "WaitForFirstConsumer" delays until scheduling.
- SchedulingGates allow external admission.
- Volcano/Yunikorn for batch/multi-tenant; Karpenter for autoscale + scheduling.
