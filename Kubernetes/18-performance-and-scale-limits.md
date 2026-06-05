# 18 · Performance and Scale Limits

The famous K8s limits. Staff candidates should know the numbers cold AND the underlying reason each limit exists AND how to push past.

## 18.1 The published limits (per cluster)

```
   Nodes:                ≤ 5,000     (officially supported)
   Pods total:           ≤ 150,000
   Pods per node:        ≤ 110 (default)  (can be raised with caveats)
   Containers total:     ≤ 300,000
   Services:             ≤ 10,000        (kube-proxy iptables) or much higher with eBPF
```

These come from k/k's [SLO doc](https://github.com/kubernetes/community/blob/master/sig-scalability/slos/slos.md). Tested in scale-test infrastructure.

## 18.2 Where each limit comes from

### 5000 nodes

Multiple bottlenecks pile up:
- **Watch traffic on apiserver**: each node has a kubelet watching its own pods + nodes + secrets. 5000 watchers × event rates = lots of CPU.
- **etcd write rate**: node heartbeats (Lease object every 10s) × 5000 nodes = 500/s baseline writes.
- **Scheduler latency**: O(N) Filter for some plugins.
- **kube-controller-manager memory**: caches all objects in informers.

Hyperscale shops push past with custom apiservers, watch optimizations, separate etcd per resource type.

### 150,000 pods

- **etcd object count**: each pod is ~5KB in etcd (with status). 150K × 5KB = 750MB. Plus replicas, secrets, etc.
- **Watch event rate**: pod churn from rolling updates is the heaviest write source.
- **GC controller memory**: tracks owner-ref graph.

### 110 pods per node

- **PLEG poll time**: scans all containers every 1s. At 110 pods × 3 containers each = 330 CRI calls per second. Legacy.
- **iptables/IPVS rule count**: kube-proxy programs node for service VIPs.
- **conntrack table size**: each pod is multiple flows.
- **kubelet stat collection**: cadvisor on each container.

Evented PLEG (1.27+) lets you go higher (~250 pods/node). Custom builds at hyperscale go to 500+.

### 300,000 containers

Bounded by 150K pods × 2 containers average. Container runtime (containerd) can handle ~1000 per node fine.

### 10,000 services (iptables)

- Each Service + endpoint = ~3 iptables rules.
- 10K services × 10 endpoints = 100K-300K rules.
- iptables linear scan → packet path slows.
- Reload of full table during a service change = seconds.

With **IPVS** or **eBPF** (Cilium), this limit is ~100K+ services with no penalty.

## 18.3 The first thing that breaks at scale

Each layer breaks at a different threshold:

| At... | What hits the wall |
|-------|-------------------|
| ~500 nodes | Default `--watch-cache-sizes` too small; relist storm; tune up |
| ~1000 nodes | iptables kube-proxy reload time becomes noticeable |
| ~2000 nodes | etcd DB size from events + endpoints; need event-split |
| ~3000 nodes | apiserver memory; APF tuning matters; multiple apiserver replicas |
| ~5000 nodes | etcd write throughput; node-lifecycle delays; you're at the ceiling |
| >5000 nodes | Hyperscaler-only; custom apiserver, sharding |

## 18.4 Scaling the control plane

### Multiple apiservers

```
   Load balancer (cloud LB)
         │
    ┌────┼────┐
    ▼    ▼    ▼
  apiserver replicas (e.g., 3-5)
    │    │    │
    └────┼────┘
         ▼
       etcd
```

Watch streams are per-replica; bookmarks help clients survive disconnects.

### Separate etcd for events

```
   apiserver flag: --etcd-servers-overrides=/events#http://events-etcd:2379
```

Events are high-write, low-value-long-term. Routing to separate etcd prevents them from filling main etcd. Hyperscale clusters always do this.

### Encryption KMS plugin perf

The KMS plugin processes every Secret write. At scale (high SA token rotation), this matters. Use KMS v2 (1.27+) with caching.

### APF tuning

Default APF gives `system` priority to control-plane components. At scale, also:
- Bump `workload-high` for HPA / autoscaler / cluster-autoscaler.
- Add a custom FlowSchema for high-traffic controllers.

## 18.5 Scaling etcd

