# 23 · Staff Topics and Cheatsheets

The cross-cutting "what does staff look like" file + the night-before flashcards.

## 23.1 Scope and leadership signals

Staff SWE in a Kubernetes role is usually:

- **Owner of a sub-system** (the apiserver scaling work; the CSI driver suite; the operator framework; the cost-optimization operator).
- **Maintainer of an OSS component** (Cilium component, custom operator, contributor to k/k or sigs).
- **Architecture reviewer** for k8s-adjacent changes — when a senior engineer proposes a CRD, you ask: "what's the conversion strategy? finalizer plan? rate-limit story?"
- **Bridge** — between application teams (think in Deployments) and platform teams (think in CRDs).

## 23.2 KEP literacy

KEPs are the lingua franca of k8s. Read these to internalize the project's design taste:

- **KEP-25** — CRDs
- **KEP-95** — Admission webhooks
- **KEP-127** — CRI
- **KEP-235** — CRD validation rules
- **KEP-365** — Watch list streaming
- **KEP-589** — NodeLease
- **KEP-624** — In-place pod resize
- **KEP-753** — Sidecar containers
- **KEP-1287** — In-place pod resize (renamed)
- **KEP-2433** — Topology-aware routing
- **KEP-2535** — DaemonSet scheduler integration
- **KEP-3157** — Efficient watch resumption
- **KEP-3094** — Pod scheduling readiness gates
- **KEP-3243** — Authentication structured config
- **KEP-3386** — Evented PLEG
- **KEP-3476** — Group volume snapshots
- **KEP-3691** — Stable PV node affinity
- **KEP-3744** — Extended kubelet version skew
- **KEP-4006** — Watch list pagination

A staff candidate reading a KEP can identify the unanswered questions (backwards compat, version skew, scale-test, security implications).

## 23.3 Migration / deprecation campaigns

Staff-level project examples:
- iptables kube-proxy → IPVS → Cilium eBPF.
- PSP → PSA.
- Endpoints → EndpointSlices.
- In-tree storage plugins → CSI drivers.
- CRD v1beta1 → v1.
- Service ipFamilies single-stack → dual-stack.

Execution pattern:
1. **Build parallel path**; don't yank old.
2. **Per-cluster rollout** with rollback.
3. **Telemetry** to compare; declare old dead when metrics agree.
4. **Document** for posterity.

## 23.4 The 60-second elevator pitches (memorize)

### K8s architecture

> "Cluster state in etcd (Raft, MVCC). kube-apiserver is the only thing that talks to etcd; it exposes a versioned REST API, runs admission (authn → authz → mutating → validating → quota), writes, broadcasts via watch. Controllers (in kube-controller-manager, plus operators) watch via informers (local cache + work queue + reconcile), reconcile desired vs actual. Scheduler watches unscheduled pods, Filter→Score→Bind via plugin framework, writes spec.nodeName. On each node, kubelet talks to a CRI runtime (containerd/CRI-O), uses CSI for volumes, CNI for networking, cgroups for limits. kube-proxy or eBPF (Cilium) implements Service VIPs. The whole design is eventually consistent, watch-driven, idempotent reconciliation."

### Reconcile loop

> "Informer watches a resource, dispatches add/update/delete to a deduplicated work queue. Worker pops, fetches current state from cache, computes desired state, writes diff to apiserver. Idempotent — running twice = once. Work queue handles dedup + rate-limit + exponential backoff. Leader election via Lease for HA controllers. Never trust cache for writes; always issue PATCH with version-checked optimistic concurrency."

### etcd

> "Raft-replicated KV store (3 or 5 voters; tolerates ⌊n/2⌋ failures). MVCC: every write a new revision; watch streams are revision-range subscriptions. Lease for ephemeral keys (node heartbeats, leader election). 8GB DB limit; events bloat; mitigations are compaction, defrag, separate events etcd, KMS v2 for Secrets encryption."

### CRDs and operators

> "CRD registers a new resource type with apiserver. Operator = controller + CRD + domain knowledge. Schema validation via OpenAPI + CEL. Status subresource for separate spec/status RBAC. Finalizers turn delete into a workflow. Owner references for GC cascade. controller-runtime + kubebuilder is the canonical Go framework."

### Scaling

> "5000 nodes / 150K pods / 110 pods/node / 10K services iptables. Bottlenecks: etcd write throughput, watch fanout, PLEG poll, iptables linear scan. Push past by: separate events etcd, IPVS/eBPF for Services, evented PLEG, multiple apiserver replicas with APF tuning, multi-cluster for >5000 nodes."

### Networking

> "Pod IP unique cluster-wide, no NAT pod-to-pod. CNI ADDs with netns path → veth + IP. CNI plugins: flannel (overlay), Calico (BGP-routed), Cilium (eBPF), AWS VPC CNI (cloud-native). Services: ClusterIP (iptables/IPVS/eBPF), NodePort, LoadBalancer (cloud LB), ExternalName. EndpointSlices shard endpoints; topology-aware routing hints. DNS via CoreDNS; ndots:5 gotcha; NodeLocalDNS for caching."

