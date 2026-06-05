# 03 · API Server Deep Dive

`kube-apiserver` is the only component that talks to etcd. Everything else is a client. Every interesting design constraint in the cluster eventually shows up here.

## 3.1 What the apiserver actually does

```
   HTTPS port 6443
        │
        ▼
   ┌────────────────────────────────────────────┐
   │ go-restful router                          │
   │   /api, /apis, /healthz, /metrics, /openapi│
   └────────────────────────────────────────────┘
        │
        ▼
   ┌────────────────────────────────────────────┐
   │ Filters (in order):                        │
   │   - PanicRecovery                          │
   │   - RequestInfo (parse verb+resource)      │
   │   - WithRequestDeadline                    │
   │   - WithAudit (event creation)             │
   │   - WithAuthentication (x509/token/OIDC)   │
   │   - WithImpersonation                      │
   │   - WithMaxInFlightLimit / WithPriorityAndFairness │
   │   - WithAuthorization (RBAC/Webhook)       │
   └────────────────────────────────────────────┘
        │
        ▼
   ┌────────────────────────────────────────────┐
   │ Handler:                                   │
   │   - Decode (JSON/Protobuf/YAML)            │
   │   - Mutating admission webhooks            │
   │   - Validation (OpenAPI / CRD schema)      │
   │   - Validating admission webhooks          │
   │   - Storage (etcd write)                   │
   │   - Watch broadcast                        │
   └────────────────────────────────────────────┘
        │
        ▼
   ┌────────────────────────────────────────────┐
   │ Response                                   │
   │   - audit completion event                 │
   │   - metrics                                │
   └────────────────────────────────────────────┘
```

## 3.2 Authentication — the chain

Configured via `--authentication-config` (1.30+) or legacy flags:

1. **x509 client cert** — matches `Subject: CN` against allowed CAs. `kubeconfig` uses this.
2. **Bearer token (service account)** — JWT signed by the apiserver's SA signing key.
3. **OIDC** — JWT from an external provider (Google, Azure AD, Okta). `--oidc-issuer-url`.
4. **Webhook authenticator** — apiserver calls external service with the token; gets back `UserInfo`.
5. **Authenticating proxy** (`X-Remote-User` header) — used by `kube-aggregator`.
6. **Anonymous** — for unauthenticated requests, mapped to `system:anonymous`.

Each authenticator runs in order. First one to return a verdict wins. `UserInfo` includes username, groups, UID, extra map.

### Common gotchas

