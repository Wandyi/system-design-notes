# 2 · Engineering Culture and Tech Stack

LinkedIn engineering is **JVM-heavy, data-driven, deeply opinionated about ownership**, and one of the most open-source-prolific orgs in the industry. This file describes the stack from end to end so that when an interviewer asks you a question about *their* system, you can frame your answer in *their* vocabulary.

## 2.1 The languages

### Java + Scala — the dominant duo

- **Java** is the lingua franca. Most online services, batch jobs, and infra are Java.
- **Scala** is heavily used in data processing (Spark, Samza pipelines) and some online services that grew out of streaming teams.
- Versions: most LinkedIn code is on Java 11 / 17 these days; some still on 8. Scala 2.12 dominant.

### Python

- Data science, ML model training, notebooks, glue scripts.
- A growing footprint in ML inference serving (Triton, vLLM-based services).

### Go and Rust

- **Go** in infrastructure: some service-mesh / proxy components, some new K8s-native control planes, some observability tooling. Not the dominant language but growing.
- **Rust** appears in newer perf-critical components (a small set of teams). Not pervasive.

### JavaScript / TypeScript

- React + TypeScript for `linkedin.com` and most product surfaces.
- Ember.js powered linkedin.com for years; now mostly migrated to React.
- Node.js for BFF (backend-for-frontend) layers in some places.

### Mobile

- **Android**: Kotlin (Java legacy still present).
- **iOS**: Swift (Objective-C legacy still present).

### SQL & SQL-ish

