# 6 · System Design — LinkedIn Search

LinkedIn's search platform is **Galene** — a homegrown distributed search system originally Lucene-based, evolved over a decade to handle people/job/company/content search at LinkedIn scale. The retrieval-side companion is **Bobcat**, the structured query layer for typed search (e.g., Recruiter's faceted searches over the Economic Graph). Recent work has added **vector / embedding** retrieval for AI surfaces.

A staff candidate is expected to know all three modes (lexical, structured/faceted, vector) and when to use each.

## 6.1 Requirements

### Functional

- **Member search** — type a name, get ranked people with rich previews.
- **Typeahead** — autocomplete on names, companies, skills, schools as the user types.
- **Faceted / filtered search** (Recruiter) — Boolean filters (location, current title, seniority, skill, school, current employer, years of experience) over millions of candidate profiles.
- **Content search** — find posts, articles, comments by keyword.
- **Job search** — find postings by title, location, skill, salary, remote.
- **Cross-entity search** — a query like "Vaibhav Kumar" might hit People, Posts, Companies, Jobs simultaneously.
- **Vector / semantic** — for AI surfaces ("find me senior engineers in Toronto with Kafka experience and a public Github profile"), embedding similarity + filters.

### Non-functional

- **Scale**: hundreds of millions of indexed documents per index type. Total index size: tens of TB.
- **Query latency**: p95 typeahead < 100ms server-side; p95 full search < 300ms; Recruiter advanced search < 1s.
- **Freshness**: profile edits visible in search within ~1 minute. New posts within seconds.
- **Availability**: 99.95% — search degradation cascades to most products.
- **Throughput**: tens of thousands of queries/sec at peak.

## 6.2 Galene — the search platform

Galene was the internal name for the platform that succeeded an earlier Lucene-based search service. It is:

- A **sharded inverted index** built on Lucene.
- A **search-broker layer** that fans queries to shards and merges results.
- An **indexing pipeline** (offline + real-time) that updates shards from Kafka events.
- A **scoring pipeline** that supports both classical IR scoring (BM25, TF-IDF) and LinkedIn's ML scoring.

```
   Query                                Index updates
     │                                        ▲
     ▼                                        │
   ┌──────────────┐                    ┌────────────────┐
   │ Search Broker│                    │ Indexing Svc   │
   │ (federator)  │                    │ (real-time +   │
   └──────┬───────┘                    │  offline build)│
          │                            └────────────────┘
          │ scatter
          ▼
   ┌────────┬────────┬────────┬────────┐
   │ Shard 1│ Shard 2│ Shard 3│  ...   │   ← Lucene segment stores
   └────────┴────────┴────────┴────────┘
          │ gather
          ▼
   ┌──────────────┐
   │  Merger /    │
   │  Reranker    │
   └──────┬───────┘
          │
          ▼
   ┌──────────────┐
   │  Hydration   │  ← fetch full doc previews from primary store
   └──────────────┘
```

### Sharding strategy

- **People index**: sharded by `member_id mod N` — uniform distribution, no hot shards. Each shard holds all features (name, title, skills, etc.) for its members.
- **Job index**: sharded by `job_id mod N` similarly. Replicas tuned for read QPS.
- **Posts index**: sharded by time-bucket (week) × hash(post_id) — older shards become read-only, new shards absorb writes.
- **Replication**: each shard has 3+ replicas behind a load balancer.

### Indexing pipeline

- **Source-of-truth**: Espresso (members, jobs), separate stores for posts/companies.
- **CDC** (Brooklin) → Kafka → an indexing service.
- Indexing service materializes the searchable document (normalized title, expanded skills, denormalized company info, geo-coordinates, etc.) and pushes updates to the relevant shard.
- Two pipelines:
  - **Real-time**: low-latency updates for high-importance fields (name, current title).
  - **Batch** (offline rebuild): nightly Hadoop job builds a complete index from scratch — used for major schema changes, ranking model updates, or fixing skew/corruption.

