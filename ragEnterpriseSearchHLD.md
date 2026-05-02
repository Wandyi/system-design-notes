# RAG-Based Enterprise Search at MAANG Scale — Staff-Level HLD

How a 200 K-employee organization builds a search-and-answer system over PB-scale internal corpora — Confluence wikis, Slack threads, Google Drive documents, code repositories, design RFCs, Jira tickets, meeting transcripts — that returns *cited, ACL-respecting, sub-second* answers with bounded hallucination risk.

The deceptively simple story: "ingest, chunk, embed, retrieve, generate." The actual production system has 12 distinct services, three orthogonal scaling axes, four caching layers, two types of vector index, hybrid retrieval, ACL enforcement that can't be retrofitted, an evaluation pipeline that is its own product surface, and a cost model where the LLM bill is bigger than the rest of the platform combined.

---

## 1. Problem at MAANG Scale

### 1.1 The corpus shape

| Source | Volume | Mutation rate | ACL model |
|---|---|---|---|
| Google Docs / Confluence | 100s of millions of docs | High (new/edit/delete every minute) | Per-doc ACL, group-based |
| Slack | Trillions of messages | Stream | Per-channel; many channels per user |
| Code (GitHub/internal) | Billions of files | Continuous | Repo-level + path-level |
| Jira / Issues | Billions of tickets | High | Project ACL |
| Email / archival | Hundreds of TB | Append-only | Per-recipient |
| Meeting transcripts | TB+ daily | Stream | Per-meeting attendee |
| Wikis / runbooks | Millions of pages | Lower | Org/team |

Aggregate: **multi-PB corpus, 100s of millions of authoritative documents, billions of ephemeral messages, mutating constantly.**

### 1.2 Functional requirements

1. Natural-language Q&A: "What's the rollout plan for project Phoenix?" → an answer with citations to the source docs.
2. Filter facets: by source, author, date, team, project.
3. **ACL fidelity**: never surface a chunk to a user not authorized to see the source. Zero tolerance.
4. **Citations / grounding**: every claim links to the source span. No bare LLM output.
5. **Freshness**: doc edited at 10:00 should be findable by 10:01.
6. **Personalization**: the same query from a SRE vs a PM should bias toward different sources.
7. **Feedback**: thumbs up/down → rewires retrieval over time.
8. **Multi-modal** (longer term): images in design docs, diagrams in RFCs, code snippets.

### 1.3 Non-functional

| Dimension | Target |
|---|---|
| Query latency p50 / p99 | 150 ms / 300 ms (retrieval-only); 1.5 s / 4 s (with LLM stream-to-first-token) |
| Indexing freshness | < 60 s for high-priority sources |
| Availability (read) | 99.95 % |
| ACL correctness | 100 % (any false positive is a security incident) |
| Recall@10 | > 0.85 on internal eval set |
| Faithfulness (no hallucination) | > 0.95 (LLM outputs must be supported by retrieved context) |
| Cost per query (avg) | < $0.01 (with aggressive caching; LLM dominates) |

### 1.4 Why naive RAG fails at this scale

- **Pinecone with 1 B vectors single-tenant** → 10× cost vs sharded self-hosted; ACL filtering is per-query metadata scan, slow.
- **OpenAI embeddings for everything, all the time** → $millions / month on **re-embedding stable docs**.
- **Single cross-encoder reranker** → 100s of ms per query; tail-latency-dominated.
- **No hybrid retrieval** → semantic-only misses exact-match queries ("find the doc with config flag `enable_payment_v2`").
- **Post-filtering ACL** → returns 0 results when the user has narrow access; pagination breaks.
- **No eval harness** → you ship and never know if quality regressed.

---

## 2. Architecture Overview

