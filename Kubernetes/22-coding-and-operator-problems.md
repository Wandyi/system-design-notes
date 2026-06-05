# 22 · Coding and Operator Problems

The Go-coding round for K8s SWEs. Expect one of these. All sketches use Go + controller-runtime / client-go.

## 22.1 Build a tiny informer

**Prompt**: watch all Pods in a namespace; print add/update/delete events.

```go
package main

import (
    "log"
    "time"

    corev1 "k8s.io/api/core/v1"
    "k8s.io/client-go/informers"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/tools/clientcmd"
    "k8s.io/client-go/util/homedir"
    "path/filepath"
)

func main() {
    kubeconfig := filepath.Join(homedir.HomeDir(), ".kube", "config")
    cfg, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
    if err != nil { log.Fatal(err) }
    cs, err := kubernetes.NewForConfig(cfg)
    if err != nil { log.Fatal(err) }

    factory := informers.NewSharedInformerFactoryWithOptions(cs, 30*time.Second,
        informers.WithNamespace("default"))
    podInformer := factory.Core().V1().Pods().Informer()

    podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc:    func(obj interface{}) { p := obj.(*corev1.Pod); log.Printf("ADD %s", p.Name) },
        UpdateFunc: func(old, new interface{}) { p := new.(*corev1.Pod); log.Printf("UPDATE %s", p.Name) },
        DeleteFunc: func(obj interface{}) {
            p, ok := obj.(*corev1.Pod)
            if !ok {
                // DeletedFinalStateUnknown for missed delete events
                t := obj.(cache.DeletedFinalStateUnknown)
                p = t.Obj.(*corev1.Pod)
            }
            log.Printf("DELETE %s", p.Name)
        },
    })

    stop := make(chan struct{})
    factory.Start(stop)
    factory.WaitForCacheSync(stop)
    log.Println("synced; watching...")
    <-stop
}
```

**Discussion**:
- SharedInformerFactory caches resources; one watch per type.
- `DeletedFinalStateUnknown` for events the cache missed.
- Always `WaitForCacheSync` before serving.

## 22.2 Build a simple reconciler with controller-runtime

**Prompt**: implement a `Foo` reconciler that maintains a Deployment matching `foo.spec.replicas`.

```go
package controllers

import (
    "context"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"

    examplev1 "example.com/api/v1"
)

type FooReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

const finalizer = "foo.example.com/finalizer"

func (r *FooReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var foo examplev1.Foo
    if err := r.Get(ctx, req.NamespacedName, &foo); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Handle deletion
    if !foo.DeletionTimestamp.IsZero() {
        if controllerutil.ContainsFinalizer(&foo, finalizer) {
            // External cleanup (none here)
            controllerutil.RemoveFinalizer(&foo, finalizer)
            return ctrl.Result{}, r.Update(ctx, &foo)
        }
        return ctrl.Result{}, nil
    }

    // Ensure finalizer
    if !controllerutil.ContainsFinalizer(&foo, finalizer) {
        controllerutil.AddFinalizer(&foo, finalizer)
        if err := r.Update(ctx, &foo); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{Requeue: true}, nil
    }

    // Reconcile owned Deployment
    dep := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{Name: foo.Name, Namespace: foo.Namespace},
    }
    op, err := controllerutil.CreateOrUpdate(ctx, r.Client, dep, func() error {
        dep.Spec.Replicas = &foo.Spec.Replicas
        dep.Spec.Selector = &metav1.LabelSelector{MatchLabels: map[string]string{"foo": foo.Name}}
        dep.Spec.Template = corev1.PodTemplateSpec{
            ObjectMeta: metav1.ObjectMeta{Labels: map[string]string{"foo": foo.Name}},
            Spec: corev1.PodSpec{Containers: []corev1.Container{{Name: "main", Image: foo.Spec.Image}}},
        }
        return controllerutil.SetControllerReference(&foo, dep, r.Scheme)
    })
    if err != nil {
        return ctrl.Result{}, err
    }
    _ = op

    // Update status
    foo.Status.AvailableReplicas = dep.Status.AvailableReplicas
    return ctrl.Result{}, r.Status().Update(ctx, &foo)
}

func (r *FooReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&examplev1.Foo{}).
        Owns(&appsv1.Deployment{}).
        Complete(r)
}
```

**Discussion**:
- `CreateOrUpdate` handles create + update in one helper.
- `SetControllerReference` sets the owner ref for GC.
- `r.Status().Update()` uses the status subresource.

## 22.3 Build an admission webhook

