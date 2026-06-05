# 14 · Scaling — HPA, VPA, Cluster Autoscaler, Karpenter

K8s ships with three autoscaling axes: pod-count, pod-resource, and node-count. Plus modern node provisioners (Karpenter) that compress provisioning + scheduling.

## 14.1 The three axes

| Axis | What | When |
|------|------|------|
| **Horizontal Pod Autoscaler (HPA)** | Pod replica count | CPU/memory/custom load |
| **Vertical Pod Autoscaler (VPA)** | Pod resources (requests/limits) | Right-sizing over time |
| **Cluster Autoscaler / Karpenter** | Node count | Pending pods need more capacity |

These can compose but interact subtly.

## 14.2 HPA — Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: {name: web-hpa}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric: {name: requests_per_second}
      target: {type: AverageValue, averageValue: 1000}
  - type: External
    external:
      metric:
        name: sqs_queue_depth
        selector: {matchLabels: {queue: my-queue}}
      target: {type: AverageValue, averageValue: 30}
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

### The algorithm

Every `--horizontal-pod-autoscaler-sync-period` (default 15s):
1. Fetch current metrics from metrics.k8s.io (resource metrics) or custom.metrics.k8s.io (pod/object metrics).
2. For each metric, compute `desiredReplicas = ceil(currentReplicas * currentValue / targetValue)`.
3. Pick the **max** across metrics.
4. Apply behavior policies (rate limits, stabilization).
5. Patch the scale subresource.

### Metric types

