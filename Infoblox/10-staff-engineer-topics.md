# 10 · Staff-Engineer Topics — Scale, Tradeoffs, and the Conversations That Matter

At staff level, the technical bar is no longer "can you code it" — it's "can you reason about a system as it evolves, name the right tradeoffs, and lead the room to a decision". This file collects the topics that show up in staff-deep-dive rounds where the interviewer is looking for senior judgment, not algorithm recitation.

## 10.1 The CAP triangle, distilled for DDI

The classical CAP statement: a distributed system can only guarantee 2 of {Consistency, Availability, Partition tolerance}. In practice, partition tolerance is non-negotiable, so the real choice is **C vs. A under partition**.

Where each DDI subsystem sits:

| System | Choice | Why |
|--------|--------|-----|
| **DNS recursive resolver** | AP | Stale cached answers >> outage. Eventual consistency is fine. |
| **DNS authoritative** | CP within a zone, AP across replicas | Stale zone data is acceptable; conflicting "current" answers are not. Replicas serve last-known good. |
| **DHCP lease allocation** | CP | Two clients with the same IP is a disaster; better to delay a lease than to grant a conflict. |
| **IPAM allocation** | CP | Same reason as DHCP. |
| **Threat intel feed** | AP | Stale block list is better than no block list. |
| **NIOS Grid Master** | CP for control, AP for data plane | The data plane keeps working; you lose only the ability to make changes. |
| **Audit log** | CP eventually | Lose nothing; tolerate some lag. |

You'll get credit for naming these explicitly.

## 10.2 Consistency models — vocabulary you must use precisely

- **Linearizable**: every operation appears to happen at a single instant between invocation and response, in a global total order. Hardest to scale.
- **Sequential consistency**: total order, but a process sees its own operations in program order. (Slightly weaker than linearizable.)
- **Causal consistency**: writes that are causally related are seen in order; concurrent writes may be reordered.
- **Read-your-writes**: a client always sees its own previous writes.
- **Monotonic reads**: a client never sees data go "back in time".
- **Eventual consistency**: replicas converge if updates stop.

For an IPAM allocate-IP API, you want at least **linearizable per subnet** (no two clients get the same IP). Across subnets, eventual is fine.

For a DNS cache, **eventual** is the model and you spend your engineering on minimizing staleness.

## 10.3 Replication strategies

| | Sync replication | Async replication | Quorum (e.g., Raft) |
|---|------------------|-------------------|---------------------|
| Latency | Highest | Lowest | Medium |
| Durability | Best | Risk of loss on master crash | Strong, configurable |
| Availability under failure | Poor (writes block) | Best | Good (N/2+1) |
| Use case | Tight HA pairs | Cross-region replicas | Strongly consistent multi-DC |

The NIOS GM↔member replication is functionally **async with checkpointing**. Modern BloxOne services typically use Raft (etcd, CockroachDB) for state that must be strongly consistent.

## 10.4 Sharding strategies

- **Hash sharding** — uniform distribution; bad for range scans.
- **Range sharding** — good for ranges; risk of hot spots.
- **Consistent hashing** — adds/removes nodes with minimal rebalance.
- **Directory-based** — explicit mapping, more flexible, requires a lookup tier.

For DNS caches: shard by `hash(qname)`. For IPAM: range-shard by CIDR prefix (so a /16 stays on one shard).

## 10.5 Backpressure

Every fast producer hitting a slow consumer needs an explicit backpressure strategy. The choices:

1. **Block the producer** (synchronous). Simple. Hurts latency, propagates pressure upward.
2. **Buffer with bounds**. Bounded queue; producer blocks when full. Adds memory, delays the back-propagation.
3. **Drop**. Either oldest or newest. Acceptable for telemetry; never for transactions.
4. **Shed load** at the boundary. Return 429/503 to callers before resources are saturated. Best for HTTP APIs.

A staff answer always specifies *which* of these is in use, not "we have a queue".

## 10.6 Multi-tenancy at SaaS scale

The bullet points that matter:

- **Logical**: tenant_id everywhere. Cross-tenant data access is a P0 bug.
- **Quotas**: rate, storage, compute. Enforced at boundaries (API gateway, ingest endpoints).
- **Noisy neighbor**: per-tenant resource isolation. Worst-case: shard whales onto dedicated infra.
- **Cost attribution**: per-tenant metering even if you don't bill by usage; lets you see which tenants cost what.
- **Tenant lifecycle**: provision, suspend, delete, restore. Cleaning up data on delete is a regulatory requirement (GDPR right-to-erasure).
- **Schema evolution**: migrations must be safe for every tenant; can't rely on coordinated downtime.

## 10.7 Observability — SLI/SLO/SLA in the language they expect

- **SLI** — the measurement (p99 latency, error rate, availability).
- **SLO** — the target you set internally (p99 < 50 ms; error rate < 0.1%; 99.95% available).
- **SLA** — the contractual promise to customers, usually weaker than the SLO.
- **Error budget** — `1 - SLO`. The amount of unreliability you're allowed.

For each major service, a staff engineer should be able to name:
1. Three SLIs that matter.
2. Their SLO.
3. The alert thresholds (typically based on burn-rate).

Example for the BloxOne DNS data plane:
- SLI: query success rate (non-SERVFAIL), SLO 99.99%, alert if 1h burn rate > 10× budget.
- SLI: p99 query latency, SLO < 20 ms, alert if 1h average > 50 ms.
- SLI: cache freshness (last threat-intel refresh < 5 min ago), SLO > 99.5%, alert if stale > 15 min.

## 10.8 Migration playbook (the perennial staff topic)

Almost every staff project at Infoblox involves migration — NIOS to BloxOne, one DB engine to another, monolith to microservices. The playbook:

