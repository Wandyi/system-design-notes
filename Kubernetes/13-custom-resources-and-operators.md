# 13 · Custom Resources and Operators

The extensibility surface that turns k8s into a platform for arbitrary domain APIs. Most staff interviews include an "implement an operator for X" prompt.

## 13.1 What a CRD does

A **CustomResourceDefinition** registers a new resource type with the apiserver. Once registered, your custom resource behaves like any built-in: kubectl get/apply, RBAC, watch streams, status subresource, etc.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: foos.example.com
spec:
  group: example.com
  scope: Namespaced
  names:
    plural: foos
    singular: foo
    kind: Foo
    shortNames: [foo]
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [size]
            properties:
              size:
                type: integer
                minimum: 1
                maximum: 100
              image:
                type: string
                pattern: '^[a-zA-Z0-9./_-]+$'
          status:
            type: object
            properties:
              phase: {type: string, enum: [Pending, Running, Failed]}
              replicas: {type: integer}
    subresources:
      status: {}
      scale:
        specReplicasPath: .spec.size
        statusReplicasPath: .status.replicas
    additionalPrinterColumns:
    - name: Size
      type: integer
      jsonPath: .spec.size
    - name: Phase
      type: string
      jsonPath: .status.phase
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
```

### Once registered:
```bash
kubectl get foos.example.com
kubectl apply -f my-foo.yaml
# Foo has its own RBAC verbs, watches, etc.
```

## 13.2 Schema validation — OpenAPI v3

The CRD's schema validates instances at admission time:
- Type checking (string, integer, object, array).
- Required fields.
- Constraints: minimum, maximum, pattern, enum, minLength, maxLength.
- `additionalProperties`: whether unknown fields are allowed.

### CRD validation extensions (1.25+)

CEL validation in schema:

```yaml
spec:
  versions:
  - name: v1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              minReplicas: {type: integer}
              maxReplicas: {type: integer}
            x-kubernetes-validations:
            - rule: "self.maxReplicas >= self.minReplicas"
              message: "maxReplicas must be >= minReplicas"
```

Replaces the need for a validating webhook for simple cross-field validation.

## 13.3 Subresources

### status subresource

Splits `spec` and `status` into separate endpoints with separate RBAC:
- `PATCH /foos/<name>` updates spec only.
- `PATCH /foos/<name>/status` updates status only.

Crucial for: avoiding clobbering between controllers updating status and users updating spec.

### scale subresource

Makes the CR target for HPA / `kubectl scale`:
- `specReplicasPath` — where to read/write the desired count.
- `statusReplicasPath` — current count.

```bash
kubectl scale foos my-foo --replicas=5
```

## 13.4 Multi-version CRDs and conversion

A CRD can serve multiple versions:

```yaml
spec:
  versions:
  - name: v1beta1
    served: true
    storage: false
  - name: v1
    served: true
    storage: true   # only one storage version
  conversion:
    strategy: Webhook
    webhook:
      conversionReviewVersions: [v1]
      clientConfig:
        service: {name: convert, namespace: default, path: /convert}
```

When a client requests v1beta1 but storage is v1, apiserver calls the conversion webhook to translate. The webhook is responsible for being correct in both directions.

### conversion strategy

- **None**: no conversion; only valid if schemas are byte-identical (rare).
- **Webhook**: external service handles conversion.

Building a conversion webhook is more work than it looks; if you can avoid multi-version, do.

## 13.5 The operator pattern

An **operator** = a controller that watches a CRD + reconciles to make reality match. Coined by CoreOS.

```
                      User creates a Foo CR
                                │
                                ▼
                       My-operator informer
                                │
                                ▼
                      Reconcile(foo) called:
                       - get desired state from foo.spec
                       - get actual state (own Pods, Services, etc.)
                       - compute diff
                       - apply via apiserver
                       - update foo.status
