# 19 · Case Studies and Realistic Scenarios

Production stories from companies running Kubernetes at scale. Memorize at least one per category — concrete numbers and lessons make staff answers vivid.

## 19.1 Lyft — Envoy + Kubernetes (the service mesh origin story)

**Context**: Lyft built Envoy (2016) because their monolith was splitting into hundreds of services and they needed mesh networking before "service mesh" was a term.

**Stack**:
- ~hundreds of microservices.
- Envoy as sidecar before Istio existed.
- Custom control plane (Envoy xDS).
- Eventually adopted Istio.

**Lesson**: Service mesh observability is invaluable at high microservice count. The sidecar overhead (per-pod) was worth it for the auto-instrumentation.

**Quote-worthy**: "The hardest problem with microservices isn't the services — it's the network between them."

## 19.2 Spotify — Backstage and the developer platform

**Context**: Spotify runs ~2000 microservices on k8s. They built Backstage to give developers a curated platform UI.

**Stack**:
- Multi-cluster (different regions, different teams).
- GKE (managed k8s).
- Backstage CRDs for services, components, APIs.
- Custom operators on top.

**Lesson**: Platform engineering matters as much as the runtime. Developers shouldn't write raw YAML; they should consume a higher-level API.

**Quote-worthy**: "We have more YAML than code."

## 19.3 Pinterest — Karpenter adoption (the cost optimization story)

**Context**: Pinterest's EKS bill was massive. Cluster Autoscaler with rigid node groups wasn't optimal.

**Move**: Adopted Karpenter (2022-2023).

**Results**:
- ~50% EC2 cost reduction.
- Consolidation kept clusters at high utilization.
- Spot adoption increased; diverse instance types reduced spot interruption impact.

**Lesson**: The default scaling story (CA + node groups) leaves money on the table. Karpenter's just-in-time + bin-packing is materially better.

## 19.4 Airbnb — etcd separation (the scaling story)

**Context**: Airbnb's large clusters were hitting etcd write throughput limits.

**Move**:
- Separated events to dedicated etcd cluster (`--etcd-servers-overrides`).
- Tuned `--quota-backend-bytes` up.
- Cron-driven defrag.
- Multiple apiserver replicas behind LB.

**Lesson**: etcd is the universal scaling pinch point. Splitting events is the first staff-level move.

## 19.5 Datadog — Cilium and high-cardinality (the observability scaling story)

**Context**: Datadog runs k8s for their own backend (a meta-cluster: k8s for their k8s monitoring SaaS).

**Stack**:
- Cilium for everything (network policy, kube-proxy replacement, ClusterMesh).
- Datadog Agent DaemonSet on every node.
- High-cardinality metrics; need very careful labeling.

**Lesson**: At hyperscale, cardinality is the bottleneck. Cilium's eBPF approach was the only way to get O(1) Service VIP lookup.

## 19.6 GitHub Actions — Kubernetes runner scaling (the ephemeral workload story)

**Context**: GitHub Actions runners are ephemeral pods that run a CI job and exit.

**Stack**:
- ~hundreds of clusters globally.
- Each runner = one Pod.
- Pod startup latency matters (developer experience).
- High pod churn (millions per day).

**Lessons**:
- Image lazy-load (stargz) saves seconds per startup.
- Pod startup latency drove image-size discipline.
- Bin-packing tighter for cost; over-provisioning gives latency.

## 19.7 OpenAI — training cluster scale (the hyperscale story)

**Context**: OpenAI trains models on clusters of thousands of nodes (each with multiple GPUs).

**Stack**:
- 7500-node clusters reported.
- Custom apiserver patches.
- GPU operator + driver + nvidia-container-runtime.
- Job + StatefulSet for distributed training.
- High-bandwidth interconnect (NVLink, InfiniBand).

**Lessons**:
- K8s scales beyond official limits with patches.
- GPU scheduling needs awareness (NUMA, topology, MIG, NVLink).
- Storage: shared file system (Ceph, Lustre) + per-job ephemeral.

OpenAI's blog on scaling to 7500 nodes is required reading.

## 19.8 Tesla — multiple-AZ and edge (the geographic distribution story)

**Context**: Tesla runs k8s for backend + for in-vehicle services.

**Stack**:
- Multi-region clusters for backend.
- Edge clusters (single-node or 3-node) in factories and data centers.
- Cluster API + GitOps for cluster lifecycle.

