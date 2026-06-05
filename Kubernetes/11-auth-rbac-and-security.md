# 11 · Authentication, RBAC, and Security

The security surface is large. Staff candidates should know: how clients identify themselves (authn), how the apiserver decides what's allowed (authz), how admission controls policy, how pods are restricted (PSA, capabilities), and how secrets flow.

## 11.1 The big picture

```
   client (kubectl, kubelet, controller)
       │
       │ HTTPS with client cert OR Bearer token
       ▼
   ┌──────────────────────────┐
   │ Authentication           │ → UserInfo {name, groups, uid, extra}
   │ - x509 cert              │
   │ - SA token (JWT)         │
   │ - OIDC                   │
   │ - Webhook                │
   └────────────┬─────────────┘
                ▼
   ┌──────────────────────────┐
   │ Authorization (RBAC)     │ → allow / deny
   │ - RBAC                   │
   │ - Node                   │
   │ - Webhook                │
   └────────────┬─────────────┘
                ▼
   ┌──────────────────────────┐
   │ Admission                │ → mutate / validate
   │ - built-in               │
   │ - webhooks               │
   │ - VAP (CEL)              │
   └──────────────────────────┘
```

## 11.2 Authentication — the modes

### x509 client certificates
```
kubectl uses ~/.kube/config:
- cluster CA + endpoint
- user: client-cert + client-key (Subject: CN=alice, O=devs)
```

The apiserver verifies the cert was signed by the configured CA. `CN` becomes the username; `O`'s become groups.

Caveats:
- No revocation. Once issued, valid until expiry. Use short-lived certs (RBAC-only auth, sa-token) instead.
- Pre-shared CA — if compromised, all certs invalid.

### ServiceAccount tokens (the workload identity)

Each pod gets a projected ServiceAccount token:

```yaml
spec:
  serviceAccountName: my-sa
  containers:
  - volumeMounts:
    - name: kube-api-access
      mountPath: /var/run/secrets/kubernetes.io/serviceaccount
  volumes:
  - name: kube-api-access
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600
          audience: api
      - configMap: {name: kube-root-ca.crt, items: [{key: ca.crt, path: ca.crt}]}
      - downwardAPI: {items: [{path: namespace, fieldRef: {fieldPath: metadata.namespace}}]}
```

The token is a JWT signed by the apiserver's SA signing key. It contains:
- subject: `system:serviceaccount:<ns>:<sa>`
- audience (validated by receiver)
- exp (short — 1h, kubelet refreshes)

### OIDC

```
--oidc-issuer-url=https://accounts.google.com
--oidc-client-id=k8s-cluster
--oidc-username-claim=email
--oidc-groups-claim=groups
```

apiserver verifies JWT signature (using JWKs from the issuer), extracts username + groups from claims.

Used for human users. Bind RBAC to email or group.

### Webhook token authenticator
External service validates tokens. apiserver calls it; result cached briefly.

### Authentication v1 structured config (1.30+ GA)

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AuthenticationConfiguration
jwt:
- issuer:
    url: https://accounts.google.com
    audiences: ["k8s"]
  claimMappings:
    username: {claim: email}
    groups: {claim: groups}
  claimValidationRules:
  - claim: hd
    requiredValue: "mycompany.com"
```

Replaces the old `--oidc-*` flags with structured config; supports multiple issuers.

## 11.3 RBAC — the canonical authorization

```
Role         (namespace-scoped)
ClusterRole  (cluster-wide)
RoleBinding         (binds subjects to a Role inside a namespace)
ClusterRoleBinding  (binds subjects to a ClusterRole cluster-wide)
```

### Roles

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {namespace: foo, name: pod-reader}
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  resourceNames: ["my-app"]   # optional: restrict to named resources
  verbs: ["get", "update", "patch"]
```

### Bindings

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {namespace: foo, name: alice-can-read}
subjects:
- kind: User
  name: alice@company.com
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: ci-bot
  namespace: foo
- kind: Group
  name: devs
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Verbs