### Security

> "Authn: x509 / SA token (JWT) / OIDC / webhook → UserInfo. Authz: RBAC primary + Node + Webhook; first-allow wins. Admission: mutating → validation → validating, with VAP (CEL) replacing simple webhooks. PSA (Pod Security Admission) at namespace label: Restricted profile by default. Secrets: KMS encryption at rest; ESO or CSI Secret Store for external. Image signing via cosign at admission."

## 23.5 Cheatsheet — defaults to memorize

```
NodeMonitorGracePeriod (controller-manager)       40s
NodeUnreachable toleration (pod default)          300s
PodGracefulTerminationPeriod (default)            30s
tcp_keepalive_time                                7200s (!)
kubelet sync period                               1s
PLEG relist period                                1s (now evented)
SchedulingFramework percentage of nodes to score  50% (small), 5% (large)
HPA sync period                                   15s
HPA scale-down stabilization window               300s
LeaseDuration (default controllers)               15s
LeaseRenewDeadline                                10s
LeaseRetryPeriod                                  2s
nf_conntrack_tcp_timeout_established              5 days (tune to 1d)
nf_conntrack_max                                  default ~256K (tune up)
etcd auto-compaction-retention                    8h (recommended)
etcd quota-backend-bytes                          2GB (recommended 8GB max)
container_log_max_size                            10Mi
container_log_max_files                           5
PodMaxPidsLimit                                   default unlimited
maxPods per node                                  110 (default)
somaxconn                                         4096 (modern)
tcp_rmem.max                                      6MB (raise for cross-region)
```

## 23.6 Cheatsheet — ports

| Service | Port | Protocol |
|---------|------|----------|
| kube-apiserver | 6443 | TCP/TLS |
| etcd client | 2379 | TCP/TLS |
| etcd peer | 2380 | TCP/TLS |
| kubelet API | 10250 | TCP/TLS |
| kubelet read-only | 10255 | TCP (deprecated) |
| kube-proxy | 10256 | TCP (healthz) |
| kube-scheduler | 10259 | TCP/TLS |
| kube-controller-manager | 10257 | TCP/TLS |
| NodePort range | 30000-32767 | TCP/UDP |
| CoreDNS | 53 | UDP/TCP |
| Calico BGP | 179 | TCP |
| Calico Typha | 5473 | TCP |
| Cilium agent | 4244 | TCP (hubble) |
| Service Mesh (Istio) | 15001/15006 | TCP (sidecar) |

## 23.7 Cheatsheet — RBAC verbs

```
get          Read one
list         Read many
watch        Stream changes
create       Create
update       Replace
patch        Partial update
delete       Delete one
deletecollection  Delete many
proxy        Connect via apiserver to a Service/Pod
escalate     Create role with more permissions than self (special)
bind         Bind role to subject (special)
impersonate  As-another-user header
```

## 23.8 Cheatsheet — common commands

```bash
# Cluster info
kubectl cluster-info
kubectl get componentstatuses

# Debugging pod
kubectl describe pod <pod>
kubectl logs <pod> -c <container> --previous
kubectl exec -it <pod> -- /bin/sh
kubectl debug -it <pod> --image=busybox --target=<container>
kubectl get events --sort-by='.lastTimestamp' -A

# Auth check
kubectl auth can-i create pods --as=alice
kubectl auth can-i list secrets --as=system:serviceaccount:default:my-sa

# Resource state
kubectl top pod
kubectl top node
kubectl get pdb -A
kubectl get crd

# Drain / cordon
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
kubectl cordon node-1
kubectl uncordon node-1

# Rolling restart
kubectl rollout restart deployment/<name>
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>

# SSA
kubectl apply --server-side -f manifest.yaml --field-manager=my-tool

# Wait
kubectl wait --for=condition=Ready pod/my-pod --timeout=120s

# port-forward / proxy
kubectl port-forward pod/<pod> 8080:80
kubectl proxy --port=8080
```

## 23.9 Cheatsheet — etcd ops

```bash
# Health
ETCDCTL_API=3 etcdctl endpoint health --endpoints=$ENDPOINTS

# Status
etcdctl endpoint status --write-out=table

# Backup
etcdctl snapshot save /backup/etcd-$(date +%F).db

# Compact
REV=$(etcdctl endpoint status --write-out=fields | grep '"Revision"' | awk '{print $3}')
etcdctl compact $REV

# Defrag
etcdctl defrag

# DB size
etcdctl endpoint status --write-out=fields | grep dbSize
```

## 23.10 Cheatsheet — performance metrics

