# 17 · Upgrades and Version Policy

K8s ships a minor version every ~4 months. Upgrading is a perennial operational topic; staff candidates should know the skew policy, the upgrade order, and how to handle long-running clusters.

## 17.1 The version skew policy

```
   kube-apiserver:    must be the newest version in the cluster
   kube-controller-manager: up to 1 minor older than apiserver
   kube-scheduler:    up to 1 minor older than apiserver
   cloud-controller-manager: up to 1 minor older
   kubelet:           up to 2 minor older (extending to 3 in some releases)
   kube-proxy:        same minor as kubelet on its node, or 1 newer
   kubectl:           1 minor older OR newer than apiserver
```

Skew matrix as of 1.30:

```
1.30 apiserver  ←  1.29, 1.30 controller/scheduler/CCM
                   1.28, 1.29, 1.30 kubelet (with feature gates)
                   1.29, 1.30, 1.31 kubectl
```

KEP-3744 (1.28+): supports kubelet up to 3 minor versions behind, giving more upgrade flexibility.

## 17.2 The upgrade order

For a self-managed cluster:

1. **Control plane**: upgrade kube-apiserver first (then controller-manager + scheduler + CCM).
   - With HA control plane (3+ apiservers): rolling upgrade, one at a time.
2. **kubelet**: upgrade nodes one at a time, draining first.
3. **kube-proxy** (and other DaemonSets): rolling update.
4. **Add-ons**: cluster-autoscaler, CoreDNS, ingress controller, etc.

For managed (EKS / GKE / AKS): cloud provider handles this; you select the target version.

### Each step

```bash
# Drain node
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# Upgrade kubelet (Ubuntu, kubeadm-installed)
apt-get install -y kubelet=1.30.0-00
systemctl restart kubelet

# Uncordon
kubectl uncordon node-1
```

## 17.3 The feature lifecycle

```
   alpha    →    beta    →    GA / stable
   (default off)  (default on, usually)  (default on)
   ~ minor versions:  1-2 each
```

Total alpha→GA: typically 3-5 minor versions = 12-20 months.

### Feature gates

```
kubelet:
  --feature-gates=InPlacePodVerticalScaling=true,SidecarContainers=true
```

Used to opt into alpha or out of beta features.

### Deprecation policy

| API stage | Deprecation timeline |
|-----------|----------------------|
| Alpha | Can be removed in any release |
| Beta | Deprecated 9 months or 3 releases before removal |
| GA | Deprecated 12 months or 1 minor release before removal (but with longer grace usually) |

## 17.4 API removal — the migration

When `extensions/v1beta1 Ingress` was removed in 1.22, every manifest using it broke. Staff must:
1. Read removal notes for target version.
2. Inventory cluster for removed APIs (`kubectl-convert`, `kubectl-deprecations`).
3. Migrate manifests.
4. Test in staging.
5. Plan rollout.

Tools:
- **kube-no-trouble (kubent)** — scans cluster + git for deprecated APIs.
- **pluto** — similar, by FairwindsOps.

## 17.5 Etcd upgrades

Etcd version is somewhat decoupled from k8s minor:
- Each k8s minor recommends a specific etcd minor.
- Upgrade etcd separately, before or after k8s upgrade.
- Same Raft protocol → no major downtime.

```bash
# Per member:
systemctl stop etcd
# replace binary
systemctl start etcd
# wait for "healthy" check
# move to next member
```

## 17.6 CRD versioning + upgrades

CRDs are user-defined; you control their lifecycle. K8s won't break them, but:
- Old CRD `apiextensions.k8s.io/v1beta1` was removed; only v1 now.
- CRD schema changes: must be backward-compatible OR use conversion webhook.

Strategy:
- Always serve v1; bump to v1beta1 only when truly experimental.
- Have a conversion webhook before adding v2.

## 17.7 Cloud-managed upgrades

### EKS

1. Upgrade control plane: AWS handles; one click.
2. Upgrade node groups: managed-node-group rolling upgrade; or replace ASG.
3. Upgrade add-ons: kube-proxy, CoreDNS, VPC CNI versions follow.

EKS supports 4 latest versions; older are End-of-Life. Forced upgrade at EOL.

### GKE

1. Control plane auto-upgrade by default; can pin.
2. Node pools auto-upgrade by default; opt out.
3. Release channels: rapid / regular / stable.

### AKS

Similar; explicit one-click upgrade.

## 17.8 Add-on lifecycle

Things that aren't k8s itself but live in the cluster:

| Add-on | Upgrade considerations |
|--------|----------------------|
| **CoreDNS** | Pin version; large change at 1.20+ |
| **kube-proxy** | Follows kubelet; restart on change |
| **CNI plugin** | Plugin-specific; check for k8s compatibility (Cilium release notes for k8s 1.x) |
| **Metrics-server** | Compatibility matrix in repo |
| **Ingress controller (NGINX, etc.)** | Independent lifecycle; pinned to specific k8s |
| **Cert-manager** | Check API version compat |
| **Cluster Autoscaler** | One CA version per k8s minor |
| **CSI drivers** | Per-driver compat matrix |

## 17.9 The drain procedure