Beyond defaults:
- `--quota-backend-bytes=8589934592` (8GB).
- `--auto-compaction-mode=periodic --auto-compaction-retention=8h`.
- Defrag cron every week.
- 5-node Raft cluster for higher tolerance.
- Pin to NVMe local SSD.

Beyond 8GB DB: sharded etcd (split by resource type), or replace with alternative (kine + RDBMS for k3s, custom at hyperscale).

## 18.6 Scaling the data plane

### Networking

| At scale | Switch from | Switch to |
|----------|-------------|-----------|
| iptables | kube-proxy iptables | kube-proxy IPVS or Cilium eBPF |
| Service count | full iptables | EndpointSlices + IPVS/eBPF |
| Network policy | iptables-based (Calico) | eBPF-based (Cilium) |
| DNS | central CoreDNS | NodeLocalDNS DaemonSet |

### Storage

CSI drivers have per-driver scaling limits:
- EBS: max 39 volumes per EC2 (varies by instance type).
- EFS: thousands per cluster.
- Ceph: depends on Ceph cluster.

Bin-packing pods with disks: scheduler's `MaxVolumeCount` plugin.

### Container runtime

containerd has been pushed to 250+ pods/node; the limit is kubelet bookkeeping (PLEG, status updates).

## 18.7 Workload-level scaling

Per-namespace limits prevent one tenant from blowing up:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata: {name: my-ns-quota, namespace: my-ns}
spec:
  hard:
    pods: "100"
    services: "20"
    persistentvolumeclaims: "10"
    configmaps: "200"
    secrets: "100"
    requests.cpu: "20"
    requests.memory: "20Gi"
    limits.cpu: "40"
    limits.memory: "40Gi"
    requests.storage: "200Gi"
```

`LimitRange` adds per-pod defaults / maxes:

```yaml
apiVersion: v1
kind: LimitRange
spec:
  limits:
  - type: Container
    default: {cpu: "500m", memory: "512Mi"}
    defaultRequest: {cpu: "100m", memory: "128Mi"}
    max: {cpu: "4", memory: "8Gi"}
```

ResourceQuota + LimitRange = essential at multi-tenant scale.

## 18.8 The scale-test methodology

K8s sig-scalability runs ClusterLoader2 + perf-dash tests:
- Provision a cluster with N nodes.
- Create M pods.
- Measure SLOs.

Reproducing locally:
- kind / minikube to ~500 nodes (with significant resources).
- kwok (k8s without kubelet) — simulates nodes/pods without real runtime; tests apiserver/etcd scaling.

For a real org: run scale tests on staging hardware before claiming N-node support.

### SLO targets (sig-scalability)

- **API latency**: 99% of mutating requests < 1s; 99% of reads < 1s.
- **Pod start latency**: 99% of pods running within 5s of scheduling.
- **DNS latency**: 99% of queries < 5ms.
- **Pod throughput**: ≥ 100 pods/s creation.

## 18.9 Real-world numbers from hyperscalers

| Org | Cluster size | Notes |
|-----|--------------|-------|
| Google | thousands of clusters, many at 5000+ nodes | GKE Enterprise scale |
| AWS EKS | up to ~5000 nodes | Limit lifted on request |
| Lyft | ~1000 nodes per cluster | Multi-cluster strategy |
| Spotify | 10s of clusters, ~500-1000 nodes each | Backstage maintains |
| Alibaba | clusters up to 10K nodes | Significant patches |
| OpenAI | clusters up to 7500 nodes for training | Customized apiserver |
| Tesla | unknown but reportedly large | Public talks 2023+ |

## 18.10 Cost-related scale

Scale and cost are tangled:
- Each node has overhead (kubelet ~250MB memory, kube-proxy, monitoring agents, CSI driver).
- Smaller node = more overhead per workload pod.
- Larger node = better packing but bigger blast radius.

Sweet spot for most workloads: 4-16 vCPU nodes; 16-64GB memory.

Karpenter consolidation finds this automatically.

## 18.11 Limits monitoring — what to alert on

```
   ALERTS:
     - apiserver_request_total{code=~"5.."} rate > threshold
     - etcd_disk_wal_fsync_duration_seconds p99 > 50ms
     - etcd_mvcc_db_total_size_in_bytes > 6GB (warning) / 8GB (critical)
     - kubelet_pleg_relist_duration_seconds p99 > 5s
     - kube_pod_status_phase{phase="Pending"} count > N
     - apiserver_dropped_requests_total > 0 (APF rejections)
     - kube_resourcequota usage approaching limit
     - cluster_autoscaler_unschedulable_pods_count > 0 for > N min
