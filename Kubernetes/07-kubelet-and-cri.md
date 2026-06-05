# 07 · kubelet and CRI

`kubelet` is the node-side agent. The most code-heavy single binary in k8s (after apiserver). Every pod lifecycle event passes through it.

## 7.1 What kubelet does

- Watches the apiserver for pods bound to its node (`spec.nodeName == myhostname`).
- Maintains pod state via the **SyncLoop**.
- Talks to the container runtime via **CRI** (gRPC).
- Mounts volumes via **CSI** drivers.
- Sets up pod network via **CNI** plugins.
- Manages **cgroups** (resource limits) via systemd or cgroupfs driver.
- Runs **probes** (liveness, readiness, startup).
- Reports node + pod status to apiserver.
- Serves debug endpoints (logs, exec, port-forward).
- Watches kubelet-config and node objects.

## 7.2 The syncloop

```go
// kubelet/pkg/kubelet/kubelet.go (paraphrased)
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
    for {
        select {
        case u := <-updates:                  // apiserver pod change
            handler.HandlePodAdditions(u.Pods)
        case e := <-kl.pleg.Watch():          // PLEG container event
            handler.HandlePodSyncs([]*v1.Pod{p})
        case <-housekeepingTicker.C:          // periodic 2s
            handler.HandlePodCleanups()
        case <-syncTicker.C:                   // periodic 1s sync
            handler.HandlePodSyncs(podWorkers.SyncPodsWithRetries())
        }
    }
}
```

The syncloop is the heartbeat. Every second (default), and on every event, kubelet runs `HandlePodSyncs` for each pod.

## 7.3 PLEG — Pod Lifecycle Event Generator

PLEG polls the container runtime every 1s, comparing current container states to last-known. Differences → events fed into syncloop.

```
   PLEG every 1s:
     - call CRI ListContainers
     - compute deltas vs last snapshot
     - for each delta: emit ContainerStarted, ContainerDied, etc.
     - syncLoop reacts
```

The legacy concern: at high pod count (>110 per node), PLEG list call gets slow → `PLEG is not healthy` warnings → kubelet considers itself unhealthy. KEP-3386 (Evented PLEG, GA 1.27) replaces polling with CRI events.

## 7.4 The pod sandbox

Each pod has a **sandbox** — the network/IPC/UTS namespace shared by containers in the pod:

```
   pod foo:
     sandbox (pause container, holds the namespaces)
       ├── container A (joins network/IPC/UTS ns)
       ├── container B (joins network/IPC/UTS ns)
       └── container C (joins network/IPC/UTS ns)
```

The **pause container** is the canonical empty container that just runs `pause()` and never exits. It holds the network namespace so other containers can come and go.

Kubelet's sandbox lifecycle:
1. `RunPodSandbox` (CRI): create network ns, run pause container.
2. Setup network: kubelet calls CNI ADD with the netns path. CNI plugin creates veth, allocates IP.
3. `CreateContainer` + `StartContainer` for each app container.

## 7.5 CRI — Container Runtime Interface

gRPC protocol between kubelet and the runtime. Two services:

- **RuntimeService** — sandbox + container lifecycle, exec, logs.
- **ImageService** — image pull, list, remove.

```protobuf
service RuntimeService {
  rpc RunPodSandbox(RunPodSandboxRequest) returns (RunPodSandboxResponse) {}
  rpc StopPodSandbox(StopPodSandboxRequest) returns (StopPodSandboxResponse) {}
  rpc RemovePodSandbox(RemovePodSandboxRequest) returns (RemovePodSandboxResponse) {}
  rpc PodSandboxStatus(PodSandboxStatusRequest) returns (PodSandboxStatusResponse) {}
  rpc ListPodSandbox(ListPodSandboxRequest) returns (ListPodSandboxResponse) {}

  rpc CreateContainer(CreateContainerRequest) returns (CreateContainerResponse) {}
  rpc StartContainer(StartContainerRequest) returns (StartContainerResponse) {}
  rpc StopContainer(StopContainerRequest) returns (StopContainerResponse) {}
  rpc RemoveContainer(RemoveContainerRequest) returns (RemoveContainerResponse) {}
  rpc ContainerStatus(ContainerStatusRequest) returns (ContainerStatusResponse) {}
  rpc ListContainers(ListContainersRequest) returns (ListContainersResponse) {}

  rpc ExecSync(ExecSyncRequest) returns (ExecSyncResponse) {}
  rpc Exec(ExecRequest) returns (ExecResponse) {}
  rpc Attach(AttachRequest) returns (AttachResponse) {}
  rpc PortForward(PortForwardRequest) returns (PortForwardResponse) {}

  rpc ContainerStats(ContainerStatsRequest) returns (ContainerStatsResponse) {}
  rpc Status(StatusRequest) returns (StatusResponse) {}
}
```

Implementations: **containerd** (default in most distros), **CRI-O** (Red Hat OpenShift default).

## 7.6 The runtime stack

