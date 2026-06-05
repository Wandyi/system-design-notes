# 16 · Multi-Cluster and Cluster API

K8s is single-cluster by design. Multi-cluster is layered on top via separate projects. This is the area where staff candidates often differentiate themselves — knowing the trade-offs of each approach.

## 16.1 Why multi-cluster

- **Fault isolation**: one cluster's incident doesn't affect others.
- **Geographic distribution**: edge POPs, regions.
- **Compliance**: tenant data in a tenant-specific cluster.
- **Scale**: each cluster has the 5000-node limit; sharding beats stretching.
- **Upgrade isolation**: pre-prod cluster runs new version.
- **Cost**: different node groups, different cloud accounts.

## 16.2 The patterns

### Hub-and-spoke (centralized management)

```
                  Management cluster (hub)
                          │
                          │ (operators push config)
                          ▼
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        Cluster A    Cluster B    Cluster C
        (workload)   (workload)   (workload)
```

Hub has operators that manage spokes via their kubeconfigs / APIServers.

Examples: Cluster API, Karmada, Argo CD multi-cluster, Open Cluster Management.

### Federation (single API surface)

A virtual API server that fans out to multiple member clusters.

Examples: KubeFed v2 (deprecated), Karmada, Crossplane's compositions.

### Service-mesh multi-cluster

Each cluster has its own control plane; meshes know about each other for cross-cluster service-to-service.

Examples: Istio multi-cluster, Cilium ClusterMesh, Submariner.

### Sidecar-style

Apps in different clusters federate via gateway / tunneling.

Less common; more app-aware.

## 16.3 Cluster API (CAPI)

The CNCF project for declarative cluster lifecycle.

```
   Management cluster
      │
      │ has Cluster CRs, Machine CRs, MachineDeployment CRs
      │
      ▼
   Provider controllers (CAPA for AWS, CAPZ for Azure, ...)
      │
      ▼
   Cloud APIs create infrastructure
      │
      ▼
   Bootstrap (kubeadm) initializes the new cluster
```

### Resources

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata: {name: prod-us-east}
spec:
  infrastructureRef:
    kind: AWSCluster
    name: prod-us-east
  controlPlaneRef:
    kind: KubeadmControlPlane
    name: prod-us-east-cp

---
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata: {name: prod-workers}
spec:
  clusterName: prod-us-east
  replicas: 10
  template:
    spec:
      clusterName: prod-us-east
      version: v1.30.0
      bootstrap:
        configRef:
          kind: KubeadmConfigTemplate
          name: prod-workers
      infrastructureRef:
        kind: AWSMachineTemplate
        name: prod-workers
```

### Providers

| Provider | Cloud |
|----------|-------|
| CAPA | AWS |
| CAPZ | Azure |
| CAPG | GCP |
| CAPV | vSphere |
| CAPO | OpenStack |
| CAPM | Metal3 (bare metal) |
| CAPI-Operator | Manage CAPI components |

### Upgrade

Update `MachineDeployment.spec.template.spec.version: v1.31.0` → controller rolls workers; KubeadmControlPlane upgrade rolls apiserver.

### Why CAPI

The standard, cross-cloud answer for "how do I provision k8s clusters from k8s." Cloud providers' managed-k8s offerings (EKS, GKE, AKS) use CAPI-like internal layers.

## 16.4 Karmada — federation

Lets you submit a Deployment "once" and have it propagate to N clusters.

```yaml
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
spec:
  resourceSelectors:
  - apiVersion: apps/v1
    kind: Deployment
    name: web
  placement:
    clusterAffinity:
      clusterNames: [us-east, us-west, eu-central]
    replicaScheduling:
      replicaSchedulingType: Divided
      replicaDivisionPreference: Weighted
      weightPreference:
        staticWeightList:
        - targetCluster: {clusterNames: [us-east]}
          weight: 50
        - targetCluster: {clusterNames: [us-west]}
          weight: 30
        - targetCluster: {clusterNames: [eu-central]}
          weight: 20
