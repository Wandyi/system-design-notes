# 20 · Corner Cases and Alternative Solutions

The "name three ways to solve this" file. For each common K8s problem: the naive solution, what breaks, then 2-3 alternatives with trade-offs.

## 20.1 Pattern: Pod stuck in `Terminating`

**Naive**: wait and hope.

### Approach A: Identify and resolve the finalizer
```bash
kubectl get pod my-pod -o yaml | grep -A3 finalizers
# Find the finalizer; investigate why owning controller can't remove it.
```

### Approach B: Force-remove finalizer (last resort)
```bash
kubectl patch pod my-pod -p '{"metadata":{"finalizers":null}}'
```
Object then disappears. External resources may leak; cleanup may be incomplete.

### Approach C: Force delete the Pod
```bash
kubectl delete pod my-pod --force --grace-period=0
```
Removes from etcd immediately. Pod may still be running on the node (kubelet has no way to "ungracefully" know). The node will clean up eventually.

### Approach D: Restart the kubelet on the node
If the issue is kubelet not processing the termination event.

### Selection rubric
| Situation | Choice |
|-----------|--------|
| Operator-managed; controller alive | Wait + investigate |
| Operator dead; cleanup external | Manual cleanup + remove finalizer |
| Emergency / non-critical | Force delete |

## 20.2 Pattern: CrashLoopBackOff with no useful logs

### Approach A: Add `command: ["sleep", "infinity"]` temporarily
Edit the Pod template; container won't crash; `kubectl exec` into it; investigate filesystem, env, configs.

### Approach B: `kubectl debug` with ephemeral container
```bash
kubectl debug -it my-pod --image=busybox --target=my-container
```
Joins the same Pod namespace; can inspect process tree, network.

### Approach C: Check `kubectl get events` and `kubectl describe pod`
Often reveals: image pull failed, init container failed, mount failed, OOMKilled.

### Approach D: Adjust startupProbe
If the app needs more time to initialize; default may declare unhealthy too fast.

### Approach E: Add `securityContext` to log debugging info
Allow capabilities (NET_RAW for tcpdump in-pod, etc.) only in non-prod.

## 20.3 Pattern: Apiserver returning 504s under load

### Approach A: Tune API Priority and Fairness
Bump FlowSchema priorities for control-plane components; cap experimental controllers.

### Approach B: Add apiserver replicas
Horizontal scale. Update LB to include new replicas.

### Approach C: Reduce watch fan-out
Bundle multiple controllers in fewer processes; use SharedInformerFactory.

### Approach D: Diagnose etcd
Check `etcd_disk_wal_fsync_duration` — if etcd is slow, apiserver is too. Move to faster disk.

### Approach E: Rate-limit specific clients
APF FlowSchema with low concurrency for known-hot clients.

## 20.4 Pattern: Service VIPs are flaky (5-10% requests fail)

### Approach A: Check EndpointSlice readiness
```bash
kubectl get endpointslice -n my-ns -l kubernetes.io/service-name=my-svc -o yaml
```
Pods with `ready: false` shouldn't be in the rotation but may be lingering.

### Approach B: Inspect kube-proxy logs
iptables/IPVS programming errors; sync lag.

### Approach C: Switch from iptables to IPVS or eBPF (Cilium)
If scale-related (>5K services), the iptables linear scan adds latency tails.

### Approach D: Check conntrack table
`conntrack -S`; if `insert_failed` is rising, conntrack is full.

### Approach E: NetworkPolicy debugging
Wrong policy could be silently dropping. Hubble (Cilium) shows drops with reason.

## 20.5 Pattern: HPA flapping

### Approach A: Tune stabilization windows
```yaml
behavior:
  scaleDown: {stabilizationWindowSeconds: 600}
  scaleUp:   {stabilizationWindowSeconds: 30}
```

### Approach B: Change metric to custom (RPS, queue depth)
CPU-based can flap during load patterns; RPS more stable.

### Approach C: Add behavior rate limits
Cap scale-down to 10% per minute.

### Approach D: KEDA with cooldown
For event-driven; longer cooldownPeriod.

### Approach E: Reduce target utilization
Run at 60% target instead of 80%; less likely to thrash near edges.

## 20.6 Pattern: PVC stuck in `Pending`