```

## 18.12 Common interview probes

- **"What's the cluster size limit and why?"** 5000 nodes / 150K pods. Multiple bottlenecks: watch traffic, etcd writes, scheduler latency.
- **"How would you push past 5000 nodes?"** Separate events etcd; APF tuning; multiple apiserver replicas; per-resource etcd; custom scheduler / patch.
- **"Why is etcd the bottleneck?"** Single-writer Raft consensus; fsync per write; 8GB DB limit. Sharding (per resource) helps.
- **"Why limit 110 pods/node?"** PLEG poll time, iptables/IPVS rule count, conntrack table, kubelet bookkeeping. Evented PLEG (1.27+) raises this.
- **"How do you debug apiserver overload?"** apiserver_request_duration metric; APF rejection counters; etcd commit latency. Tune APF; scale apiserver replicas; add nodes to etcd.
- **"What's the workload-level cap mechanism?"** ResourceQuota (namespace-wide) + LimitRange (per-pod defaults).

## 18.13 Corner cases

- **Webhook with 1s latency × 1000 pods/sec = 1000 req/s on webhook** → bottleneck. Webhook capacity matters.
- **Massive Watch reconnects** (control plane bounce) cause "relist storm" — all clients re-LIST every resource. Memory spike.
- **Single namespace with 10K Pods** — namespace watcher gets blown; FieldSelector by namespace helps but only so much.
- **etcd compaction during high churn** — write latency spikes; tune compaction interval.
- **kube-proxy iptables-restore time** at 5000 services — seconds; service updates lag.
- **NodeIPAM exhausts** (per-node pod CIDR) at high density → pods stuck.
- **Endpoints (legacy) at 1000 endpoints in a single Service** — one giant object; watch broadcast blows.

## 18.14 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| >5000 nodes | Multi-cluster | Custom apiserver patches | Hyperscaler-managed (GKE) |
| High Service count | iptables kube-proxy | IPVS | Cilium eBPF |
| Event spam | event-rate-limit admission | separate events etcd | suppress at source (controller fix) |
| etcd DB growth | larger DB | events split | per-resource etcd sharding |
| High pod churn | apiserver replicas | APF tuning | reduce watch verbosity (fewer event handlers) |
| Pod startup latency | smaller images | image preloading | image lazy loading (stargz) |

## 18.15 Image-related scale

Image pulls dominate startup time. Mitigations:
- **Smaller images** (distroless, alpine).
- **ImagePullPolicy: IfNotPresent**.
- **Pre-pull on nodes** via DaemonSet that pulls common images.
- **Lazy image loading** (stargz-snapshotter, eStargz): fetch chunks as the container reads them.
- **Image cache / registry mirror** in cluster.

KEP-3936 (Lazy Pull / stargz, alpha): startup time drops from minutes to seconds for big images.

## 18.16 The "10x scale" framework

For an interview "how do you scale to 10× current":

1. **Identify the binding constraint**. Usually one of: etcd, apiserver, watch fanout, scheduler latency, network plane (iptables vs eBPF).
2. **Address that constraint first**. If it's etcd, split events; if it's apiserver, add replicas + tune APF.
3. **Multi-cluster if necessary**. Beyond 5000 nodes, the answer is shard.
4. **Measure**. Use ClusterLoader2 or similar; verify SLOs.

## Must-internalize

- The numbers: 5000 nodes / 150K pods / 110 pods/node / 10K services iptables.
- Where each comes from: etcd Raft, PLEG, iptables linear scan, watch traffic.
- Mitigations: IPVS/eBPF, event split, evented PLEG, multiple apiservers, APF.
- For >5000 nodes: multi-cluster is the answer.
- ResourceQuota + LimitRange for tenant containment.
- Beyond 8GB etcd DB: shard or replace.
- Track key SLOs: apiserver latency, etcd fsync, pod start time.
