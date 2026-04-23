# AWS Cost Optimization Techniques for Production

Comprehensive, staff-level playbook for cutting AWS spend without sacrificing reliability. Organized by service category with traps and leverage-ranked takeaways at the end.

---

## 1. Compute (EC2 / ECS / EKS / Lambda)

### Pricing model arbitrage
- **Savings Plans** (Compute SP > EC2 Instance SP for flexibility): 1-yr no-upfront typically hits ~27% discount; 3-yr all-upfront ~55%. Compute SP covers Fargate and Lambda too — often the right default.
- **Reserved Instances** only when you need capacity reservation guarantees (zonal RI); otherwise Savings Plans are strictly better (same discount, more flexible).
- **Spot instances** for anything fault-tolerant: batch, CI runners, stateless web tiers with ASG mixed-instance policy, EKS node groups via Karpenter. 70–90% discount. Use multiple instance families + AZs in the allocation strategy to minimize interruption.
- **Graviton (ARM)** instances: 20–40% cheaper than x86 equivalents with equal-or-better perf on most workloads. Requires multi-arch container builds.
- **Fargate Spot** for bursty ECS workloads.

### Rightsizing
- **Compute Optimizer** recommendations — filter to "Underprovisioned" and "Over-provisioned" weekly.
- **Downsize by generation**: an `m7i.large` is cheaper and faster than an `m5.large`. Always prefer the latest generation when rightsizing.
- **Kill idle** EC2 (< 5% CPU for 14 days), unattached EBS, idle RDS, idle load balancers. Use AWS Trusted Advisor + Cost Explorer "Idle resources" report.

### Autoscaling
- **Scale to zero** for dev/staging off-hours (cron via EventBridge → Lambda). A full stg environment for 168 hrs vs 40 working hrs = 76% savings.
- **Predictive scaling** on ASG when traffic has daily/weekly seasonality — avoids the reactive-scale lag cost (keeping headroom warm).
- **Karpenter** (EKS) instead of Cluster Autoscaler — chooses cheapest instance that fits pending pods, consolidates aggressively, supports Spot natively.

### Lambda-specific
- **Power Tuning** (AWS Lambda Power Tuning state machine): memory ↔ cost is not linear; often 1024 MB runs 2× faster than 512 MB for same cost.
- **ARM (Graviton2)** Lambda: 20% cheaper flat.
- **Provisioned concurrency** only for latency-critical paths; for the rest, cold starts are a cheaper tradeoff.
- **SnapStart** (Java) to avoid provisioned concurrency costs.

---

## 2. Storage (S3 / EBS / EFS / FSx)

### S3
- **Intelligent-Tiering** by default for any bucket with unknown access patterns. Auto-moves objects across tiers; monitoring fee is trivial (~$0.0025/1000 objects/month). For 90% of buckets this is set-and-forget.
- **Lifecycle policies**: Standard → IA (30d) → Glacier Instant (90d) → Glacier Deep Archive (180d). Match to access SLA.
- **Delete incomplete multipart uploads** — lifecycle rule `AbortIncompleteMultipartUpload` after 7 days. Easy forgotten cost.
- **S3 Storage Lens** for bucket-level waste discovery (duplicate data, non-current versions bloat).
- **Object versioning**: noncurrent version lifecycle expiry. Without it, versioning silently 2–5×'s your storage over years.
- **Requester Pays** for egress-heavy public datasets.
- **S3 Express One Zone** for ultra-low-latency small objects — 50% cheaper than Standard for that access pattern, but single-AZ.

### EBS
- **gp3 instead of gp2**: 20% cheaper, and you provision IOPS/throughput independently. gp2's IOPS scales with volume size, forcing you to over-provision capacity to get IOPS.
- **Delete unattached volumes** (weekly sweep — they keep accruing charges).
- **Snapshot lifecycle (DLM)**: expire old snapshots; delete AMIs tied to deregistered images (snapshots are orphaned otherwise).
- **Snapshot archive tier** (75% cheaper) for compliance-only snapshots rarely restored.

