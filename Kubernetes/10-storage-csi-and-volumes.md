# 10 · Storage — CSI, Volumes, and Persistence

The most operationally fraught subsystem. Storage failures cause the highest-impact incidents.

## 10.1 The volume types — quick taxonomy

| Type | Lifetime | Use |
|------|----------|-----|
| **emptyDir** | Pod | Scratch space, shared between containers |
| **hostPath** | Forever (host) | Privileged daemons; rare for apps |
| **configMap, secret, downwardAPI** | Pod | Inject config/secrets/pod metadata |
| **projected** | Pod | Combine multiple of above + service-account tokens |
| **persistentVolumeClaim (PVC)** | PVC lifetime | The standard for stateful workloads |
| **ephemeral inline** | Pod | CSI volume created per-pod |
| **image** (1.31+ alpha) | Pod | Mount OCI image as a read-only volume |
| **CSI volume** | Driver-managed | Block / file storage via CSI plugin |

For application persistence, the canonical answer is **PVC → PV → CSI driver**.

## 10.2 PV / PVC / StorageClass — the three-level model

```
   StorageClass (cluster-admin defines provisioner + params)
            │
            │ (dynamic provisioning)
            ▼
   PersistentVolume (PV — the actual storage)
            │
            │ (binding by PVC)
            ▼
   PersistentVolumeClaim (PVC — the user's request)
            │
            │ (referenced by pod)
            ▼
   Pod (mounts the PVC)
```

### StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: {name: gp3}
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iopsPerGB: "3"
  throughput: "125"
reclaimPolicy: Delete           # Delete | Retain
volumeBindingMode: WaitForFirstConsumer   # Immediate | WaitForFirstConsumer
allowVolumeExpansion: true
```

### PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: data}
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: gp3
  resources:
    requests:
      storage: 100Gi
```

When user creates PVC:
1. PVC controller sees it, matches against existing PVs (static) or invokes the StorageClass's provisioner (dynamic).
2. CSI external-provisioner creates the volume in the backing storage.
3. PV created representing the new volume.
4. PV + PVC bound.
5. Pod can mount.

### Access modes

| Mode | What |
|------|------|
| **ReadWriteOnce (RWO)** | Mounted r/w on a single node |
| **ReadOnlyMany (ROX)** | Mounted r/o on many nodes |
| **ReadWriteMany (RWX)** | Mounted r/w on many nodes |
| **ReadWriteOncePod (RWOP)** | Mounted r/w by a single pod |

Block storage (EBS, GCE PD, Azure Disk) is typically RWO.
File storage (NFS, EFS, CephFS) is RWX.
RWOP is newer; ensures truly single-pod access (vs RWO which allows multiple pods on same node).

## 10.3 CSI — Container Storage Interface

Standardized plugin interface. Each CSI driver implements:
- **Controller Service**: provision/delete volume; attach/detach to node.
- **Node Service**: stage (one-time setup per node) and publish (mount to pod) volume.
- **Identity Service**: driver metadata.

```
                kube-apiserver
                    │
       ┌────────────┼────────────┐
       ▼            ▼            ▼
  external-     external-    csi driver
  provisioner   attacher     (DaemonSet on each node)
       │            │            │
       └────────────┴────────────┘
                    │ gRPC over Unix socket
                    ▼
              cloud storage API
              (AWS EBS, GCE PD, Ceph, etc.)
```

The "external" sidecars (external-provisioner, external-attacher, external-resizer, external-snapshotter) are generic; they watch k8s resources and call the CSI driver via gRPC. The driver is the storage-specific code.

### Driver registration

```yaml
apiVersion: storage.k8s.io/v1
kind: CSIDriver
metadata: {name: ebs.csi.aws.com}
spec:
  attachRequired: true
  podInfoOnMount: false
  volumeLifecycleModes: [Persistent, Ephemeral]
  storageCapacity: true
```

### Volume operations from a pod's perspective

