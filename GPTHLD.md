# High-Level Design: ChatGPT-Style LLM Application

Staff-level HLD for building a production LLM chat platform at ChatGPT scale — 100M+ MAU, 10M+ concurrent sessions, 1B+ conversations/day. Focus on **scalability**, **availability**, and **fault tolerance** across the unique challenges of serving LLM inference.

---

## 1. Requirements

### Functional
- User signup/auth, chat sessions with multi-turn history, streaming token responses.
- Multiple models (GPT-4-class, GPT-3.5-class, fast/cheap vs slow/smart).
- File uploads (images, PDFs, code) with multimodal support.
- Web search / tool use / code execution.
- Conversation persistence, share links, memory across sessions.
- Rate limits per user tier (free, plus, team, enterprise).
- Content moderation (input + output).

### Non-Functional
- **Availability**: 99.95% (≤22 min downtime/month).
- **Latency**: Time-to-first-token (TTFT) P95 < 800 ms; inter-token P95 < 50 ms.
- **Throughput**: 100K+ concurrent streaming sessions per region.
- **Scale**: 10B+ tokens generated/day; petabytes of conversation history.
- **Cost**: GPU inference dominates — every optimization matters.
- **Compliance**: SOC 2, GDPR (data residency, deletion), enterprise data isolation.

---

## 2. Top-Level Architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                        Clients (web/mobile/API)                 │
  └─────────────────────────────────────────────────────────────────┘
                                │
                     (TLS, HTTP/2, SSE/WebSocket)
                                ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │   CDN / Edge (CloudFront + POPs) — static assets, TLS term      │
  └─────────────────────────────────────────────────────────────────┘
                                │
  ┌─────────────────────────────────────────────────────────────────┐
  │   Global LB (Anycast) → regional API Gateway (rate limit, auth) │
  └─────────────────────────────────────────────────────────────────┘
                                │
  ┌───────────────┬────────────┬────────────┬────────────┬──────────┐
  │ Auth Service  │ Chat API   │ Files API  │ Tools API  │ Admin    │
  └───────────────┴────────────┴────────────┴────────────┴──────────┘
                                │
                                ▼
           ┌──────────────────────────────────────────┐
           │   Orchestrator / Agent Runtime           │
           │   (prompt build, tool calls, streaming)  │
           └──────────────────────────────────────────┘
                                │
            ┌───────────────────┼────────────────────┐
            ▼                   ▼                    ▼
    ┌──────────────┐    ┌──────────────┐    ┌────────────────┐
    │ Inference    │    │ Retrieval    │    │ Tool Executors │
    │ (GPU fleet)  │    │ (vector + kv)│    │ (code, web,..) │
    └──────────────┘    └──────────────┘    └────────────────┘
            │                   │                    │
            ▼                   ▼                    ▼
    ┌──────────────────────────────────────────────────────┐
    │   Data plane: Postgres (metadata), S3 (blobs),       │
    │   Cassandra/DynamoDB (conversations),                │
    │   Pinecone/pgvector (embeddings), Redis (sessions),  │
    │   Kafka (events/analytics), ClickHouse (analytics)   │
    └──────────────────────────────────────────────────────┘
```

---

## 3. API Surface (Abbreviated)

```
POST   /v1/auth/login             → session_token (JWT)
POST   /v1/chat/completions       → SSE stream { delta: "token" } ...
  Body: { model, messages, tools?, stream: true, temperature, ... }