### Data transfer (often the silent top-3 line item)
- **Cross-AZ traffic is $0.01/GB each way** — 2x for request+response. At scale this dwarfs compute.
  - Use **topology-aware routing** (EKS topology-aware hints, ALB zonal routing) to keep traffic intra-AZ.
  - Place chatty services in the same AZ; accept the availability tradeoff consciously.
- **VPC Endpoints** (Gateway for S3/DynamoDB is free; Interface endpoints cost ~$7/mo per AZ but kill NAT egress). A single NAT-routed S3 call path at scale is often $10K+/month vs $0 via gateway endpoint.
- **NAT Gateway** costs $0.045/hr + $0.045/GB processed. At high throughput, NAT is often the #1 line item. Alternatives: VPC endpoints for AWS services, VPC peering for private AWS traffic, fck-nat or Egress-only IGW where applicable.
- **CloudFront** for egress-heavy workloads — $0.085/GB via CF vs $0.09/GB direct is misleading; real win is CloudFront egress is cheaper at commit tiers, has free tier, and hits cache.

---

## 3. Databases

### RDS / Aurora
- **Reserved Instances** for baseline capacity; 3-yr all-upfront ~60% off.
- **Aurora Serverless v2** for variable load (scales ACU in 0.5 increments); Serverless **v1** is deprecated — migrate.
- **Aurora I/O-Optimized** cluster: 30% more for compute, but I/O is free. Breakeven at ~25% of total cost being I/O — typical for write-heavy workloads.
- **Downsize read replicas** — often over-provisioned. Use Aurora Auto Scaling for replicas.
- **gp3 storage** on RDS (same logic as EBS).
- **Stop non-prod RDS** on nights/weekends (RDS allows stop for up to 7 days; automate restart via Lambda).

### DynamoDB
- **On-Demand vs Provisioned**: switch to Provisioned with auto-scaling once traffic is predictable — 40–60% cheaper at sustained load.
- **Reserved Capacity** for Provisioned baseline.
- **Infrequent Access (IA) table class**: 60% cheaper storage, 25% more expensive reads — net win if read:storage ratio is low.
- **Item size**: DynamoDB charges in 1-KB RCU/4-KB WCU units. Compressing or splitting large items saves hard money.
- **TTL** on ephemeral data (sessions, audit logs) — deletion is free, avoids storage bloat.
- **Global Tables** cost 2× write (replicated) — use only where cross-region writes are required, not for DR.

### ElastiCache / MemoryDB
- **Reserved Nodes** for steady-state.
- **Graviton** node types.
- **Right-size by evictions, not CPU**: if your cluster has no evictions and low memory, you're over-provisioned.

---

## 4. Networking

- **Eliminate NAT Gateway where possible**: gateway/interface VPC endpoints for S3, DynamoDB, ECR, Secrets Manager, SSM, STS, CloudWatch Logs. Each one removes NAT processing charges.
- **Transit Gateway** vs VPC peering: TGW is $0.05/attachment/hr + $0.02/GB. For N VPCs, peering is cheaper until mesh complexity dominates (~6 VPCs).
- **PrivateLink** to consume SaaS over AWS private backbone — avoids internet egress, often cheaper than public egress at scale.
- **CloudFront + Origin Shield** for caching — cuts origin egress and compute.
- **Accept 'EC2 - Other'** in Cost Explorer is almost always NAT + inter-AZ + EBS snapshots. Always investigate.

---

## 5. Observability (often 5–15% of total bill)

### CloudWatch Logs
- **Log retention**: default is "Never Expire". Set retention per log group (7d for debug, 30d for app, 90d for audit). Apply via infra-as-code default.
- **Log Infrequent Access class**: 50% cheaper ingestion, same query — good for audit/compliance logs.
- **Drop noisy logs at the source** — DEBUG logs in prod are pure waste. Ingestion is ~$0.50/GB (us-east-1); a single chatty service can run $10K/month.
- **Metric filters vs custom metrics**: custom metrics are $0.30 each; filter logs into one metric rather than emitting hundreds.
- **Contributor Insights / Logs Insights queries**: stop scheduled queries scanning months of logs.