```
                ┌────────────────────────────────────────────────────────┐
                │                       Ingestion Plane                  │
                │                                                        │
   ┌──────────┐ │  ┌────────────┐  ┌──────────────┐  ┌─────────────────┐ │
   │ Source   │ │  │ Connectors │→ │   Parser /   │→ │   Chunker /     │ │
   │ Systems  │─┼─▶│  (CDC)     │  │   Extract    │  │   Normalizer    │ │
   │ (Drive,  │ │  └────────────┘  │  (PDF, OCR,  │  └────────┬────────┘ │
   │  Slack,  │ │                  │   markdown,  │           │          │
   │  Code,   │ │                  │   code AST)  │           ▼          │
   │  Jira)   │ │                  └──────────────┘  ┌─────────────────┐ │
   └──────────┘ │                                    │ Embedding Svc   │ │
                │                                    │ (batched GPU)   │ │
                │                                    └────────┬────────┘ │
                │                                             ▼          │
                │  ┌─────────────────────────────────────────────────┐   │
                │  │  Storage Plane                                  │   │
                │  │  ┌────────────┐ ┌──────────┐ ┌────────────────┐ │   │
                │  │  │  Vector    │ │ Inverted │ │  Doc Store /   │ │   │
                │  │  │  Index     │ │ Index    │ │  Chunk Store   │ │   │
                │  │  │  (HNSW /   │ │ (BM25,   │ │  (S3 + meta-   │ │   │
                │  │  │   IVF-PQ)  │ │  ES)     │ │   data DB)     │ │   │
                │  │  └────────────┘ └──────────┘ └────────────────┘ │   │
                │  └─────────────────────────────────────────────────┘   │
                └──────────────────────────┬─────────────────────────────┘
                                           │
                                           ▼
                ┌────────────────────────────────────────────────────────┐
                │                      Query Plane                       │
                │                                                        │
                │   User → Query Gateway → Query Planner →               │
                │             ▼                  ▼                       │
                │     ACL Resolver         Hybrid Retriever              │
                │     (Zanzibar)         (Dense + Sparse)                │
                │             ▼                  ▼                       │
                │     ┌──────────────────────────────┐                   │
                │     │   Reranker (cross-encoder)   │                   │
                │     └──────────────────────────────┘                   │
                │                  ▼                                     │
                │     ┌──────────────────────────────┐                   │
                │     │   Context Assembler          │                   │
                │     │   (dedup, compress, cite)    │                   │
                │     └──────────────────────────────┘                   │
                │                  ▼                                     │
                │     ┌──────────────────────────────┐                   │
                │     │   LLM Inference Svc          │                   │
                │     │   (streaming + grounding)    │                   │
                │     └──────────────────────────────┘                   │
                │                  ▼                                     │
                │   Response → Cache → Feedback log → Eval pipeline      │
                │                                                        │
                └────────────────────────────────────────────────────────┘
```

### Component map

| Component | Responsibility | Storage / Tech |
|---|---|---|
| **Connectors** | Pull/CDC from source systems; preserve ACL metadata | Per-source service; Kafka outbound |
| **Parser** | Bytes → structured text (PDF, OCR, AST for code, markdown) | Stateless workers |
| **Chunker** | Split into retrievable units; carry parent linkage | Stateless workers |
| **Embedding Service** | Text → vector(s); batched on GPU pools | GPU clusters; mid-tier model self-hosted |
| **Vector Index** | ANN search over embeddings | HNSW (hot) + IVF-PQ on disk (cold) |
| **Inverted Index** | BM25 / lexical retrieval | Elasticsearch / OpenSearch |
| **Doc / Chunk Store** | Source-of-truth content + metadata | S3 + Postgres / Spanner |
| **ACL Service** | Resolve _user → accessible-doc-set_ | Zanzibar-style relation graph |
| **Query Planner** | Rewrite, expand, multi-route the query | Stateless |
| **Hybrid Retriever** | Dense + sparse, fused via RRF | Stateless |
| **Reranker** | Cross-encoder; second-stage scoring | GPU |
| **Context Assembler** | Build LLM prompt; dedup; ordering; citation tags | Stateless |
| **LLM Inference** | Generate grounded, cited answer | GPU pool; vLLM/TensorRT-LLM |
| **Cache** | Query, embedding, retrieval, generation | Redis (multi-cluster) |
| **Eval Harness** | Continuous offline + online quality measurement | Airflow + custom |
| **Feedback Loop** | User signals → retraining datasets | Kafka → DWH → fine-tune |

---

## 3. Schema Design

### 3.1 Document model

```sql
documents (
  doc_id          UUID PK,
  source_id       VARCHAR,            -- "google_drive:abc123"
  source_type     ENUM('drive','slack','confluence','code','jira','email',...),
  uri             TEXT,               -- canonical link back to source
  title           TEXT,
  author_id       UUID,
  team_id         UUID,
  created_at, modified_at, indexed_at,
  source_etag     VARCHAR,            -- monotonic per source-doc; drives idempotent re-index
  language        VARCHAR,
  is_deleted      BOOL,
  deletion_reason ENUM,               -- 'source_deleted','retention','user_request_gdpr'
  embedding_model_version VARCHAR,    -- which embedding model produced its chunks
  acl_signature   BYTES,              -- hash of (acl_principals, acl_version) for cache busting
  ...
)

chunks (
  chunk_id        UUID PK,
  doc_id          UUID FK,
  parent_chunk_id UUID NULLABLE,      -- for parent-child / hierarchical chunking
  position        INT,                -- ordering within doc
  start_offset    INT, end_offset INT,
  text            TEXT,               -- the actual snippet
  token_count     INT,
  semantic_role   ENUM('title','heading','body','code','table','caption'),
  embedding_id    UUID,               -- pointer to embedding row (separate store)
  bm25_doc_id     BIGINT,             -- inverted-index doc id
  acl_signature   BYTES,
  ...
)

embeddings (
  embedding_id    UUID PK,
  vector          VECTOR(d),          -- 384 / 768 / 1024 / 1536 dims by model
  model_version   VARCHAR,
  norm            FLOAT,              -- precomputed L2 norm; used by IVF
  ...
)

acl_index (
  doc_id          UUID,
  principal_id    UUID,               -- user or group id
  permission      ENUM('view','comment','edit','admin'),
  inherited_from  UUID,               -- the parent folder/space that granted this
  granted_at, expires_at,
  PRIMARY KEY (doc_id, principal_id)
)

acl_groups (
  group_id        UUID PK,
  members         SET<UUID>,          -- transitive group → user expansion
  parent_groups   SET<UUID>           -- group hierarchy
)

query_log (
  query_id        UUID PK,
  user_id         UUID,
  raw_query       TEXT,
  rewritten_query TEXT,
  retrieved_chunks UUID[],
  rerank_scores   FLOAT[],
  generated_answer TEXT,
  citations       JSONB,
  latency_breakdown JSONB,            -- {parse: 5, retrieve: 80, rerank: 60, llm_ttft: 200, llm_total: 1200}
  feedback        ENUM,
  ts              TIMESTAMP
)
```

