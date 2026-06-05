# 01 · Role and Interview Context

## Where K8s SWEs work — and what each looks like

"Interviewing on Kubernetes" can mean several different shops. The technical bar overlaps, but the *flavor* differs:

| Org type | Examples | Flavor |
|----------|----------|--------|
| **Upstream contributor at a vendor** | Red Hat, Google (GKE), Microsoft (AKS), AWS (EKS), VMware Tanzu, SUSE Rancher | Deep code; subscribe to SIG mailing lists; submit KEPs; on-call for managed k8s clusters |
| **CNCF project SWE** | Kubernetes project, Cilium, Karpenter, Crossplane, Knative | OSS process, contributor ladder, public review |
| **Hyperscaler managed-k8s team** | GKE / EKS / AKS / IBM Cloud | Build the control plane *as a service*; tenant isolation; auto-upgrade orchestration |
| **End-user infra team** | Lyft, Spotify, Pinterest, Airbnb, Stripe, GitHub | Run k8s at scale in production; build CRDs/operators on top; cost optimization |
| **K8s product company** | Datadog (k8s observability), Buoyant (Linkerd), Tetrate (Istio), Fairwinds | Sell k8s tooling; talk to many customers; deep on a narrow domain |

If you're at "Kubernetes" the upstream project: there is no traditional employer — most maintainers are paid by member companies. But the *project itself* funds interns and contributors via CNCF mentorship → eventually upstream maintainers. Most staff-level k8s positions are at one of the above categories.

## The interview loop

Typical staff loop (5–7 rounds, ~6 hours):

1. **Recruiter screen** (30 min) — comp, scope, team alignment.
2. **Hiring-manager technical screen** (45–60 min) — résumé deep dive, "what's your k8s war story," one shallow probe ("what's a Service?", or "trace kubectl apply").
3. **Coding** (1–2 × 60 min) — Go coding, often k8s-adjacent: build a tiny informer; implement an admission webhook; write a CRD reconciler.
4. **System design** (60 min) — design a k8s control-plane feature; design a multi-cluster service; design something k8s-specific (HPA replacement, custom scheduler).
5. **K8s deep dive** (60 min) — the technical anchor. Pick one: API server / etcd / scheduler / kubelet / CRDs / networking / scaling. Expect 3-layer-deep probing.
6. **Behavioral / bar-raiser** (45 min) — STAR stories. For CNCF/k8s: "describe a time you contributed upstream"; "how did you handle a maintainer pushback"; "what's a contentious technical decision you owned."
7. **Sometimes a take-home** — read a KEP and critique it; design an operator end-to-end; write a small CRD spec.

## What "staff" means in a Kubernetes role

A staff SWE on K8s is expected to:

- **Own a critical sub-system.** Examples: "I own the apiserver watch cache scaling work"; "I own the CSI snapshot subsystem"; "I own our cluster's cost-optimization operator suite."
- **Be the architecture review** for changes touching k8s. When a senior engineer proposes a CRD, you ask: "what's your conversion strategy? finalizer plan? rate-limit story? watch budget? RBAC surface?"
- **Read KEPs critically.** A staff candidate can read a 5-page KEP and identify the unanswered questions (backwards compatibility, version skew, scale-test plan).
- **Bridge** — translate between application teams (who think in Deployments) and platform teams (who think in CRDs and webhooks).
- **Have produced upstream impact** — at least a PR merged to k/k or a major OSS controller/operator. Not required for every role, but a strong positive signal.

## Calibration — Junior vs Staff answers

| Question | Junior | Staff |
|----------|--------|-------|
| **"How does a Service VIP work?"** | "It's an IP that load-balances to pods." | "Service objects have ClusterIP, NodePort, LoadBalancer, ExternalName modes. ClusterIP is implemented by kube-proxy: in iptables mode it's a DNAT rule per service+endpoint; in IPVS mode it's a real LB with WLC/SH algorithms; in Cilium eBPF mode it's a BPF map keyed by ServiceID. EndpointSlices (replacing Endpoints in 1.21+) shard the endpoint list so that a 1000-pod service doesn't blow up watch traffic. The DNS resolution for the VIP is via CoreDNS, which watches Service/EndpointSlice objects." |
| **"What's a Deployment?"** | "It manages pods." | "Deployment owns ReplicaSets; ReplicaSet owns Pods. The Deployment controller maintains the desired replica count and orchestrates rollouts (RollingUpdate or Recreate) by creating new ReplicaSets, scaling them up while scaling the old one down, respecting maxSurge / maxUnavailable. Revision history is kept (revisionHistoryLimit default 10) for rollback. The owner reference chain (Deployment → ReplicaSet → Pod) drives garbage collection: delete the Deployment and the kube-controller-manager's garbage-collector controller cascades. Pod hash labels (`pod-template-hash`) make ReplicaSet → Pod selector unambiguous." |
| **"What happens if etcd is slow?"** | "Cluster gets slow." | "Watch streams from the apiserver to controllers lag → controllers reconcile against stale cache → apiserver request latency rises (etcd lock contention) → eventually apiserver returns 504 to clients → operators retry with backoff. New pods don't get scheduled (scheduler can't update PodSpec.Node); kubelet heartbeats may be late, triggering node-not-ready false positives. Below ~50ms etcd write latency the cluster is healthy; >100ms is degraded; >500ms is broken. The fix is etcd disk performance: NVMe local SSD, write IOPS provisioned, fsync hot. At scale, split events into a separate etcd cluster and turn on Object Count Quota to cap blast radius." |
| **"How would you design a custom scheduler?"** | "I'd write a controller." | "Two paths: (a) **scheduler plugin** in the Scheduling Framework — register a Filter, Score, Reserve, Permit, Bind plugin via the scheduler config; this re-uses the default scheduler's queueing, preemption, pod prioritization. (b) **separate scheduler binary** that watches unscheduled pods (with `spec.schedulerName == mine`) and writes `spec.nodeName` via a bind subresource. Path (a) is right when you augment the default; path (b) when you have a fully different model (Volcano for batch, Yunikorn for multi-tenant queueing). For staff-level design, also discuss preemption interaction, scheduling-extender vs framework plugin, scheduling latency budget, scheduler cache sync." |