POST   /v1/conversations          → { conversation_id }
GET    /v1/conversations/{id}     → full message history
POST   /v1/conversations/{id}/messages
POST   /v1/files                  → presigned S3 upload; returns file_id
GET    /v1/models                 → available models + pricing/quotas
POST   /v1/feedback               → thumbs up/down on message_id
```

**Streaming contract**: Server-Sent Events over HTTP/1.1 or HTTP/2 (not WebSocket — unidirectional server→client fits better; less infra). Each event is `data: {"delta": "...", "finish_reason": null}\n\n`.

---

## 4. The Inference Layer (The Hard Part)

### Why LLMs are different from "normal" services

| Property | Regular service | LLM inference |
|---|---|---|
| Request duration | 10–500 ms | **10–60 seconds (streaming)** |
| Unit of work | CPU cycles | **GPU tokens** |
| Memory pattern | Heap/cache | **KV cache grows per-token per-session** |
| Scaling unit | Threads/pods | **GPU batches + KV memory** |
| Cold start | Milliseconds | **Model load: 10–60 seconds** |
| Cost per request | fractions of a cent | **Dollars** |

### Model serving stack

- **Inference engine**: vLLM, TensorRT-LLM, or SGLang — not raw PyTorch. These implement:
  - **Continuous batching** (iteration-level scheduling): instead of fixed batch-per-step, new requests join mid-generation. 10–20× throughput improvement over naive batching.
  - **PagedAttention**: treats KV cache like virtual memory (pages), eliminates fragmentation, enables higher batch sizes on same GPU memory.
  - **Speculative decoding**: draft model generates N tokens, target verifies in parallel. 2–3× latency improvement.
  - **Quantization**: FP16 → INT8/INT4 → FP8. Modest quality loss for 2–4× throughput.

- **Model sharding across GPUs**:
  - **Tensor parallelism** (TP): split weights across GPUs within a node (NVLink). For big models that don't fit one GPU.
  - **Pipeline parallelism** (PP): layers across nodes. Higher latency (bubble) but enables massive models.
  - **Expert parallelism** (EP): for MoE models (Mixtral, GPT-4 rumored MoE) — different experts on different GPUs.

- **Hardware topology**: H100/H200 SXM with NVLink for TP, InfiniBand between nodes. Your scheduler must be topology-aware — scheduling a TP group across non-NVLink-connected GPUs costs 10× bandwidth.

### Routing + Load Balancing

This is where most teams get it wrong. **GPUs are stateful during a stream** — you can't round-robin.

- **Session affinity for active streams**: once a stream starts on GPU pod A, all subsequent tokens go there. Pod A holds the KV cache for that conversation.
- **Cache-aware routing for new requests** (SGLang RadixAttention, vLLM prefix cache): if two conversations share a long system prompt, route both to the same replica so the prompt's KV cache is reused. This cuts prefill compute by 50–90% for chat-heavy workloads.
- **Load signals**: don't use CPU — use **running batch size** + **KV cache utilization** + **queue depth** per replica. Inference engines expose these via metrics.
- **Multi-tier fleet**: route by model size + tier.
  - Tier 1: biggest model, most expensive GPUs (H100), premium users.
  - Tier 2: distilled/quantized model on A100s, free tier.
  - Tier 3: small models (7B) on L4/T4 for "quick reply" features and classification.

### Autoscaling GPU fleets — why it's painful

- **Cold start is 10–60s**: loading a 70B model from S3 into GPU memory. HPA reactively scaling is too late — the spike is over before pods are Ready.
- **Solutions**:
  1. **Predictive scaling** on historical patterns (traffic correlates with business hours by region).
  2. **Warm pools**: spare GPUs with model loaded, idle. Expensive but necessary for tail availability.
  3. **Model streaming + fast loaders** (Safetensors mmap, Run:AI model streamer): start serving before full load completes.
  4. **Shared model weights** via read-only volume (EFS/FSx Lustre) — pods mount instead of pulling from S3.
- **Scale down carefully**: a pod with active streams can't just terminate. Drain = stop accepting new requests, wait for in-flight streams to finish (up to conversation timeout, typically 5 min).

### Fault tolerance during streaming

- **Resume tokens**: each streamed chunk carries an opaque cursor. On disconnect, client reconnects with cursor; orchestrator can re-attach to the same inference pod (if alive) or regenerate from conversation history (if not).
- **Graceful GPU failure**: OOM or CUDA error kills the pod. Orchestrator detects via timeout/503, falls back to a different replica. User sees a brief stall, then resumes — ideally the client auto-retries the last delta.
- **Partial-response saving**: every N tokens, orchestrator persists partial message to DB. If the stream dies after 500 tokens, the user doesn't lose them.

---

## 5. Orchestration / Agent Runtime

The orchestrator is the brain between the API and the GPU.

### Responsibilities
1. **Prompt assembly**: system prompt + conversation history + retrieved context + tool schemas.
2. **Context window management**: if conversation exceeds model context, summarize or sliding-window.
3. **Tool use loop**: parse model's tool call → execute (with timeout) → inject result → re-invoke model.
4. **Streaming passthrough**: forward tokens from inference to client, with logging + safety filters inline.
5. **Budget enforcement**: per-request token budget, per-user daily budget, cost-based circuit breakers.

### Design
- **Stateless orchestrator pods**, horizontally scaled. Conversation state lives in Redis + durable DB.
- **Async/event-loop runtime** (Go or Python asyncio) — a single pod holds 1000s of streams concurrently (one goroutine + one HTTP connection per client).
- **Backpressure**: if downstream GPU is saturated, orchestrator queues or rejects with 503 + Retry-After. Don't buffer indefinitely.

### Tool calling fault tolerance
- **Every tool has a timeout** (web search 10s, code exec 30s). Model is told "tool timed out; respond without result."
- **Tool failures shouldn't fail the response** — inject an error message as tool output and let the model explain to the user.
- **Idempotent tools or de-duped side effects** — model may retry; calling `send_email` twice is bad.

---

## 6. Conversation & Data Storage

### Conversations
- **Primary store**: DynamoDB or Cassandra. Partition key = `user_id`, sort key = `conversation_id#message_id`. Reasons:
  - Infinite horizontal scale.
  - Writes are append-only (new messages); no updates on old messages.
  - Range queries for "fetch conversation N–M".