```
   kubelet ── CRI ──► containerd / CRI-O
                              │
                              │ uses OCI runtime spec (config.json)
                              ▼
                          runc (or kata, gVisor, youki)
                              │
                              │ creates namespaces, cgroups, mounts
                              ▼
                         Linux kernel — pid/net/ipc/mount/uts/user ns
                                       cgroup v1 / v2
```

- **OCI image spec** — the format of images on disk and in registries.
- **OCI runtime spec** — `config.json` describing how to run (rootfs, command, mounts, namespaces, cgroups).
- **runc** — the reference low-level runtime; pure Linux. Most common.
- **kata** — runs each pod in a lightweight VM (KVM). Higher isolation.
- **gVisor** — userspace kernel. Higher isolation, some compatibility loss.
- **Wasm runtimes** (wasmtime, krustlet) — emerging.

## 7.7 RuntimeClass

Lets a pod opt into a non-default runtime:

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata: {name: kata}
handler: kata
---
apiVersion: v1
kind: Pod
spec:
  runtimeClassName: kata
  containers:
  - name: app
    image: nginx
```

The handler name maps to runtime config in the CRI implementation (containerd has a `runtimes` config block).

## 7.8 Image pull

Kubelet decides whether to pull based on `imagePullPolicy`:
- `Always`: pull on every container creation.
- `IfNotPresent`: pull only if not in cache (default for tags other than `:latest`).
- `Never`: never pull; image must be pre-loaded.

Image pull is gated by **image pull secrets** (registry credentials):
- Pod's `spec.imagePullSecrets` references Secret(s) of type `kubernetes.io/dockerconfigjson`.
- Plus ServiceAccount's `imagePullSecrets`.
- Plus kubelet's `--image-credential-provider-config` (cloud-provider integration; replaces deprecated in-tree credential providers).

### Image pull credential providers (post-1.26)

Pluggable model: kubelet calls an external binary (configured via `--image-credential-provider-config`) for credentials per registry. Each cloud provides one (AWS ECR, GCR, ACR).

### Image pull throttling

```
--registry-qps=5
--registry-burst=10
```

Prevents bombing a registry on node startup.

### Image GC

```
--image-gc-high-threshold=85  # start GC at 85% disk
--image-gc-low-threshold=80   # target 80%
```

Kubelet evicts unused images (by usage time) when disk fills.

## 7.9 Volume management

Kubelet's volume manager:
1. Watches Pod for volume requirements.
2. For PVCs: waits for PV to be bound; calls CSI driver's `NodeStageVolume` (one-time format/attach) and `NodePublishVolume` (mount to pod's dir).
3. For ConfigMaps/Secrets/DownwardAPI: kubelet writes files directly into a `emptyDir`-style location, then bind-mounts.
4. For Projected volumes (e.g., service account tokens): kubelet auto-refreshes content.

`/var/lib/kubelet/pods/<pod-uid>/volumes/` is where mounts land.

## 7.10 Pod eviction

Kubelet evicts pods when node is under resource pressure. Two phases:

### Soft eviction
Configured threshold + grace period. Example:
```
--eviction-soft=memory.available<200Mi,nodefs.available<10%
--eviction-soft-grace-period=memory.available=2m,nodefs.available=2m
```
When breached for grace period, kubelet picks pods to evict (lowest QoS first, then by priority and excess usage).

### Hard eviction
Immediate when breached:
```
--eviction-hard=memory.available<100Mi,nodefs.available<5%
```

### QoS classes (eviction priority order)

| QoS | When |
|-----|------|
| **BestEffort** | No resources set | Evicted first |
| **Burstable** | Some resources set, requests < limits | Middle |
| **Guaranteed** | requests == limits for all resources | Last |

### Soft vs OOM kill

Soft eviction is a kubelet decision (graceful pod termination). OOMKill is the kernel killing a process when cgroup memory limit is hit. Both happen; OOMKill is faster but not pod-aware (can kill main app container while sidecar lives).

## 7.11 Probes

Three types:

| Probe | When | If fails |
|-------|------|----------|
| **Startup** | At container start, until success | Restart container; pauses other probes |
| **Liveness** | Throughout pod lifecycle | Restart container |
| **Readiness** | Throughout pod lifecycle | Remove from Service endpoints |

Probe types: HTTP GET, TCP socket, exec command, gRPC (1.24+).

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 1
  failureThreshold: 3
  successThreshold: 1
```

### Startup probe (KEP-950)

For slow-starting apps. Liveness checks pause until startup succeeds (or initialDelaySeconds × failureThreshold × periodSeconds exhausted). Avoids the common "long-init liveness restart loop" anti-pattern.

## 7.12 cgroups and resource limits

kubelet sets up cgroups for each pod:
- `/sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/kubepods-burstable-<podid>.slice/`
- Within: `<containerid>.scope` per container.

Memory limit → cgroup `memory.max` (v2) or `memory.limit_in_bytes` (v1).
CPU request → `cpu.shares` (v1) or `cpu.weight` (v2) (weight for fair scheduling).
CPU limit → CFS throttling: `cpu.cfs_period_us` + `cpu.cfs_quota_us` (v1) or `cpu.max` (v2).