### Approach A: Check StorageClass exists
```bash
kubectl get storageclass
```
Default missing? `kubectl annotate sc <name> storageclass.kubernetes.io/is-default-class=true`.

### Approach B: Capacity available?
Cloud quota; CSI driver health.

### Approach C: Volume binding mode mismatch
If `Immediate` and pod is in different zone; switch to `WaitForFirstConsumer`.

### Approach D: Resource quota on PVC
Check namespace's ResourceQuota; PVCs may be capped.

### Approach E: Check CSI external-provisioner pod logs
The provisioner may be erroring.

## 20.7 Pattern: Etcd database growing without bound

### Approach A: Identify the source
```bash
ETCDCTL_API=3 etcdctl get --prefix --keys-only / | sort | uniq -c | sort -rn | head
```
Often: Events bloating, ConfigMap reconcile loop, leftover orphan resources.

### Approach B: Compact + defrag
```bash
etcdctl compact $(etcdctl endpoint status --write-out=fields | grep "\"Revision\"" | awk '{print $3}')
etcdctl defrag --endpoints=<each member>
```

### Approach C: Increase quota-backend-bytes
Temporary relief; up to 8GB. Not a long-term fix.

### Approach D: Split events to separate etcd
```yaml
apiserver:
  --etcd-servers-overrides=/events#http://events-etcd:2379
```

### Approach E: Audit ResourceQuotas
Cap number of objects per namespace.

## 20.8 Pattern: Pods being evicted at random

### Approach A: Check QoS class
BestEffort pods evicted first. Set requests on prod pods.

### Approach B: Check eviction thresholds
```bash
ps -ef | grep kubelet | grep eviction
```
Tune `--eviction-soft`/`--eviction-hard` if too aggressive.

### Approach C: Disk pressure?
Image cache filling node disk. Image GC config.

### Approach D: Memory pressure
Some workload using more than requested. VPA recommendation can help.

### Approach E: Preempted by higher priority?
Check PriorityClass. Use PDB to protect.

## 20.9 Pattern: Deployment rollout stuck

### Approach A: Check progressDeadlineSeconds
```bash
kubectl describe deployment my-app | grep Conditions
```
Look for `ProgressDeadlineExceeded`.

### Approach B: New pods crashing
Check the new ReplicaSet's pods; usually it's readiness probe failing or container crash.

### Approach C: PDB blocking
Old pods can't be evicted without violating PDB; rollout stuck.

### Approach D: Scale-down can't proceed
maxUnavailable too tight; new RS scales up but old can't scale down.

### Approach E: Rollback
```bash
kubectl rollout undo deployment/my-app
```

## 20.10 Pattern: DNS resolution slow / failing

### Approach A: Check CoreDNS logs and metrics
`kubectl logs -n kube-system -l k8s-app=kube-dns`. Look for query rate, errors.

### Approach B: Reduce ndots
Per pod:
```yaml
spec:
  dnsConfig:
    options:
    - name: ndots
      value: "1"
```
Or use FQDN with trailing dot: `google.com.`

### Approach C: Install NodeLocalDNS
DaemonSet caches DNS on each node; saves the round-trip to CoreDNS.

### Approach D: Check NetworkPolicy
Egress to kube-system:53 allowed?

### Approach E: Conntrack table full
`conntrack -S`; UDP conntrack entries can blow up.

## 20.11 Pattern: One namespace eating all cluster resources

### Approach A: ResourceQuota
```yaml
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    pods: "100"
```

### Approach B: LimitRange for per-pod defaults
Prevent pods with no requests.

### Approach C: PriorityClass with low value
Other tenants' workloads get scheduled first.

### Approach D: Run tenant in separate cluster
Hard isolation.

### Approach E: NetworkPolicy to limit traffic
Prevent tenant from saturating shared resources.

## 20.12 Pattern: Cluster Autoscaler can't scale up

### Approach A: Check unschedulable pods
```bash
kubectl get pods --field-selector status.phase=Pending
```

### Approach B: CA logs
```bash
kubectl logs -n kube-system -l app=cluster-autoscaler
```
"No node group" / "max size reached" / "cloud provider error."

### Approach C: Instance type fit
Pod requests too big for any configured node type.

### Approach D: ASG capacity
AWS quota; check service quotas.

### Approach E: Move to Karpenter
JIT provisioning bypasses node-group limits.

## 20.13 Pattern: Pod cannot reach external internet

