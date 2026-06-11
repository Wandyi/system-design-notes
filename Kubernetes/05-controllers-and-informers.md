# 05 · Controllers and Informers — The Reconcile Pattern

The single most important *programming* pattern in Kubernetes. Every k8s component, every operator, every CRD controller is a variation of this.

## 5.1 The pattern

```
    ┌────────────┐  watch  ┌────────────┐
    │ apiserver  │ ◄────── │  Informer  │
    └────────────┘         └─────┬──────┘
                                 │ (add/update/delete event)
                                 ▼
                       ┌─────────────────────┐
                       │  Event Handler      │
                       │  (per-controller)   │
                       │  - extract key      │
                       │  - workqueue.Add    │
                       └─────────┬───────────┘
                                 │
                                 ▼
                       ┌─────────────────────┐
                       │  Work Queue         │   (rate-limited, deduplicated)
                       └─────────┬───────────┘
                                 │
                                 ▼
                       ┌─────────────────────┐
                       │  Reconcile worker(s)│
                       │  - load object      │
                       │  - compute desired  │
                       │  - apply diff       │
                       │  - on failure:      │
                       │    re-queue with    │
                       │    backoff          │
                       └─────────────────────┘
```

The pattern is **idempotent reconciliation** — running reconcile twice for the same object produces the same end state. No procedural orchestration.

## 5.2 Informers

An **informer** is a long-running goroutine that:
1. LISTs all objects of a type (full set).
2. Stores them in a local **store** (a thread-safe cache).
3. Begins a WATCH from the listed `resourceVersion`.
4. Dispatches events to registered handlers.

```go
factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)
podInformer := factory.Core().V1().Pods().Informer()

podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: func(obj interface{}) {
        pod := obj.(*v1.Pod)
        queue.Add(pod.Namespace + "/" + pod.Name)
    },
    UpdateFunc: func(old, new interface{}) {
        queue.Add(...)
    },
    DeleteFunc: func(obj interface{}) {
        queue.Add(...)
    },
})

factory.Start(ctx.Done())
factory.WaitForCacheSync(ctx.Done())
```

### Shared informer factory

Multiple controllers watching the same resource share one informer → one watch stream to apiserver, one local cache. 
This is critical for scale: 100 operators on a cluster all watching Pods would be 100× the watch load without shared informers.

### Lister

```go
podLister := factory.Core().V1().Pods().Lister()
pod, err := podLister.Pods("default").Get("my-pod")  // reads from cache, no apiserver call
```

The lister is the canonical "read" surface. **Never** call the apiserver directly to read — always use the lister.

### Cache freshness

The cache is **eventually consistent**. There can be a brief window where:
- You create an object via apiserver.
- The informer hasn't yet processed the watch event.
- The lister returns "not found."

This is normal. Reconcilers must handle "not found in cache" gracefully (often by requeuing).

## 5.3 Work queues

The standard work queue:
- **Deduplicated** — Adding the same key twice is a no-op until processed.
- **Rate-limited** — exponential backoff on requeue after failure.
- **Thread-safe** — many workers can pop concurrently.

```go
queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

// Worker:
go func() {
    for {
        key, shutdown := queue.Get()
        if shutdown { return }
        err := reconcile(key.(string))
        if err != nil {
            queue.AddRateLimited(key)  // backoff
        } else {
            queue.Forget(key)  // reset backoff
        }
        queue.Done(key)
    }
}()
```

Default rate limiter: 5ms → 1s exponential per-item + 10qps overall bucket.

## 5.4 The reconciler

```go
func reconcile(key string) error {
    namespace, name, _ := cache.SplitMetaNamespaceKey(key)

    obj, err := lister.Get(name)
    if errors.IsNotFound(err) {
        // Object was deleted; clean up downstream
        return cleanup(namespace, name)
    }
    if err != nil {
        return err  // requeue
    }

    // Compute desired state
    desired := computeDesired(obj)

    // Apply diff
    err = applyDiff(obj, desired)
    if err != nil {
        return err
    }

    // Update status
    obj.Status.Phase = "Ready"
    _, err = clientset.UpdateStatus(obj)
    return err
}
```

### Idempotency rules

- Reconciling twice for the same input must produce the same output state.
- Don't keep state in the reconciler memory; always re-derive from the apiserver.
- Don't assume the previous reconcile completed — it may have crashed.

### Patching vs updating