```bash
kubectl drain node-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force \
  --grace-period=120 \
  --timeout=600s
```

What drain does:
1. Cordon node (`spec.unschedulable=true`): no new pods.
2. Eviction API for each non-DaemonSet pod (respects PDB).
3. Waits for eviction completion.

Caveats:
- DaemonSets aren't evicted (would just respawn).
- emptyDir data lost (--delete-emptydir-data acknowledges).
- Pods without controller (bare Pods) need --force.

## 17.10 Long-running clusters — staying current

Real production clusters: 1-3 years old; lots of CRDs; many add-ons. Upgrade challenges:

- **CRD compat**: old CRDs may not work with new k8s.
- **Webhook compat**: admission webhooks must accept v1 AdmissionReview.
- **CSI compat**: old driver may not support new feature gates.
- **kubelet config drift**: file-based config may have stale options.
- **Custom controllers**: their client-go version may not match cluster version.

Best practices:
- **Upgrade every minor** — falling behind makes catching up dangerous.
- **Stage in non-prod** for 1+ minor versions first.
- **Tag clusters with version**; track in inventory.

## 17.11 Rollback

Rolling back k8s minor versions is **hard**. Most teams treat upgrade as one-way:
- Storage version migrations may not reverse.
- Newer fields in resources may not work with older apiservers.
- etcd version compatibility may not extend backward.

In practice: if upgrade is bad, fix forward (patch release in newer version), not roll back.

Cloud-managed: most providers explicitly don't support downgrade.

## 17.12 Blue-green cluster upgrades

For high-stakes upgrades:
- Provision new cluster at new version.
- Migrate workloads (GitOps + DNS flip + data replication).
- Decommission old cluster.

Slow but safe. Common for major version jumps or risk-averse orgs.

## 17.13 In-place vs blue-green

| Property | In-place | Blue-green |
|----------|----------|------------|
| Cost | Lower (no double infra) | Higher (~2x briefly) |
| Risk | Higher (live cluster) | Lower (test before flip) |
| Time | Faster (hours) | Slower (days) |
| Reversibility | Hard | Easy (flip back) |
| Pro use | Most clusters | Major-risk upgrades |

## 17.14 Common interview probes

- **"What's the version skew policy?"** apiserver newest; kubelet up to 2 minor older; rest 1 minor older.
- **"How do you handle a deprecated API?"** Scan with kubent/pluto; migrate manifests; bump CRD versions; deploy in staging first.
- **"What's the upgrade order?"** Control plane → kubelet → DaemonSets → add-ons.
- **"How do you upgrade etcd?"** Per-member rolling; same Raft protocol; check health between.
- **"How do you roll back a botched upgrade?"** Usually you don't; fix forward. Blue-green if you anticipated.
- **"What's a feature gate?"** Per-component flag controlling alpha/beta features. Once GA, removed.

## 17.15 Corner cases

- **kubelet restart loses ephemeral storage** — restarts are typically pod-preserving, but kubelet kill mid-restart can clobber.
- **Drain stuck on PDB** — PDB blocks eviction. Either reduce minAvailable (risky) or wait.
- **Bare pod without controller** — drain --force needed; pod just dies, no recreate.
- **DaemonSet pod with terminationGracePeriodSeconds: 0** — kill immediately on drain.
- **Pod with hostNetwork during upgrade** — its port may conflict with new pod after restart.
- **In-flight workload during apiserver upgrade** — rolling control plane → some requests fail; clients retry. APF + slow rollout helps.
- **CRD with no conversion webhook + schema change** — upgrade breaks reads.

## 17.16 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Self-managed upgrade | kubeadm | Cluster API | manual |
| Managed upgrade | EKS one-click | GKE auto-upgrade | AKS portal |
| Risk control | Stage in non-prod first | Blue-green | Canary clusters |
| API migration | kubent + sed | OPA detection | Webhook intercept old API |
| CRD migration | Conversion webhook | Migration job | Manual scripts |
| Add-on lifecycle | Per-add-on update | Argo CD | Operator with desired version |

## 17.17 Best practices

1. **Stay close to current**: max 1-2 minor versions behind. Maintain currency budget.
2. **Test in staging**: run new k8s version with prod-like workload for 1+ week before prod.
3. **Have a runbook** per add-on with version notes.
4. **Use GitOps** for cluster config; upgrades become a PR.
5. **Track deprecations**: subscribe to k8s deprecation announcements.
6. **Monitor APF rejections, apiserver latency, etcd commit during rollout**.
7. **Schedule upgrades during low-traffic windows**.
8. **Drain slowly**: respect PDB; limit concurrency.

## Must-internalize

- Skew: apiserver newest; kubelet up to 2 minor older (or 3, new KEP).
- Order: control plane → kubelet → DaemonSets → add-ons.
- Feature gates control alpha/beta; deprecated APIs are removed on schedule.
- CRD versioning is your responsibility; conversion webhook for non-trivial changes.
- Rollback rarely supported; fix forward or blue-green.
- Cloud-managed (EKS/GKE/AKS): one-button; pinned to recent versions.
- Drain respects PDB; ignores DaemonSets.
- Tools: kubent, pluto for deprecation scans; kubectl-convert for manifest migration.