### Approach A: NetworkPolicy egress
Explicit deny? Check policies.

### Approach B: NAT gateway / DNS
For private subnet: NAT gateway? DNS server reachable?

### Approach C: CNI dataplane health
Calico nodeStatus, Cilium status.

### Approach D: Service mesh egress
Istio: ServiceEntry needed for external access if `outboundTrafficPolicy: REGISTRY_ONLY`.

### Approach E: Cloud-specific egress (SNAT issues)
AWS EKS: pod gets ENI IP that may not be route-able to internet unless via NAT gateway.

## 20.14 Pattern: Statefulset stuck during rolling update

### Approach A: Check pod-1 ready
StatefulSet updates sequentially (default `OrderedReady`); pod-0 must become Ready before pod-1.

### Approach B: PVC corruption / data issue
Pod can't start because volume's data is bad. Check pod logs.

### Approach C: Manual intervention
Delete pod (StatefulSet recreates with new image).

### Approach D: Switch to parallel update
```yaml
spec:
  podManagementPolicy: Parallel
```
NOT default; loses ordering guarantees. Only for some workloads.

### Approach E: Partition for canary
`partition: 2` updates only pods 2+; 0,1 stay old until you decrement partition.

## 20.15 Pattern: Webhook latency adding 500ms to every write

### Approach A: Identify the slow webhook
```bash
kubectl get validatingwebhookconfigurations -o yaml
```
Look at `timeoutSeconds`; cluster metrics show webhook latency.

### Approach B: Move to ValidatingAdmissionPolicy (VAP)
CEL in-process; no network hop. 1.30 GA.

### Approach C: Add namespace selector to exclude hot paths
`namespaceSelector`: exclude kube-system, hot tenant namespaces.

### Approach D: Scale webhook deployment + cache
Multiple replicas; persistent connections; cached upstream queries.

### Approach E: Change `failurePolicy: Ignore`
Webhook errors don't fail writes; trade-off correctness for availability.

## 20.16 Pattern: Multi-tenant cluster — one tenant noisy

### Approach A: ResourceQuota + LimitRange
Per-tenant namespace cap.

### Approach B: PriorityClass tiers
Critical tenants have higher priority.

### Approach C: PodOverhead for RuntimeClass (gVisor for noisy tenant)
Isolate process via lightweight VM.

### Approach D: NetworkPolicy isolation
Tenant can't reach other tenants.

### Approach E: Separate node pool with taints
"Noisy tenant" tolerates a specific taint; only their pods land there.

## 20.17 Pattern: Need to upgrade k8s but stuck add-on

### Approach A: Check compat matrix per add-on
Sometimes there's a path: add-on N supports k8s X..Y.

### Approach B: Migrate add-on first to newer version
Most add-ons release before k8s major version; can pre-update.

### Approach C: Replace add-on
Sometimes the right answer: switch CNI from Calico to Cilium during the upgrade window.

### Approach D: Pin to N-2
Stay 2 minor versions back to give add-ons time.

### Approach E: Blue-green new cluster
Build new cluster with new k8s + new add-ons; migrate.

## 20.18 Pattern: ConfigMap update doesn't reach pod

### Approach A: ConfigMap as volume mount
Atomic updates propagate to pod automatically (within ~minute). But the pod must be reading from disk.

### Approach B: ConfigMap as env vars
Env vars NOT updated when ConfigMap changes; pod must restart.

### Approach C: Restart with annotation
```bash
kubectl rollout restart deployment my-app
```

### Approach D: ConfigMap watcher (Reloader operator)
Watches ConfigMap changes; triggers rollout of dependent deployments.

### Approach E: App watches its own config
App-level config watch (some apps support SIGHUP for config reload).

## 20.19 Pattern: Operator can't reconcile fast enough

### Approach A: Increase MaxConcurrentReconciles
```go
ctrl.NewControllerManagedBy(mgr).
    WithOptions(controller.Options{MaxConcurrentReconciles: 50}).
    Complete(r)
```

### Approach B: Shard operator across replicas
Leader election for active control; shard by namespace.

### Approach C: Reduce reconcile work
Cache more; only do expensive work on actual change.

### Approach D: Use status conditions to track state machine
Avoid redoing work; status reflects done sub-steps.

### Approach E: Move to event-driven
React only on watch events, not periodic reconciles.