| Verb | Resource state |
|------|----------------|
| `get` | Read one |
| `list` | Read many |
| `watch` | Stream changes |
| `create` | New |
| `update` | Replace |
| `patch` | Partial update |
| `delete` | Remove one |
| `deletecollection` | Remove many |

Plus resource-specific verbs: `pods/exec`, `pods/log`, `pods/portforward`, etc. The slash indicates a subresource.

### Aggregated cluster roles

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-admin
  labels:
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
- apiGroups: ["monitoring.coreos.com"]
  resources: ["prometheuses", "alertmanagers"]
  verbs: ["*"]
```

The `aggregate-to-admin` label causes the `admin` ClusterRole to include these rules. Extends cluster roles without modifying built-ins.

### Built-in roles to remember

- `cluster-admin` — god mode.
- `admin` — almost everything in a namespace.
- `edit` — read + write, no RBAC.
- `view` — read-only.
- `system:node` — kubelet identity (Node authorizer adds restrictions).
- `system:masters` — cert-based superuser (CN in this group bypasses RBAC).

## 11.4 Node authorizer

Special mode that restricts kubelet identity (`system:node:<nodename>` in group `system:nodes`) to only:
- Read pods scheduled on its node.
- Read secrets/configmaps that those pods use.
- Update its own Node and Pod status.
- Get its own Lease.

Critical for cluster security: a compromised node can't read all secrets.

`NodeRestriction` admission plugin enforces it (kubelet can only modify its own resources).

## 11.5 ServiceAccount best practices

```yaml
# Don't reuse default SA for app pods
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-ns
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123:role/my-role  # IRSA
---
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: my-app-sa
  automountServiceAccountToken: false   # if pod doesn't need API access
```

- **One SA per workload** — least privilege.
- **automountServiceAccountToken: false** for pods that don't talk to the apiserver.
- **Bound tokens** (projected, expiration_seconds) — automatic 1h tokens replacing legacy long-lived tokens.

### Cloud workload identity

- AWS: **IRSA** (IAM Roles for Service Accounts) — annotation on SA links to IAM role; pod's projected token is exchanged via STS for AWS creds.
- GCP: **Workload Identity** — SA annotation links to GCP SA; token exchange.
- Azure: **Workload Identity** (AAD) — similar pattern.

All use the projected SA token + audience verification.

## 11.6 Admission control — the third gate

After authn + authz, admission runs:

### Built-in mutating admission
- `ServiceAccount` — adds SA token volume; defaults SA.
- `Priority` — sets priorityClass.
- `RuntimeClass` — applies pod overhead from RuntimeClass.

### Built-in validating admission
- `PodSecurity` — Pod Security Standards (next section).
- `ResourceQuota` — enforces namespace quotas.
- `LimitRanger` — applies LimitRange defaults/maxes.
- `NamespaceLifecycle` — blocks writes to terminating namespaces.
- `NodeRestriction` — kubelet identity restrictions.

### External webhooks
ValidatingWebhookConfiguration / MutatingWebhookConfiguration — covered in 03.

### ValidatingAdmissionPolicy (CEL, 1.30 GA)
Inline CEL expressions — no webhook hop.

## 11.7 Pod Security Admission (PSA)

Replaces deprecated PodSecurityPolicy (removed 1.25). Built-in admission plugin enforcing **Pod Security Standards**:

| Profile | Restrictions |
|---------|--------------|
| **Privileged** | Anything goes |
| **Baseline** | No privilege escalation; no host namespaces; allowed capabilities limited |
| **Restricted** | Above + runAsNonRoot; seccomp RuntimeDefault; capabilities = drop-all + NET_BIND_SERVICE; readOnlyRootFilesystem optional |

Applied per-namespace via labels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/warn: restricted     # warn for create
    pod-security.kubernetes.io/audit: restricted    # log to audit
```

Three modes per profile:
- `enforce` — block non-compliant pods.
- `warn` — emit warning to user, allow.
- `audit` — log to audit, allow.

## 11.8 Container security context