- **Service account tokens**: pre-1.20 were stored in a Secret of type `kubernetes.io/service-account-token`; 1.21+ are projected (audience-scoped, short-lived, mounted as projected volume). The `Secret` approach still works but is deprecated.
- **OIDC token lifetime**: must be re-validated on every request (apiserver doesn't cache). Slow OIDC = slow apiserver.
- **Webhook TokenReview**: cache size and TTL matter. Misconfig = every request hits external authenticator.

## 3.3 Authorization — RBAC primary, others available

Modes via `--authorization-mode=Node,RBAC` (default in kubeadm clusters):

1. **Node** — special mode; lets each node's kubelet only access pods/secrets bound to itself.
2. **RBAC** — role-based access control via Role/ClusterRole + RoleBinding/ClusterRoleBinding.
3. **Webhook** — call external for authorization decision.
4. **AlwaysAllow** / **AlwaysDeny** — testing only.
5. **ABAC** — legacy; static file-based policy. Don't use.

RBAC objects:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {namespace: foo, name: pod-reader}
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {namespace: foo, name: alice-can-read}
subjects:
- kind: User
  name: alice
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Authorization is a sequence; first ALLOW wins. RBAC is deny-by-default.

### Aggregated cluster roles (1.9+)

A ClusterRole can include `aggregationRule` selecting other ClusterRoles by label. The controller aggregates rules. Used by `edit`, `admin`, `view` defaults.

## 3.4 Admission control — the most powerful extensibility point

Two phases: **mutating** (can change the object) and **validating** (yes/no).

Built-in admission plugins:
- `NamespaceLifecycle` — block writes to terminating namespaces.
- `LimitRanger` — enforce LimitRange defaults / max.
- `ServiceAccount` — auto-assign default SA, project volume.
- `ResourceQuota` — enforce ResourceQuota.
- `PodSecurity` — Pod Security Admission (PSS profiles).
- `DefaultIngressClass`, `DefaultStorageClass` — annotate defaults.
- `Priority` — assign PriorityClass to pods.
- `RuntimeClass` — set runtime class.
- `NodeRestriction` — kubelet can only modify its own node + pods on it.
- `MutatingAdmissionWebhook`, `ValidatingAdmissionWebhook` — external.
- `ValidatingAdmissionPolicy` (1.30 GA) — CEL-based inline validation.

### External admission webhooks

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata: {name: my-validator}
webhooks:
- name: my-validator.example.com
  clientConfig:
    service: {name: my-validator-svc, namespace: kube-system, path: /validate}
    caBundle: <base64 ca>
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE", "UPDATE"]
    resources: ["pods"]
  failurePolicy: Fail
  sideEffects: None
  admissionReviewVersions: ["v1"]
  timeoutSeconds: 5
```

The webhook receives an `AdmissionReview` (JSON), responds with `allowed: true|false` + optional `patch` (JSON Patch, base64-encoded) for mutating.

### Webhook hazards

| Hazard | Mitigation |
|--------|------------|
| Webhook down → cluster broken (failurePolicy: Fail) | `failurePolicy: Ignore` for non-critical; namespaceSelector to exclude kube-system |
| Webhook latency adds to every write | `timeoutSeconds: 5` strict; profile; consider VAP for hot paths |
| Webhook creates feedback loops (mutating own resources) | `reinvocationPolicy: Never` or careful design |
| TLS cert expires | cert-manager auto-rotation; monitor |
| Webhook only configured for one apiVersion (v1) but apiserver sends v1beta1 | List all versions in `admissionReviewVersions` |
| Webhook resource bloats writes | Filter by `objectSelector`, `namespaceSelector` |

### ValidatingAdmissionPolicy (1.30 GA)

CEL expressions evaluated in-apiserver, no webhook hop:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata: {name: replica-cap}
spec:
  matchConstraints:
    resourceRules:
    - apiGroups: ["apps"]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["deployments"]
  validations:
  - expression: "object.spec.replicas <= 10"
    message: "Replicas cannot exceed 10"
```

This is **much** faster than a webhook (no network hop). Use VAP for simple checks; webhooks for anything stateful (querying external systems).

## 3.5 The watch story — internals

Apiserver maintains a **watch cache** per resource type:

```
   ┌────────────────────────┐
   │ Ring buffer            │  size: --watch-cache-sizes (default 1000)
   │  [event @ rv=100]      │
   │  [event @ rv=101]      │
   │  ...                   │
   │  [event @ rv=1098]     │
   │  [event @ rv=1099]     │  ← head
   └────────────────────────┘
            ▲
            │
   etcd progress notifier feeds events
```

Watcher sends `resourceVersion=N` on connect:
- If N is within the cache → stream from N.
- If N is older → `410 Gone` → client must full-list + watch.

### Watch cache scaling

At 5000 nodes, watching Pods → ~50 events/sec from controllers, plus kubelet status updates. The cache fills fast. Tunings:
- `--watch-cache-sizes=pods#10000` per-resource sizes.
- `--max-mutating-requests-inflight=2000` to control writes.
- `--max-requests-inflight=10000` to control reads.

### Bookmarks

Servers periodically (configurable) emit a `BOOKMARK` event with the current RV. Watchers checkpoint without depending on actual changes. This prevents a stale watcher that's just been idle from getting a `410 Gone`.

## 3.6 Conversion — multiple versions of one resource

Each resource lives at multiple API versions: `v1alpha1`, `v1beta1`, `v1`. The apiserver picks one as the **storage version**; the others convert on read/write.

For native resources, conversion functions are compiled in (`pkg/apis/.../conversion.go`).

For CRDs, two strategies:
- **None** — strict schema, no conversion. Only works if versions are byte-identical.
- **Webhook** — apiserver calls a conversion webhook. Required for non-trivial migrations.

CRD conversion webhook surface:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata: {name: foos.example.com}
spec:
  conversion:
    strategy: Webhook
    webhook:
      conversionReviewVersions: ["v1"]
      clientConfig:
        service: {name: foo-convert, namespace: default, path: /convert}
```

## 3.7 API Priority and Fairness (APF)

Replaces the simple "max in-flight" limit. Categories:
- **FlowSchema** — pattern-match incoming requests (by user, group, namespace, verb).
- **PriorityLevelConfiguration** — concurrency limit + queueing config.
- **PriorityLevel** — class.

Default categories: `system`, `leader-election`, `workload-high`, `workload-low`, `global-default`, `exempt`.

`exempt` priority bypasses all limits (kubectl exec, leader election).

Tuning at scale: bump nominal concurrency for `workload-high` (where controller-manager runs).

## 3.8 Storage — encryption at rest

`--encryption-provider-config` enables encryption of Secrets (and optionally other types) in etcd:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources: ["secrets"]
  providers:
  - aescbc: {keys: [{name: key1, secret: <base64>}]}
  - identity: {}
```

Identity at the end = fallback to no encryption for unencrypted entries. Rotating keys requires reading + rewriting every Secret.

KMS providers (`kms`) integrate with cloud KMS (AWS KMS, GCP Cloud KMS) so master key never leaves HSM.

## 3.9 Aggregation layer

External apiservers can serve specific API groups, fronted by kube-apiserver:

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata: {name: v1beta1.metrics.k8s.io}
spec:
  service: {name: metrics-server, namespace: kube-system}
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
```

`metrics-server`, `custom-metrics-apiserver`, `kube-state-metrics` (sometimes) use this. The aggregated apiserver implements the k8s API surface (list/watch/get) for its types; kube-apiserver proxies traffic to it.

## 3.10 Server-Side Apply (SSA)

Strategic merge patches were complex; SSA replaces them.

Every `apply` from a manager is tagged with a `fieldManager`. The apiserver tracks **which manager owns which field** in `metadata.managedFields`. Two managers can co-edit; conflicts are detected and reported.

```bash
kubectl apply --server-side -f deployment.yaml --field-manager=ci-bot
```

When the deployment-controller wants to change `spec.replicas` and HPA also wants to, they each touch their fields; SSA reports a conflict only if they touch the same field.

Used heavily by controllers; `client-go` library has SSA support.

## 3.11 Common interview probes

- **"Walk me through the admission chain."** Auth → admission mutating → validation → admission validating → storage.
- **"How does watch work?"** Long-lived HTTP/2 stream; resourceVersion checkpoint; watch cache ring buffer; bookmark events; 410 Gone for stale RV.
- **"How would you ensure all Pods have an annotation?"** MutatingWebhook (or VAP) that adds annotation on CREATE. Discuss `failurePolicy`, exclusion of kube-system.
- **"What if my admission webhook is down?"** With `failurePolicy: Fail`, every Pod create fails → cluster broken. With `Ignore`, requests skip the webhook silently. Use `Fail` for critical (security policies); `Ignore` for nice-to-have (logging, defaulting).
- **"What's the difference between VAP and a ValidatingWebhook?"** VAP runs CEL in-process (no network hop, no TLS, no separate service); webhook is external (more powerful but with operational cost). Use VAP for simple structural validation; webhook for anything requiring external state.
- **"How does the apiserver scale horizontally?"** Stateless; multiple replicas behind LB. Watch cache is per-process so each apiserver has its own; etcd is shared. Watching clients may bounce between apiservers; bookmarks help resume.

## 3.12 Corner cases and alternatives

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Validate resource | Built-in OpenAPI schema | ValidatingWebhook | VAP (CEL) |
| Default fields | Built-in `defaulting` | MutatingWebhook | SSA + controller |
| Rate limit clients | APF | OPA Gatekeeper rate limit | gateway-level (Envoy) |
| Audit | apiserver `--audit-policy-file` | audit-webhook | audit2rbac for analysis |
| Encrypted Secrets | aescbc | aesgcm | KMS (cloud-integrated) |
| Multi-version CRD | None strategy | Webhook conversion | Single version + rename |

## Must-internalize

- The filter chain: panic-recovery → audit → authn → authz → admission mutating → validation → admission validating → storage.
- Watch cache is per-apiserver-replica; bookmarks prevent stale clients from getting 410.
- VAP (1.30 GA) replaces simple validating webhooks; faster, no network hop.
- APF replaced max-in-flight; tune FlowSchemas + PriorityLevelConfigurations at scale.
- Server-Side Apply tracks fieldManager → multi-owner reconciliation.
- Encryption at rest = `--encryption-provider-config` with KMS in production.
- Aggregation layer = external apiservers for new API groups (metrics-server is the classic).