```

## 13.6 Frameworks

### controller-runtime

The de facto Go framework. Built on client-go.

```go
import "sigs.k8s.io/controller-runtime"

type FooReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

func (r *FooReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var foo examplev1.Foo
    if err := r.Get(ctx, req.NamespacedName, &foo); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Handle deletion
    if !foo.DeletionTimestamp.IsZero() {
        return r.cleanup(ctx, &foo)
    }

    // Ensure finalizer
    if !controllerutil.ContainsFinalizer(&foo, finalizerName) {
        controllerutil.AddFinalizer(&foo, finalizerName)
        if err := r.Update(ctx, &foo); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{Requeue: true}, nil
    }

    // Reconcile owned resources
    if err := r.reconcileDeployment(ctx, &foo); err != nil {
        return ctrl.Result{}, err
    }

    // Update status
    foo.Status.Phase = "Running"
    if err := r.Status().Update(ctx, &foo); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}

func (r *FooReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&examplev1.Foo{}).
        Owns(&appsv1.Deployment{}).
        Watches(&corev1.ConfigMap{}, handler.EnqueueRequestsFromMapFunc(r.mapConfigMapToFoo)).
        WithOptions(controller.Options{MaxConcurrentReconciles: 10}).
        Complete(r)
}
```

### Kubebuilder

CLI scaffolding on top of controller-runtime:
```bash
kubebuilder init --domain example.com
kubebuilder create api --group example --version v1 --kind Foo
```

Generates CRD YAML, controller stub, RBAC manifests.

### Operator SDK (Red Hat)

Similar; richer scaffolding for Helm/Ansible-based operators.

### Other languages

- **kopf** (Python).
- **operator-sdk Ansible/Helm** (no Go).
- **java-operator-sdk** (Java).
- **kube-rs** (Rust).

## 13.7 Owner references — cascade delete

```go
import "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"

// In your reconciler:
if err := controllerutil.SetControllerReference(&foo, &dep, r.Scheme); err != nil {
    return err
}
return r.Create(ctx, &dep)
```

This sets `metadata.ownerReferences[]` on the Deployment with `controller: true`. When Foo is deleted, the GC controller cascades the deletion to the Deployment.

Only one "controller" owner reference per object (the controller owner); other owner refs are non-controller.

Constraints:
- Owner must be in the same namespace (or cluster-scoped).
- An owner ref to a cluster-scoped Foo can own namespaced objects only if `foo` is itself cluster-scoped.

## 13.8 Finalizers

Already covered in 05. Recap:

```go
const finalizerName = "myoperator.example.com/cleanup"

if !foo.DeletionTimestamp.IsZero() {
    if controllerutil.ContainsFinalizer(&foo, finalizerName) {
        if err := r.deleteExternalResources(ctx, &foo); err != nil {
            return err
        }
        controllerutil.RemoveFinalizer(&foo, finalizerName)
        if err := r.Update(ctx, &foo); err != nil {
            return err
        }
    }
    return nil  // foo will be GC'd by apiserver now
}

if !controllerutil.ContainsFinalizer(&foo, finalizerName) {
    controllerutil.AddFinalizer(&foo, finalizerName)
    return r.Update(ctx, &foo)
}
```

The finalizer prevents the apiserver from actually deleting the object until your controller has done its cleanup (deleted cloud resources, etc.).

## 13.9 Reconcile idempotency — the rules

1. **Always re-derive state from apiserver.** Don't trust local variables across reconciles.
2. **Use SSA or PATCH, not UPDATE.** UPDATE replaces the full object; multiple controllers fight.
3. **Compute diffs; only patch on change.** Avoids infinite reconcile loops.
4. **Status update is separate.** Use `r.Status().Update()` or status patch.
5. **No work in reconcile init**; do all the work each call.
6. **Return `RequeueAfter`** when you need periodic re-check (TTL-style).
7. **Return `Requeue: true`** when work isn't done but no specific time.
8. **Return `nil, nil`** when done.

## 13.10 Webhook patterns for CRDs

### Validation webhook
Check structural invariants beyond OpenAPI. E.g., "owner field must reference an existing User."

### Mutation webhook
Set defaults (e.g., default image tag), add annotations, inject sidecars.

### Conversion webhook
Translate between API versions.

### Implementation

```go
import "sigs.k8s.io/controller-runtime/pkg/webhook/admission"

