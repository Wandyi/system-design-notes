# 15 · Observability — Metrics, Logs, Events, Audit, Traces

The observability surface in a K8s cluster. Multiple signals, multiple destinations, each with its own scale story.

## 15.1 The four signals

```
   Metrics:  numeric time-series       Prometheus, Datadog, Cortex
   Logs:     unstructured/structured   Loki, ELK, CloudWatch, Splunk
   Traces:   per-request spans         Jaeger, Tempo, Honeycomb
   Events:   cluster-internal notifications  apiserver Event resource → audit log
```

Plus **profiles** (continuous profiling, Pyroscope, Parca) as a fourth-and-a-half.

## 15.2 Metrics — the stack

### Layered architecture

```
   App (instrumented with Prometheus client) ── /metrics
                                                    │ HTTP scrape
                                                    ▼
   Prometheus / Cortex / Mimir                   collects + stores
                                                    │
                                                    ▼
   Grafana                                       visualizes
                                                    │
                                                    ▼
   Alertmanager                                  fires alerts
```

### k8s-specific exporters

| Exporter | Scope | Metrics |
|----------|-------|---------|
| **kubelet** (built-in `/metrics`) | Node | Resource stats, container stats (cadvisor), volume stats |
| **kube-state-metrics** | Cluster | Object counts/conditions/status: pods.phase, deployments.replicas.ready, etc. |
| **node-exporter** | Node | OS-level: CPU, memory, disk, network, IO |
| **metrics-server** | Cluster | For HPA (NOT for long-term storage) |
| **kube-proxy /metrics** | Node | Sync rate, sync duration |
| **etcd /metrics** | Control plane | wal_fsync, db size, leader changes |
| **apiserver /metrics** | Control plane | request rate, latency by verb/resource |

### Prometheus discovery

Prometheus discovers scrape targets via kube_sd_configs:
- `kubernetes_sd_configs.role: pod` — discovers pods (with annotations or selectors).
- `kubernetes_sd_configs.role: service` — discovers Services.
- `kubernetes_sd_configs.role: endpoints` — Endpoints.
- `kubernetes_sd_configs.role: node` — Nodes (for node-exporter, kubelet).

Or annotation-based:
```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
```

### Prometheus Operator + ServiceMonitor

Higher-level abstraction:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata: {name: my-app, labels: {team: platform}}
spec:
  selector: {matchLabels: {app: my-app}}
  endpoints:
  - port: metrics
    interval: 30s
```

The operator generates Prometheus config from ServiceMonitor + PodMonitor + Probe CRDs.

## 15.3 Cardinality — the silent killer

Prometheus time-series:
```
   metric{label1="value1", label2="value2", ...} 
```

Each unique combination of labels is a distinct **series**. Cardinality = total series count.

- Container metric `kube_pod_status_ready{pod, namespace, condition, status}` with 1000 pods, 50 namespaces, 5 conditions, 3 statuses = 750K series. Per scrape.
- 10K pods churning daily = many series rotating in/out, each retained for retention period.

**Cardinality bombs**:
- Pod UIDs in labels.
- Request IDs.
- User IDs.

**Mitigations**:
- Drop high-cardinality labels via `metric_relabel_configs`.
- Use exemplars for tracing (instead of putting trace ID as label).
- Pre-aggregate via recording rules.
- Use Thanos / Mimir for horizontal storage.

## 15.4 Long-term storage — Thanos, Mimir, Cortex

Prometheus single-node is limited (~1M series). Long-term scale options:

| Tool | Architecture |
|------|--------------|
| **Thanos** | Sidecar uploads chunks to S3; Querier fans out across multiple Prometheus + S3 |
| **Cortex** | Multi-tenant, horizontal Prometheus alternative |
| **Mimir** | Grafana's fork of Cortex; performance improvements |
| **VictoriaMetrics** | Single-binary Prometheus replacement, very memory-efficient |

For staff context: hyperscale shops run dozens of Prometheus replicas + Thanos/Mimir for global query.

## 15.5 Logs — pipeline patterns

### Sidecar pattern

```yaml
spec:
  containers:
  - name: app
    image: my-app
    volumeMounts: [{name: logs, mountPath: /var/log/app}]
  - name: log-shipper
    image: fluent-bit
    volumeMounts: [{name: logs, mountPath: /var/log/app, readOnly: true}]
  volumes: [{name: logs, emptyDir: {}}]