**Prompt**: validate that every Deployment has a `team` label.

```go
package webhook

import (
    "context"
    "fmt"
    "net/http"
    "encoding/json"

    admissionv1 "k8s.io/api/admission/v1"
    appsv1 "k8s.io/api/apps/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
)

func handleAdmit(w http.ResponseWriter, r *http.Request) {
    var ar admissionv1.AdmissionReview
    if err := json.NewDecoder(r.Body).Decode(&ar); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    review := ar.Request
    var dep appsv1.Deployment
    if err := json.Unmarshal(review.Object.Raw, &dep); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    resp := &admissionv1.AdmissionResponse{
        UID:     review.UID,
        Allowed: true,
    }

    if _, ok := dep.Labels["team"]; !ok {
        resp.Allowed = false
        resp.Result = &metav1.Status{
            Message: "deployment must have a 'team' label",
            Reason:  metav1.StatusReasonInvalid,
        }
    }

    ar.Response = resp
    json.NewEncoder(w).Encode(ar)
}

func main() {
    http.HandleFunc("/validate", handleAdmit)
    http.ListenAndServeTLS(":8443", "/etc/certs/tls.crt", "/etc/certs/tls.key", nil)
}
```

**Discussion**:
- AdmissionReview struct from `k8s.io/api/admission/v1`.
- Must serve TLS; apiserver doesn't accept plain HTTP.
- ValidatingWebhookConfiguration registers this with caBundle.

## 22.4 Implement an LRU cache for pod metadata

**Prompt**: thread-safe LRU keyed by pod UID. Bounded size.

```go
package cache

import (
    "container/list"
    "sync"
)

type entry struct {
    key string
    val interface{}
}

type LRU struct {
    mu    sync.Mutex
    list  *list.List
    items map[string]*list.Element
    cap   int
}

func New(cap int) *LRU {
    return &LRU{
        list:  list.New(),
        items: make(map[string]*list.Element),
        cap:   cap,
    }
}

func (c *LRU) Get(k string) (interface{}, bool) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if e, ok := c.items[k]; ok {
        c.list.MoveToFront(e)
        return e.Value.(*entry).val, true
    }
    return nil, false
}

func (c *LRU) Set(k string, v interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if e, ok := c.items[k]; ok {
        e.Value.(*entry).val = v
        c.list.MoveToFront(e)
        return
    }
    if c.list.Len() >= c.cap {
        e := c.list.Back()
        c.list.Remove(e)
        delete(c.items, e.Value.(*entry).key)
    }
    e := c.list.PushFront(&entry{k, v})
    c.items[k] = e
}

func (c *LRU) Delete(k string) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if e, ok := c.items[k]; ok {
        c.list.Remove(e)
        delete(c.items, k)
    }
}

func (c *LRU) Len() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.list.Len()
}
```

**Discussion**:
- One mutex; consider sharding for scale.
- For 10M ops/sec: per-shard mutex by hash of key.

## 22.5 Implement a rate-limited work queue

**Prompt**: like client-go's workqueue but custom. Tokens replenish; exponential backoff on retry.

```go
package wq

import (
    "math"
    "sync"
    "time"
)

type RateLimitedQueue struct {
    mu       sync.Mutex
    queue    []string
    seen     map[string]struct{}
    failures map[string]int
    cond     *sync.Cond
}

func New() *RateLimitedQueue {
    q := &RateLimitedQueue{
        seen:     map[string]struct{}{},
        failures: map[string]int{},
    }
    q.cond = sync.NewCond(&q.mu)
    return q
}

func (q *RateLimitedQueue) Add(key string) {
    q.mu.Lock()
    defer q.mu.Unlock()
    if _, ok := q.seen[key]; ok {
        return // dedup
    }
    q.seen[key] = struct{}{}
    q.queue = append(q.queue, key)
    q.cond.Signal()
}

func (q *RateLimitedQueue) AddRateLimited(key string) {
    q.mu.Lock()
    q.failures[key]++
    n := q.failures[key]
    q.mu.Unlock()

    backoff := time.Duration(math.Pow(2, float64(min(n, 10)))) * 10 * time.Millisecond
    go func() {
        time.Sleep(backoff)
        q.Add(key)
    }()
}

func (q *RateLimitedQueue) Forget(key string) {
    q.mu.Lock()
    defer q.mu.Unlock()
    delete(q.failures, key)
}

func (q *RateLimitedQueue) Get() (key string, shutdown bool) {
    q.mu.Lock()
    defer q.mu.Unlock()
    for len(q.queue) == 0 {
        q.cond.Wait()
    }
    key = q.queue[0]
    q.queue = q.queue[1:]
    delete(q.seen, key)
    return key, false
}

func min(a, b int) int { if a < b { return a }; return b }
```

