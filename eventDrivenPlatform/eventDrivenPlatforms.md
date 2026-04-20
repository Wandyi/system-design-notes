Multi-Level Event Bus Architecture — When You Need It

A multi-level (hierarchical) event bus uses multiple buses scoped at different levels (e.g., global → module → component), rather than one flat bus. You need it in these
situations:

  ---
1. Large-Scale Micro-frontend / Modular Apps

When independent modules/MFEs must communicate internally without polluting a global namespace. Each module has its own bus; a root bus handles cross-module events.
Prevents event name collisions and unintended listeners.

2. Event Isolation & Encapsulation

When a subsystem should not leak its internal events to the rest of the app. Example: a payment widget fires card.validated locally, but only emits payment.ready upward to
the parent bus.

3. Performance at Scale

A single global bus with thousands of listeners becomes a bottleneck. Scoped buses reduce fan-out — most events never leave their local scope and don't trigger irrelevant
handlers.

4. Multi-Tenant / Plugin Systems

Each tenant or plugin gets an isolated bus. The host app bridges only what it chooses to. Prevents tenant A's events from reaching tenant B.

5. Scoped Undo/Redo or Event Replay

When you need replay or audit trails at different granularities (e.g., replay all events in a document section vs. the whole app), each level maintains its own event
history.

6. Cross-Process / Cross-Service Boundaries

In distributed systems: an in-process bus handles local component coordination; a process-level bus (e.g., Redis pub/sub) handles inter-service events. You don't want every
local event hitting the network.

7. Access Control

Higher-level buses can act as gatekeepers — only privileged events are promoted from child to parent bus. This enforces security boundaries between components with
different trust levels.

8. Testing in Isolation

Each module's bus can be mocked independently without affecting global state. Integration tests can verify what events cross bus boundaries without noise from sibling
modules.

  ---
When You Don't Need It
❯ explain with different realistic examples
- Small/medium apps where a single bus (or simple state manager) is sufficient.
  ⏺ Multi-Level Event Bus — Realistic Example seen levels exceeds the benefit.
- Teams unfamiliar with the pattern — the indirection adds cognitive load.
  ---
1. E-Commerce Platform with Micro-frontendsen a flat bus starts requiring event namespacing conventions, listener filtering, or you're drowning in unrelated events.

Setup: A site like Amazon where Header, ProductCatalog, Cart, and Checkout are independently deployed micro-frontends.

Global Bus
├── Header MFE Bus
├── Catalog MFE Bus
│   ├── ProductCard Component Bus
│   └── Filters Component Bus
├── Cart MFE Bus
└── Checkout MFE Bus

Flow:
- User clicks "Add to Cart" inside ProductCard → fires item.selected on Catalog MFE Bus
- Catalog MFE decides this is cross-cutting → promotes it as cart.item.added to Global Bus
- Cart MFE listens on Global Bus, updates count
- Header MFE listens on Global Bus, updates cart badge

Why not flat? ProductCard fires dozens of internal events (image.hovered, price.toggled, review.expanded). The Cart and Header should never hear those — they'd be filtering
noise constantly.

  ---
2. Real-Time Collaborative Document Editor (like Notion/Google Docs)

Setup: Multiple users editing a doc with nested blocks (page → section → block).