```

Pros: explicit; per-app config. Cons: every pod adds a sidecar.

### DaemonSet pattern

A log-shipper DaemonSet reads `/var/log/pods/*` (kubelet writes here):

```yaml
apiVersion: apps/v1
kind: DaemonSet
spec:
  template:
    spec:
      volumes:
      - name: varlog
        hostPath: {path: /var/log/pods}
```

Pros: one shipper per node. Cons: less per-app control.

### Modern: structured logs from stdout

Apps write JSON to stdout; container runtime captures via CRI; kubelet writes to `/var/log/pods/<ns>_<pod>_<uid>/<container>/0.log`. Log shipper tails these.

```
/var/log/pods/default_my-app-abc_uid/my-app/0.log
```

Format: each line is JSON containing the original log + timestamp + stream (stdout/stderr).

### Log rotation

kubelet rotates per `--container-log-max-size` (default 10Mi) + `--container-log-max-files` (5). When exceeded, old files are deleted; **logs are lost** if shipper hasn't picked them up.

### Loki vs ELK

| Property | Loki | ELK (Elasticsearch) |
|----------|------|---------------------|
| Indexing | Labels only (Prom-style) | Full-text |
| Storage | Object store (S3) | Hot SSD |
| Cost | Low | High |
| Query | LogQL (limited full-text) | Lucene (rich) |
| Setup | Simple | Complex |

Loki at hyperscale: cheaper for keep-everything; limited rich queries.

## 15.6 Events — the audit-of-control-plane

```yaml
apiVersion: v1
kind: Event
metadata: {name: pod-1.abc.def, namespace: default}
involvedObject:
  kind: Pod
  name: my-pod
  uid: <uid>
reason: SuccessfulCreate
message: "Created pod: my-pod-xyz"
source: {component: deployment-controller}
firstTimestamp: ...
lastTimestamp: ...
count: 1
type: Normal   # Normal | Warning
```

Events are first-class resources. `kubectl get events` / `kubectl describe pod` show them.

Default retention: 1 hour (controlled by apiserver `--event-ttl`).

### Event spam

Bad code = thousands of events. apiserver `--event-rate-limit-config`:

```yaml
apiVersion: eventratelimit.admission.k8s.io/v1alpha1
kind: Configuration
limits:
- type: Namespace
  qps: 50
  burst: 100
- type: Source+Object
  qps: 1
  burst: 5
```

Production tip: route events to a separate etcd cluster via apiserver flag `--etcd-servers-overrides=/events#http://events-etcd:2379`. Keeps the main etcd lean.

## 15.7 Audit logs — the security signal

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages: ["RequestReceived"]
rules:
- level: Metadata
  verbs: ["get", "list", "watch"]
  resources:
  - group: "" 
    resources: ["secrets", "configmaps"]
- level: RequestResponse
  resources:
  - group: "" 
    resources: ["pods", "services"]
- level: None  # don't log
  users: ["system:apiserver"]
```

Audit levels:
- **None** — don't log.
- **Metadata** — request metadata only.
- **Request** — request metadata + body.
- **RequestResponse** — metadata + request body + response body.

apiserver `--audit-policy-file --audit-log-path /var/log/k8s-audit.log`.

### Forwarding

In production: don't store on disk. Use `audit-webhook-config-file` to send to fluentd → SIEM (Splunk, Datadog, ELK).

```yaml
apiserver flags:
  --audit-webhook-config-file=/etc/audit-webhook.yaml
  --audit-webhook-batch-max-size=400
  --audit-webhook-batch-throttle-qps=10
```

## 15.8 Distributed tracing

K8s itself doesn't trace; apps do (via OpenTelemetry SDK). The cluster role:
- **Propagate context** through service mesh (Istio, Linkerd inject headers).
- **Aggregate** via OpenTelemetry Collector DaemonSet.
- **Export** to Jaeger / Tempo / Honeycomb.

```yaml
# OpenTelemetry Collector
receivers:
  otlp:
    protocols:
      grpc: {endpoint: 0.0.0.0:4317}
exporters:
  otlphttp:
    endpoint: tempo:4318
service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [otlphttp]
```

### Tail-based sampling

Random sampling = miss anomalies. Tail-based sampling (in the Collector) keeps traces matching:
- p99 latency.
- Error responses.
- Specific routes.

Expensive in compute but high signal value.

## 15.9 Pixie / Hubble — eBPF-native observability

Modern alternative: eBPF auto-instrumentation, no SDK changes.

- **Pixie** (Apache, now part of New Relic): eBPF on every node; captures HTTP/gRPC/MySQL/PG/Redis automatically; stores last N hours in node-local memory.
- **Hubble** (Cilium): flow-level observability; every TCP connect/HTTP request visible.
- **Parca / Pyroscope**: continuous CPU profiling via eBPF.

These don't replace OpenTelemetry traces but complement: get instant flow visibility without code changes.

## 15.10 Prometheus-style metrics — the staff cheat sheet

```promql
# Pod CPU usage rate
rate(container_cpu_usage_seconds_total{pod="my-pod"}[5m])

# Memory usage
container_memory_working_set_bytes{pod="my-pod"}

# Pod restart count (24h)
increase(kube_pod_container_status_restarts_total[24h])

# Deployment availability
kube_deployment_status_replicas_available / kube_deployment_spec_replicas

# Apiserver request latency p99
histogram_quantile(0.99, rate(apiserver_request_duration_seconds_bucket[5m]))

# Etcd write latency
histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m]))

# kube-state-metrics: pods Pending too long
sum(kube_pod_status_phase{phase="Pending"}) by (namespace)
```

## 15.11 Alerts — SLO-driven

Modern best practice: alert on **SLOs** (Service Level Objectives), not low-level signals.

```yaml
groups:
- name: api-server-slo
  rules:
  - alert: ApiServerErrorRateHigh
    expr: |
      sum(rate(apiserver_request_total{code=~"5.."}[5m]))
      / sum(rate(apiserver_request_total[5m]))
      > 0.05
    for: 10m
    annotations:
      summary: "apiserver 5xx rate above 5% for 10m"

  - alert: ApiServerLatencyHigh
    expr: histogram_quantile(0.99, rate(apiserver_request_duration_seconds_bucket[5m])) > 5
    for: 10m
    annotations:
      summary: "apiserver p99 latency > 5s"
```

SLO patterns:
- **Burn rate**: alert when error budget is being consumed too fast.
- **Multi-window**: short window (1h) + long window (24h) both must trigger.

## 15.12 Common interview probes

- **"How does the apiserver expose metrics?"** Prometheus-format at `/metrics`. Scraped by Prometheus or compatible.
- **"What's the metrics-server?"** Aggregated API for HPA. Scrapes kubelet's `/metrics/resource`. NOT for long-term storage.
- **"What's kube-state-metrics?"** Exposes k8s object state as metrics (kube_pod_status_*, kube_deployment_*).
- **"How would you alert on apiserver overload?"** P99 latency, request rate, APF rejected requests, etcd commit latency.
- **"How would you debug a slow pod?"** Logs (`kubectl logs`), describe (`events`), metrics (cAdvisor), traces (if instrumented), eBPF tools (bpftrace, Pixie).
- **"Why is your audit log so noisy?"** Default policy logs everything; tune policy to skip `get/list/watch` on non-sensitive resources.
- **"Where do events live?"** apiserver writes to etcd as Event resources; TTL 1h by default; can route to separate etcd.

## 15.13 Corner cases

- **metrics-server isn't scrape target** — it's an aggregated apiserver; query via `kubectl top` or HPA, not via Prometheus.
- **kube-state-metrics has high cardinality** — every Pod has labels per condition; 1000 pods → 5000 series per metric.
- **Events lost during apiserver restart** — events written in flight may not persist if write was in progress.
- **Logs lost on rotation** — log shipper must keep up; alert on log_shipper_lag.
- **OOM in log shipper** — fluentd/fluent-bit memory caps; configure backpressure.
- **Cardinality explosion** — kill Prometheus. Always cap label cardinality.
- **Distributed trace + log linking** — need consistent traceid in logs (slog with trace context).

## 15.14 Alternative approaches

| Need | Path 1 | Path 2 | Path 3 |
|------|--------|--------|--------|
| Metrics | Prometheus + ServiceMonitor | VictoriaMetrics | Datadog Agent |
| Long-term storage | Thanos + S3 | Mimir | Cortex |
| Logs | Loki + fluent-bit | ELK + filebeat | Datadog Logs / Splunk |
| Traces | Jaeger | Tempo + OTEL | Honeycomb |
| eBPF observability | Pixie | Hubble (Cilium) | Parca / Pyroscope |
| Audit | apiserver audit policy + webhook | OPA admission + log | Falco (runtime) |
| Events | apiserver Events | external event store | hubble.observe |

## 15.15 Falco — runtime security

Falco watches syscalls via eBPF; alerts on suspicious patterns (shell in container, sensitive mount, etc.):

```yaml
- rule: Suspicious shell in container
  desc: A shell was spawned in a container.
  condition: container and proc.name in (bash, sh, zsh)
  output: "Shell in container (user=%user.name container_id=%container.id)"
  priority: WARNING
```

Complements admission control; catches what gets past it.

## 15.16 The full observability stack — a sketch

```
   Cluster:
     Prometheus Operator       → 6 Prometheus replicas + Thanos sidecar
     Grafana                    → dashboards
     Alertmanager + on-call routing (PagerDuty)
     Loki                       → fluent-bit DaemonSet ships logs
     Tempo                      → OTEL Collector DaemonSet aggregates traces
     OpenCost                   → cost visibility (kube cost optimization)
     Pixie / Hubble             → eBPF auto-instrumentation
     Falco                      → runtime security alerts
     audit log → fluentd → Splunk
```

This stack is what hyperscale shops run. Each layer is independent.

## Must-internalize

- Four signals: metrics, logs, traces, events. Plus profiles.
- metrics-server (HPA), kube-state-metrics (object state), kubelet /metrics (resource), node-exporter (OS).
- Prometheus + ServiceMonitor (operator) — the standard.
- Cardinality is the silent killer — labels with unbounded values.
- Long-term storage: Thanos/Mimir/Cortex/VictoriaMetrics + S3.
- Loki = logs at scale; full-text limited.
- Audit policy levels: None / Metadata / Request / RequestResponse.
- Events default 1h TTL; route to separate etcd to keep main etcd clean.
- OpenTelemetry Collector DaemonSet for traces.
- Pixie / Hubble for eBPF auto-instrumentation.
- Falco for runtime security alerts via syscall monitoring.
