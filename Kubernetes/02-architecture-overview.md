# 02 · Kubernetes Architecture Overview

The single most important file. Almost every other answer compresses to "...because k8s works like this." Read twice; sketch the diagrams from memory.

## 2.1 The mental model — the cluster as a whole

```
                       ┌──────────────────── Control Plane ────────────────────┐
                       │                                                          │
   ┌──────────┐        │   ┌─────────────────┐         ┌──────────────────────┐  │
   │ kubectl  │ ─HTTPS►│   │  kube-apiserver │ ◄──────►│       etcd           │  │
   │  + RBAC  │        │   │  (REST + watch) │         │  (3-5 Raft voters)   │  │
   └──────────┘        │   └────────┬────────┘         └──────────────────────┘  │
                       │            │  (watch + write)                            │
                       │            ▼                                              │
                       │   ┌─────────────────┐                                    │
                       │   │ kube-controller │  (deployments, rs, sa-tokens, gc) │
                       │   │     manager     │                                    │
                       │   └─────────────────┘                                    │
                       │   ┌─────────────────┐                                    │
                       │   │ kube-scheduler  │  (Filter + Score plugin chain)    │
                       │   └─────────────────┘                                    │
                       │   ┌─────────────────┐                                    │
                       │   │ cloud-controller│  (node, route, LB, volume)        │
                       │   │      manager    │                                    │
                       │   └─────────────────┘                                    │
                       └──────────────────────────────────────────────────────────┘
                                       │   ▲
                                       │   │ (watch + status)
                                       ▼   │
   ┌─────────────────────────── Each Node ────────────────────────────────────┐
   │                                                                            │
   │  ┌─────────────────┐    ┌──────────────────┐    ┌──────────────────┐     │
   │  │     kubelet     │    │   kube-proxy /   │    │   CNI plugin     │     │
   │  │  (watches pods  │    │   (or eBPF       │    │  (calico/cilium/ │     │
   │  │   bound to me)  │    │    replacement)  │    │   flannel)       │     │
   │  └────────┬────────┘    └──────────────────┘    └──────────────────┘     │
   │           │                                                                 │
   │           ▼ (CRI gRPC)                                                     │
   │  ┌─────────────────┐                                                       │
   │  │  containerd /   │ ── (OCI runtime spec) ──► runc ── (cgroup, ns) ──►    │
   │  │     CRI-O       │                                                       │
   │  └─────────────────┘                                                       │
   │           │                                                                 │
   │           ▼                                                                 │
   │  ┌─────────────────┐                                                       │
   │  │   Containers /   │                                                       │
   │  │   Pod sandbox    │                                                       │
   │  └─────────────────┘                                                       │
   └────────────────────────────────────────────────────────────────────────────┘
```

Five rules:

1. **Only apiserver talks to etcd.** Every other component talks to apiserver.
2. **Everything is declarative + eventually consistent.** Controllers reconcile; nothing is procedurally orchestrated.
3. **Watch streams are the nervous system.** Components don't poll — they watch.
4. **Pods are the atomic unit of scheduling.** Not containers.
5. **Nothing trusts the cache for writes.** Every write goes through apiserver, which validates against the live etcd state with optimistic concurrency (resourceVersion).

## 2.2 The control plane components

### kube-apiserver

```
                        ┌─────────────────────┐
   client request  ───► │ Authentication      │   x.509, bearer token, OIDC, webhook
                        ├─────────────────────┤
                        │ Authorization       │   RBAC / Webhook / ABAC / Node
                        ├─────────────────────┤
                        │ Mutating Admission  │   webhooks, defaulting, ServiceAccount, ResourceQuota
                        ├─────────────────────┤
                        │ Validation          │   OpenAPI schema, CRD schema
                        ├─────────────────────┤
                        │ Validating Admission│   webhooks, ResourceQuota (final)
                        ├─────────────────────┤
                        │ Storage             │   write to etcd through KV interface
                        └─────────────────────┘
                                 │
                                 ▼
                         Watch broadcast to all watchers
```

Key points:
- **Stateless.** Behind a load balancer, multiple replicas, all read/write the same etcd.
- **Watch streams** carry deltas to every controller, every kubelet, every operator.
- **Aggregation layer**: routes specific API groups to a different apiserver (metrics-server, custom-metrics-apiserver).
- **API priority and fairness (APF)** — replaces max-in-flight limits; fairness across users/groups.

### etcd