### Reading

- Broker receives query, routes to relevant indexes (federation — see below).
- For each index: parses the query, expands synonyms / skill embeddings, evaluates against each shard in parallel.
- Each shard returns its top-K with scores; broker merges + reranks with an ML model.

## 6.3 Bobcat / structured retrieval (Recruiter)

Recruiter's faceted search is materially different from member search. The query is a Boolean expression over typed facets:

```
location IN ("San Francisco", "Toronto")
  AND skills CONTAINS "Kafka"
  AND current_title MATCHES "senior engineer"
  AND years_experience BETWEEN 5 AND 15
  AND open_to_work = true
```

Requirements:
- Must return **counts per facet** ("123 engineers in SF with Kafka") — a roll-up over the entire result set.
- Must support **complex Boolean** queries with arbitrary nesting.
- Must filter against **billions of edges** (Economic Graph relationships).

**Bobcat** is the structured query layer that translates Boolean predicates into shard queries, runs Lucene filter chains, aggregates counts (BKD trees, doc-value scans), and returns ranked + counted results.

Underneath, much of this still rides on Lucene, but the query model is structured rather than free-text. Bobcat may also dip into Pinot for some counting aggregations.

## 6.4 Typeahead

Typeahead is its own beast — separate infrastructure from full search because:
- Latency budget is 50–100ms server-side.
- Query throughput is much higher than full search (every keystroke).
- Ranking is simpler — popularity + match prefix + member-affinity.

Architecture:
- A **prefix tree (trie) or FST** for fast prefix lookups, sharded by first character bucket.
- Per-query: lookup ~200 candidates, rank by combined score = log-popularity + prefix match quality + member-network proximity.
- Cached aggressively at the edge.

Subtle: typeahead needs to handle name diacritics, case-folding, partial-token matches ("Vai Kum" should suggest "Vaibhav Kumar"), and personalize ("Vai" should surface my colleagues, not random Vais).

## 6.5 Federated / cross-entity search

When a member types "Microsoft" into the global search bar, the system must determine:

- Are they looking for the company?
- A person at Microsoft?
- Job postings at Microsoft?
- Recent posts about Microsoft?

Approach:
- Broker fans out queries to all relevant indexes in parallel.
- Each index returns top-K with its own scores.
- A **federator / blender** reranks across entity types using:
  - Member's recent behavior (do they usually click company pages?).
  - Query intent signals (is "Microsoft" usually a company vs. a person?).
  - Click feedback loops from past queries.
- Returns a blended result list with type-aware UI.

This is one of the more subtle design questions — the candidate must reason about the **multi-objective ranking** problem.

## 6.6 Vector / semantic search

Increasingly important for AI-driven surfaces. Pattern:

- **Embedding generation**: profile text → embedding via a fine-tuned transformer. Generated by an offline batch + online incremental job; stored in Venice or an embedding-specialized store.
- **ANN index**: HNSW or IVF-PQ over those embeddings.
- **Hybrid retrieval**: a query is converted to an embedding and (typically) also tokenized into structured facets. Retrieval uses both vector ANN and lexical/structured filters, intersected.
- **Reranking**: cross-encoder over the top candidates for precision.

When designed in an interview, talk about:
- **Storage**: dimensions × bytes × docs. 768-dim float embeddings on 500M members = ~1.5 TB raw. Quantization gets you to ~300 GB.
- **Latency**: HNSW p99 < 50ms at this scale, with sufficient replicas.
- **Index freshness**: how do you update embeddings when a profile changes? (Batch nightly with online deltas for top fields.)

## 6.7 Ranking

Search at LinkedIn uses learning-to-rank. Three phases:

1. **Retrieval**: cheap, recall-focused — fetch ~1000 candidates per shard, merge to ~500.
2. **Reranking (Layer 1)**: a fast linear model over the top 500.
3. **Reranking (Layer 2)**: a heavier GBDT or DNN over top 50–100.