CPU limit is **soft enforcement** — uses CFS throttling. Container can be paused for milliseconds at a time. Causes notorious latency tail issues; many production deployments **don't set CPU limits** for latency-sensitive workloads.

## 7.13 Node lifecycle

Kubelet writes Node status:
- Conditions: Ready, MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable.
- Capacity (total) and Allocatable (after kubelet/system reservations).
- Heartbeats via NodeLease (kubelet writes a Lease object every 10s — KEP-589).

Node-lifecycle controller (in kube-controller-manager) watches:
- Lease too old → mark NodeReady=Unknown, add `unreachable` taint.
- Stays unknown → eventually node is candidate for eviction.

## 7.14 Static pods

Kubelet can run pods specified in static YAML files in `--pod-manifest-path=/etc/kubernetes/manifests/`. The apiserver, etcd, scheduler, controller-manager in kubeadm clusters are static pods.

Kubelet mirrors them as `Pod` objects (with name suffix `-<nodename>`) so they're visible. Editing the mirror does nothing; you must edit the YAML on disk.

## 7.15 cgroup driver — systemd vs cgroupfs

Two drivers:
- **systemd** (preferred when systemd is PID 1; default on most distros 1.22+).
- **cgroupfs** (direct manipulation of `/sys/fs/cgroup`).

`KubeletConfiguration.cgroupDriver: systemd`. Must match the runtime's setting (containerd config). Mismatch → confusing failures.

## 7.16 cgroup v2

Modern Linux (Ubuntu 22.04+, RHEL 9+) uses cgroup v2 by default. Kubelet supports v2 since 1.22; full features GA in 1.25.

Differences:
- Unified hierarchy (no separate cpu/memory/etc. trees).
- `memory.max` enforced more reliably.
- New controllers (io, hugetlb).
- Some sysctls move (e.g., `memory.oom_group`).

## 7.17 Common interview probes

- **"What happens when a pod is created on a node?"** kubelet sees via watch; PLEG idle → syncLoop kicks; volume manager mounts; CRI RunPodSandbox → CNI ADD → IP; CRI CreateContainer + StartContainer per container; probes start; status update.
- **"Why does PLEG sometimes report unhealthy?"** Slow CRI ListContainers at high pod count. Mitigation: evented PLEG (1.27+).
- **"How does a Pod get its IP?"** kubelet calls CNI ADD; CNI plugin allocates from IPAM, creates veth, returns IP; kubelet records in Pod.Status.
- **"What's the difference between liveness and readiness?"** Liveness restart; readiness removes from Service endpoints. Failed startup probe = restart; pauses liveness/readiness until success.
- **"Why don't people set CPU limits?"** CFS throttling causes tail latency spikes. For p99-sensitive services, requests-only is common.
- **"How does kubelet handle node disk full?"** Eviction by QoS class; image GC.

## 7.18 Corner cases

- **Pod gets stuck in `ContainerCreating`** — usually CNI failure, volume mount failure, or image pull. Check kubelet logs.
- **Pod gets `CrashLoopBackOff`** — exponential backoff between restarts (10s, 20s, 40s... up to 5m).
- **Pod gets `ImagePullBackOff`** — registry unavailable or creds wrong.
- **Pod evicted but no Pod.Status reason** — kubelet shutdown without grace; check `kubectl describe`.
- **Sidecar runs forever after main exits** — pre-1.28 pattern: pod stays Running. Post-1.28 sidecar containers: proper lifecycle.
- **Node Ready but apiserver thinks not** — Lease lag or apiserver clock skew.
- **Containerd OOMs** — high cardinality of containers; bump cgroup limits.

## 7.19 Static vs dynamic kubelet config

Static: command-line flags. Hard to change without restart.
Dynamic (deprecated 1.22+; removed): per-node config via ConfigMap.
Current best practice: KubeletConfiguration file (`--config=/var/lib/kubelet/config.yaml`) with rolling restarts.

## 7.20 Alternative runtimes — when

| Need | Runtime |
|------|---------|
| Default Linux containers | runc (default) |
| Strong isolation (multi-tenant) | gVisor, Kata Containers |
| GPU workloads | nvidia-container-runtime (wraps runc) |
| Wasm workloads | wasmtime, runwasi |
| Rootless | runc with userns + rootless setup |

## Must-internalize

- kubelet syncloop runs every 1s + on events; uses PLEG (evented in 1.27+).
- CRI = gRPC protocol; containerd or CRI-O are the impls.
- Sandbox + pause container holds namespaces; app containers join.
- CSI = volumes; CNI = network; cgroups = limits.
- QoS classes: Guaranteed > Burstable > BestEffort for eviction priority.
- Probes: startup (gates liveness/readiness), liveness (restart), readiness (LB).
- CPU limit causes CFS throttling — careful with latency-sensitive apps.
- Static pods in `/etc/kubernetes/manifests/`; mirror to apiserver.
- cgroup v2 is the modern default; systemd cgroup driver preferred.