- **Raft consensus** — 3 or 5 voters; tolerates ⌊n/2⌋ failures.
- **MVCC** — every write gets a new revision; watch streams are revision-range subscriptions.
- **Lease** — TTL-bound keys (used for leader election, node heartbeats).
- **Hard limit** — `--quota-backend-bytes` defaults to 2GB, recommended max 8GB.
- **Compaction + defrag** required periodically to reclaim space.

Production etcd lives on local NVMe SSD, with `--quota-backend-bytes=8589934592`, `--auto-compaction-retention=8h`, and a defrag cron job.

### kube-controller-manager

A single binary that runs ~30 controllers:
- **Deployment**, **ReplicaSet**, **StatefulSet**, **DaemonSet**, **Job**, **CronJob**.
- **Endpoint** (legacy) + **EndpointSlice** controller.
- **PersistentVolume**, **PVC binder**.
- **ServiceAccount** + **Token** controllers.
- **Garbage collector** (owner-reference-driven cascade delete).
- **Namespace** controller (handles namespace finalization).
- **TTL-after-finished**, **NodeIPAM**, **NodeLifecycle**.
- **Service-account-token** controller signs JWTs.

Each controller runs an informer + work queue + reconcile loop (see file 05).

### kube-scheduler

Watches Pods with `spec.nodeName == ""`, runs the scheduling framework (Filter → Score → Reserve → Permit → Bind), picks a node, writes back via the bind subresource.

### cloud-controller-manager

Cloud-provider-specific controllers:
- **Node controller** — detect VM deletion, mark node as deleted.
- **Route controller** — set up routes between pod CIDRs in cloud networking.
- **Service controller** — provision cloud LBs for `Service type: LoadBalancer`.
- **Volume controller** — attach/detach EBS, GCE PD, Azure Disk.

This is what made k8s portable: extract cloud-specific logic to a plugin binary.

## 2.3 The data plane — what happens on each node

### kubelet

The node-side agent. Watches the apiserver for pods bound to its node. Maintains state through the **syncloop**:

```
                        ┌──────────────────┐
   Pod added/changed ──►│   SyncLoop       │
   (from apiserver       │                  │
   watch, PLEG event,    │   - get latest   │
   timer, etc.)          │   - sync pods    │
                        │   - update status│
                        └────────┬─────────┘
                                 │
                                 ▼
                   ┌────────────────────────┐
                   │  PLEG (Pod Lifecycle   │   polls CRI every 1s
                   │  Event Generator)      │
                   └────────────────────────┘
                                 │
                                 ▼
                   ┌────────────────────────┐
                   │  Container Manager     │   creates sandbox, containers via CRI
                   │  Volume Manager        │   mounts via CSI
                   │  CNI                   │   sets up network ns + veth
                   │  cgroups               │   sets limits via systemd or cgroupfs
                   │  Probes                │   liveness/readiness/startup
                   └────────────────────────┘
```

### Container runtime — via CRI

`kubelet` ↔ runtime over a Unix socket at `/var/run/containerd/containerd.sock` (or CRI-O). Protocol is gRPC. Operations:

- `RunPodSandbox` — create the pod's network namespace + pause container.
- `CreateContainer` + `StartContainer` — start each container.
- `StopPodSandbox`, `RemovePodSandbox`.
- `PullImage` — pull from registry.
- `ListContainerStats`, `ContainerStatus`.

The runtime (containerd or CRI-O) translates CRI calls into OCI runtime spec invocations, which are executed by **runc** (the default low-level runtime that uses cgroups + namespaces).

### kube-proxy (or eBPF replacement)

Implements Service VIPs:
- **iptables mode** — default; DNAT rules per Service+Endpoint. Linear scan; degrades at >5K services.
- **IPVS mode** — kernel IPVS; faster lookup, supports many algorithms.
- **eBPF replacement** (Cilium) — no kube-proxy at all; eBPF programs at tc-ingress.

### CNI

Reads `/etc/cni/net.d/*.conflist` (the CNI configuration), invokes the plugin binary (e.g., `/opt/cni/bin/calico`). Plugin sets up the pod's network namespace, allocates IP from IPAM, configures routes.

## 2.4 The request flow — `kubectl apply -f deployment.yaml`

1. `kubectl` parses YAML; computes a strategic merge patch; sends `PATCH` to apiserver.
2. **Apiserver**:
   - Authenticates (x509 client cert from kubeconfig).
   - Authorizes (RBAC check: `apps/deployments` resource, `patch` verb, namespace).
   - Mutating admission: defaults applied, owner refs set (if SSA, fields tracked).
   - Validates against OpenAPI schema.
   - Validating admission: webhooks (e.g., OPA Gatekeeper) check.
   - Resource quota (final, atomic).
   - Writes to etcd via `Apply` → new ResourceVersion.