```yaml
spec:
  securityContext:                # pod-level
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile: {type: RuntimeDefault}
  containers:
  - securityContext:              # container-level
      allowPrivilegeEscalation: false
      capabilities:
        drop: [ALL]
        add: [NET_BIND_SERVICE]
      readOnlyRootFilesystem: true
      seccompProfile: {type: RuntimeDefault}
```

### Capabilities

Linux capabilities are sub-root privileges. Common ones:
- `NET_BIND_SERVICE` — bind ports < 1024.
- `NET_ADMIN` — manage networking.
- `SYS_ADMIN` — many privileged ops.
- `SYS_TIME` — set clock.
- `CHOWN`, `DAC_OVERRIDE` — bypass file ownership/perms.

Best practice: `drop: [ALL]` then `add: [NET_BIND_SERVICE]` only if needed.

### Seccomp
Filter the set of syscalls a container can make. `RuntimeDefault` uses the runtime's default profile (containerd ships a sensible one).

### SELinux / AppArmor
Mandatory access control. Apply via annotations (AppArmor) or securityContext.seLinuxOptions.

## 11.9 OPA / Gatekeeper / Kyverno

External policy engines (validating webhooks) for cluster-wide rules:

### Gatekeeper (OPA)

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata: {name: k8srequiredlabels}
spec:
  crd:
    spec:
      names: {kind: K8sRequiredLabels}
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels: {type: array, items: {type: string}}
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels
      violation[{"msg": msg}] {
        provided := {label | input.review.object.metadata.labels[label]}
        required := {label | label := input.parameters.labels[_]}
        missing := required - provided
        count(missing) > 0
        msg := sprintf("missing labels: %v", [missing])
      }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata: {name: require-owner}
spec:
  match:
    kinds: [{apiGroups: [""], kinds: ["Namespace"]}]
  parameters:
    labels: ["owner"]
```

### Kyverno

YAML-based (no Rego required), more k8s-native. Generally easier to adopt.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: {name: require-labels}
spec:
  validationFailureAction: enforce
  rules:
  - name: check-for-labels
    match:
      any:
      - resources: {kinds: [Namespace]}
    validate:
      message: "label 'owner' is required"
      pattern:
        metadata:
          labels:
            owner: "?*"
```

### Choose between

- **VAP (CEL)** for simple structural checks. Fastest, no webhook hop.
- **Kyverno** for policy-as-data; YAML-native; easier for non-experts.
- **Gatekeeper** for advanced Rego use cases; broader OPA ecosystem.

## 11.10 Secrets — at rest, in transit, in use

### At rest

By default, etcd holds Secrets as base64 (NOT encrypted). Enable encryption:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources: ["secrets"]
  providers:
  - kms:
      apiVersion: v2
      name: aws-kms
      endpoint: unix:///var/run/kmsplugin/socket.sock
  - aescbc:
      keys: [{name: key1, secret: <base64>}]
  - identity: {}
```

Order matters: writes use first; reads try in order. After enabling, **re-encrypt existing secrets**:
```bash
kubectl get secrets -A -o json | kubectl replace -f -
```

### In transit
Always TLS between apiserver ↔ etcd and apiserver ↔ kubelet. Verify with cert chains.

### In use (pod)
Mount secrets as volumes, not env vars (env vars leak to logs, child processes). Use projected SA tokens (short-lived).

### Better: external secret managers
- **External Secrets Operator** — syncs from AWS Secrets Manager, GCP Secret Manager, Vault.
- **Secrets Store CSI Driver** — mounts secrets directly from external store.
- **Vault Agent Sidecar** — injects.

Don't store production secrets as plain Secrets if you can help it.

## 11.11 Image security

### Image signing
**Sigstore / cosign** — sign images, verify at admission.

```yaml
apiVersion: cosigned.sigstore.dev/v1alpha1
kind: ClusterImagePolicy
spec:
  images:
  - glob: "ghcr.io/myorg/*"
  authorities:
  - keyless:
      identities:
      - issuer: https://accounts.google.com
        subject: alice@myorg.com