The shift from junior to staff: **named components, named limits, named alternatives, end-to-end traceability.**

## Behavioral signals for k8s/CNCF interviews

If the role is CNCF or upstream-adjacent:

- **Upstream contribution** — even a doc fix counts. List PRs in your résumé.
- **KEP authorship** or co-authorship — strong staff signal.
- **SIG attendance** — sig-api-machinery, sig-network, sig-scalability, sig-scheduling.
- **Slack participation** in #sig-foo channels — public collaboration.
- **A maintainer relationship** — being on first-name basis with another contributor.

If you don't have upstream contribs, prep STAR stories around:
- "How I designed an operator for X" (talk about CRD shape, reconciliation, edge cases).
- "How I migrated us from raw Deployments to an internal abstraction" (platform engineering).
- "How I scaled our cluster from N to 10N nodes" (limits hit, fixes applied).
- "How I debugged a production k8s incident" (be specific about which component).

## Stack you should be ready to discuss

Pick 1–2 to be *deeply* fluent on:

- A **CNI** (Calico, Cilium, AWS VPC CNI, flannel) — know the dataplane.
- A **service mesh** (Istio, Linkerd, Cilium service mesh) — know mTLS, traffic policy.
- An **observability stack** (Prometheus + Thanos, Grafana, Loki, OpenTelemetry) — know cardinality limits.
- A **cluster autoscaler** (CA, Karpenter) — know the decision algorithm.
- An **operator framework** (Operator SDK, controller-runtime, kubebuilder, Metacontroller, kopf) — know the controller-runtime patterns.

## Anti-patterns in your answers

- **"Just use a sidecar"** — but did you mention the 1.28+ sidecar container lifecycle, the resource accounting, the network namespace sharing?
- **Vague "it's eventually consistent"** — for what state? With what convergence time? Under what failure mode?
- **"K8s handles X for me"** — staff knows where the boundaries are. K8s does NOT handle: backup of stateful data, secrets rotation in running pods (without restart), inter-cluster networking by default, cost optimization.
- **Confusing the etcd resource quota with the namespace ResourceQuota** — first is `ETCDCTL_API=3 etcdctl alarm list`, second is `kubectl get resourcequota`. Both are real, very different.
- **Missing the version-skew implications** — apiserver N, kubelet N-2; if you propose a feature in 1.30, you can't rely on it on a 1.28 node.

## Three preparation principles

1. **Run a cluster.** Stand up `kind` or `minikube`; deploy nginx; trace it through `kubectl get events`, audit logs (enable `--audit-log-path`), and apiserver metrics. Reading docs ≠ knowing the system.
2. **Read one KEP per week.** Start with: KEP-25 (CRD), KEP-95 (admission webhook), KEP-127 (CRI), KEP-95 (volume snapshot), KEP-235 (CRD validation), KEP-624 (in-place pod resize), KEP-753 (sidecar containers), KEP-3243 (auth structured config), KEP-4 (event-rate-limit). The phrasing in interview answers mirrors KEP language.
3. **Build a CRD + operator.** With kubebuilder, scaffold a `Foo` CRD; write a controller that reconciles. Add a finalizer; add a webhook; add a status subresource. Doing this once teaches more than reading 10 blog posts.

## Must-internalize

- The four pillars: apiserver, etcd, controller-manager, scheduler. Plus the data plane: kubelet, runtime, kube-proxy/CNI.
- The reconcile loop is THE pattern: informer → work queue → reconcile → apiserver write.
- Read KEPs; they're the staff vocabulary.
- Build one operator with finalizers + webhooks before the interview.
- Have one strong upstream-contribution story or one strong internal-operator story.