**Lessons**:
- Edge needs a lightweight k8s (k3s, microk8s).
- Standardize the management plane; let the data plane vary by location.

## 19.9 Reddit — Helm and the configuration story

**Context**: Reddit migrated from Mesos to k8s in 2019-2021.

**Stack**:
- Argo CD + Helm.
- Custom helm chart library for shared patterns.
- Strict CI on chart updates.

**Lessons**:
- Helm + GitOps is the dominant config story.
- Custom-helm-chart-library = platform engineering discipline.
- Without strict CI, helm chart drift kills production.

## 19.10 Real war stories — incidents

### Incident: kubelet certificate expired
**Symptom**: Cluster appeared healthy; new pods couldn't start (auth failures); existing pods running. After ~12h, etcd full of failed sa-token requests.

**Root cause**: kubelet's serving cert expired; controller-manager couldn't reach kubelet's /metrics; PLEG events still flowed but heartbeats started failing.

**Fix**: Rotate kubelet certs. Use cert-manager's CSR signing.

**Lesson**: Cert expiry hits all of k8s eventually. Monitor expiry; rotate automatically.

### Incident: etcd disk full → cluster frozen
**Symptom**: All apiserver writes returning 500. Pods stuck. Logs full of `mvcc: database space exceeded`.

**Root cause**: Misbehaving controller emitting thousands of events per second; etcd hit 8GB; no compaction.

**Fix**:
1. Add disk capacity.
2. Compact + defrag.
3. Fix the controller.
4. Route events to separate etcd.

**Lesson**: Event spam is a real production threat. Set `--event-rate-limit-config`; alert on etcd db size > 4GB.

### Incident: webhook outage → cluster broken
**Symptom**: Can't create any pod (or other resource); apiserver returns "validating webhook X is unreachable."

**Root cause**: A namespace-mutating webhook with `failurePolicy: Fail` had its pod down. The webhook itself was a pod — but it required webhook approval to start (chicken-and-egg).

**Fix**:
1. Set `failurePolicy: Ignore` temporarily.
2. Fix the webhook.
3. Configure `namespaceSelector` to exclude kube-system from this webhook.

**Lesson**: Admission webhooks are critical infrastructure. `failurePolicy: Fail` is dangerous; exclude kube-system always; have a recovery plan.

### Incident: misconfigured NetworkPolicy → DNS dead
**Symptom**: Pods can't resolve any hostname; everything broken; users can't even see logs of why.

**Root cause**: A "deny all egress" NetworkPolicy applied to namespace; no allow for DNS.

**Fix**: Add egress rule for kube-system/CoreDNS.

**Lesson**: Always allow DNS in restrictive policies. Test policies in non-prod first.

### Incident: HPA scaled wrong direction during incident
**Symptom**: Production downtime during a backend issue; HPA scaled UP its target Deployment because pods were starting but failing, signaling high CPU because of startup work.

**Root cause**: HPA based on CPU; under failure, pods used more CPU; HPA decided more pods needed; more pods → more load on broken backend → cascading.