```go
// UPDATE — replaces the full object; race-prone if other writers exist
obj.Status.Phase = "Ready"
_, err := clientset.Update(obj)

// PATCH — sends only the changed fields; better for multi-writer
patch := []byte(`{"status":{"phase":"Ready"}}`)
_, err := clientset.Patch(name, types.StrategicMergePatchType, patch)

// SSA (Server-Side Apply) — declarative apply with field ownership
applyConfig := corev1ac.Pod(name, namespace).WithStatus(...)
_, err := clientset.Apply(ctx, applyConfig, metav1.ApplyOptions{FieldManager: "my-controller"})
```

Modern controllers use SSA where possible.

## 5.5 Leader election

For HA controllers (run multiple replicas, one active at a time):

```go
import "k8s.io/client-go/tools/leaderelection"

leLock := &resourcelock.LeaseLock{
    LeaseMeta: metav1.ObjectMeta{
        Name:      "my-controller",
        Namespace: "kube-system",
    },
    Client: client.CoordinationV1(),
    LockConfig: resourcelock.ResourceLockConfig{
        Identity: hostname,
    },
}

leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
    Lock:          leLock,
    LeaseDuration: 15 * time.Second,
    RenewDeadline: 10 * time.Second,
    RetryPeriod:   2 * time.Second,
    Callbacks: leaderelection.LeaderCallbacks{
        OnStartedLeading: func(ctx context.Context) {
            // Start your controller loops
            startReconcileWorkers()
        },
        OnStoppedLeading: func() {
            os.Exit(0)  // step down → exit; next leader takes over
        },
    },
})
```

Three timeouts:
- **LeaseDuration**: time the lease is valid (default 15s).
- **RenewDeadline**: deadline to renew (default 10s).
- **RetryPeriod**: retry interval while not leader (default 2s).

Failover: ~15-30s (lease expires, another replica acquires).

## 5.6 The reconcile event flow — a Deployment example

```
   user kubectl apply Deployment foo
                       ↓
   apiserver writes Deployment object to etcd (rev=100)
                       ↓
   watch broadcast → Deployment controller's informer
                       ↓
   informer event handler → queue.Add("default/foo")
                       ↓
   worker pops "default/foo" → reconcile(foo)
                       ↓
   reconcile:
     - get Deployment from lister
     - compute desired ReplicaSet (hash of PodTemplate)
     - check if matching ReplicaSet exists
     - if not: create RS via apiserver POST
     - scale old RS down, new RS up per RollingUpdate
                       ↓
   apiserver writes new ReplicaSet (rev=101)
                       ↓
   watch broadcast → ReplicaSet controller's informer
                       ↓
   ReplicaSet controller reconciles:
     - get RS from lister
     - count Pods owned by it
     - if < replicas: create N pods via apiserver POST
                       ↓
   apiserver writes Pods (rev=102, 103, ...)
                       ↓
   watch broadcast → scheduler informer
                       ↓
   scheduler binds each pod to a node
                       ↓
   ... (and so on)
```

Each controller is unaware of the others; coordination happens via apiserver state.

## 5.7 Finalizers — the cleanup hook

A **finalizer** is a string in `metadata.finalizers[]`. When the object is deleted via apiserver:
1. apiserver sets `metadata.deletionTimestamp` (object enters "deleting" state).
2. Object is NOT removed from etcd while any finalizer remains.
3. Each controller responsible for a finalizer does its cleanup, then removes its finalizer.
4. When `len(finalizers) == 0`, apiserver actually deletes from etcd.

```go
const myFinalizer = "myoperator.example.com/cleanup"

func reconcile(obj *MyResource) error {
    if obj.DeletionTimestamp != nil {
        // Object being deleted
        if !containsString(obj.Finalizers, myFinalizer) {
            return nil  // not ours, nothing to do
        }
        // Do cleanup (delete cloud resources, etc.)
        err := cleanupExternal(obj)
        if err != nil {
            return err  // requeue
        }
        // Remove our finalizer
        obj.Finalizers = removeString(obj.Finalizers, myFinalizer)
        return clientset.Update(obj)
    }

    // Normal reconciliation
    if !containsString(obj.Finalizers, myFinalizer) {
        obj.Finalizers = append(obj.Finalizers, myFinalizer)
        if err := clientset.Update(obj); err != nil {
            return err
        }
    }
    // ... rest of reconcile
}
```

Finalizers turn delete into a deletion *workflow* — you can ensure cleanup happens before the object disappears.

## 5.8 Owner references — cascade delete