## 22.6 Implement leader election with Lease

**Prompt**: use `coordination.k8s.io/v1` Lease for leader election; only the leader runs reconcile.

```go
package leader

import (
    "context"
    "log"
    "os"
    "time"

    coordinationv1 "k8s.io/api/coordination/v1"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/leaderelection"
    "k8s.io/client-go/tools/leaderelection/resourcelock"
    _ = coordinationv1.Lease{}
)

func Run(ctx context.Context, cs kubernetes.Interface) {
    hostname, _ := os.Hostname()
    lock := &resourcelock.LeaseLock{
        LeaseMeta: metav1.ObjectMeta{Name: "my-controller", Namespace: "kube-system"},
        Client:    cs.CoordinationV1(),
        LockConfig: resourcelock.ResourceLockConfig{Identity: hostname},
    }

    leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
        Lock:            lock,
        LeaseDuration:   15 * time.Second,
        RenewDeadline:   10 * time.Second,
        RetryPeriod:     2 * time.Second,
        ReleaseOnCancel: true,
        Callbacks: leaderelection.LeaderCallbacks{
            OnStartedLeading: func(ctx context.Context) {
                log.Println("became leader; starting controllers")
                // start your reconcile workers
            },
            OnStoppedLeading: func() {
                log.Println("lost leadership; exiting")
                os.Exit(0)
            },
            OnNewLeader: func(identity string) {
                log.Printf("new leader: %s", identity)
            },
        },
    })
}
```

## 22.7 Implement a custom scheduler binary

**Prompt**: watch unscheduled pods with `schedulerName=my-sched`, pick the least-loaded node, bind.

```go
package main

import (
    "context"
    "log"
    "math"

    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/cache"
    "k8s.io/client-go/informers"
    "k8s.io/client-go/tools/clientcmd"
)

func main() {
    cfg, _ := clientcmd.BuildConfigFromFlags("", "/path/kubeconfig")
    cs, _ := kubernetes.NewForConfig(cfg)

    factory := informers.NewSharedInformerFactory(cs, 0)
    podInformer := factory.Core().V1().Pods().Informer()
    nodeInformer := factory.Core().V1().Nodes().Informer()

    nodeLister := factory.Core().V1().Nodes().Lister()

    podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
        AddFunc: func(obj interface{}) {
            pod := obj.(*corev1.Pod)
            if pod.Spec.SchedulerName != "my-sched" || pod.Spec.NodeName != "" {
                return
            }
            nodes, _ := nodeLister.List(labels.Everything())
            node := pickLeastLoaded(nodes)
            if node == nil { return }

            binding := &corev1.Binding{
                ObjectMeta: metav1.ObjectMeta{Name: pod.Name, Namespace: pod.Namespace},
                Target:     corev1.ObjectReference{Kind: "Node", Name: node.Name},
            }
            err := cs.CoreV1().Pods(pod.Namespace).Bind(context.Background(), binding, metav1.CreateOptions{})
            if err != nil { log.Printf("bind error: %v", err) }
        },
    })

    stop := make(chan struct{})
    factory.Start(stop)
    factory.WaitForCacheSync(stop)
    <-stop
}

func pickLeastLoaded(nodes []*corev1.Node) *corev1.Node {
    var best *corev1.Node
    bestScore := math.MaxFloat64
    for _, n := range nodes {
        // Skip if unschedulable
        if n.Spec.Unschedulable { continue }
        // Score: lower is better; use Allocatable - Pods-on-node
        score := computeLoad(n)
        if score < bestScore {
            best = n
            bestScore = score
        }
    }
    return best
}
```

**Discussion**:
- This is a *real* working scheduler, just for pods marked with `schedulerName: my-sched`.
- For production: respect taints/tolerations, node affinity, capacity.

## 22.8 Build an exponential backoff retry

**Prompt**: retry a function up to N times with exponential backoff.

```go
func Retry(ctx context.Context, fn func() error, maxAttempts int, baseDelay time.Duration) error {
    var err error
    for attempt := 0; attempt < maxAttempts; attempt++ {
        if err = fn(); err == nil {
            return nil
        }
        if !isRetryable(err) {
            return err
        }
        delay := baseDelay * time.Duration(1<<attempt)
        if delay > 30*time.Second { delay = 30*time.Second }
        select {
        case <-ctx.Done():
            return ctx.Err()
        case <-time.After(delay):
        }
    }
    return err
}

func isRetryable(err error) bool {
    return errors.IsServerTimeout(err) || errors.IsTooManyRequests(err) || errors.IsServiceUnavailable(err)
}
```