```

Karmada's controller fans out: `Deployment` → `Work` resources per cluster → applied to member clusters.

### Limits
- Fanning out only works for resources that fan out cleanly (Deployments, ConfigMaps).
- StatefulSets harder (identity per cluster).
- Hard to debug "why isn't my pod in cluster X."

## 16.5 Argo CD ApplicationSet

GitOps-driven multi-cluster:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
spec:
  generators:
  - clusters:
      selector:
        matchLabels: {env: prod}
  template:
    metadata: {name: '{{name}}-app'}
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/manifests
        path: 'environments/{{name}}'
      destination:
        server: '{{server}}'
        namespace: my-app
```

Generates one Argo Application per matching cluster. Cluster registration + manifests live in git.

This is the most popular pattern for cross-cluster GitOps.

## 16.6 Istio multi-cluster

Three topologies:

### Primary-remote
```
   Primary cluster: full Istio control plane (Istiod)
   Remote cluster: only the data plane (Envoy sidecars)
              │
              │ remote sidecars connect to primary Istiod via gateway
              ▼
          Cross-cluster mTLS
```

### Multi-primary
Each cluster has Istiod. They share a root CA. Pods in cluster A can mTLS to pods in cluster B via gateway → remote sidecars.

### Multi-network
Clusters are on different L3 networks; traffic flows via east-west gateways.

## 16.7 Cilium ClusterMesh

eBPF-based multi-cluster. Each cluster's Cilium agent maintains a global service map.

```yaml
# Global service: any cluster's pods serve it
apiVersion: v1
kind: Service
metadata:
  name: global-svc
  annotations:
    service.cilium.io/global: "true"
    service.cilium.io/shared: "true"
```

Pods in cluster A connecting to `global-svc.default.svc.cluster.local` are load-balanced across endpoints in cluster A AND cluster B.

## 16.8 Submariner

Cluster-mesh that creates IPSec or WireGuard tunnels between clusters' pod CIDRs:

```
   Cluster A pod (10.244.1.5)
     │
     ▼
   Submariner Gateway A
     │ IPSec tunnel
     ▼
   Submariner Gateway B
     │
     ▼
   Cluster B pod (10.250.1.7)
```

Routes pod CIDR of B from A's gateway. Each cluster needs unique pod CIDR.

Use case: legacy apps that need flat pod-to-pod IP.

## 16.9 Multi-cluster Services API (MCS API)

A k8s SIG project to standardize cross-cluster Service discovery:

```yaml
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata: {name: my-svc, namespace: my-ns}
---
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceImport
metadata: {name: my-svc, namespace: my-ns}
spec:
  ports:
  - port: 80
    protocol: TCP
  type: ClusterSetIP
```

DNS: `my-svc.my-ns.svc.clusterset.local` resolves to endpoints across the ClusterSet.

Implementations: Cilium ClusterMesh, AWS Cloud Map MCS, KubeFed.

## 16.10 GitOps + multi-cluster

GitOps (Argo CD, Flux) shines in multi-cluster:
- Cluster state in git.
- Argo CD continuously reconciles each cluster against its git path.
- Single source of truth.

Pattern: one git folder per cluster; ApplicationSet generates Applications.

### Bootstrap pattern (Argo CD self-managed)

Argo CD installs itself, then takes over managing itself + everything else. Cluster lifecycle is git-tracked.

## 16.11 Hub-of-hubs

For enterprises with many clusters across many teams: a meta-management cluster manages many management clusters.

Open Cluster Management (OCM, Red Hat ACM) implements this. Each managed cluster has an agent (`klusterlet`); the hub orchestrates via `ManifestWork` CRs.

## 16.12 Common multi-cluster questions

### "How do you do disaster recovery across clusters?"

- App data via cross-region replicated storage (or app-level replication).
- Config in git (GitOps).
- Cluster-internal state (Secrets, Custom Resources): backed up via Velero with S3.
- DNS failover: external GSLB / Route 53 health checks → flip traffic.

### "How do you upgrade clusters one at a time?"

- Cluster API: bump `version` in MachineDeployment → CAPI rolls.
- Self-managed: drain control plane node, upgrade, repeat; then nodes via DaemonSet upgrade-controller.
- Managed (EKS/GKE/AKS): one-button.

### "How do you debug a cross-cluster service mesh issue?"

- Hubble (Cilium) shows flows; cross-cluster flows visible.
- Istio: `istioctl proxy-status` shows sync; access logs at sidecar.
- mTLS issue: cert chain validation; usually identity mismatch.