```yaml
metadata:
  name: pod-xyz
  ownerReferences:
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: rs-abc
    uid: <uid>
    controller: true
    blockOwnerDeletion: true
```

The kube-controller-mana    ger's **garbage collector** watches owner refs. When `rs-abc` is deleted, the GC finds all objects owning it 
(Pods with that owner ref) and deletes them.

Three deletion modes:
- **Foreground**: deletion blocks until dependents are gone (`metadata.finalizers: [foregroundDeletion]`).
- **Background** (default): apiserver deletes immediately; GC cleans up dependents asynchronously.
- **Orphan**: dependents are *not* deleted (set on `kubectl delete --cascade=orphan`).

## 5.9 Status subresource

By default, updating `status` requires the same permissions as `spec`. The **status subresource** (`/foo/<name>/status` separate endpoint) splits this:
- Controllers patch status with their own permissions.
- Users patch spec with theirs.
- An update to `spec` doesn't accidentally clobber `status` and vice versa.

CRDs enable status subresource via:
```yaml
spec:
  versions:
  - name: v1
    served: true
    storage: true
    subresources:
      status: {}
```

## 5.10 controller-runtime — modern abstraction

`sigs.k8s.io/controller-runtime` is the canonical abstraction (used by Kubebuilder, Operator SDK). It hides much of the boilerplate:

```go
import "sigs.k8s.io/controller-runtime"

type MyReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var obj MyResource
    if err := r.Get(ctx, req.NamespacedName, &obj); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // reconcile logic

    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}

func (r *MyReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&MyResource{}).
        Owns(&corev1.Pod{}).
        Complete(r)
}
```

- `For(&MyResource{})`: primary type the reconciler manages.
- `Owns(&corev1.Pod{})`: secondary types; their changes trigger reconcile on the parent.
- `Watches`: arbitrary other events (for cross-resource reconciliation).

The manager handles informer setup, work queue, leader election, metrics, health.

## 5.11 Common interview probes

- **"How would you build a controller for X?"** Informer + work queue + reconcile loop; idempotent; uses status subresource; has finalizer.
- **"What if reconcile takes 30s?"** Other items wait in queue → throughput drops. Either parallelize (more workers), break work into smaller pieces, or async (kick off work, requeue to check).
- **"How do you handle a resource being deleted mid-reconcile?"** Get returns 404 → return nil to forget; lister will eventually catch up.
  - **"What if your controller writes back to apiserver in a tight loop?"** Watch-event → reconcile → write → watch-event → infinite loop. 
        Mitigations: check if state already matches before writing; use a status hash to detect change; check `metadata.resourceVersion`.
- **"How do you handle multiple controllers reconciling the same resource?"** Use SSA with separate fieldManagers; or assign owner controller (only one writes spec, others only watch).

## 5.12 Corner cases

- **Informer cache stale on startup** — call `factory.WaitForCacheSync()` before serving.
- **Informer doesn't see events after long pause** — relist storm if watch cache expired. Use bookmark events.
- **Work queue stuck** — bad reconcile that always errors → item requeued forever with backoff. Add max retries or alarming.
- **Two leaders briefly active** — if `LeaseDuration` is too short relative to clock drift. Standard recommendations: 15s lease, 10s renew, 2s retry.
- **Status thrashing** — controller writes Status repeatedly. Add hash check before writing.
- **Resource version conflict** — `409 Conflict` on update → re-fetch + retry.

## 5.13 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Watch a resource | client-go informer | controller-runtime | dynamic informer (any GVK) |
| Reconcile | manual workqueue | controller-runtime | metacontroller |
| Cross-resource | Watches in CR | Owns + secondary informer | Watch + custom EventHandler |
| Cleanup on delete | finalizer | owner reference cascade | external (cron) |
| Multi-tenant operator | one ctrl per ns | namespace selector | dynamic ns scoping |
| Status persistence | status subresource | annotation | external store |

## Must-internalize

- The pattern: informer → event → workqueue → reconcile → apiserver write.
- Reconcile is idempotent — re-running yields the same state.
- Shared informer factory: many controllers, one watch per type.
- Lister is the read surface; never read directly from apiserver.
- Work queue dedup + rate limit + exponential backoff are built-in.
- Finalizers turn delete into a workflow; remove finalizer when cleanup is done.
- Owner references → garbage collector cascade-deletes dependents.
- controller-runtime + kubebuilder is the modern operator pattern.
- Leader election via Lease resource for HA controllers.
- Status subresource separates spec/status RBAC and prevents clobbering.