1. User creates PVC.
2. external-provisioner sees it, calls `CreateVolume` RPC → cloud creates EBS.
3. PV created; PVC binds.
4. User creates Pod referencing PVC.
5. Scheduler picks a node (Volume Binding plugin checks feasibility).
6. external-attacher creates a VolumeAttachment object.
7. external-attacher calls `ControllerPublishVolume` → cloud attaches EBS to node's VM.
8. Pod is scheduled to node.
9. Node CSI driver calls `NodeStageVolume` (format + mount to /var/lib/kubelet/plugins/.../...).
10. `NodePublishVolume` (bind-mount into pod's mount namespace).
11. kubelet starts the container.

Detach reverses on pod deletion.

## 10.4 WaitForFirstConsumer (WFFC)

The crucial scheduling mode.

```yaml
spec:
  volumeBindingMode: WaitForFirstConsumer
```

- **Immediate**: PVC is bound to a PV as soon as it's created. PV picks the zone; scheduler is constrained.
- **WaitForFirstConsumer**: PVC is *not* bound until a pod is scheduled. Then scheduler picks the node, PVC is bound to a PV in that node's zone.

WFFC is critical for multi-zone clusters: avoids "PVC in us-east-1a, pod needed in us-east-1b → can't bind."

## 10.5 Volume expansion

```bash
kubectl patch pvc data -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
```

StorageClass must have `allowVolumeExpansion: true`. The external-resizer:
1. Sees PVC size change.
2. Calls `ControllerExpandVolume` → cloud resizes EBS.
3. If FS expansion required: marks PVC with annotation; kubelet calls `NodeExpandVolume` on next pod (filesystem grown online for ext4, online for XFS with caveats).

## 10.6 Snapshots

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata: {name: snap-1}
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: data
```

external-snapshotter creates VolumeSnapshotContent, calls `CreateSnapshot` on CSI driver → cloud snapshot.

Restore: new PVC with `dataSource: kind: VolumeSnapshot, name: snap-1`.

Group snapshots (KEP-3476, beta 1.27+): atomic snapshot across multiple PVCs.

## 10.7 Ephemeral CSI volumes

Inline volume that lives for pod lifetime:

```yaml
spec:
  volumes:
  - name: scratch
    csi:
      driver: scratch.csi.example.com
      volumeAttributes:
        size: 100Gi
```

No PVC needed; driver creates the volume on pod creation, destroys on deletion. Used for scratch/temp where persistence is unwanted.

## 10.8 Generic ephemeral volumes (1.23+)

The newer pattern. PVC template inline in pod spec:

```yaml
spec:
  volumes:
  - name: scratch
    ephemeral:
      volumeClaimTemplate:
        spec:
          accessModes: [ReadWriteOnce]
          storageClassName: fast
          resources: {requests: {storage: 100Gi}}
```

A PVC is created with pod's name + suffix; deleted when pod deletes. Uses standard StorageClass — works with any CSI driver.

## 10.9 StatefulSet + volumeClaimTemplates

```yaml
apiVersion: apps/v1
kind: StatefulSet
spec:
  serviceName: my-svc
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        volumeMounts: [{name: data, mountPath: /data}]
  volumeClaimTemplates:
  - metadata: {name: data}
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: gp3
      resources: {requests: {storage: 100Gi}}
```

StatefulSet creates PVCs named `data-<sts-name>-0`, `data-<sts-name>-1`, etc. PVCs **persist** even when pods are deleted — so when `pod-0` is recreated, it gets the same PVC (and thus same volume).

Deletion: `kubectl delete sts` does NOT delete PVCs by default. Manual cleanup or use `persistentVolumeClaimRetentionPolicy` (1.27+ GA).

## 10.10 Reclaim policies

When a PVC is deleted, what happens to the underlying PV/storage?

| Policy | What |
|--------|------|
| **Retain** | PV stays; admin must clean up manually |
| **Delete** | PV and underlying volume are deleted |
| **Recycle** | (deprecated) Old `rm -rf /vol/*` then make available |

Default for dynamically provisioned PVs is **Delete**. For production-critical data, set **Retain** so accidental PVC deletion doesn't lose data.

## 10.11 The CSI driver landscape

| Driver | Backend | Modes |
|--------|---------|-------|
| **ebs.csi.aws.com** | AWS EBS | Block, RWO |
| **efs.csi.aws.com** | AWS EFS | File, RWX |
| **pd.csi.storage.gke.io** | GCE Persistent Disk | Block, RWO |
| **disk.csi.azure.com** | Azure Disk | Block, RWO |
| **file.csi.azure.com** | Azure Files | File, RWX |
| **rbd.csi.ceph.com** | Ceph RBD | Block, RWO |
| **cephfs.csi.ceph.com** | CephFS | File, RWX |
| **rook-ceph** | Rook-managed Ceph | mixed |
| **longhorn.io** | Longhorn (k8s-native) | Block, RWO |
| **openebs.io** | OpenEBS | Block/File |
| **portworx.io** | Portworx | Block |
| **local-path** (rancher) | Local node disk | Block, RWO |

For testing: `local-path-provisioner` or `kind`'s built-in.

## 10.12 Local storage

Two options for local-disk PVs:

### hostPath
Simple but not portable; pod tied to a node by `nodeAffinity` if you want this to make sense.

### Local PV (KEP-121, GA 1.14)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata: {name: local-pv-0}
spec:
  capacity: {storage: 100Gi}
  volumeMode: Filesystem
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/disk0
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values: [node-1]
```

CSI driver alternative: `topolvm` (creates LVM volumes per node).

## 10.13 Snapshot-restore + clone

```yaml
# Clone (PVC from PVC)
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  dataSource:
    kind: PersistentVolumeClaim
    name: source-pvc
  accessModes: [ReadWriteOnce]
  resources: {requests: {storage: 100Gi}}
```

CSI driver must support `CLONE_VOLUME` capability. Faster than copy if backend supports it (most cloud block storage does).

## 10.14 Common interview probes

- **"Walk me through the lifecycle of a PVC."** PVC created → provisioner watches → CSI `CreateVolume` → PV created + bound → pod uses → attacher attaches → kubelet stages + publishes → mount in pod.
- **"What's WaitForFirstConsumer?"** PV not provisioned until pod is scheduled; ensures PV is in the same zone as the pod.
- **"Why doesn't deleting a StatefulSet delete PVCs?"** Stateful data is precious; default is to retain. Use `persistentVolumeClaimRetentionPolicy: {whenDeleted: Delete}`.
- **"How do you back up cluster data?"** Velero — k8s-native backup; handles resources + PV snapshots via CSI snapshotter.
- **"What's the difference between attach and stage?"** Attach = make the volume visible to the node OS (cloud API); Stage = format/mount to a node-shared dir; Publish = bind-mount to pod's namespace.
- **"How does volume expansion work?"** external-resizer calls CSI ControllerExpandVolume; FS expansion via kubelet on the node.

## 10.15 Corner cases

- **PVC stuck in Pending** — usually StorageClass not found, capacity unavailable, or zone mismatch.
- **Pod stuck in ContainerCreating with volume error** — CSI driver issue or PV not attached.
- **`Multi-Attach error` for RWO volume** — pod scheduled to new node but old pod's PV hasn't detached. Usually with non-graceful node failure. Force-detach if needed.
- **Volume zone-mismatch** — pod scheduled to zone B, PV in zone A. Use WFFC to avoid.
- **Snapshot fails when source PVC is being written** — many CSI drivers want freeze; some don't. Test before production reliance.
- **Backing storage runs out of capacity** — quotas; alerting; ResourceQuota on PVC count + size.
- **Orphan PVs** — PV in Released state with Retain policy; need manual cleanup or recycler.
- **PV recycling vs deletion** — Recycle is deprecated; use Delete or manual.

## 10.16 Best practices

- **WFFC for any multi-zone cluster.** Always.
- **Retain for production.** Recovery from accidental delete.
- **Volume snapshots scheduled** (Velero or similar).
- **ResourceQuota on PVC** to prevent quota blow-up.
- **`fsGroup` security context** when pod needs to write to volume as non-root user.
- **Avoid hostPath for apps** — security and portability nightmare.
- **Test failover** — kill a pod, ensure new pod gets same data.

## 10.17 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Pod-local scratch | emptyDir | inline ephemeral CSI | generic ephemeral PVC |
| Shared file storage | NFS via CSI | EFS (AWS) / Filestore (GCP) | CephFS (rook-ceph) |
| Block storage | EBS / PD / Disk CSI | Longhorn (k8s-native) | OpenEBS |
| Stateful app | StatefulSet + PVCs | Operator with custom storage logic | DaemonSet + hostPath (rare) |
| Backup | Velero | CSI snapshots only | Application-level (mysqldump) |
| Multi-AZ resiliency | StorageClass with regional disk (regional-pd) | Cross-zone replication at app level | CockroachDB / TiDB (DB-native) |

## Must-internalize

- StorageClass (params) → PVC (request) → PV (resource) → Pod (mount).
- CSI drivers via external-* sidecars + node DaemonSet.
- WaitForFirstConsumer delays PV provision until pod schedule; multi-zone safety.
- Reclaim policy: Retain for prod, Delete for ephemeral.
- StatefulSet PVCs persist across pod recreation; deletion is opt-in.
- Snapshots via VolumeSnapshot CR; clones via dataSource.
- Local PV with nodeAffinity for fast local storage.
- Attach (cloud API) vs Stage (node FS) vs Publish (pod bind-mount) — three CSI levels.