## 16.13 Multi-cluster anti-patterns

- **One huge cluster** instead of multi-cluster — hits 5000-node limit; blast radius huge.
- **Strong consistency across clusters** — k8s is per-cluster eventually consistent; cross-cluster on top is multiplied eventually.
- **Federation with no fallback** — federation control plane down → no scheduling cross-cluster. Each cluster should be operable standalone.
- **Cross-cluster joins with PVC** — block storage rarely federates.
- **Manual cluster setup** without IaC — drift across clusters; impossible to maintain.

## 16.14 Common interview probes

- **"How do you scale beyond 5000 nodes?"** Multi-cluster. Federation for visibility; CAPI for lifecycle; service mesh for connectivity.
- **"How do you do multi-region active-active?"** Anycast + GSLB; multi-region clusters; app-level replication. Cross-cluster service mesh for service-to-service.
- **"What's Cluster API?"** Declarative cluster lifecycle: Cluster, MachineDeployment, etc. CRDs in management cluster; provider-specific impls.
- **"How does Karmada compare to KubeFed?"** Karmada is the active-developed successor; better scheduling, propagation.
- **"How would you migrate workloads between clusters?"** GitOps + DNS flip; service mesh canary; per-app strategy.
- **"What's the trade-off of one cluster per team vs shared?"** Shared: lower ops cost, noisier neighbor. Per-team: stronger isolation, more ops. Sweet spot depends on team count + maturity.

## 16.15 Corner cases

- **Cross-cluster DNS** — pod in A asks for `svc.B.cluster.local` — needs MCS API or CoreDNS rewrite + Submariner.
- **mTLS across clusters with different CAs** — requires CA federation (shared root or cross-signed).
- **Resource version skew** between clusters — CAPI manages version per cluster; can lag.
- **CAPI workload cluster API server unreachable** → CAPI controller stuck; events stop flowing.
- **Service mesh trust domain mismatch** — each cluster needs unique trustDomain or shared.
- **Namespace name collision** — multiple clusters with same `prod` ns; metric dashboards confused.

## 16.16 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Provision clusters | Cluster API | Cloud-specific (eksctl, gcloud) | Terraform |
| Apply config to many clusters | Argo CD ApplicationSet | Flux + GitRepository | Karmada propagation |
| Cross-cluster service | Cilium ClusterMesh | Istio multi-cluster | Submariner |
| Cross-cluster discovery | MCS API | Service mesh service entry | external DNS |
| Disaster recovery | Velero + cross-region S3 | App-level multi-region | Stretch cluster (rare) |
| Hub management | Open Cluster Management (ACM) | Argo CD as hub | custom operator |

## 16.17 The reference architecture — fleet of clusters

```
                    Management cluster (hub)
                          │
                          ├── Cluster API (CAPI) controllers
                          ├── Argo CD ApplicationSet
                          ├── Karmada (optional)
                          ├── Open Cluster Management
                          ├── External Secrets Operator (cross-cluster sync)
                          ├── Prometheus federation (collects metrics from all)
                          ├── OPA Gatekeeper (policy enforcement)
                          └── ChartMuseum / Harbor (cluster-shared)
                          
                          │ (federates to)
                          ▼
        Cluster 1 (us-east-1 prod)
        Cluster 2 (us-west-2 prod)
        Cluster 3 (eu-central-1 prod)
        Cluster 4 (us-east-1 staging)
        Cluster 5 (per-team dev)
        ...
```

Each cluster operates standalone (can survive hub outage). Hub provides "fleet-wide" visibility + policy + provisioning.

## Must-internalize

- Single cluster scales to ~5000 nodes; beyond is multi-cluster.
- Cluster API (CAPI) is the standard for declarative cluster lifecycle.
- Karmada / Open Cluster Management for federation.
- Argo CD ApplicationSet for GitOps-driven multi-cluster apps.
- Service mesh multi-cluster: Istio (primary-remote / multi-primary), Cilium ClusterMesh.
- MCS API standardizes ServiceExport / ServiceImport across clusters.
- Each cluster should be standalone-functional; federation is value-add not single-point-of-failure.
- DR pattern: GitOps + Velero + cross-region object store + GSLB DNS failover.