Features include:
- Query-doc text features (BM25, term overlap, embedding cosine).
- Doc features (member popularity, profile completeness, recency).
- Query-side features (intent classifier output, geo).
- Personalization features (who's the searcher, their history, their network).
- Cross-features (searcher × doc interaction history).

For Recruiter, ranking is additionally tied to **commercial signals** — how good is this candidate for *this specific Recruiter's typical hires*?

## 6.8 Indexing freshness — the hard tradeoffs

- **Real-time** indexing has cost: every Lucene segment merge is expensive.
- LinkedIn uses a **near-real-time (NRT)** pattern: a small in-memory index per shard that holds recent updates, queried alongside the on-disk index, periodically flushed.
- For very high-update fields (likes, comments counts), don't store in Lucene at all — read from a counters service at query/result time.

Discussion points:
- The trade-off between segment size, merge frequency, and query latency.
- When to use **doc-values** vs. **stored fields**.
- How to handle deletes (tombstones) and reclaim space.

## 6.9 Multi-region

- Each region has full read replicas of all indexes.
- Indexing pipeline writes to a "primary" region; cross-region replication via Brooklin tails Kafka events for each index.
- Reads served locally; consistency is eventual.
- Failover: if a region's index is stale by > N minutes, route queries to a healthy region.

## 6.10 Failure modes

- **Shard outage**: serve with partial results, return a "we may be missing some content" flag to the UI. Better than a 5xx.
- **Ranking model failure**: fall back to retrieval-only ranking (BM25 + popularity).
- **Index corruption**: rebuild from offline Hadoop job; meanwhile shadow shard the affected partition.
- **Hot query**: cache top queries aggressively (a few hundred queries account for tens of % of traffic).
- **Adversarial / abuse**: rate-limit per IP; bot detection on query patterns.

## 6.11 Operational concerns

- **Schema evolution**: every new index field needs a deprecation path. Lucene schema changes typically require reindexing — expensive.
- **A/B testing**: ranking changes ramp via LIX; held to clicks-per-search, member-time-to-first-click, satisfaction metrics.
- **Capacity**: shard count growth tied to corpus growth. Online resharding is a beast — LinkedIn rebuilds offline + cuts over.
- **Cost**: index storage dominates; quantization, segment compaction, tiered storage (hot/warm/cold shards) all matter.

## 6.12 Things to say in the interview

- Acknowledge the **retrieval/ranking split** unprompted.
- Distinguish **typeahead** vs. full search vs. structured search vs. vector search; describe trade-offs.
- Estimate **storage** by doc count × avg doc size, with realistic compression.
- Discuss **freshness vs. cost** — NRT for important fields, batch for derived/cold.
- Multi-region — explicitly state which writes are primary-region vs. globally consistent.
- Bring up **vector search** — interviewers love that you can speak to AI surfaces.
- Bring up **explainability** for Recruiter — why was this candidate ranked here?

## 6.13 Common follow-ups

> **"How would you handle 100 million Boolean-faceted queries from a new abuse pattern?"**
Rate-limit per Recruiter seat; per-IP; pre-aggregated counts cached; if a single seat exceeds a threshold, soft-block with a CAPTCHA-like step-up.

> **"How would you migrate to a new ranking model?"**
Dark-launch with shadow scoring. Compare top-K overlap and clickthrough lift on a small ramp. Watch guardrails (latency, complaint rate, false-positive rate for spam). Promote incrementally.

> **"How do you decide to add a new facet for Recruiter?"**
Cost of indexing × query frequency × incremental conversion lift. Often gated on whether the facet improves Recruiter satisfaction without skewing toward a small power-user cohort.

> **"How would you serve a personalized embedding-based search for an AI feature like 'find me candidates similar to this one'?"**
Compute query embedding from the example candidate's profile; ANN search over candidate embeddings with filters; rerank with a cross-encoder; explain top features.