## 20.20 Pattern: Need to canary deploy

### Approach A: Two Deployments + Service selector
v1 + v2; Service selects all (gets both); proportions controlled by replica count.

### Approach B: Service mesh weighted routing
Istio VirtualService 90/10; precise control.

### Approach C: Argo Rollouts
Native canary controller; metric-driven promotion.

### Approach D: Flagger
Similar; built on Flux.

### Approach E: Manual rollout with extra Service
Pre-deploy v2; flip Service selector at the right moment.

## 20.21 Pattern: Secrets management

### Approach A: Native Secrets with encryption at rest
Built-in; baseline.

### Approach B: External Secrets Operator
ESO syncs from Vault/AWS SM/GCP SM to k8s Secret.

### Approach C: Secret Store CSI Driver
Mounts secrets directly from external store; no k8s Secret created.

### Approach D: Sealed Secrets (Bitnami)
Encrypt at-rest in git; decrypted only by controller in cluster.

### Approach E: Vault Agent Sidecar
Inject secrets at pod start; rotate.

## 20.22 Pattern: Custom controller leaks memory

### Approach A: Profile
Go pprof on the controller's `/debug/pprof`.

### Approach B: Leaking informer
Restart on every reconcile? Each `factory.Start` leaks goroutines. Use one factory for the whole controller.

### Approach C: Holding object references
Cache may pin all observed objects; large clusters → big cache. Use field/label selectors.

### Approach D: Goroutine leak in reconcile
Reconciles spawning goroutines that never finish.

### Approach E: Add metric for cache size + goroutine count
Catch before OOM.

## 20.23 Pattern: Audit log volume too high

### Approach A: Tune policy
Reduce `RequestResponse` to `Metadata` for non-sensitive resources.

### Approach B: omitStages
`omitStages: ["RequestReceived"]` halves volume.

### Approach C: Sample
Webhook side can sample (e.g., only log 1% of `get` requests).

### Approach D: Filter at audit-webhook
Filter before forwarding to SIEM.

### Approach E: Drop non-prod from audit pipeline
Only ship prod audit to SIEM; dev to local disk.

## 20.24 Pattern: GPU node utilization low

### Approach A: GPU pod sharing (MPS, MIG)
NVIDIA Multi-Process Service or Multi-Instance GPU.

### Approach B: Time-slicing (nvidia GPU operator config)
Multiple pods sharing a GPU via time-slice.

### Approach C: Pack jobs with batch scheduler
Volcano for batch jobs; gang scheduling.

### Approach D: Different GPU types for different workloads
Inference (smaller GPUs) vs training (bigger).

### Approach E: Karpenter with GPU constraints
Dynamic provisioning fits the workload.

## 20.25 Selection rubrics (consolidated)

| Problem | First step | Persistent? |
|---------|-----------|--------------|
| Pod terminating | Investigate finalizer | Force-remove if controller dead |
| Apiserver overload | APF tuning | Scale + etcd disk + multi-cluster |
| Etcd full | Compact + defrag | Split events; reduce object count |
| Service flaky | Check EndpointSlice | Migrate to eBPF |
| HPA flap | Stabilization windows | Custom metric |
| PVC pending | StorageClass + zone | WaitForFirstConsumer |
| Eviction | QoS + thresholds | VPA right-sizing |
| Rollout stuck | progressDeadlineSeconds | Manual debug / rollback |
| DNS slow | NodeLocalDNS + ndots | FQDN + cache TTL |
| Tenant noisy | ResourceQuota | Separate cluster |
| Add-on incompat | Pin or replace | Blue-green |
| ConfigMap not refreshing | Volume mount + rollout | Reloader |
| Operator slow | Concurrency + sharding | Event-driven |
| Canary | Service mesh weighted | Argo Rollouts |
| Secrets | External Secrets / CSI | Vault Agent |

## Must-internalize

- For every problem in production, name **3** solutions.
- Cheapest fix is often application or namespace-level (ResourceQuota, FQDN, custom metric).
- Kernel-level fixes (eBPF, sysctls) come after exhausting app-level.
- "Force-remove finalizer" is a last resort with cleanup implications.
- Webhook brick: always have `failurePolicy: Ignore` for non-critical.
- Etcd full: compact + defrag + split events.
- Most "broken cluster" stories trace to: cert expiry, etcd full, webhook down, NetPolicy too strict.