1. **Define the cutover criteria** in measurable terms (functional parity, performance targets, data consistency).
2. **Build the new system parallel to the old**. Both writable.
3. **Dual-write** with reconciliation. Diff old vs. new in shadow mode.
4. **Dual-read with consistency check**. Promote the new system to authoritative read.
5. **Cut writes** with a kill-switch and reversibility for N days.
6. **Decommission** only after stability + data retention requirements satisfied.

Failure mode to discuss: a dual-write where one side fails silently → divergence accumulates → audits surface gaps months later. Mitigation: continuous reconciliation, not just at migration time.

## 10.9 Cost-aware architecture

At staff level the right answer to "use Aurora vs. Postgres vs. Cassandra" includes *cost*. Cloud costs scale with engineering choices:

- **Egress** is the highest-margin AWS line item. Architectures that pull large data cross-region cost a lot.
- **Cross-AZ traffic** is metered; intra-AZ is free. Architects who care put replicas in the same AZ until durability requires otherwise.
- **Per-request pricing** of managed services (DynamoDB, Lambda) is great until you hit scale; provisioned tiers may be cheaper.
- **Idle compute** is real money — autoscaling and request-batching matter.

Be ready to talk about a cost vs. performance tradeoff you've made. Concrete numbers (in dollars or % savings) impress.

## 10.10 Security — the table-stakes vocabulary

For Infoblox specifically, expect at least one round to touch security:

- **mTLS** for service-to-service; cert rotation strategy.
- **OAuth 2.0 / OIDC** for human auth; tenant-scoped tokens.
- **Defense in depth** — DNS-layer + firewall + endpoint, with no single point of failure.
- **Secret management** — Vault, AWS KMS, cloud-native equivalents. Never in code, never in env vars for prod.
- **Audit log** — append-only, signed, retained per compliance (SOC 2 = 1 year, FedRAMP = 3 years).
- **Compliance frameworks** — SOC 2 Type 2, ISO 27001, FedRAMP Moderate/High, HIPAA, PCI. Know roughly what each requires.
- **Threat modeling** — STRIDE for systems, MITRE ATT&CK for adversary behavior. Brought up casually in design discussions.
- **Supply chain** — SBOM, signed builds (Sigstore / cosign), reproducible builds.

## 10.11 Failure-mode tabletop (the deep-dive round)

A common staff round: "walk me through what happens to your system if X breaks." The drill:

1. What's the first metric that moves?
2. What alerts fire?
3. What does the on-call do in the first 5 minutes?
4. What's the blast radius?
5. What's the RTO (recovery time objective)?
6. What's the RPO (recovery point objective, max data loss)?
7. What's the postmortem action that prevents recurrence?

Rehearse this for two or three of your past systems. Use it for any new design you propose.

## 10.12 Influence without authority — the soft side of staff

A staff engineer doesn't manage people, so leverage is technical and rhetorical. The patterns that work:

- **Design docs** with explicit context, alternatives considered, and rejected options. Make decisions visible and rebuttable.
- **Architecture review forums** — propose your idea, invite criticism, integrate. The point isn't being right; it's being a good steward of decisions.
- **Mentorship** — staff time spent making mid-level engineers more effective compounds. Code review with intent. Pair on incident response.
- **Saying no with a reason and an alternative** — "we shouldn't add this column to the hot table; if you need it, here's the projection table that fits."
- **Owning the migration of someone else's project** — staff people unblock; they don't only build new things.

Behavioral questions in this category: "tell me about a time you disagreed with a senior leader / changed someone's mind / mentored someone through a hard problem".

## 10.13 Architecture decision records (ADRs)

A staff engineer who hasn't written ADRs is unusual. The standard format:

```
# ADR 0042: Use Patricia trie for IPAM in-memory representation

## Status
Accepted, 2026-04-12

## Context
We need fast next-available-subnet lookups over ~10M allocations. The DB query
is O(n) without good indexing.

## Decision
In-memory Patricia trie per VRF, rebuilt from DB at boot and updated on writes.

## Consequences
+ O(label-count) lookups; matches access patterns.
+ Memory ~200 MB per VRF for our largest customer.
- Service must hold full state; tenant-scale upper-bound enforced.
- Recovery on boot is slow (build trie from DB); mitigation: snapshot to S3.

## Alternatives Considered
- Pure-DB query with GiST index. Acceptable but 50× slower at scale.
- Distributed in-memory store (Hazelcast / Redis). Adds operational surface.
```

If asked "how did your team make decisions", referencing ADRs is the cleanest answer.

## 10.14 The leadership signals interviewers look for

From the candidate's words during a system-design round, the interviewer is trying to score:

1. **Clarification before design** — did you ask scope/scale/SLO questions, or jump to a diagram?
2. **Tradeoff articulation** — "we could do X, which gets us Y at the cost of Z" said multiple times.
3. **Operational thinking** — observability, failure modes, on-call burden mentioned voluntarily.
4. **Cost awareness** — at least once, a cost consideration named.
5. **Pragmatism** — willing to call something "good enough for v1, we revisit at scale".
6. **Curiosity** — asks the interviewer interesting questions back.

Practice: every time you finish describing a component in a mock interview, say one tradeoff out loud.

## 10.15 Must-internalize

- CAP and consistency-model vocabulary used precisely.
- Replication / sharding choices articulated.
- Multi-tenancy isolation patterns named.
- SLI/SLO/error-budget vocabulary; specific numbers for *your* systems.
- The dual-write migration playbook with cutover criteria and reversibility.
- Cost-awareness with at least one concrete example.
- The failure-mode tabletop for any design you propose.
- Influence-without-authority via ADRs and design reviews.