func (r *Foo) ValidateCreate() (admission.Warnings, error) {
    if r.Spec.Size > 100 && r.Annotations["override-size-limit"] != "true" {
        return nil, fmt.Errorf("size cannot exceed 100 without override")
    }
    return nil, nil
}

func (r *Foo) Default() {
    if r.Spec.Image == "" {
        r.Spec.Image = "default:latest"
    }
}
```

Kubebuilder scaffolds the webhook server, cert mounts, WebhookConfiguration.

## 13.11 Status conditions — the standard pattern

```yaml
status:
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2026-06-02T12:00:00Z"
    reason: AllReplicasReady
    message: "All 5 replicas are ready"
  - type: Progressing
    status: "False"
    lastTransitionTime: "2026-06-02T11:55:00Z"
    reason: NewReplicaSetAvailable
    message: "Deployment has minimum availability"
```

Use the meta/v1 Condition type:
```go
import "k8s.io/apimachinery/pkg/api/meta"
meta.SetStatusCondition(&foo.Status.Conditions, metav1.Condition{
    Type: "Ready", Status: metav1.ConditionTrue,
    Reason: "AllReady", Message: "All 5 replicas ready",
})
```

`meta.SetStatusCondition` is idempotent — doesn't bump `lastTransitionTime` unless status actually changed.

## 13.12 Operator lifecycle

### Operator Lifecycle Manager (OLM)

For distribution: package your operator as a CSV (ClusterServiceVersion) + CRDs. OLM manages installation, upgrade, dependencies.

Mostly Red Hat / OperatorHub ecosystem. For internal operators, plain Helm charts often suffice.

### Helm + operator
Operator-SDK Helm-style: operator wraps a Helm chart and reconciles by `helm upgrade`. Lowest-effort.

## 13.13 Common operator patterns

### Single-resource ownership

Foo creates Deployment + Service + ConfigMap. Owner refs cascade delete. Simple.

### Multi-resource with dependencies

Foo creates Deployment + waits for it to be Ready + then creates a follow-up. Status conditions tracking the multi-step rollout.

### Cluster-singleton

Foo is a cluster-scoped resource; only one allowed (validate via webhook). Manages cluster-wide config.

### Multi-tenant

Operator runs cluster-wide but Foo is namespaced; each namespace has its own Foo. Use a label predicate or namespace selector.

### Hub-and-spoke

Operator in management cluster watches Foo; reconciles by applying YAML to many other clusters (multi-cluster). Cluster API does this.

### Database operator

Stateful, multi-step:
- StatefulSet for replicas.
- Headless Service for pod DNS.
- Periodic backup CronJob.
- Status reflects replication lag, version.
- Webhook validates upgrade path.

Examples: postgres-operator (zalando), kubedb, mysql-operator.

## 13.14 RBAC for operators

Operator's ServiceAccount needs permissions on:
- The CR it manages (`get/list/watch/update`).
- The resources it creates (Deployments, Services, etc.) — `create/update/delete/get/list/watch`.
- Status subresource of the CR (`update/patch`).
- Optionally: ConfigMaps, Secrets, Events.

Kubebuilder generates this from `// +kubebuilder:rbac:...` markers.

```go
// +kubebuilder:rbac:groups=example.com,resources=foos,verbs=get;list;watch;update;patch
// +kubebuilder:rbac:groups=example.com,resources=foos/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=events,verbs=create;patch
```

## 13.15 Watching secondary resources