App Bus (global)
├── Document Bus
│   ├── Section-1 Bus
│   │   ├── Block-A Bus
│   │   └── Block-B Bus
│   └── Section-2 Bus
└── Presence Bus (who's online)

Flow:
- User types in Block-A → keystroke fires on Block-A Bus (very high frequency)
- After debounce, Block-A emits block.content.changed up to Section-1 Bus
- Section-1 aggregates changes → emits section.dirty to Document Bus
- Document Bus triggers autosave and sends delta to server via App Bus

Why not flat? Keystrokes fire hundreds of times per second. If every keystroke hit the global bus, the autosave listener, the presence tracker, and the undo manager would
all be triggered unnecessarily.

  ---
3. Multi-Tenant SaaS Dashboard (like Salesforce)

Setup: Multiple tenant workspaces loaded in the same shell app.

Shell Bus (global)
├── Tenant-A Workspace Bus
│   ├── CRM Module Bus
│   └── Analytics Module Bus
├── Tenant-B Workspace Bus
│   ├── CRM Module Bus
│   └── Billing Module Bus

Flow:
- Tenant-A's CRM fires lead.converted on Tenant-A Bus
- Triggers Tenant-A's analytics refresh — stays scoped
- Shell Bus only hears workspace.notification when Tenant-A wants a toast shown in the global nav

Why not flat? Tenant-A's lead.converted must never trigger Tenant-B's analytics. A flat bus would require every listener to check tenant ID on every single event — brittle
and a security risk.

  ---
4. IoT Device Management Console

Setup: A dashboard managing thousands of devices grouped by factory → floor → device.

Global App Bus
├── Factory-1 Bus
│   ├── Floor-1 Bus
│   │   ├── Device-101 Bus
│   │   └── Device-102 Bus
│   └── Floor-2 Bus
├── Factory-2 Bus

Flow:
- Device-101 sends a temperature reading every 500ms → fires on Device-101 Bus
- Floor-1 aggregates readings → fires floor.avg.temperature.updated on Floor-1 Bus
- Factory-1 detects floor anomaly → fires factory.alert on Global Bus
- Global Bus triggers alarm UI and sends notification email

Why not flat? 1000 devices × 2 events/sec = 2000 events/sec on one bus. Most of these never need to leave the floor level. Global listeners would be overwhelmed.

  ---
5. Game Engine (like Unity's event system)

Setup: A game with scenes containing game objects with components.

Game Bus (global)
├── Scene Bus
│   ├── Player GameObject Bus
│   │   ├── HealthComponent Bus
│   │   └── InventoryComponent Bus
│   └── Enemy-1 GameObject Bus
└── UI Bus

Flow:
- Player takes damage → damage.received fires on HealthComponent Bus
- Health component checks if dead → fires player.died on Player Bus
- Scene listens → fires scene.game_over on Scene Bus
- Global Bus catches it → triggers save, analytics, achievement unlock
- UI Bus independently hears player.died → shows death screen

Why not flat? damage.received fires on every hit — could be 60x/sec in combat. The achievement system, save manager, and analytics should not process every individual
damage tick.

  ---
6. CI/CD Pipeline Dashboard (like GitHub Actions UI)

Setup: Monitoring runs with jobs containing steps.

App Bus
├── Pipeline-Run-#123 Bus
│   ├── Build-Job Bus
│   │   ├── Step-1 Bus (checkout)
│   │   └── Step-2 Bus (compile)
│   └── Test-Job Bus
│       ├── Step-1 Bus (unit tests)
│       └── Step-2 Bus (integration tests)

Flow:
- Step-2 (compile) emits log.line 500 times on Step-2 Bus — only the log viewer for that step listens
- Step-2 finishes → emits step.failed → Build-Job Bus catches it
- Build-Job marks itself failed → emits job.failed → Pipeline Bus catches it
- Pipeline Bus emits run.failed → App Bus sends Slack notification

Why not flat? Log lines from Step-2 should never trigger the Slack notification listener. Each level only cares about the aggregated outcome from the level below it.

  ---
The Common Pattern Across All Examples

Low-level bus  →  high-frequency, fine-grained, scoped events
↓  (selective promotion)
Mid-level bus  →  aggregated, module-scoped events
↓  (selective promotion)
❯ KAfka vs SQS vs Jetstream vs Pulsar vs Redpanda when should one use

⏺ Kafka vs SQS vs JetStream vs Pulsar vs Redpandas promoted vs. what stays local. That boundary IS your architecture.

  ---
Quick Mental Model First

SQS         → managed queue, fire-and-forget, ops-free
Kafka       → distributed log, high-throughput, ecosystem king
Redpanda    → Kafka but simpler to operate, lower latency
JetStream   → lightweight, edge/embedded, multi-pattern
Pulsar       → multi-tenant, geo-distributed, tiered storage

  ---
Deep Dive by System

  ---
SQS (AWS Simple Queue Service)

What it is: A fully managed message queue. Not a log, not a stream — a queue. Messages are consumed and deleted.

Use when:
- You just need reliable async task dispatch — no replay, no ordering needed
- Teams without infra expertise (zero ops burden)
- Workloads that naturally spike: image processing, email sending, background jobs
- Already all-in on AWS and don't want another system to manage
- At-least-once delivery is acceptable

Don't use when:
- You need message replay (SQS deletes after consumption)
- Multiple independent consumers need the same message (use SNS+SQS fanout)
- You need strict ordering at scale (FIFO SQS exists but throughput is capped at 300 msg/s per queue)
- Event sourcing, audit logs, stream processing

Real example:
User uploads video
→ SQS queue
→ transcoding worker (picks job, processes, deletes message)
→ thumbnail worker (separate queue via SNS fanout)
Nobody needs to replay "transcode this video" from 3 days ago. SQS is perfect.

  ---
Kafka

What it is: A distributed, partitioned, replicated commit log. Messages are retained regardless of consumption. Multiple consumer groups read independently.

Use when:
- High throughput: millions of events/sec (clickstreams, metrics, logs)
- Multiple independent consumers need the same data (analytics + search index + ML pipeline all reading the same event stream)
- Replay is critical — reprocess historical data after a bug fix or new consumer
- Event sourcing / CDC (Change Data Capture from databases)
- Stream processing with Kafka Streams or Flink
- You have dedicated infra/platform team to manage it

Don't use when:
- Small team, no ops capacity — Kafka is genuinely complex to operate (ZooKeeper/KRaft, partition rebalancing, consumer lag monitoring)
- Message-level TTL, delay queues, per-message routing — Kafka doesn't do these well
- Sub-millisecond latency requirements (Kafka batches for throughput)
- Simple job queues — massive overkill

Real example:
Uber ride events (trip.started, location.updated, trip.ended)
→ Kafka topic: ride-events
→ Consumer Group A: real-time driver tracking (reads live)
→ Consumer Group B: surge pricing ML model (reads live)
→ Consumer Group C: data warehouse ETL (reads at its own pace)
→ Consumer Group D: fraud detection (replays last 7 days after model update)
All four systems read the same topic independently. Replay after model update is a core feature.

  ---
Redpanda

What it is: Kafka-compatible API, rewritten in C++ with no JVM, no ZooKeeper. Drop-in replacement for Kafka.

Use when:
- You want Kafka semantics but Kafka's operational complexity is killing you
- Latency matters more: Redpanda's p99 latency is significantly lower than Kafka
- Smaller teams that can't staff a Kafka platform team
- Kubernetes-native deployments (single binary, simpler k8s story)
- Dev environments where running Kafka locally is painful
- Edge deployments where JVM overhead is unacceptable

Don't use when:
- You need the full Kafka ecosystem (Kafka Connect connectors, Kafka Streams) — compatibility is high but not 100%
- Your org already has mature Kafka infrastructure — switching costs outweigh gains
- Managed cloud is preferred — Redpanda Cloud exists but less mature than Confluent Cloud

Real example:
Fintech startup, 5-person infra team
Need Kafka semantics for transaction event streaming
Can't afford a dedicated Kafka platform engineer
→ Redpanda on k8s: same consumer/producer code, 1/3 the operational overhead

Kafka vs Redpanda decision:
Have Kafka expertise + large scale?  → Kafka
Want Kafka without the pain?         → Redpanda

  ---
NATS JetStream

What it is: NATS is a lightweight messaging system. JetStream adds persistence, replay, and consumer ACKs on top of core NATS pub/sub.

Use when:
- Edge computing, IoT, embedded systems — NATS binary is ~20MB, extremely low resource usage
- Multi-pattern needs in one system: pub/sub + queue groups + request/reply + streams — NATS does all of them
- Service mesh / microservice RPC + eventing — NATS handles both, Kafka doesn't do RPC
- Low-latency requirements — NATS is one of the fastest messaging systems
- Multi-cloud or hybrid environments — NATS clustering across clouds is simpler than Kafka
- You need at-most-once AND at-least-once in the same system

Don't use when:
- Very high throughput log streaming at Kafka scale (Kafka still wins here)
- You need the rich stream processing ecosystem (Kafka Streams, ksqlDB)
- Long-term message retention as a primary use case (JetStream can do it but it's not the sweet spot)
- Team is already Kafka-fluent

Real example:
Connected car platform:
- Car ECU (edge) → publishes telemetry via NATS (lightweight, works on 4G with reconnect)
- Backend services → subscribe to telemetry streams via JetStream (persistence + replay)
- Diagnostics service → request/reply via NATS core (synchronous RPC, same system)
- Fleet command → publishes commands to specific cars (point-to-point via NATS subjects)
  One system handles all messaging patterns — pub/sub, queue, RPC, streaming.

  ---
Apache Pulsar

What it is: A messaging system that separates compute (brokers) from storage (BookKeeper). Native multi-tenancy, geo-replication, and tiered storage built-in.

Use when:
- Multi-tenant platforms where teams need isolated namespaces with quotas and ACLs
- Geo-replication is a first-class requirement — Pulsar's active-active geo-replication is simpler than Kafka's MirrorMaker
- Infinite retention with tiered storage — Pulsar can offload cold data to S3/GCS natively
- You need both queuing (exclusive consumers) and streaming (shared consumers) from the same topic
- Large-scale cloud-native deployments where elastic scaling is critical (compute and storage scale independently)

Don't use when:
- Small/medium scale — Pulsar requires BookKeeper + ZooKeeper + brokers — more components than Kafka
- Team isn't familiar with it — ecosystem and community are smaller than Kafka
- Simple use cases — like using a rocket to go to the grocery store

Real example:
Multi-cloud messaging platform for a bank:
- EU region and US region both produce and consume (active-active geo-replication)
- Retail banking team gets tenant A (isolated namespace, 100 MB/s quota)
- Investment banking gets tenant B (isolated namespace, separate ACLs)
- 7-year audit log retention → hot data in BookKeeper, cold data auto-offloaded to S3
- Compliance consumer replays 3-year-old events after regulatory audit
  Kafka could do parts of this but each feature (geo-replication, tiered storage, multi-tenancy) requires separate tooling. Pulsar has it built-in.

  ---
Decision Matrix

┌────────────────────┬──────────────┬─────────────┬──────────────┬────────────┬───────────┐
│      Criteria      │     SQS      │    Kafka    │   Redpanda   │ JetStream  │  Pulsar   │
├────────────────────┼──────────────┼─────────────┼──────────────┼────────────┼───────────┤
│ Ops complexity     │ None         │ High        │ Medium       │ Low        │ Very High │
├────────────────────┼──────────────┼─────────────┼──────────────┼────────────┼───────────┤
│ Throughput ceiling │ Medium       │ Very High   │ Very High    │ High       │ Very High │
├────────────────────┼──────────────┼─────────────┼──────────────┼────────────┼───────────┤
│ Message replay     │ No           │ Yes         │ Yes          │ Yes        │ Yes       │
├────────────────────┼──────────────┼─────────────┼──────────────┼────────────┼───────────┤
│ Multi-consumer     │ Via SNS      │ Yes         │ Yes          │ Yes        │ Yes       │
├────────────────────┼──────────────┼─────────────┼──────────────┼────────────┼───────────┤
│ Latency            │ Medium       │ Medium      │ Low          │ Very Low   │ Medium    │
├────────────────────┼──────────────┼─────────────┼──────────────┼────────────┼───────────┤
│ Geo-replication    │ Native (AWS) │ MirrorMaker │ Manual       │ Manual     │ Native    │
├────────────────────┼──────────────┼─────────────┼──────────────┼────────────┼───────────┤
│ Multi-tenancy      │ No           │ Limited     │ Limited      │ Namespaces │ Native    │
├────────────────────┼──────────────┼─────────────┼──────────────┼────────────┼───────────┤
│ Ecosystem          │ AWS          │ Huge        │ Kafka-compat │ Small      │ Medium    │
├────────────────────┼──────────────┼─────────────┼──────────────┼────────────┼───────────┤
│ Edge/IoT fit       │ Poor         │ Poor        │ Poor         │ Excellent  │ Poor      │
├────────────────────┼──────────────┼─────────────┼──────────────┼────────────┼───────────┤
│ Tiered storage     │ No           │ Plugin      │ Yes          │ Limited    │ Native    │
└────────────────────┴──────────────┴─────────────┴──────────────┴────────────┴───────────┘

  ---
One-Line Decision Rules

"I just need background jobs"                     → SQS
"Multiple teams consume the same events"          → Kafka
"I want Kafka but my team hates running it"       → Redpanda
"IoT / edge / I need RPC + pub/sub together"      → JetStream
"Multi-region active-active + multi-tenant SaaS"  → Pulsar
"Event sourcing at scale with replay"             → Kafka or Redpanda
"Already on AWS, ops team is small"               → SQS → upgrade to Kafka later

  ---
Common Upgrade Path

Early stage startup:
SQS (zero ops, good enough)
↓  (when you need replay or multiple consumers)
Redpanda or Kafka
↓  (when you expand to multiple regions + tenants)
Pulsar  (or Kafka + Confluent Platform at cost)

The biggest mistake teams make: jumping to Kafka on day one because it's prestigious, then spending 6 months tuning brokers instead of building product.