3. **Watch broadcast**: apiserver pushes the new Deployment object to every connected watcher.
4. **Deployment controller** receives event, reconciles:
   - Computes desired ReplicaSet (based on PodTemplate hash).
   - Creates new RS via apiserver POST.
   - Scales old RS down + new RS up per RollingUpdate strategy.
5. **ReplicaSet controller** sees new RS, computes "I need N more pods," creates Pods via apiserver POST.
6. **Apiserver** writes Pod (with `spec.nodeName == ""`) to etcd.
7. **Scheduler** receives Pod-added event:
   - PreFilter / Filter plugins: which nodes are *feasible*?
   - Score plugins: pick the best.
   - Bind plugin: write `spec.nodeName` via Pod's binding subresource.
8. **kubelet** on the chosen node sees the Pod via its watch:
   - **PLEG** processes the new pod.
   - **Volume manager** mounts volumes (PVCs via CSI; configmaps/secrets via shared volume).
   - **CRI**: `RunPodSandbox` — runc creates network namespace.
   - **CNI**: plugin called → veth created, IP allocated, routes set.
   - **CRI**: `PullImage`, `CreateContainer`, `StartContainer` for each container.
   - Probes start running.
9. **kubelet** updates Pod.Status via apiserver PATCH; events emitted; metrics scraped by metrics-server.
10. **kube-proxy / Cilium** updates Service VIP → pod-IP mapping when Pod becomes Ready.

This is the canonical k8s flow. Staff candidates should be able to walk through it without notes.

## 2.5 The watch story

The most interesting protocol detail.

```
   client ──► GET /api/v1/pods?watch=true&resourceVersion=12345
                                          │
                                          ▼
                              apiserver opens a chunked HTTP response
                                          │
                                          ▼
                              [Event: ADDED  {pod1@rv=12346}]
                              [Event: MODIFIED {pod2@rv=12347}]
                              [Event: DELETED {pod3@rv=12348}]
                              ... (keeps streaming until disconnect)
```

Behavior:
- **resourceVersion** monotonic per type. Client provides "what I've seen"; server streams from there.
- **Watch cache** in apiserver: a ring buffer per resource type, holds last ~1000 events. If client's `resourceVersion` is older than the cache window, apiserver responds with `410 Gone` → client must full-list-then-watch (relist storm).
- **Bookmark events** (`?allowWatchBookmarks=true`) — periodically the server emits a bookmark with the latest RV, so the client can checkpoint without depending on actual changes.
- **HTTP/2 multiplexing** lets one connection carry many watch streams.

Watch is **the** scaling pivot point. At >1000 nodes, watch traffic dominates apiserver CPU. KEP-3157 (efficient watch resumption), KEP-365 (watch list option), KEP-4006 (watch list pagination) are the active fronts.

## 2.6 The schema and the API groups

API groups are like packages:

| Group | Resources |
|-------|-----------|
| `""` (core) | Pod, Service, ConfigMap, Secret, Node, Namespace, PersistentVolume, ServiceAccount |
| `apps/v1` | Deployment, ReplicaSet, StatefulSet, DaemonSet |
| `batch/v1` | Job, CronJob |
| `networking.k8s.io/v1` | NetworkPolicy, Ingress, IngressClass |
| `gateway.networking.k8s.io/v1` | Gateway, HTTPRoute, GRPCRoute |
| `storage.k8s.io/v1` | StorageClass, CSIDriver, VolumeAttachment |
| `rbac.authorization.k8s.io/v1` | Role, ClusterRole, RoleBinding, ClusterRoleBinding |
| `policy/v1` | PodDisruptionBudget |
| `coordination.k8s.io/v1` | Lease |
| `admissionregistration.k8s.io/v1` | ValidatingWebhookConfiguration, MutatingWebhookConfiguration |
| `apiextensions.k8s.io/v1` | CustomResourceDefinition |
| `apiregistration.k8s.io/v1` | APIService (aggregation) |

Each resource has:
- A schema (OpenAPI v3).
- A storage version (which version is canonical in etcd).
- Conversion functions between served versions.

CRDs add user-defined resources; the apiextensions apiserver embeds the schema and serves them like native resources.

## 2.7 The hidden service: kubelet exposing endpoints

kubelet has its own HTTP API on each node (port 10250):

- `/metrics` — Prometheus-format kubelet metrics.
- `/metrics/cadvisor` — container resource metrics.
- `/metrics/resource` — pod-level resource metrics.
- `/stats/summary` — used by metrics-server.
- `/logs/<container>` — log streaming.
- `/exec` / `/portforward` — used by `kubectl exec` and `kubectl port-forward`.
- `/runningpods` — debug.