### 3.2 Why these shapes

- **`source_etag`**: lets the ingest pipeline skip unchanged docs on re-pull, idempotently (we'll re-pull a doc 10×; we should embed it once).
- **Separate `embeddings` table**: lets us re-embed (new model) without rewriting `chunks`. The chunk just points at a new embedding_id.
- **`embedding_model_version`**: critical. Different models produce non-comparable vectors. Mixing → garbage retrieval. Every query must specify the version it wants and the index must filter accordingly during a migration.
- **`acl_signature`**: hash of the doc's ACL state. When ACL changes, signature changes, cached query results invalidated; chunks themselves don't move.
- **`parent_chunk_id`** + **`semantic_role`**: enables parent-child retrieval (small chunk for search precision, larger parent context returned for grounding).

### 3.3 Index data structures

| Index | What it stores | Where |
|---|---|---|
| **Dense vector index** | (chunk_id, embedding) | HNSW in RAM for hot tier; IVF-PQ on NVMe for warm; flat scan on object store for cold |
| **Inverted index** | tokens → posting lists per chunk | Elasticsearch/OpenSearch, sharded by tenant or source |
| **Doc/Chunk store** | full chunk text, metadata | S3 + Postgres/Spanner |
| **ACL index** | principal → reachable doc_ids | Zanzibar-style; queryable as "list user's docs" or "is this doc accessible?" |
| **Cache layer** | query → answer; chunk_id → embedding; query_text → result list | Redis cluster, sharded |

---

## 4. Ingestion Pipeline — In Depth

### 4.1 Connectors and CDC

Each source has a dedicated connector. Connectors must:
- Pull initial snapshot, then **stream changes** (Drive Push notifications, Confluence webhooks, Slack RTM/Events API, GitHub webhooks, Jira webhooks).
- Carry **ACL metadata** alongside content. Without this, ACL-correctness is impossible to recover later.
- Emit to Kafka with `(source_id, source_etag, content, acl_metadata, op = upsert | delete)`.
- **Idempotent re-ingest**: keying on `source_id + source_etag`, so replaying 10× changes nothing.
- Handle deletes properly. A "deleted from source" must produce a tombstone that propagates through the pipeline and removes the doc from indexes within the freshness SLO.

Schema-of-record at this stage: a normalized `RawDocument` envelope. The downstream pipeline does not care which source it came from.

### 4.2 Parsing

- **PDF** → text + structure (tables, sections, footnotes). Tools: Apache Tika, PyMuPDF, custom layout-aware parsers (LayoutLM-style for complex layouts).
- **OCR** for scanned PDFs and images: Tesseract, PaddleOCR, or commercial.
- **Code** → AST + symbol table; preserve language tags; chunk by function boundaries, not bytes.
- **Markdown / HTML** → preserve heading hierarchy; strip nav/sidebars.
- **Slack** → thread-aware: a message + its replies form a unit; cross-thread @-mentions are metadata.
- **Meeting transcripts** → speaker-segmented, timestamped.

Parsing is **expensive** (PDF OCR can be seconds per page) and **failure-prone**. Failures emit to a DLQ for human or model-assisted recovery; don't let one bad PDF block a batch.

### 4.3 Chunking — the highest-leverage decision

A single chunking strategy across all sources is wrong. Each content type needs its own.

#### 4.3.1 Strategies

**Fixed-size with overlap**: 512 tokens, 50-token overlap. Simple, decent baseline. Loses structure; cuts mid-sentence.

**Recursive character splitter**: split by `\n\n` → `\n` → `.` → ` ` greedily until under size limit. Used by LangChain. Keeps paragraphs together. Good default.

**Document-aware (markdown / HTML / code)**: split on heading levels and code blocks. A H2 section becomes a chunk if under size; otherwise recurse into H3s.

**Semantic chunking**: sliding window; compute embedding similarity between consecutive sentences; cut where similarity drops sharply. Better topical coherence; expensive — needs embeddings *during* chunking.

**Late chunking** (Jina, 2024): embed the *entire* document with a long-context encoder, then derive chunk embeddings from token-level outputs of that encode. Keeps cross-chunk context in each chunk's vector. State-of-the-art recall; cost is the long-context encoder.

**Parent-child / hierarchical**: small "search chunks" (~256 tok) point to larger "context chunks" (~1024 tok). Search the small ones; return the large ones to the LLM. Precision *and* context.

**Code chunking**: respect AST. A function (with signature + body + docstring) is one chunk; a class is parent of its methods; never split mid-function.

**Slack/conversation chunking**: a thread (parent + replies) is one chunk; if too long, sliding window over messages preserving speaker boundaries.

#### 4.3.2 What we actually do at MAANG scale

A **router** picks strategy per source/content-type:
- Confluence/Drive markdown → document-aware + parent-child.
- Code → AST.
- Slack → thread-aware.
- PDFs → recursive character + heading-aware where layout extracted.
- Mixed/unknown → recursive char baseline.

Chunk size sweet spot: **256–512 tokens for search, 1024–2048 for the parent context returned to the LLM.** Larger search chunks dilute embedding signal; smaller parent contexts starve the LLM.

### 4.4 Embedding service

#### 4.4.1 Model choice — the cost lever

| Tier | Model | Dim | Quality | Cost / 1M tokens |
|---|---|---|---|---|
| **Cheap, self-hosted** | E5-small / BGE-small / GTE-small | 384 | Good | ~$0.01 (your GPU) |
| **Mid, self-hosted** | BGE-large, E5-large, MPNet | 768–1024 | Strong | ~$0.05 |
| **Hosted SOTA** | OpenAI text-embedding-3-large, Cohere embed-v3 | 1024–3072 | Best | $0.13 (OpenAI) |
| **Domain-fine-tuned** | Custom on internal corpus | 768 | Best for the domain | One-time train + serving |

At 100 M docs × 5 chunks × 500 tokens = 250 B tokens. At OpenAI prices: **$32 K just to embed once**. At self-hosted BGE: ~$5 K (electricity + amortized GPU). Re-embed every 90 days for model upgrades: same again.

The right answer is usually:
- **Self-host a strong open model** (BGE-M3, E5-Mistral, NV-Embed) for the bulk of content.
- **Fine-tune** on internal Q&A pairs to lift recall by 5–15 points.
- **Reserve hosted SOTA** for edge cases or experiments.

#### 4.4.2 Matryoshka representation learning

Modern embedding models (OpenAI v3, Nomic, NV-Embed) support **Matryoshka**: the same vector is meaningful at 256, 512, 1024 dims. 
We store at full dim, *search* at lower dim for speed, fall back to full dim for reranking. ~3× search speedup with negligible recall loss.

#### 4.4.3 Batching and throughput

Embedding is GPU-bound. The service:
- Buffers requests up to 10 ms or 1024 inputs, whichever first.
- Pads to common length per batch (or uses dynamic batching à la vLLM).
- Pipelines tokenize → forward → write.
- Spikes go to autoscaled GPU pool; sustained load on dedicated pool.

Throughput: ~50–200 K embeddings/sec per A100 for small models. Backfill of a 100-M-doc corpus takes ~1–10 days on a 10–100 A100 cluster.

#### 4.4.4 Versioning

Every embedding row carries `model_version`. **Mixed-model search is silently broken** (vectors not comparable). Migration pattern:
1. Build new index in parallel ("dual-write": embed each new doc with both v1 and v2; old docs lazily re-embedded by a backfill job).
2. Shadow-traffic new index; compare quality on eval set.
3. Cutover atomically once quality threshold met.
4. Decommission v1 after monitoring window.

### 4.5 Indexing

- **Vector index**: HNSW for hot tier (in-RAM, low latency, 95–99 % recall). IVF-PQ on NVMe for warm (10× more capacity, 2× the latency, 1–2 points lower recall). Flat scan on object store for cold archival.
- **Inverted index** (Elasticsearch/OpenSearch): for BM25 baseline + lexical filters (`status:open`, `team:platform`, `created:[2024-01-01 TO *]`).
- **Sharding**:
  - **By tenant** (org/team) is rarely useful — most queries are organization-wide.
  - **By source** (`drive`, `slack`, etc.) helps query routing if planner can prune.
  - **By embedding-model-version** (mandatory during migrations).
  - **By time bucket** (recent vs archive) helps both freshness and tiering.
- **Replication**: 3× for hot, 2× for warm. Reads load-balanced across replicas.

---

## 5. Query Pipeline — In Depth

### 5.1 Query rewriting / expansion

- **Query rewriter** (small LLM): "Phoenix rollout plan" → "What is the rollout plan, schedule, and milestones for project Phoenix?". Improves recall on terse queries.
- **Multi-query expansion**: generate K variants; retrieve for each; union results. Costly (K× retrieval); used selectively.
- **HyDE (Hypothetical Document Embeddings)**: have an LLM draft a hypothetical answer; embed *that*; search with it. Often outperforms raw query embeddings on ambiguous questions. Costs one LLM call.
- **Decomposition**: complex questions → sub-questions; retrieve per sub-question; aggregate.

Trade-off: expansion improves recall but adds 100–500 ms and $$. Apply selectively (heuristic: short query, ambiguous intent → expand; long detailed query → don't).

### 5.2 ACL pre-resolution

The user has access to *some* docs. We resolve *which* before search:

```
accessible_set = ACL.docs_for_user(user_id, scope=current_org)   // Zanzibar query
```

Three implementation patterns, picked by ACL cardinality:

**(a) Pre-filter by ID set** — when accessible_set is small (< 100 K docs): pass as a `WHERE doc_id IN (...)` to retrieval. Vector index supports filtered ANN.

**(b) Indexed ACL bitmap** — when accessible_set is large but ACL groups are coarse: index per chunk a Roaring bitmap of authorized group IDs; 
                             query intersects user's group bitmap with chunk's at search time.

**(c) Post-filter** — only when the above fail. Search → fetch ACL → drop disallowed. Risk: retrieve 100 results, all 100 disallowed → 0 returned. 
                      Mitigate by oversampling 10×; still not robust.

In practice (b) is the common case: chunks tagged with the union of group-IDs that can read them; query has the user's resolved-group bitmap; intersection at search is cheap.

ACL **freshness** is a separate problem: when access is revoked, cached query results must invalidate. Pattern: query results keyed by `(query, embedding_model, user_id, acl_signature)` — the acl_signature changes when the user's resolved permissions change, busting the cache.

### 5.3 Retrieval — Hybrid (Dense + Sparse) with RRF

#### 5.3.1 Why hybrid

Pure dense retrieval misses exact-match queries: "find the doc with config flag `enable_payment_v2`" — the embedding doesn't strongly anchor on the rare token. BM25 nails it.

Pure sparse misses semantic queries: "how do we handle backpressure in the data plane?" — synonyms (rate limiting, throttling, flow control) escape BM25.

Hybrid wins both:
1. Run dense ANN: top-K_d (= 100) chunks.
2. Run BM25: top-K_s (= 100) chunks.
3. Fuse with **Reciprocal Rank Fusion**:
```
RRF_score(c) = Σ over rankers (1 / (k + rank(c)))
   k = 60 (constant; smooths)
```
4. Take top-N (= 30) from RRF.

RRF is parameter-light (no per-corpus tuning), robust, and the actual production default at most large RAG shops.

#### 5.3.2 Filtered ANN

Filters (`source:drive`, `team:platform`, `modified:>2024-01-01`) must apply *during* search, not after, or recall collapses. Vector indexes increasingly support metadata filters natively (Pinecone, Weaviate, Qdrant); for self-hosted HNSW, this is pre-filter via attribute index → filtered ANN.

For ACL specifically: see §5.2.

### 5.4 Reranking — the precision lift

The retriever returns 30 candidates; not all are great. A **cross-encoder** scores each `(query, chunk)` pair jointly:

```
for chunk in candidates:
    score = cross_encoder([query, chunk.text])
    
return top_K(candidates, by=score)
```

Cross-encoders (BGE-reranker, Cohere Rerank-3, Jina Reranker) are 10–100× slower per pair than embedding lookups (they re-encode the query and chunk *together*) — but they only run on a small candidate set, not the whole corpus.

**ColBERT / late-interaction** is the middle ground: per-token embeddings stored at index time; query computes max-sim against stored token embeddings. Faster than full cross-encoder, more precise than bi-encoder ANN. Storage is 10–50× of regular embeddings → use for high-precision verticals.

### 5.5 Context assembly

Top-K reranked chunks go to the LLM. Three problems to solve:

**(a) Deduplication**: chunks from near-duplicate docs (forks, copy-paste) inflate context with noise. Cluster by embedding similarity > 0.95; keep one per cluster.

**(b) Diversity (MMR — Maximal Marginal Relevance)**: sometimes you want top-K chunks that *cover different aspects* of the query, not the K most similar (which may all be the same point). MMR balances relevance and novelty.

**(c) Token budget**: LLM context is bounded (32 K – 200 K). Even with 200 K, more context isn't always better — "needle in haystack" performance degrades past ~30 K tokens for most models. Pick chunks that fit budget; sort by relevance then truncate; preserve cite-able boundaries.

The assembled prompt:
```
[system]   You are a helpful enterprise search assistant. Answer the user's
           question using ONLY the provided context. Cite each claim with
           [source N]. If the context is insufficient, say so.

[context]  [source 1] {chunk1.text}
           [source 2] {chunk2.text}
           ...

[user]     {query}
```

### 5.6 LLM generation

- **Streaming** to first token < 800 ms; stream tokens via SSE/WebSocket so user sees progress.
- **Function calling / structured output** for downstream usage (e.g., "return a JSON of cited claims").
- **Smaller model first** (e.g., Llama-3-70B on vLLM), escalate to GPT-4-class only if a gate flag triggers (low retrieval confidence, complex query).
- **Citation enforcement**: post-process LLM output to verify every `[source N]` references an actually-included context source. Strip unsupported citations.
- **Hallucination check** (optional): re-prompt a smaller model with "is this answer supported by these sources?" → flag unsupported claims for UI badging.

#### 5.6.1 Latency budget breakdown for p99 = 4 s end-to-end

```
parse query                   ~10 ms
ACL resolve                   ~20 ms (Zanzibar w/ cache hit)
query rewrite (optional)      ~150 ms (small LLM)
hybrid retrieval (dense+BM25)  ~80 ms
rerank top 30                  ~150 ms (cross-encoder, batched)
context assembly               ~20 ms
LLM time-to-first-token        ~600 ms
LLM stream completion         ~3000 ms (350 tokens at ~120 tok/s)
```

The LLM dominates. Every other component combined is < 500 ms.

---

## 6. Caching — Multi-Layer

| Cache | Key | Why | Invalidation |
|---|---|---|---|
| **Embedding cache** | hash(text) | Re-encoding the same text is wasted cycles | Permanent (text deterministic) |
| **Retrieval cache** | hash(rewritten_query, filters, acl_signature, model_ver) | Many users ask similar things; many dashboards trigger the same lookups | TTL or on-acl-change |
| **Rerank cache** | hash(query, chunk_id_set) | Reranker is expensive; result is deterministic | Permanent until model upgrade |
| **Generation cache** | hash(query, retrieved_chunk_ids, model_ver) | Same Q same context → same A | TTL (LLM is non-deterministic w/ temperature; cache for low-temp / official answers only) |
| **Result cache** (final) | hash(query, user_acl_signature) | End-to-end short-circuit | TTL ~ minutes; bust on doc updates |

Hit rates in production: embedding cache > 99 %, retrieval cache 30–60 %, generation cache 10–30 %. Each cache layer is in Redis with sharding by hash; some at edge (CDN) for the very-hot paths.

---

## 7. Bottlenecks — Identified and Addressed

| Bottleneck | Symptom | Mitigation |
|---|---|---|
| **Embedding compute** at backfill | 100M docs × 90-day re-embed cycle = continuous GPU saturation | Self-host strong open models; batch aggressively; tier embedding service; reuse embeddings via cache; **only re-embed on real model upgrade |**
| **Vector ANN tail latency** | p99 spikes when shard hot | HNSW per-shard; shard rebalancing; replica reads; `ef_search` tuning; warm-cache JVM/HNSW |
| **BM25 (Elasticsearch) query slow** | Wildcards, large facets | Force-merge segments overnight; eager-global-ordinals on hot facets; refresh interval ≥ 5 s |
| **ACL resolution** | Slow per-query Zanzibar walk | Cache user→group expansion in Redis; bust on group change events; use bitmap intersection |
| **Reranker latency** | 30 candidates × 50 ms = 1.5 s | Smaller reranker, batched on GPU; ColBERT for very-high QPS; cap candidates at 20 |
| **LLM TTFT** | First-token 1 s+ on cold | Self-host with vLLM/TensorRT-LLM continuous batching; prefix cache (system prompt + frequent context); speculative decoding |
| **LLM throughput** | tokens/sec doesn't scale linearly | Tensor + pipeline parallelism; quantization (FP8 / INT4); paged attention |
| **Long tail of slow sources** | One bad PDF blocks ingest | Per-source DLQ; ingest SLA per source; partial freshness OK |
| **Hot doc** (everyone querying the same RFC) | Replicated chunk read amplification | CDN at chunk-fetch tier; multi-tier cache |
| **Stale ACL** | User gets revoked, still sees cached results | Acl_signature in cache key; bust cache on revoke event |
| **Cost** | LLM bill 10× the rest | Smaller default model; cache more; **rerank harder so context is tighte**r; quantize self-hosted; A/B test smaller models on quality |

---

## 8. Scalability — Three Orthogonal Axes

### 8.1 Corpus size (PB → 10 PB)

- Tiered indexes (HNSW hot, IVF-PQ warm, flat-on-S3 cold).
- Time partitioning: queries default to last 2 years; older requires opt-in.
- Per-source sharding so source-filtered queries hit fewer shards.
- Compression: PQ for vectors (16× memory reduction vs FP32, ~2 % recall loss), zstd for chunk text.
- IPAM-equivalent: capacity-plan vector storage per region; reserve 20 % headroom.

### 8.2 Query QPS (1 K → 100 K)

- Stateless query plane scales horizontally.
- Read replicas for vector and inverted indexes.
- Aggressive multi-layer caching.
- LLM inference pool autoscales with continuous batching.
- Per-tenant rate limits (token bucket) prevent noisy neighbors.

### 8.3 Freshness (hours → < 60 s)

- CDC-driven streaming ingest (Kafka, not nightly batch).
- Chunk + embed on the hot path within 30 s of source change.
- Hot tier vector index supports incremental updates (HNSW does, with care; IVF requires periodic re-train).
- A "newly-arrived" merge tier accumulates recent vectors; merged into main hot tier every ~10 min.
- For the user's own writes: when a user creates a doc, prioritize that user's index update so they see it in their next search ("write-through").

---

## 9. Fault Tolerance and Availability

### 9.1 Failure domains

| Failure | Impact | Mitigation |
|---|---|---|
| Embedding GPU pool down | New docs not indexed | Buffer in Kafka; backlog drains when restored; freshness SLO violated, search still works on stale index |
| Vector index shard down | Recall drops on that shard's slice | 3× replication; shard remapping; RAM is the single largest recovery time |
| Elasticsearch shard down | BM25 leg of hybrid degrades | Hybrid still returns dense results; quality slightly down |
| LLM inference cluster overloaded | TTFT spikes | Queue with timeout → fallback to "retrieved snippets only, no synthesis"; degrade gracefully |
| Redis cluster down | All caching gone; cold-load on backends | Backends sized for cache-miss baseline (typically 3× cache-hit baseline); circuit break to "retrieval only, no LLM" if backend overloaded |
| ACL service down | Cannot serve any query (security-critical) | **Fail closed**: return error rather than risk leaking content. ACL replicas in 3 regions, multi-AZ; this is the most reliable subsystem |
| Region outage | Whole-region request loss | Active-active across regions; client routing via Route 53 / Global Accelerator; shared eventual-consistent index across regions |
| Connector dies | One source goes stale | Per-source SLA; alerting; per-source DLQ |

### 9.2 Read availability — graceful degradation

A query path with multiple stages → many places to fall back:

```
Try: full pipeline (rewrite → hybrid → rerank → LLM)
Fallback if rerank fails: skip rerank, use top-K of hybrid
Fallback if LLM fails: return ranked snippets without generation
Fallback if vector index degraded: serve BM25-only results with banner
Fallback if ACL service degraded: fail closed (no results, with explanation)
```

Each fallback is explicit code path with metrics. The user might see a degraded UX, but never wrong content (e.g., never bypass ACL).

### 9.3 Write durability — ingestion plane

- Source events buffered in Kafka with replication-factor 3, retention 7 days. Loss requires triple-AZ failure.
- Idempotent re-ingest by `source_id + source_etag` → safe to replay any window.
- Per-source ingestion checkpoint persisted in durable store; resume from last checkpoint on restart.
- Periodic full-source reconciliation: compare source's current state to our index; emit deltas. Catches missed events.

### 9.4 Multi-region

- **Indexes replicated** across regions. Each region serves locally for latency.
- **Eventual consistency** across regions: a doc edited in us-east-1 is searchable in eu-west-1 within ~60 s.
- ACL service is **strongly consistent globally** (regional reads of replicated state, with consistency tokens for "must observe at-or-after this ACL change") — fail-closed semantics demand it.
- LLM inference pools are regional (data residency may require it; latency definitely benefits).

---

## 10. Evaluation and Feedback Loop

A RAG system without an eval harness silently regresses. This is the part teams skip and regret.

### 10.1 Offline eval

- **Golden Q&A set**: 5–50 K human-curated `(query, expected_doc_ids, expected_answer)` triples spanning the corpus and intents.
- Metrics:
  - **Recall@K** at retrieval (did we fetch the right docs?).
  - **MRR / NDCG** at rerank (is the right doc in the top spot?).
  - **Faithfulness** (LLM-judge): is the generated answer supported by the retrieved context? Self-consistency: re-prompt and check stability.
  - **Answer correctness** (LLM-judge or human-judge against expected answer).
  - **Citation accuracy**: cited spans actually support the cited claims.
- CI gate: every model/index change must clear the eval set within ε before merging.

### 10.2 Online eval

- **Click-through and dwell time** as implicit feedback.
- **Thumbs up / down** explicit.
- **Cite-clicks**: did the user open a citation? Strong signal of trust.
- **Re-query within 30 s**: dissatisfaction signal.
- A/B framework with significance testing across these signals.

### 10.3 Feedback into the loop

- Negative-feedback queries → into eval set.
- Positive-pair (query, top-clicked-source) → into fine-tuning data for embedding model and reranker.
- Quarterly retraining: small but reliable lift; biggest lift comes from domain-specific fine-tunes on this organic data.

### 10.4 Hallucination defense

- **Retrieval gating**: if reranker top score is below threshold, tell the LLM "context likely insufficient" — and the LLM is prompted to admit that rather than fabricate.
- **Citation-required answers**: any sentence in the answer without a citation is flagged.
- **Self-RAG / reflection**: after generation, prompt the model to check each claim against the retrieved chunks. Flag mismatches.

---

## 11. Cost Model

Rough cost per 1 M queries (large enterprise self-host with hosted-LLM):

| Component | Cost / 1M queries |
|---|---|
| Embedding (90 % cached) | $5 |
| Vector search (self-host HNSW, 3 replicas) | $30 |
| BM25 (Elastic) | $20 |
| Reranker (self-host) | $80 |
| LLM (mix of self-host Llama-3 + hosted GPT-4 fallback) | $2 000 – $8 000 |
| Cache (Redis cluster) | $10 |
| Storage (chunks + indexes) | amortized |
| **Total** | $2 100 – $8 200 |

The LLM **dominates** by 1–2 orders of magnitude. The cost-reduction levers in priority order:
1. **Cache more aggressively** (every 1 % cache-hit lift = 1 % LLM bill cut).
2. **Smaller default model**, escalate selectively.
3. **Better retrieval** — tighter context = fewer tokens = cheaper LLM call.
4. **Self-host quantized models** for fallback paths.
5. **Batch low-priority queries** on slower paths.

---

## 12. Anti-Patterns

- **One vector index for all sources** — sharding mistakes; can't tune retrieval per source.
- **Embedding the whole corpus once and forgetting** — model upgrades, schema changes; you'll re-embed.
- **No `embedding_model_version`** in chunks — silent corruption when migrating.
- **Post-filter ACL** — leaks count, breaks pagination, security risk on edge cases.
- **Aggressive query rewriting on every query** — latency tax for marginal recall gain.
- **Reranking 100s of candidates** — 90 % of the gain is in the first 20.
- **No eval harness** — quality regresses silently; team blames "the LLM."
- **LLM with 200 K context, dump-everything-in** — needle-in-haystack collapse; expensive; tighter retrieval is better.
- **Cache by raw query string** — case-sensitivity, whitespace, ACL ignored. Always canonicalize.
- **Same embedding for query and document** — many models have asymmetric instructions ("query: ..." vs "passage: ..."). Get this wrong and recall craters.
- **Single-tenant Pinecone for everything** — cost explodes; switch to sharded self-host past ~10 M vectors.
- **Trust the LLM for citations** — must be programmatically validated against the actually-retrieved chunk IDs.
- **No streaming** — UX feels like a 4-second hang.
- **Treating Slack like documents** — threads are conversations; chunking and ranking models must reflect that.

---

## 13. Trade-Offs Worth Defending

| Decision | Alternative | Why we picked this |
|---|---|---|
| **Hybrid (dense + sparse + RRF)** | Dense-only | RRF handles exact-match queries that dense retrieval misses; parameter-free fusion |
| **Cross-encoder reranker** | Bi-encoder only | 10–20 point precision lift on top-10; only on small candidate set, so cost is bounded |
| **Self-host strong open embedding model** | Hosted SOTA | 10× cheaper at scale, fine-tunable on domain; trade ~3–5 points of generic recall for fine-tune lift |
| **Parent-child chunking** | Flat chunks | Search precision (small) + LLM context (large); only real cost is one extra fetch |
| **Pre-resolved ACL + signature in cache key** | Post-filter ACL | Correctness + cache-friendliness; ACL subsystem becomes the most reliable component, intentionally |
| **Streaming LLM** | Wait-for-complete | UX win; user sees progress at 800 ms instead of 4 s |
| **Self-host LLM with GPT-4-fallback** | All hosted | 5–10× cost reduction on the bulk; fallback covers quality edge cases |
| **Multi-region active-active** | Single region | Latency + DR; cost is replication overhead, justified at MAANG scale |
| **Eval harness in CI** | Ship and watch | Catches regressions before customers do; non-negotiable |
| **Citation-required generation** | Free-form LLM | Hallucination defense; if model can't cite, it must admit insufficient context |
| **Per-source chunkers** | One generic chunker | 10–20 point recall lift on code/Slack/PDFs over generic |
| **Matryoshka embeddings + reduced search dim** | Full-dim search | 3× speed; recall loss imperceptible |
| **CDC ingest** | Nightly batch | < 60 s freshness; cost is per-source connector engineering |
| **Per-tenant rate limit** | Fair-share scheduling | Simpler; noisy neighbor isolation; trade off some efficiency |

---

## 14. What Makes This Staff-Level

1. **Naming the failure modes of naive RAG up front** (post-filter ACL, single embedding model, no eval harness) and addressing each with a *structural* solution, not a checkbox.
2. **ACL fidelity treated as a first-class architectural concern** — pre-resolved, signature-cached, fail-closed; not a metadata tag bolted on at the end.
3. **Hybrid retrieval (dense + sparse + RRF)** as a default, not a tuning experiment.
4. **Two embedding tiers** — small/cheap for bulk + cross-encoder rerank for precision — instead of either bi-encoder-only or cross-encoder-everywhere.
5. **Embedding model versioning** baked into the schema; migration patterns documented; no implicit cross-version search.
6. **Latency budget broken down per stage** — knowing the LLM owns 80 % of the budget changes which optimization you reach for.
7. **Eval harness elevated to first-class** — golden set, faithfulness checks, online + offline, CI gate.
8. **Cost model with concrete dollar figures** — LLM dominates by 100×; optimizations ranked by impact, not ease.
9. **Graceful degradation paths** explicitly enumerated for every subsystem failure — including "ACL fails closed."
10. **Multi-layer cache architecture** — embedding > retrieval > rerank > generation > result; each with documented hit-rate expectations and invalidation rules.
11. **Schema choices defended** — `source_etag` for idempotent re-ingest, `acl_signature` for cache busting, separate `embeddings` table for model migrations.
12. **Anti-patterns named** — the things that look like obvious wins but ship regressions: unbounded context, rerank-100, asymmetric query/passage embeddings, single-tenant vector DB.

The deeper insight: enterprise RAG at MAANG scale isn't an LLM application — it's a **search and retrieval system that happens to use an LLM as the answer formatter**. The 80 % of engineering effort is in the parts that aren't the LLM: ingestion fidelity, ACL correctness, hybrid retrieval, reranking, evaluation, and caching. Get those right and the LLM is the thin top layer; get them wrong and no LLM saves you.