## 22.9 Implement a finalizer-based cascading delete

**Prompt**: a CR Foo deletes external cloud resources before removing finalizer.

```go
func (r *FooReconciler) reconcileDelete(ctx context.Context, foo *examplev1.Foo) (ctrl.Result, error) {
    if !controllerutil.ContainsFinalizer(foo, finalizer) {
        return ctrl.Result{}, nil
    }

    // Delete external resources (cloud API)
    if err := r.cloud.DeleteVPC(ctx, foo.Status.VPCID); err != nil {
        if isRetryable(err) {
            return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
        }
        // Log and skip — don't block forever
        log.Printf("cloud delete failed: %v", err)
    }

    // Remove finalizer
    controllerutil.RemoveFinalizer(foo, finalizer)
    return ctrl.Result{}, r.Update(ctx, foo)
}
```

## 22.10 Implement consistent hashing for endpoints

**Prompt**: given N endpoints, route each request to a consistent endpoint by source IP.

```go
import "hash/crc32"

func pickEndpoint(srcIP string, endpoints []string) string {
    if len(endpoints) == 0 { return "" }
    sort.Strings(endpoints)  // stable order
    h := crc32.ChecksumIEEE([]byte(srcIP))
    return endpoints[int(h)%len(endpoints)]
}
```

For consistent under churn: use **Rendezvous Hashing** or **Maglev**:

```go
func pickEndpointRendezvous(srcIP string, endpoints []string) string {
    var best string
    var bestScore uint32
    for _, e := range endpoints {
        h := crc32.ChecksumIEEE([]byte(srcIP + ":" + e))
        if h > bestScore {
            best = e
            bestScore = h
        }
    }
    return best
}
```

Maglev table (for high-throughput LB): see Linux Networking pack §22.9.

## 22.11 Implement Pod startup wait

**Prompt**: wait for a Deployment's pods to become Ready; timeout.

```go
func WaitForDeployment(ctx context.Context, cs kubernetes.Interface, ns, name string, timeout time.Duration) error {
    ctx, cancel := context.WithTimeout(ctx, timeout)
    defer cancel()

    watch, err := cs.AppsV1().Deployments(ns).Watch(ctx, metav1.ListOptions{
        FieldSelector: "metadata.name=" + name,
    })
    if err != nil { return err }
    defer watch.Stop()

    for evt := range watch.ResultChan() {
        dep, ok := evt.Object.(*appsv1.Deployment)
        if !ok { continue }
        if dep.Status.AvailableReplicas == *dep.Spec.Replicas {
            return nil
        }
    }
    return ctx.Err()
}
```

## 22.12 The common interview-coding gotchas

- **Forgetting `WaitForCacheSync`** — using lister before sync returns empty.
- **Comparing slices/maps** — use `reflect.DeepEqual` or `apiequality.Semantic.DeepEqual`.
- **Updating without GET** — race; use Update with resourceVersion or PATCH.
- **Closing channels too many times** — panic.
- **Goroutine leak in custom watcher** — must Stop the watcher.
- **Forgetting status subresource** — calling `Update` updates spec; for status use `Status().Update()`.
- **Slow reconcile** — use `RequeueAfter` instead of `time.Sleep`.
- **Race on map access** — sync.Mutex or sync.Map.

## 22.13 Patterns to internalize

| Pattern | Where |
|---------|-------|
| Informer + work queue + reconcile | Every operator |
| Finalizer for cleanup | CR with external deps |
| Owner reference for GC | Owned resources |
| `CreateOrUpdate` helper | Idempotent ensure |
| Leader election | HA controllers |
| RequeueAfter for periodic | Polling-style |
| Workqueue rate limit | Failure backoff |
| Watcher with timeout | Wait-for state |
| Type assertion for DeletedFinalStateUnknown | Cache delete events |
| Lister-only reads | Don't burden apiserver |

## Must-internalize

- controller-runtime is the canonical operator framework.
- Use `CreateOrUpdate`, `SetControllerReference`, finalizer helpers.
- Status updates via `Status().Update()`.
- Wait for cache sync before serving.
- Bind subresource for scheduler.
- Lease for leader election (15s/10s/2s defaults).
- Workqueue: dedup, rate-limit, exponential backoff.