These are authenticated/authorized via the apiserver (kubelet sends the requestor's token back for authn/authz check).

## 2.8 Where the cluster persists state

| State | Where |
|-------|-------|
| All cluster objects (Pods, Deployments, etc.) | etcd, via apiserver |
| Cluster credentials (CA cert, sa-token) | etcd (Secret resources) |
| Service VIP → endpoint table | kube-proxy on each node (or eBPF map) |
| DNS records | CoreDNS pod; reads Services/EndpointSlices from apiserver |
| Pod logs | /var/log/pods/ on each node |
| Image cache | /var/lib/containerd/ on each node |
| Volume data | depends on CSI driver (EBS, EFS, Ceph, etc.) |
| Audit logs | wherever apiserver writes (file, webhook, fluentd, etc.) |

There is no central "cluster state database" beyond etcd; all running state is reconstructable from the desired state.

## 2.9 The skew window — N, N-1, N-2

K8s version skew policy:
- **apiserver:** must be the newest.
- **controller-manager / scheduler / cloud-controller-manager**: up to 1 minor version older.
- **kubelet:** up to 2 minor versions older than apiserver (some recent releases extend to 3).
- **kubectl:** up to 1 minor version older OR newer.

So if apiserver is 1.30, you can run 1.28 kubelets. A feature that requires kubelet support won't be usable on 1.28 nodes. This drives the **alpha → beta → GA** lifecycle: features need to soak for 2 minor versions before being safe to assume.

## 2.10 The control loop, abstracted

```
   for {
       observe actual_state  (via informer cache)
       compute desired_state (from spec)
       diff
       apply (apiserver PATCH/POST/DELETE)
       sleep until next event (work queue blocks)
   }
```

This is THE pattern. Every k8s component is some variation of this. Operators are this. Cloud-provider integrations are this. Schedulers are this.

The corollary: every controller must be **idempotent** (running twice = once) and **eventually consistent** (no atomicity across multiple resources without finalizers / two-phase commit).

## 2.11 Failure modes (cluster-wide)

| Failure | Symptom | Recovery |
|---------|---------|----------|
| etcd minority down | Read-only or degraded writes | Wait for majority recovery |
| etcd majority down | Cluster frozen | Restore from snapshot |
| etcd disk full | All writes fail | Compact + defrag; resize disk |
| apiserver overload | 429s / 504s | APF tuning, scale-out apiserver |
| Network partition (control plane vs nodes) | Pods keep running locally; eventually marked NotReady | Convergence on partition heal |
| Controller-manager down | New controllers don't reconcile (e.g., new Deployments don't create RS) | Restart; another replica takes leadership via Lease |
| Scheduler down | New pods stay Pending | Restart |
| kubelet down on a node | Pods on that node stuck (eventually evicted by node-lifecycle controller after `unreachable` taint) | Restart kubelet |
| CNI plugin failure | New pods fail to start | Restart CNI daemon; debug allocator |
| Container runtime crash | Existing containers die; new ones fail | Restart containerd |

## 2.12 Common interview probes

- **"Walk me through kubectl apply."** Use the §2.4 flow. Mention every named component.
- **"Why is k8s eventually consistent?"** Reconcile loops; no transactions across resources; finalizers for ordered teardown.
- **"What's the difference between Deployment and StatefulSet?"** Owner refs, pod identity, ordered rollout, headless service requirement, PVC template.
- **"How does kubelet talk to the runtime?"** CRI over gRPC on a Unix socket.
- **"What's APF?"** API Priority and Fairness — replaces simple max-in-flight; categorizes requests into FlowSchemas, queues by PriorityLevel, ensures fairness.
- **"What happens if a pod's node goes away?"** kubelet stops heartbeating → node-lifecycle controller waits `--node-monitor-grace-period` (40s default) → marks NotReady → adds `node.kubernetes.io/unreachable:NoExecute` taint → pod's tolerations decide → after `tolerationSeconds` (300 default), evicted.

## Must-internalize

- The five rules: apiserver-only-etcd; declarative; watch-driven; pods atomic; cache-not-trusted-for-writes.
- The control plane: apiserver, etcd, controller-manager, scheduler, cloud-controller-manager.
- The data plane: kubelet, container runtime (CRI), kube-proxy or eBPF, CNI.
- The kubectl-apply 10-step flow.
- Watch streams, resourceVersion, watch cache, bookmark events.
- The skew policy: apiserver newest; kubelet up to 2 minor older.
- The control loop is the universal pattern.