```
apiserver_request_total{code,verb,resource,subresource}
apiserver_request_duration_seconds_bucket{...}
apiserver_dropped_requests_total{requestKind,priorityLevel}
apiserver_flowcontrol_request_concurrency_limit
apiserver_flowcontrol_current_executing_requests
etcd_disk_wal_fsync_duration_seconds_bucket
etcd_disk_backend_commit_duration_seconds_bucket
etcd_server_proposals_pending
etcd_server_leader_changes_seen_total
etcd_mvcc_db_total_size_in_bytes
kubelet_pleg_relist_duration_seconds_bucket
kubelet_pod_start_duration_seconds_bucket
kubelet_runtime_operations_duration_seconds_bucket
kube_pod_status_phase
kube_pod_container_status_restarts_total
kube_deployment_status_replicas_available
container_cpu_usage_seconds_total
container_memory_working_set_bytes
```

## 23.11 Cheatsheet — annotations to memorize

```yaml
# Workload Identity
eks.amazonaws.com/role-arn: arn:aws:iam::123:role/my-role
iam.gke.io/gcp-service-account: gsa@project.iam.gserviceaccount.com

# Scheduler
scheduler.alpha.kubernetes.io/critical-pod: ""

# Resource
cluster-autoscaler.kubernetes.io/safe-to-evict: "true"

# Service mesh
sidecar.istio.io/inject: "true"
linkerd.io/inject: "enabled"

# Cilium
io.cilium/global-service: "true"

# Ingress
nginx.ingress.kubernetes.io/rewrite-target: /

# StatefulSet retention
apps.kubernetes.io/pod-index: "0"

# Prometheus discovery
prometheus.io/scrape: "true"
prometheus.io/port: "9090"
```

## 23.12 Cheatsheet — common cluster failure → first command

| Symptom | First command |
|---------|---------------|
| All pods Pending | kubectl describe pod | scheduler logs |
| New deployments hang | kubectl rollout status; check apiserver logs |
| Random 504s from API | check etcd_disk_wal_fsync; APF metrics |
| Node NotReady | kubelet logs; describe node conditions |
| Pod CrashLoopBackOff | kubectl logs --previous; describe pod events |
| PVC Pending | kubectl describe pvc; check StorageClass |
| Service unreachable | kubectl get endpointslice; kube-proxy logs |
| DNS slow | nslookup; CoreDNS metrics; check ndots |
| Memory pressure evictions | kubectl describe node; check threshold |
| HPA stuck | kubectl describe hpa; metrics-server health |
| Webhook errors | kubectl describe validatingwebhookconfiguration |

## 23.13 Cheatsheet — must-know objects + their controllers

| Object | Owning controller |
|--------|-------------------|
| Pod (bound) | kubelet |
| ReplicaSet | replicaset-controller (in KCM) |
| Deployment | deployment-controller |
| StatefulSet | statefulset-controller |
| DaemonSet | daemonset-controller |
| Job | job-controller |
| CronJob | cronjob-controller |
| Service (headless) | endpointslice-controller |
| Endpoints/EndpointSlice | endpoint(slice)-controller |
| PersistentVolume | pv-controller (binder) |
| PersistentVolumeClaim | pv-controller (binder) |
| Node | node-lifecycle-controller |
| Namespace | namespace-controller |
| HorizontalPodAutoscaler | hpa-controller |
| ServiceAccount | serviceaccount-controller |
| Secret (sa-token) | service-account-token-controller |
| Event | (none; written by clients) |

## 23.14 Behavioral / leadership prep

Have ~6 STAR stories ready:

1. **An upstream contribution** — even a small one; describe the review process.
2. **A migration** — old → new; cluster autoscaler → karpenter, e.g.
3. **An incident** — pages at 2am; how you led.
4. **A design call** — chose A over B; defend with trade-offs.
5. **A mentee** — helped someone grow.
6. **A failure** — what you learned.

Each: Situation, Task, Action, Result. 90s each.

## 23.15 Three preparation principles (recap)

1. **Run a cluster.** Don't just read docs.
2. **Read one KEP a week.** Internalize the design vocabulary.
3. **Build a CRD + operator.** Once cold; build deep familiarity with controller-runtime.

## 23.16 Final reading list

- Kubernetes docs (kubernetes.io)
- KEP tree (github.com/kubernetes/enhancements)
- *Kubernetes in Action, 2nd ed.* (Lukša)
- *Programming Kubernetes* (Hausenblas + Schimanski)
- *Production Kubernetes* (Dotson et al.)
- *Cloud Native Patterns* (Davis)
- sig-scalability docs (SLOs)
- KubeCon talks: search "operator" / "controller-runtime" / "scaling"
- Lyft, Spotify, Pinterest, OpenAI engineering blogs

## Must-internalize

- The 60-second pitches in this file — recite cold.
- The defaults, ports, RBAC verbs, common commands.
- 6 STAR stories ready, 90s each.
- The metric names for apiserver / etcd / kubelet.
- For each topic file in this pack: the 1-2 must-internalize lines.
- The grader watches for **specificity** (named components, named limits, named alternatives) more than perfect answers.

---

You've got this. Re-read 02 + 05 + 06 the day before. Build a CRD + operator once. Show up rested.