- **Hot path cache**: Redis for the last 20 messages of each active conversation. Reduces DB reads on each turn.
- **Long-term archival**: S3 Glacier for conversations older than 1 year (for users who opt in to training / retention).

### Files
- **Blob storage**: S3 / GCS. Presigned URLs for direct client upload (bypass API for bandwidth).
- **Extracted content**: text from PDFs, OCR from images — stored alongside as separate S3 objects for re-use.
- **Multimodal embeddings**: pre-computed and stored in vector DB on upload.

### Metadata
- **Postgres** (Aurora) for: users, accounts, billing, rate-limit quotas, feature flags, model catalog. Small, relational, strongly consistent.

### Vector / Retrieval
- **pgvector** for small-scale RAG (per-user memory), **Pinecone/Weaviate/Qdrant** for large-scale (billions of vectors).
- **Hybrid search**: BM25 + vector. Pure vector misses exact-match queries ("error code 7823").

### Analytics
- **Kafka** → **ClickHouse** for usage analytics, model performance, A/B tests. Never write analytics to the OLTP store.

---

## 7. Scalability Strategy

### Per-dimension scaling

| Dimension | Bottleneck | Mitigation |
|---|---|---|
| Concurrent streams | Orchestrator connections | Go/async runtime, horizontal scale |
| Tokens/sec | GPU fleet | Continuous batching, quant, more GPUs |
| Conversation writes | DB hot partitions | Partition by user_id, Redis write-through |
| Context length | KV cache memory | PagedAttention, prompt caching, sliding window |
| Prompt prefix reuse | Cold prefill cost | Prefix/radix cache + cache-aware routing |
| File uploads | API bandwidth | Presigned S3 URLs, client→S3 direct |
| Rate limiting | Centralized counter contention | Token bucket in Redis per-user, approximate at edge |

### Multi-region