If your operator owns Deployments and reacts to their changes:

```go
ctrl.NewControllerManagedBy(mgr).
    For(&Foo{}).
    Owns(&appsv1.Deployment{}).   // watches Deployments owned by Foo
    Complete(r)
```

`Owns` watches all Deployments and, for each, finds the owning Foo (via ownerReferences) and enqueues a reconcile.

For non-owned references (e.g., react to ConfigMap changes), use `Watches` + a mapping function.

## 13.16 Common interview probes

- **"Design a CRD for X."** Walk through schema, validation, subresources (status, scale), conversion strategy, finalizer logic, owner refs.
- **"How do you handle CRD versioning?"** Multiple versions in CRD; conversion webhook; bump storage version after migration.
- **"What's the difference between a controller and an operator?"** Operator is a controller + CRD + domain knowledge. All operators are controllers; not all controllers are operators (e.g., the Deployment controller).
- **"How do you ensure your operator scales?"** MaxConcurrentReconciles in controller options; shard by leader election; rate limiter on workqueue; avoid hot loops.
- **"How does GC work for your CRs?"** Owner references; foreground deletion if you need to wait for dependents.

## 13.17 Corner cases

- **CR schema change breaks existing instances** — must be a backward-compatible change OR use conversion webhook. Pre-1.25, no way to add required fields without breaking existing.
- **Finalizer stuck** — if your operator is broken or removed, the finalizer is never removed → CR can't be deleted. Manual `kubectl patch -p '{"metadata":{"finalizers":null}}'`. Always document this for ops.
- **Operator crash loop with infinite finalizer-add** — your reconcile fails after adding finalizer but before completing. Idempotency rules.
- **Two operators reconcile the same CR** — undefined; one always wins. Use only one controller per CR.
- **Status churn** — reconcile updates status which fires another event which reconciles again. Use `meta.SetStatusCondition` (idempotent) or hash-compare before write.
- **Cross-namespace owner references** — not allowed for namespaced owners; allowed for cluster-scoped → namespaced.
- **Webhook + CRD same controller** — webhook needs cert before apiserver can call it; chicken-and-egg on cluster init. Use cert-manager or self-signed bootstrap.

## 13.18 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Extend k8s API | CRD | Aggregation layer (custom apiserver) | Annotation on existing resource |
| Operator language | Go (controller-runtime) | Python (kopf) | Helm/Ansible (operator-sdk) |
| Validation | OpenAPI schema | CEL validation rules | ValidatingWebhook |
| Defaulting | OpenAPI default + admission | MutatingWebhook | Operator init logic |
| Conversion | Webhook | rename + drop old | data migration via finalizer |
| Multi-tenant | One operator per ns | namespace selector | label selector |
| Cluster-singleton | Webhook validate count=1 | Built-in controller pattern | OLM |

## 13.19 Bad operator patterns to avoid

- **Reconcile that fetches state from multiple APIs in series** → slow + unreliable. Cache via informers.
- **Reconcile that depends on event order** → events can be coalesced, dropped, replayed.
- **Status updates as side channel for control** → use spec; spec is for desired state.
- **Long-running operations in-reconcile** → reconcile should be quick; long ops should be kicked off and tracked via status.
- **Bypassing the work queue** — losing dedup + rate limiting.

## Must-internalize

- CRD = new resource type; status subresource separates spec/status RBAC.
- Operator = CRD + controller; built on controller-runtime / kubebuilder.
- Reconcile idempotent; always re-derive from apiserver.
- Finalizers for ordered cleanup.
- Owner references for cascade delete.
- CEL validation in CRD schema (1.25+) replaces simple webhooks.
- Conversion webhook for multi-version CRDs.
- Status conditions pattern with `meta.SetStatusCondition`.
- RBAC markers in controller code; kubebuilder generates manifests.
- `Owns` for owned resources; `Watches` for cross-resource reactions.