### CloudWatch Metrics
- **High-resolution metrics (1s)** are 10× the cost of standard — use sparingly.
- **Embedded Metric Format (EMF)**: emit metrics via logs, batch-extracted. Cheaper than PutMetricData per-call for high-cardinality metrics.

### X-Ray / Traces
- **Sampling rules** — default 1 req/sec + 5% thereafter. Raise the sample rate for high-value paths; don't trace 100% at scale.

### Third-party observability (Datadog, New Relic)
- Often 2–5× the CloudWatch cost for same data. Custom metrics cardinality is the usual killer — audit tag cardinality quarterly.

---

## 6. Kubernetes (EKS) specific

- **Karpenter over Cluster Autoscaler** — already mentioned.
- **Bin-packing**: set correct requests (not just limits). Over-requested pods leave nodes half-empty.
- **Vertical Pod Autoscaler (VPA)** in recommendation mode to right-size requests.
- **Spot node pools** for stateless workloads with PodDisruptionBudget + topology spread.
- **Cluster consolidation**: fewer, larger clusters amortize EKS control plane ($73/mo/cluster) and per-cluster add-ons.
- **Shared ingress** (one ALB per cluster with host/path routing) rather than one ALB per service.
- **kube-green / kube-downscaler** to scale dev/stg deployments to zero off-hours.

---

## 7. Process / FinOps discipline

### Tagging + accountability
- **Mandatory cost allocation tags** (`team`, `env`, `service`, `cost-center`). Enforce via SCP or AWS Config rules — untagged resources get auto-terminated in non-prod, blocked at launch in prod.
- **AWS Cost Categories** to group tags into business units for chargeback/showback.

### Budgets + alerts
- **AWS Budgets** with anomaly detection. Alert at 50/80/100% of forecast, not just actual.
- **Cost Anomaly Detection** — ML-based, catches step changes (e.g., someone turned on GuardDuty S3 Protection on a petabyte bucket).

### Organizational controls
- **SCPs** to block expensive regions, huge instance families (`*.32xlarge`), or unapproved services in non-prod accounts.
- **Service Quotas** as a guardrail — capping EC2 vCPU in dev accounts prevents runaway.
- **AWS Organizations consolidated billing** — volume discounts across accounts, one Savings Plan pool.

### Reviews
- **Monthly top-10 line item review** with service owners. Make it routine, not reactive.
- **Pre-launch cost review** for new services — estimate monthly run-rate before prod.
- **Quarterly RI/SP coverage review** — target 70–80% coverage of baseline, not 100% (leaves room for growth and risk).

---

## 8. The high-leverage 20% (if you only do this)

1. **Savings Plans on baseline compute** — 20–50% instant savings, no refactor.
2. **Kill NAT + cross-AZ traffic waste** via VPC endpoints and topology-aware routing — often the single biggest line item.
3. **CloudWatch Logs retention + drop DEBUG in prod** — typically 50% of observability cost vanishes.
4. **gp3 + Graviton** migrations — pure discount, minimal risk.
5. **Karpenter on EKS with Spot** — 40–60% node cost cut on stateless workloads.
6. **S3 Intelligent-Tiering default + incomplete-MPU lifecycle** — set-and-forget.
7. **Tagging + Anomaly Detection** — so #1–6 don't silently regress.

---

## Traps staff engineers call out

- **Don't optimize by % — optimize by $**. Don't spend a sprint saving $400/mo while a $50K/mo NAT bill rots.
- **Savings Plans aren't free** — over-committing locks you in. Start with Compute SP covering 60–70% baseline.
- **Reserved anything in a stagnant account is risk** — right before a re-architecture is the worst time to sign a 3-yr RI.
- **Cost != efficiency** — a service can be cheap and awful. Track $/request or $/MAU to catch regressions that a raw $ view misses.
- **Egress lock-in**: moving off AWS costs egress-at-retail on petabytes. Factor this into multi-cloud strategy — "AWS is cheap to enter, expensive to leave."