- **Active-active across 3+ regions** (us-east, us-west, eu-west, ap-southeast).
- **Latency-based routing** at the global LB.
- **Conversation data**: eventually consistent replication across regions (DynamoDB Global Tables / Cassandra multi-DC). A user who hops regions might briefly see stale history — acceptable.
- **GPU fleets are regional** — don't cross-region inference (latency kills streaming UX).
- **Data residency**: EU users pinned to EU region for GDPR. Route53 geo-routing + hard user-region binding at signup.

---

## 8. Availability & Fault Tolerance

### Graceful degradation hierarchy

When things break, degrade rather than error:

1. **Primary model unavailable** → fall back to smaller/cheaper model, warn user "using backup model".
2. **Retrieval/RAG down** → proceed without context (noticeably worse answers, but works).
3. **Tool executor down** → model responds without tool use.
4. **Specific GPU pod dies mid-stream** → client auto-retry, orchestrator re-runs from partial history.
5. **Entire region down** → DNS/Anycast routes to another region; active streams are lost but new ones succeed within ~60s.

### Key SLOs + error budgets

- **API availability**: 99.95% (22 min/mo).
- **TTFT P95**: 800 ms — budget burn triggers capacity review.
- **Stream completion rate**: 99.9% — a stream started must complete or meaningfully error (not just vanish).
- **Cost per 1K tokens** — track as reliability signal; a regression often indicates a caching or routing bug.

### Incident-class failure modes we've designed for

- **Cascading GPU OOM**: one model variant has a memory leak → pods OOM → HPA adds more → they OOM → fleet depletes.
  - **Defense**: per-pod memory limits + liveness probe + circuit breaker at orchestrator to stop sending to leaking replica.
- **Prompt-bomb DoS**: user sends 1M-token prompt repeatedly.
  - **Defense**: input token limits enforced at API gateway (not orchestrator — too late), per-user concurrent request cap.
- **Retry storm on outage**: GPU fleet recovers → every client retries simultaneously → re-OOMs.
  - **Defense**: exponential backoff + jitter enforced at API layer, Retry-After headers, token bucket shedding at edge.
- **Runaway agent loop**: model calls tool, tool result triggers another tool call, infinite.
  - **Defense**: max tool-call depth (usually 10), hard token budget per request.
- **Bad model deploy**: new model version degrades quality or costs.
  - **Defense**: canary with shadow traffic, online eval (user feedback, coherence metrics), automatic rollback on SLO burn.

### Safety / Moderation (reliability-adjacent)

- **Input moderation**: classifier (often a small model) on every prompt. Hard-block CSAM, weapons-of-mass-destruction prompts, etc.
- **Output moderation**: streaming classifier on output tokens — can abort mid-stream if unsafe content emerges.
- **Prompt injection defense**: treat tool outputs and retrieved docs as untrusted; wrap in clear delimiters, instruct model accordingly. Not a perfect defense — assume breaches and restrict tool permissions.

---

## 9. Cost Architecture (Because Inference Dominates the P&L)

- **Model tiering**: free users → quantized 8B model; plus → 70B; enterprise → full 400B.
- **Prompt caching** (Anthropic-style ephemeral cache): cache long system prompts / docs for 5 min TTL. 90% discount on cached tokens for chat apps with long system prompts — this alone can cut costs in half.
- **Cache-aware routing** (discussed in §4): ensure prompt cache hits are actually realized.
- **Batch API** for non-latency-sensitive work (summaries, embeddings, analytics re-processing): 50% discount.
- **Speculative decoding** with a 7B draft model generating for a 70B target: 2× speedup, 40–50% cost reduction.
- **Embeddings and small classifiers on CPU** — don't burn H100 for a 100M-param moderation model.
- **Reserved GPU capacity** for baseline, spot/preemptible for burst (with checkpointing for training, not inference).

---

## 10. Observability

### What to measure (beyond "CPU + latency")