- **Resource**: built-in metrics for CPU and memory. Comes from metrics-server.
- **Pods**: custom per-pod metric (e.g., `requests_per_second`). From custom-metrics-apiserver (Prometheus Adapter, etc.).
- **Object**: metric on another object (e.g., Service's request rate).
- **External**: metric not tied to a k8s object (e.g., SQS queue depth).
- **ContainerResource** (1.27+): like Resource but for a specific container.

### Stabilization window

`scaleDown.stabilizationWindowSeconds=300`: consider the recommendation over the last 300s; only scale down if the recommendation has been low for the whole window. Prevents flapping.

`scaleUp` typically has stabilization=0 (fast response).

### Behavior policies

Cap rate of change:
- "scale down by at most 10% per 60s."
- "scale up by at most 100% per 15s" (doubling).

`selectPolicy: Max` (default) — most aggressive. `Min` for conservative.

## 14.3 metrics-server and the metrics pipeline

```
   kubelet (each node)
        │ /stats/summary (cAdvisor data)
        ▼
   metrics-server (aggregated API)
        │ /metrics.k8s.io/v1beta1/pods/{ns}/{pod}
        ▼
   HPA controller
        │ patches scale subresource
        ▼
   Deployment / StatefulSet / ReplicaSet
```

metrics-server runs as a Deployment, registers as an APIService for `metrics.k8s.io`. Scrapes kubelet every ~15s.

`kubectl top pod` uses metrics-server.

For custom metrics, run **prometheus-adapter** alongside metrics-server (it exposes `custom.metrics.k8s.io` and `external.metrics.k8s.io`).

## 14.4 KEDA — Kubernetes-based Event-Driven Autoscaling

The de facto standard for HPA based on external systems (Kafka lag, SQS depth, Postgres row count, anything).

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: {name: kafka-consumer}
spec:
  scaleTargetRef:
    name: my-consumer
  minReplicaCount: 0   # KEDA can scale to zero
  maxReplicaCount: 100
  pollingInterval: 30
  cooldownPeriod: 300
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: my-group
      topic: events
      lagThreshold: "10"
```

KEDA creates an HPA under the covers but provides the metric source. **Scale-to-zero** is a unique KEDA feature: when no work, pod count → 0.

## 14.5 VPA — Vertical Pod Autoscaler

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata: {name: my-vpa}
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: Auto   # Off | Initial | Recreate | Auto | InPlaceOrRecreate (alpha)
  resourcePolicy:
    containerPolicies:
    - containerName: '*'
      minAllowed: {cpu: 100m, memory: 128Mi}
      maxAllowed: {cpu: 2,   memory: 4Gi}
      controlledResources: ["cpu", "memory"]
```

### Update modes

| Mode | Behavior |
|------|----------|
| **Off** | Only recommends; doesn't apply (use to gather data) |
| **Initial** | Apply on pod creation only |
| **Recreate** | Evict and recreate pods with new resources |
| **Auto** | Currently same as Recreate |
| **InPlaceOrRecreate** (alpha) | Use in-place resize if possible, else recreate |

### The three components

- **Recommender**: analyzes historical usage, computes recommendation.
- **Updater**: evicts pods so they're recreated with new resources (in Recreate mode).
- **Admission Controller**: injects new resources at pod creation.

### Warning — HPA + VPA conflict

VPA changes resources; HPA scales based on resources. Both adjusting CPU → unstable. Use VPA for memory only and HPA for CPU; or use HPA on a custom metric (RPS) and VPA on resources.

## 14.6 Cluster Autoscaler (CA)

```yaml
# Deployment with provider-specific config
spec:
  containers:
  - command:
    - ./cluster-autoscaler
    - --cloud-provider=aws
    - --nodes=2:50:my-asg
    - --balance-similar-node-groups
    - --skip-nodes-with-system-pods=false
```

### Algorithm

Every loop (default 10s):
1. **Find unschedulable pods** (Pending with `Unschedulable=True` condition).
2. **Try expansion**: for each node group, simulate "would adding a node here make pods schedulable?" Pick the best (cheapest, fewest nodes, etc.).
3. **Find unneeded nodes**: a node is unneeded if all its pods could run elsewhere. After `scaleDownUnneededTime`, evict pods and remove the node.
4. **Apply via cloud provider**: ASG resize, GCE MIG, Azure VMSS.

### Expansion strategies

- **random** (default): pick at random.
- **most-pods**: pick node group whose new node fits the most pending pods.
- **least-waste**: pick node group with least leftover resources.
- **price** (cloud-specific): cheapest.
- **priority**: explicit ordering via PriorityExpander config.

### Limits

- Doesn't pre-warm; pods wait for cloud provisioning (typically 30s–5min).
- Operates on **fixed node groups** (one instance type per group).
- Heuristic; can be slow to make "right" decisions.

## 14.7 Karpenter

Modern alternative (AWS open-source, now CNCF graduating). Replaces CA + node group config.

### How it differs

- **Provisioner-driven**: NodePool CR defines what kinds of nodes can be created (instance types, AZ, capacity-type).
- **Just-in-time**: when pods are unschedulable, Karpenter directly creates EC2 instances (no ASG).
- **Bin-packing**: chooses instance type to fit pending pods optimally.
- **Consolidation**: continually evaluates if existing pods could move to fewer/smaller nodes.
- **Fast**: ~30s from pending to scheduled (vs ~3min for CA).

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata: {name: default}
spec:
  template:
    spec:
      nodeClassRef:
        name: default
      requirements:
      - key: kubernetes.io/arch
        operator: In
        values: [amd64, arm64]
      - key: karpenter.sh/capacity-type
        operator: In
        values: [spot, on-demand]
      - key: node.kubernetes.io/instance-type
        operator: In
        values: [m5.large, m5.xlarge, m5.2xlarge, c5.large]
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
  limits:
    cpu: 1000
```

Karpenter has effectively replaced CA at major shops (Pinterest, ANZ Bank, Adobe).

## 14.8 PodDisruptionBudget + scale-down

When CA or Karpenter wants to remove a node, it calls the Eviction API on pods. PDBs constrain this — eviction returns 429 if violating.

Designs that protect:
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 80%
  selector: {matchLabels: {app: my-app}}
```

Without PDB, autoscaler can rip out all your pods at once.

## 14.9 Cost optimization patterns

| Pattern | What |
|---------|------|
| **Spot instances** | Cheap (60-90% off) but interruptible (~2min warning). Tolerate Spot termination. |
| **Diverse instance types** | Karpenter picks cheapest available; if one type out, falls back. |
| **Scale to zero** | KEDA for batch / event-driven. |
| **VPA for right-sizing** | Reduce requests; pack more. |
| **Bin-packing scheduler** | NodeResourcesFit with `mostAllocated` strategy. |
| **Consolidation (Karpenter)** | Reclaim under-utilized nodes. |
| **Reserved Instances / Savings Plans** | For baseline; pair with Spot for burst. |

## 14.10 Cluster Autoscaler vs Karpenter

| Property | CA | Karpenter |
|----------|----|-----------|
| Cloud support | Multi (AWS, GCP, Azure, …) | AWS GA; Azure preview; GCP alpha |
| Configuration | Node groups (one type each) | Pool with constraints |
| Provisioning | Resize ASG (minutes) | Direct EC2 (30s) |
| Bin-packing | Per-group | Across all types |
| Consolidation | Limited | First-class |
| Cluster size | Tested to 5000 nodes | Tested similar |
| Maturity | Mature, stable | New, fast-moving |

For multi-cloud: CA. For AWS-only and want best efficiency: Karpenter.

## 14.11 Descheduler

A non-default tool that **evicts** pods to improve placement:
- "Pods on over-utilized nodes." Move to under-utilized.
- "Pods violating topology constraints." Re-schedule.
- "Pods with anti-affinity violations." Same.

```yaml
apiVersion: descheduler/v1alpha2
kind: DeschedulerPolicy
profiles:
- name: ProfileName
  pluginConfig:
  - name: DefaultEvictor
    args:
      evictLocalStoragePods: true
  - name: LowNodeUtilization
    args:
      thresholds: {cpu: 20, memory: 20, pods: 20}
      targetThresholds: {cpu: 50, memory: 50, pods: 50}
  plugins:
    balance: {enabled: [LowNodeUtilization]}
```

Descheduler doesn't schedule; it just evicts. Scheduler then re-places.

Useful for: long-running clusters where placement degrades over time; cost reduction.

## 14.12 Common interview probes

- **"How does HPA decide when to scale?"** Fetches metrics from metrics.k8s.io; computes `desired = current × currentValue / targetValue`; applies behavior policies; patches scale subresource.
- **"What's the difference between HPA and VPA?"** HPA changes replicas; VPA changes resources. Can compose carefully.
- **"How do you scale on Kafka lag?"** KEDA's Kafka trigger; KEDA creates HPA with external.metrics.k8s.io.
- **"Karpenter vs Cluster Autoscaler?"** Karpenter is provisioner-driven (no node groups), JIT, faster, consolidates. CA is node-group based, older, multi-cloud.
- **"How do PDBs interact with autoscaler?"** Autoscaler evicts via Eviction API; PDB blocks if eviction would violate.
- **"Scale to zero — how?"** KEDA with idle trigger; pod count → 0 when no work. Wakeup pattern: KEDA scales 0→1 on trigger; first request waits for cold start.

## 14.13 Corner cases

- **HPA + VPA on same metric (CPU)** → both try to control; oscillation. Avoid.
- **HPA stuck at maxReplicas** → check `kubectl describe hpa`; metric not above threshold, or maxReplicas reached.
- **HPA scales but pods don't help** → resource exhaustion downstream (DB, cache). Scale up != solve all problems.
- **CA scales up but pods stay pending** → node group's instance type can't accommodate (taints, resource size, GPU). Misconfig.
- **Karpenter creates expensive instances** → constrain via NodePool requirements; use price-aware policies.
- **VPA Recreate kills production pods** → use Initial or InPlaceOrRecreate during business hours.
- **Spot interruption causes cascading evictions** → set `terminationGracePeriodSeconds` long enough; PDB.

## 14.14 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Scale on CPU/memory | HPA Resource metric | KEDA `cpu`/`memory` triggers | manual `kubectl scale` |
| Scale on RPS | HPA custom metric (prometheus-adapter) | KEDA Prometheus trigger | external load balancer scaling |
| Scale on queue depth | KEDA Kafka/SQS/Pub-Sub trigger | external metrics API | sidecar pattern |
| Scale to zero | KEDA | Knative | Argo Events |
| Right-size resources | VPA recommender (Off mode) | Goldilocks (open-source) | manual sizing review |
| Add nodes | Cluster Autoscaler | Karpenter | manual ASG ops |
| Bin-pack tightly | mostAllocated strategy | Karpenter consolidation | descheduler |
| Save cost | Spot + diverse types | Reserved + scheduled scale-down | Karpenter consolidation |

## 14.15 Best practices

- **HPA**: set `behavior.scaleDown.stabilizationWindowSeconds=300` to avoid flap.
- **VPA**: start in `Off` mode for a week, observe recommendations, then move to Initial.
- **CA**: enable `--balance-similar-node-groups` to keep node groups even across zones.
- **Karpenter**: define explicit NodePool constraints; otherwise it picks anything.
- **PDB on every production Deployment**: protect from over-eager scale-down.
- **Resource requests > 0 always**: otherwise scheduler treats as BestEffort, eviction-prone.
- **Don't set CPU limits** for latency-sensitive apps (CFS throttling).
- **For batch**: set CPU limits = requests; use Spot.

## Must-internalize

- HPA: replicas based on metric × threshold; metrics from metrics-server / custom-metrics-apiserver / external-metrics-apiserver.
- KEDA: HPA for external metrics; scale-to-zero pattern.
- VPA: resource right-sizing; Recreate or InPlace.
- Cluster Autoscaler: node-group-based; multi-cloud; minutes to provision.
- Karpenter: provisioner-based; JIT; AWS-mature; consolidation.
- PDB constrains voluntary evictions including autoscaler scale-down.
- Spot + diverse + consolidation = primary cost optimization stack.
- HPA + VPA on same metric → oscillation; avoid.