```

Admission validates the image has a valid signature from an approved signer.

### Image scanning
Trivy, Clair, Snyk scan images at build + at runtime. Combine with admission webhook (Trivy + Kyverno) to block CVE-laden images.

### Distroless / minimal base images
`gcr.io/distroless/static` — no shell, no package manager. Smaller attack surface.

## 11.12 Audit logging

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  verbs: ["get", "list", "watch"]
  resources: [{group: "", resources: ["secrets"]}]
- level: RequestResponse
  resources: [{group: "", resources: ["configmaps", "secrets"]}]
```

`--audit-policy-file=/etc/k8s/audit-policy.yaml --audit-log-path=/var/log/k8s-audit.log`.

Audit levels:
- **None** — don't log.
- **Metadata** — request metadata only.
- **Request** — also include the request body.
- **RequestResponse** — also include the response body.

Forward to SIEM (Splunk, Datadog) via fluentd or audit webhook.

## 11.13 Common interview probes

- **"How does a pod authenticate to the apiserver?"** Projected SA token (JWT signed by apiserver's signing key), 1h TTL, audience-bound.
- **"How does RBAC handle a permission check?"** Apiserver collects all (Cluster)RoleBindings matching the user+groups; aggregates rules; if any rule matches the verb+resource, allow. First-allow wins.
- **"How would you prevent users from creating privileged pods?"** PSA with `enforce: restricted`. Or admission webhook (Gatekeeper) for finer rules.
- **"What's IRSA?"** AWS IAM Roles for Service Accounts. SA annotation → IAM role; projected SA token → STS AssumeRoleWithWebIdentity → temp AWS creds.
- **"Where should secrets live?"** External secret manager (AWS SM, Vault) synced via External Secrets Operator or CSI; etcd encryption at rest as defense-in-depth.
- **"How does image signing work in k8s?"** cosign signs at build; admission policy (cosigned, Kyverno) verifies signature before allowing pod create.

## 11.14 Corner cases

- **Aggregated CR not appearing** — label mismatch or controller hasn't aggregated yet.
- **`system:masters` group bypasses RBAC** — be careful which certs are in this group.
- **Stale SA tokens** after revoke — projected tokens auto-refresh; legacy (Secret-stored) don't.
- **PSA doesn't apply to existing pods** — only at create/update. Existing pods continue running.
- **Bind to ClusterRole as RoleBinding** — works; restricts to the namespace of the binding.
- **`kubectl auth can-i`** — quick check; uses SelfSubjectAccessReview.
- **Webhook MITM** — if webhook serves on `Service`, apiserver verifies via CA bundle in WebhookConfig. Watch for cert rotation.

## 11.15 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Workload identity | Projected SA token | IRSA / Workload Identity | SPIFFE/SPIRE |
| Block bad images | Image scanner admission | Cosign signature verify | Private registry + signed only |
| Enforce labels | Kyverno | OPA Gatekeeper | VAP (CEL) |
| Restrict pods | PSA `restricted` | PSP (deprecated) | OPA / Kyverno |
| Secrets at rest | kms encryption provider | External Secrets Operator | CSI Secret Store |
| Audit forwarding | audit webhook + Splunk | fluentd sidecar | external audit pipeline |
| Cert rotation | cert-manager | manual scripts | service mesh auto (Istio) |

## Must-internalize

- Authn: x509 / SA token / OIDC / webhook → UserInfo.
- RBAC: Role/ClusterRole + Binding; first-allow wins; aggregated roles for extensibility.
- Node authorizer + NodeRestriction limits kubelet identity.
- PSA replaces PSP: Privileged / Baseline / Restricted at namespace label.
- securityContext: runAsNonRoot, capabilities drop-all, seccomp RuntimeDefault, readOnlyRootFilesystem.
- Workload identity (IRSA, GCP WI) bridges k8s SA → cloud IAM.
- Secrets: encryption at rest (KMS) + external store for production.
- Image signing (cosign) at admission for supply chain.
- Audit policy + forwarding to SIEM is staff-table-stakes.