- **TTFT P50/P95/P99** per model, per region, per tier.
- **Inter-token latency (ITL)**: variance matters — users feel jitter as "stuttering".
- **Stream completion rate**: started / completed / errored / timed out.
- **Batch fill ratio** per GPU: low = wasted capacity, high = good utilization but watch queueing.
- **KV cache utilization + evictions**.
- **Prompt cache hit rate**.
- **Tokens/sec per GPU** — primary capacity KPI.
- **Cost per conversation / per user / per feature** — attribution.

### Tooling
- **Metrics**: Prometheus + Thanos or Mimir (multi-cluster) → Grafana.
- **Logs**: OpenTelemetry, sample heavily (full logs of every prompt = PII + cost nightmare).
- **Traces**: OpenTelemetry with tail-based sampling for errors and slow requests.
- **Quality evals**: continuous online evals — thumbs up/down, regression suites, adversarial prompts. Treat quality as a first-class SLO.

---

## 11. Deployment & Release

- **GitOps** (ArgoCD) across 4+ Kubernetes clusters, one per region.
- **Progressive rollout**:
  - Model changes: shadow-traffic canary → 1% → 5% → 25% → 100%, with quality eval gates at each step.
  - Code changes: blue-green or rolling with readiness probes tuned for model load time.
- **Feature flags** for client behavior (new UI, new models) — detached from deploys.
- **Automated rollback** on SLO burn or eval regression.

---

## 12. Security & Compliance

- **mTLS between services**, via service mesh (Linkerd/Istio).
- **Per-tenant encryption keys** (enterprise tier) via KMS.
- **Zero training on enterprise data** — enforced via tagged data paths; audit logged.
- **GDPR "right to be forgotten"**: delete pipeline across OLTP, vector store, backups, and training data pools. Tracked via a deletion-request log — harder than it sounds because of derivative data.
- **Rate limiting**: per-IP, per-user, per-API-key. Redis token bucket at edge.
- **Abuse detection**: classifiers on prompts (CSAM, spam, scraping patterns) + behavioral signals (sudden high volume from new account).

---

## 13. The 10 Hard-Won Staff-Level Insights

1. **LLM serving is nothing like stateless web serving**. Throw out your reflexes — a request is 30 seconds, state lives on a GPU, batching is everything.
2. **KV cache is the asset**. Route to maximize cache reuse; evict wisely; don't waste it.
3. **Continuous batching + PagedAttention are not optional** at scale — use vLLM/TRT-LLM/SGLang, don't roll your own.
4. **Warm capacity is expensive but necessary**. Accept the cost of idle GPUs for tail availability.
5. **Streaming disconnects are the norm, not the exception** — design resume/partial-save into the data path, not as an afterthought.
6. **Graceful degradation > error pages**. Falling back to a cheaper model beats 500ing the user.
7. **Prompt caching is the single highest-leverage cost optimization** for chat workloads with stable system prompts.
8. **GPU autoscaling must be predictive**, reactive is too slow. Warm pools + scheduled scale-up for known traffic patterns.
9. **Tool use is a distributed system inside your request** — timeouts, idempotency, circuit breakers per tool.
10. **Quality is a reliability property**. A cheap deploy that regresses quality is as bad as an outage — wire evals into the release gate.

---

## Quick Decision Reference

| Problem | Reach for |
|---|---|
| TTFT too high | Prompt caching + speculative decoding + smaller model for prefill |
| ITL jitter | Disable CPU limits on inference pods, check batch queueing |
| GPU under-utilized | Higher max batch size, check continuous batching config |
| GPU OOM mid-stream | PagedAttention, stricter max_tokens, KV eviction policy |
| Cold-start pain | Warm pools + model streaming + shared weights volume |
| Cross-region latency | Pin user to home region, don't cross-region inference |
| Cost regression | Check prompt cache hit rate, model tier routing, batch fill |
| Retry storms | Retry-After + jitter + edge shedding + fleet circuit breakers |
| Bad deploy quality | Shadow traffic + online evals + auto-rollback |
| GDPR delete | Tag data lineage from ingestion; deletion pipeline with audit log |