**Fix**:
1. HPA stabilization window (don't scale during instability).
2. Custom metric: scale on RPS or queue depth, not CPU.
3. Circuit breaker on the upstream call.

**Lesson**: CPU-based HPA can do the wrong thing during failures. Custom metrics + stabilization windows.

### Incident: PVC volume zone mismatch after node replacement
**Symptom**: After node maintenance, pod stuck in `ContainerCreating`. Event: "failed to attach volume; volume in different zone."

**Root cause**: PVC was provisioned with `Immediate` binding mode in zone us-east-1a. Node-replacement put new node in 1b. Volume couldn't follow.

**Fix**: Switch StorageClass to WaitForFirstConsumer for future PVCs; manually attach to a node in 1a or migrate data.

**Lesson**: Always use WaitForFirstConsumer in multi-zone clusters.

### Incident: kube-proxy iptables thrash during deploy
**Symptom**: 30 seconds of broken service VIPs during a deployment that recreated many endpoints.

**Root cause**: 1000-pod deployment → endpointslice churn → kube-proxy iptables rebuild each change → temporary inconsistency.

**Fix**:
1. Move to IPVS mode.
2. Or move to Cilium eBPF.
3. Or stagger the rollout.

**Lesson**: At >1000 services or >100-pod single deploy, iptables mode hurts.

### Incident: a CRD update broke an operator
**Symptom**: Operator pods crashing with type errors; their CR objects no longer reconciled.

**Root cause**: CRD's `apiextensions.k8s.io/v1beta1` was removed in k8s 1.22; the operator's CRD was still on v1beta1.

**Fix**: Update CRD to v1; update operator's generated client.

**Lesson**: Pluto/kubent scan before upgrades. Have a CRD versioning plan.

## 19.11 Production tropes — patterns that always work

| Pattern | Why |
|---------|-----|
| **GitOps (Argo CD / Flux) for everything** | Reproducibility; audit trail; rollback |
| **Multiple clusters, federated config** | Blast radius limited |
| **Cert-manager for cert lifecycle** | Webhook + ingress certs auto-rotate |
| **External Secrets Operator for prod secrets** | Vault-backed; no etcd secrets to leak |
| **Karpenter for cost** | 30-50% savings over CA |
| **Cilium for net + policy + mesh** | One tool, eBPF-native, ambient mesh |
| **OpenTelemetry Collector DaemonSet** | One agent for metrics + logs + traces |
| **PSA `restricted` at namespace level** | Security default |
| **PDB on every prod Deployment** | Protect from scale-down |
| **Velero for backup** | DR baseline |

## 19.12 Production tropes — things teams regret

| Anti-pattern | Why bad |
|--------------|---------|
| **Default ServiceAccount with cluster-admin** | Compromised pod = compromised cluster |
| **Self-signed CA without rotation** | Cert expiry incidents |
| **Manual cluster ops (no GitOps)** | Drift; can't reproduce |
| **Custom CRDs without conversion webhooks** | Upgrade hell |
| **Putting prod secrets in etcd as plain Secret** | Easy to leak |
| **No PDB on production** | Drains kill availability |
| **kube-proxy iptables at 5000-services scale** | Reload thrash |
| **One big cluster** | Blast radius |
| **CPU limits on latency-sensitive services** | CFS throttling tail |
| **Webhook with failurePolicy: Fail + no exclusion** | Cluster brick risk |
| **HPA on CPU only, no custom metric** | Wrong direction during failure |

## 19.13 Common interview probes

- **"Describe a production K8s incident you debugged."** Use a real story or paraphrase one from this file. Hit: symptom → triage → root cause → fix → lesson.
- **"How would you prevent X from happening again?"** Concrete: monitoring, alerting, policy enforcement, runbook updates.
- **"What's the worst K8s production failure you can imagine?"** Cluster-wide cert expiry; etcd disk full + DR lost; webhook brick; control-plane outage during AZ failure. Walk through containment.
- **"Tell me about a CRD migration."** Multiple versions; conversion webhook; storage version bump; verify with kubent.
- **"How would you migrate from CA to Karpenter?"** Side-by-side: Karpenter handles new workloads; CA continues for existing. Cut over once metrics confirm Karpenter's behavior. Then disable CA.

## 19.14 Real numbers (for sizing answers)

| Workload | Numbers |
|----------|---------|
| Lyft | hundreds of services, ~thousands of pods peak |
| Spotify | ~2000 microservices, 10s of clusters |
| Pinterest | ~10 clusters; reportedly 1000s of nodes after optimization |
| Reddit | ~hundreds of services post-migration |
| Google | thousands of clusters; some at 5000-15000 nodes |
| OpenAI | 7500-node training clusters |
| Anthropic | also large training clusters; specific numbers undisclosed |
| GitHub Actions | millions of ephemeral pods per day |
| Cloudflare | many small clusters at edge POPs (300+) |

## 19.15 The narrative arc — telling a war story in 90 seconds

For interview:
1. **Context (15s)**: "Cluster size, what was running."
2. **Symptom (15s)**: "What broke; how it was discovered."
3. **Triage (20s)**: "What we checked first; what ruled it out."
4. **Root cause (15s)**: "The actual cause."
5. **Fix (15s)**: "Immediate + long-term."
6. **Lesson (10s)**: "What changed permanently."

Practice 3-5 of these out loud.

## Must-internalize

- One war story per category: scaling (Airbnb etcd), cost (Pinterest Karpenter), networking (Lyft Envoy), security (cert expiry), webhook brick, NetPolicy DNS, HPA wrong direction, PVC zone mismatch, CRD upgrade.
- Production patterns: GitOps + Cert-manager + ESO + Karpenter + Cilium + OTel.
- Anti-patterns: default-SA admin, manual ops, CPU limits, big cluster, webhook fail policy.
- Practice the 90-second narrative structure.