- **MySQL** (Espresso's underlying engine).
- **Trino / Presto** for ad-hoc query over the data lake.
- **Pinot SQL** for OLAP.
- **Spark SQL** for batch.

## 2.2 Service framework and RPC

### Rest.li — the primary RPC framework

- Built by LinkedIn, open-sourced. Strongly-typed schemas (PDSC → now PDL), code-gen for clients in Java/Scala/Python.
- Supports REST + Action endpoints + Finder endpoints (a richer model than plain REST).
- Service definitions live in a shared schema repo; clients pin to specific versions.

### D2 (Dynamic Discovery)

- Service-discovery layer underneath Rest.li.
- Uses ZooKeeper as the registry. Clients subscribe; ZooKeeper pushes updates.
- Supports degrader load balancing — clients track latency/error rate per host and shed traffic to slow ones (similar to ideas later popularized by Linkerd, Envoy outlier detection).

### GraphQL

- Adopted for some frontend BFF surfaces, especially on the consumer-facing apps.
- The "**linkedIn Frontend Compute (LFC) / GraphQL gateway**" pattern: a single GraphQL gateway federates many Rest.li services.

### gRPC

- Less common than Rest.li historically; growing for newer services, especially infra-to-infra and ML serving where lots of bytes/sec matter.

## 2.3 Data storage layer

### Espresso — the primary online document store

- Distributed, partitioned, replicated.
- **Backing engine: MySQL** per partition. Espresso adds the distribution layer, schema evolution, multi-table semantics.
- Use cases: member profile, mailbox, settings, most "primary" online state.
- Built on **Helix** for cluster management; **Databus** (now Brooklin) for change capture.
- Strong consistency for single-row writes; eventual replication across regions.

### Voldemort — older Dynamo-style KV store

- Open-sourced 2009. Heavily used historically, less so for new work.
- Two flavors: read-write (for online state) and **read-only** stores (built offline in Hadoop, then bulk-loaded into Voldemort for serving — pre-cursor to Venice).
- Eventual consistency, vector clocks, sloppy quorum / hinted handoff.

### Venice — derived-data serving (successor to Voldemort RO)

- Stores data computed offline (Spark) or via stream (Samza/Flink), served online.
- Massive at LinkedIn: feature store for ranking models, denormalized data for hot reads.
- Built on Helix + Kafka + RocksDB.
- Open-sourced 2022 — `github.com/linkedin/venice`.

### Ambry — blob store

- Immutable, large-object store (photos, attachments, video segments).
- Append-only, distributed, no global metadata index — uses an in-memory hash index per partition.
- Open-sourced; the LinkedIn answer to S3 for their internal needs.

### Pinot — real-time OLAP

- Columnar, distributed, sub-second query latency.
- Powers "Who's viewed your profile?", member dashboards, internal analytics, anomaly detection.
- Real-time tables ingest directly from Kafka; offline tables from Hive/Spark.
- Apache project since 2018; LinkedIn is the original author and largest user.

### Other stores

- **Kafka** — distributed log; the canonical event bus. Hundreds of clusters, trillions of messages per day.
- **Couchbase / Memcached** — caching layers.
- **GraphDB (LIquid / homegrown)** — backs the social graph for queries like 2nd/3rd-degree connections.
- **Hadoop / HDFS** — data-lake batch processing.
- **Iceberg / OpenHouse** — newer table format on top of HDFS / object storage; LinkedIn open-sourced **OpenHouse** as a unified table service.
- **Druid** — used in places, but Pinot has been displacing it.

## 2.4 Stream processing

### Samza

- LinkedIn-born, Apache project.
- Built on Kafka — uses Kafka for input, output, and (importantly) for **changelog/state**.
- Per-partition local state in RocksDB; state recovery from Kafka changelog topics.
- Native Kafka semantics; very predictable scaling.

### Flink

- Increasing adoption for jobs that need stronger event-time semantics, complex windowing, or exactly-once SQL.
- Not the historical default but pragmatic adoption is happening.

### Beam

- Not a primary engine; sometimes used as an abstraction.

## 2.5 Batch processing

- **Spark** dominates. Scala APIs primarily; PySpark for data science.
- **Hive** (Hadoop SQL) still significant; gradually being replaced by Trino/Pinot.
- **Azkaban** — LinkedIn-born workflow orchestrator (think Airflow predecessor). Still widely used internally.
- **Airflow** also present.

## 2.6 ML infrastructure

LinkedIn's ML stack is unusually mature because so much of the product (Feed ranking, PYMK, jobs, search, ads, notifications, trust) is ML-driven.

- **ProML** (sometimes called **Pro-ML** or **DARWIN / Pro-ML**) — LinkedIn's ML platform: feature store, model training, evaluation, A/B integration, deployment.
- **Feathr** — open-sourced feature store (joint with Microsoft).
- **Quasar** — LinkedIn's offline feature-engineering DSL.
- **TonY** — TensorFlow-on-YARN, open-sourced.
- **GDMix** — large-scale linear-model training.
- **Photon-ML** — Spark-based ML library (notable for generalized additive mixed effects, GAME).
- **Serving**: many services serve their own models in JVM (XGBoost, TFLite, ONNX). For LLMs, vLLM and Triton appear in recent infra.

For GenAI:

- **Recruiter Copilot, Premium AI features, AI writing assistance** — built on top of OpenAI (via Azure), with internal RAG pipelines over the Economic Graph.
- **GAI Platform** — LinkedIn's name for the internal GenAI serving + eval + prompt-management layer.

## 2.7 Front-end and mobile

### Web

- React + TypeScript primary stack.
- Build with internal tooling (Bazel-ish), feature-flag everything via LIX.
- BFFs in Node.js or Java, often federated through a GraphQL gateway.

### Mobile

- Single-binary Android (Kotlin) and iOS (Swift) apps.
- Heavy use of **Tetra / Voyager** — internal frameworks for screen composition.
- Hybrid: lots of native + some webview surfaces for fast-changing content.

## 2.8 Observability, deploys, and platform infrastructure

- **Monitoring**: InGraphs (LinkedIn-built metrics platform) → Grafana visualisations. Lots of histograms.
- **Logging**: Splunk + an internal pipeline through Kafka.
- **Tracing**: OpenTelemetry adoption; internal distributed-trace UI.
- **Alerting**: Iris (LinkedIn-built escalation tool; open-sourced).
- **Deploys**: **LPS (LinkedIn Platform Services)** + **inGraphs** + ramp-via-LIX. Canaries, regional rollouts, fast rollback.
- **CI/CD**: Jenkins historically; ProductionEng has been modernizing onto Azure DevOps + GitOps in places.
- **Containers**: Kubernetes ("**LIK**" or just "k8s") is the primary runtime for new services. Older bare-metal "**inframan**" pools still exist for stateful systems (Kafka, Espresso, etc.).
- **Service mesh**: D2-based for Rest.li services; Envoy-based proxies in some newer environments.

## 2.9 LIX — the experimentation platform

- LinkedIn's A/B testing framework. Internal name: **LIX (LinkedIn Experimentation)**.
- Every feature ramp goes through LIX. You request a "treatment" and get a value.
- Integrates with the analytics pipeline: define a metric, see lift, get a recommendation.
- The **causality / statistics** side is rigorous — staff candidates are expected to know about peeking, sequential testing, novelty/primacy effects.

Be ready to discuss:
- How would you ramp a risky migration? (Start at 0.1% in one DC, etc.)
- What's a "guardrail metric" vs. a "primary metric"?
- How do you avoid p-hacking when you have hundreds of metrics?

## 2.10 Engineering culture — what to expect

### Code review and design review

- Every non-trivial change has a design doc; staff candidates write the doc.
- Design docs go to an **architecture review** for cross-team impact.
- Code review is rigorous; LGTM is earned.

### Reliability culture

- Heavy on **post-mortems** — blameless, written within days of an incident.
- **Site SUP (Site Up)** dashboards visible to whole company.
- On-call rotations across teams; staff engineers expected to lead complex incidents.

### Inner-source

- Lots of cross-team contribution. Most internal libraries accept PRs from other orgs.
- A staff candidate is expected to drive cross-team code changes.

### "Operational excellence"

- A meta-competency LinkedIn explicitly tracks. Includes capacity planning, on-call hygiene, post-mortems, security posture, dependency management, paying down tech debt.
- In behavioral interviews, expect a question that touches OE explicitly.

### "Members first"

- Cultural mantra: when in doubt, prioritize member trust. This biases LinkedIn against dark patterns and biases for slower, more deliberate launches than some peers.

## 2.11 Anti-patterns the interviewers watch for

- "We'd just use DynamoDB / Redis / Postgres" without acknowledging the LinkedIn-internal equivalent. Be aware of their stack.
- Designing pure greenfield without considering migrations from existing systems.
- Ignoring multi-DC failover (LinkedIn runs in multiple data centers + Azure regions).
- Forgetting tracking events — every consumer-facing surface emits tracking events; that's the lifeblood of LIX and analytics.
- Designing without a cost lens — capacity / cost are first-class concerns at Staff level.