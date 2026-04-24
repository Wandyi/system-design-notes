# HLD: ChatGPT-Scale LLM Application — Deep Dive with Corner Cases

A staff-level high-level design for a ChatGPT-class product, rewritten to explicitly enumerate **failure domains**, **race conditions**, and **edge cases** at every layer. If the first version is the architecture, this is the architecture review where a staff engineer asks "but what happens when…"

Scale assumptions: 100M+ MAU, 10M+ concurrent sessions, 1B+ conversations/day, 10B+ tokens/day, petabytes of conversation history, multi-region active-active.

---

## Table of Contents

1. Requirements & Error-Taxonomy-First Design
2. Architecture & Failure Domains
3. Client Layer — Corner Cases
4. Edge / CDN / Global LB
5. Auth, Identity, and Session
6. API Gateway & Rate Limiting
7. Orchestrator / Agent Runtime
8. Inference Layer (Where It Gets Real)
9. Retrieval / RAG
10. Tool Use / Code Execution
11. Storage Layer
12. Multi-Region, Data Residency, Failover
13. Autoscaling & Capacity
14. Billing, Quotas, Fair-Use Enforcement
15. Safety / Moderation
16. Security & Abuse
17. Privacy, Deletion, Legal Hold
18. Observability & SLOs
19. Deployment & Release Engineering
20. Cost Engineering & Corner Cases
21. Testing: Chaos, Load, Quality Evals
22. Runbooks for Classic Incidents
23. Summary Decision Matrix

---

## 1. Requirements & Error-Taxonomy-First Design

### Functional

- Auth: email/OAuth/SSO/SAML; API keys for programmatic.
- Chat: multi-turn, streaming tokens, resumable, editable messages, regenerate, stop.
- Multiple models: small/fast, large/smart, multimodal (image/audio/video), code-specialist.
- Files: upload (image/PDF/code/audio), processed as attachments, referenced in prompts.
- Tools: web search, code execution (sandboxed), image generation, user-defined functions.
- Memory: per-conversation context + cross-conversation long-term memory.
- Sharing: shareable conversation links, read-only public snapshots.
- Teams/Enterprise: workspaces, admin policies, DLP, audit logs, data-retention controls, no-training guarantees.
- Mobile, web, desktop clients; public API; client SDKs.

### Non-Functional

- **Availability**: 99.95% (22 min/mo). Separate SLO per tier: Enterprise 99.99%.
- **Latency**: TTFT P50 < 400 ms, P95 < 800 ms, P99 < 1500 ms. ITL (inter-token latency) P95 < 60 ms.
- **Stream completion rate**: ≥ 99.9% (a stream that starts must finish or meaningfully error).
- **Durability**: conversation history 11 9s (S3-class). Partial messages saved every N tokens.
- **Throughput**: 100K+ concurrent streams per region.
- **Data residency**: EU, US, UK, IN, AU enclaves (enterprise-configurable).
- **Compliance**: SOC 2 Type II, HIPAA (BAA tier), GDPR, CCPA, ISO 27001.

### Design from the error taxonomy backwards

Before architecture, define the vocabulary of failure, because every service in the stack must agree on it:

| Class | When | User-visible |
|---|---|---|
| `invalid_request` | Bad input | 400, clear message, no retry |
| `authentication_failed` | Bad creds | 401, re-auth |
| `permission_denied` | Valid creds, no access | 403, no retry |
| `rate_limited` | Quota exceeded | 429 + Retry-After, retry with backoff |
| `context_length_exceeded` | Too many tokens | 400, surface truncation option |
| `content_policy_violation` | Safety filter | 400, with policy reference |
| `model_overloaded` | Capacity pressure | 503, retry with backoff |
| `upstream_timeout` | Tool/retrieval timeout | partial response, marker in stream |
| `internal_error` | Our bug | 500, retry once, log + alert |
| `service_unavailable` | Regional degradation | 503, client retries in different region |
| `stream_interrupted` | Stream ended before completion | client resumes via cursor |

**Why this matters**: clients, SDKs, observability, retry policies, and billing all key off these. An inconsistent error taxonomy is an operational bug that bleeds for years.

---

## 2. Architecture & Failure Domains

```
         ┌─────────────────────────────────────────────────────────┐
         │                       CLIENTS                           │
         │  web • iOS • Android • desktop • public API • SDK       │
         └──────────────────────┬──────────────────────────────────┘
                                │ TLS / HTTP2 / SSE / WebSocket
   ┌──────────────── Edge (CloudFront / Cloudflare) ───────────────┐
   │ • TLS term, TLS 1.3, HTTP/3, OCSP stapling                    │
   │ • Bot detection, WAF, DDoS scrubbing                          │
   │ • Static asset cache, SSR shell cache                         │
   └──────────────────────┬────────────────────────────────────────┘
                          │
   ┌──────── Global LB (Anycast / Global Accelerator) ─────────────┐
   │ • Geo + latency routing                                       │
   │ • Regional health checks                                      │
   │ • Failover (60–120s DNS; ≤30s with Anycast)                   │
   └──────────────────────┬────────────────────────────────────────┘
                          │
   ┌────────────────────  REGION  ─────────────────────────────────┐
   │                                                               │
   │   ┌── API Gateway / L7 proxy (Envoy/Istio) ──┐                │
   │   │ • Auth (JWT verify, API key lookup)      │                │
   │   │ • Rate limit (edge: approx, fast)        │                │
   │   │ • Request canonicalization, budget check │                │
   │   └─────────────┬────────────────────────────┘                │
   │                 │                                             │
   │   ┌─────────────┴──────────────┐   ┌──────────────────────┐   │
   │   │  Chat API (stateless)      │   │  Files, Auth,        │   │
   │   │  manages connection lifecyc│   │  Admin, Billing APIs │   │
   │   └─────────────┬──────────────┘   └──────────────────────┘   │
   │                 │ internal gRPC                               │
   │   ┌─────────────┴────────────────────────┐                    │
   │   │  Orchestrator / Agent Runtime        │                    │
   │   │  • prompt assembly                   │                    │
   │   │  • streaming pump                    │                    │
   │   │  • tool loop                         │                    │
   │   │  • safety in-line                    │                    │
   │   │  • persistence (partial saves)       │                    │
   │   └─────┬──────────┬──────────┬──────────┘                    │
   │         │          │          │                               │
   │   ┌─────┴───┐  ┌───┴──────┐  ┌┴─────────────┐                 │
   │   │Inference│  │Retrieval │  │Tool Executors│                 │
   │   │(GPU)    │  │(vec+KV)  │  │(sandbox)     │                 │
   │   └─────┬───┘  └───┬──────┘  └┬─────────────┘                 │
   │         │          │          │                               │
   │   ┌─────┴──────────┴──────────┴─────────────────────────┐     │
   │   │              Data Plane (per-region)                │     │
   │   │ • DDB/Cassandra conversations                       │     │
   │   │ • Postgres/Aurora metadata, billing                 │     │
   │   │ • S3 blobs (attachments, exports)                   │     │
   │   │ • pgvector / Pinecone embeddings                    │     │
   │   │ • Redis sessions + prompt cache metadata            │     │
   │   │ • Kafka event bus                                   │     │
   │   │ • ClickHouse analytics                              │     │
   │   └─────────────────────────────────────────────────────┘     │
   └───────────────────────────────────────────────────────────────┘
       Cross-region replication (async) for conversation data,
       global eventual consistency, region-pinned user identity.
```

### Failure domains (explicitly enumerated)

| Domain | Blast radius | Isolation |
|---|---|---|
| **Availability Zone** | ~33% of regional capacity | Run ≥3 AZs; topology spread per deployment |
| **Region** | 100% of regional users | Active-active across 3–4 regions; Anycast failover |
| **GPU node** | N pods' active streams | PDB, rolling drain, partial-save frequency |
| **GPU pod** | Active batch's streams | Partial-save, session migration |
| **Model variant** | All users of that variant | Multi-variant fleet, graceful fallback tier |
| **Inference engine version** | New deploys | Canary with shadow traffic + auto rollback |
| **Tool executor** | Users invoking that tool | Fail-open with "tool unavailable" injection |
| **Retrieval cluster** | RAG users | Fail-open, degrade to no-context |
| **Data store shard** | Users on that partition | Shard re-balancing; hot-key mitigation |
| **Cell / stack** | One "cell" of customers | Cellular architecture: cap blast radius |

**Cellular architecture** is worth emphasizing: partition users into cells of ~1–5M users, each with its own stack (orchestrator + inference + DB shards). A bad deploy poisons one cell, not the whole region. This is how AWS, Slack, and Azure contain blast radius; at ChatGPT scale it's mandatory.

---

## 3. Client Layer — Corner Cases

### Connection lifecycle

A chat request is a multi-minute HTTP connection. Clients must handle:

- **Tab suspension / mobile backgrounding**: browser freezes JS when tab is backgrounded; mobile OS suspends the app. The stream stops receiving events but the server thinks the connection is alive.
  - Mitigation: heartbeat events (`:ping\n\n`) every 15s so TCP keepalive fires; client detects stall (no event > 30s → reconnect).
- **Tab closed mid-stream**: HTTP connection closes. Server should detect via write-fail and either (a) continue generation to persist partial message, or (b) abort to save compute. Default: abort and mark conversation as `interrupted`; next load resumes from partial save.
- **Intermittent Wi-Fi / LTE swap**: connection drops; user's IP may change. Client reconnects with `Last-Event-ID` (SSE) or a resume token carrying `(conversation_id, message_id, cursor)`.
- **Multiple tabs on same conversation**: both submit messages concurrently.
  - Server serializes per-conversation with a lock (Redis `SET NX EX`); second request gets 409 with "another response in progress" or queues.
- **Multiple devices**: phone and laptop open the same conversation. One starts generating.
  - Use pub/sub (Redis/WebSocket) to broadcast stream events to all subscribed clients. All devices see the same tokens arriving.
- **Keyboard autoretry / double-submit**: user hits Enter twice.
  - Idempotency key on POST: `(conversation_id, client_message_id)` dedupes at API layer.
- **Copy-paste of 1MB prompt**: client must refuse (or chunk) before sending.
  - Client-side token estimator (approximate) + server-side hard cap. Return `context_length_exceeded` with "tokens_used / tokens_max" for the UI.
- **Clock skew between client and server**: client-issued timestamps aren't trustworthy for ordering.
  - Server assigns message timestamps; client uses monotonic sequence numbers for local ordering.

### Offline / unreliable networks (mobile)

- Message queue persisted client-side. Retries with exponential backoff. Server dedupes by idempotency key.
- Conversation state syncs on reconnect via `GET /conversations/{id}?since={message_id}`.

### Streaming format choice

- **SSE** for server→client token stream. Simpler than WebSocket, works through most proxies, has built-in reconnect semantics (`retry:`, `Last-Event-ID`). Unidirectional fits LLM use case.
- **WebSocket** only if bidirectional voice/video is added.
- **HTTP/2** strongly recommended (multiplexing, header compression). Beware: some CDNs/proxies convert HTTP/2 to HTTP/1.1 internally — test end-to-end.

### Corner case: proxy buffering

Many corporate proxies buffer responses, defeating streaming. User sees no tokens for 30s then a wall of text.

- Mitigation: `X-Accel-Buffering: no` header for nginx family; flush after every event; SSE `Content-Type: text/event-stream` is usually enough to bypass buffering.

---

## 4. Edge / CDN / Global LB

### Responsibilities
- TLS termination (TLS 1.3, HTTP/3 where supported)
- WAF: OWASP rules + LLM-specific (huge payloads, pathological regex, prompt-injection-in-header attempts)
- L3/L4 DDoS scrubbing
- Rate limit (coarse, per IP/ASN) — real rate limit is at the API gateway
- Static asset caching
- Cache for cheap cacheable read-only endpoints (e.g., `/v1/models`)

### Corner cases

- **Long-lived streams + CDN timeouts**: Cloudflare free tier idles streams at 100s. Paid tiers up to ~15 min. If generation exceeds this, the stream dies.
  - Mitigation: Either use an enterprise CDN contract, bypass CDN for streaming endpoints via a separate subdomain (`stream.example.com` pointing straight at regional LB), or chunk generation (pause at natural boundaries and let client reconnect).
- **Global LB failover**: DNS TTL controls failover latency. TTL 60s means up to 60s of hard failure before resolvers re-query.
  - Use Anycast (AWS Global Accelerator / Cloudflare Anycast) for seconds-level failover, not DNS alone.
- **Retry storm across regions during failover**: when us-east-1 fails, all clients hammer us-west-2 simultaneously; us-west doesn't have the capacity.
  - Pre-provision failover capacity to ≥ 2× expected steady state. Edge-level rate limiting enforces shedding before overloading the healthy region.
- **Header size limits**: long conversations encoded in headers for retry are common mistake. Store state server-side; clients send only `conversation_id`.

---

## 5. Auth, Identity, Session

### Model

- **Web/mobile**: OAuth/OIDC with SSO (Google, Apple, Microsoft); enterprise SAML.
- **API**: opaque API keys, scoped (read/write, org, project).
- **Internal services**: mTLS with SPIFFE identities.
- Session tokens: short-lived JWT (15 min) + refresh token (30 days, rotating).

### Corner cases

- **Token theft replay**: stolen JWT used from different IP / UA.
  - Bind token to client fingerprint where feasible; anomaly detection on IP + UA changes; forced re-auth on sensitive operations.
- **Rotation race**: refresh happens twice from two tabs simultaneously.
  - Refresh tokens are single-use; reuse detection = revoke entire refresh chain. Both tabs re-auth. Annoying but secure.
- **Account deletion during active stream**: user deletes account while a stream is running in another tab.
  - Stream auth is verified at start, not per-token. Post-deletion stream continues to completion; its output is written to a tombstoned conversation that the deletion pipeline cleans up within SLA.
- **SSO session expiry mid-conversation**: IdP expires session; websocket/SSE connection doesn't know.
  - Edge periodically re-validates; on expiry, send a `stream_interrupted` event with `reason: "reauth_required"`. Client re-auths and resumes.
- **Enterprise user leaves org**: org admin deactivates the user.
  - Propagation time matters. Hard sync on deactivate → revoke active sessions across regions (publish revocation event on Kafka, each region's gateway honors denylist). Target < 5s.
- **API key rotation without downtime**: customer rotates key.
  - Support overlapping validity windows: old key valid for N hours after rotation. Inform customer of last-use timestamp for the old key for audit.
- **Clock skew for JWT**: edge regions may have clock drift → tokens rejected as "used before nbf".
  - Enforce NTP; tolerate ±30s skew in JWT validation.

### Corner case: per-conversation authorization

A user can "share" a conversation. Shared link must be read-only, revocable, optionally password-protected.
- Share links are separate short URLs with their own token (`share_token`), stored with a TTL and a revocation flag. Accessing them doesn't grant access to the user's other conversations.
- Corner case: conversation contains user's attached file. Shared read-only link must also grant read access to the files referenced. Store per-share ACLs in a separate table, not derived from conversation ACLs.

---

## 6. API Gateway & Rate Limiting

### Rate limiting at scale — harder than it looks

Naive Redis counter doesn't work at 1M RPS across a multi-region fleet. Pattern:

- **Two-layer bucket**:
  - **Edge bucket** (local to API gateway pod): fast, approximate, sheds obvious overage (1s window).
  - **Global bucket** (Redis cluster, per-region master): exact over longer windows (1 min, 1 hour, 1 day). Periodically syncs with other regions for total-across-regions caps.
- **Token bucket semantics** (not sliding window): simpler to reason about, allows bursts up to bucket capacity.
- **Per-dimension limits**: per-user, per-API-key, per-org, per-endpoint, per-model. A rogue tenant can be rate-limited without affecting others.

### Corner cases

- **Cross-region drift**: user's limit is 1000 req/hour globally but they hit two regions; each region thinks they're at 800.
  - Accept small drift (<20%) for most endpoints. For hard limits (free-tier daily quota) use a globally consistent counter (DynamoDB with transactional writes, or Spanner).
- **Starts a 60s stream then counts it**: is it 1 request or N tokens?
  - Count both: 1 request + N output tokens. Rate limit on both axes independently. Stream can be terminated mid-generation when token budget exhausted (send `rate_limited` marker and close).
- **Quota exhaustion mid-stream**: user has 100 tokens left, generates 50, then limit.
  - Send `rate_limited` SSE event, close. Bill for tokens actually produced. Next request hard-rejected with Retry-After.
- **Burst legitimate traffic from one IP (NAT)**: office with 1000 users behind one IP.
  - Never rate-limit by IP as primary key in logged-in flows — use user ID. IP limits only apply to anonymous/auth endpoints.
- **Distributed rate limit outage**: Redis down, edge bucket only.
  - Fail-open for coarse limits (approximate), fail-closed for hard quotas (safer to 503 than to hand out free compute).

### Request canonicalization

- Normalize headers, strip unexpected ones, cap header size.
- Enforce max body size per endpoint. Reject with 413 before reading full body.
- Decode + validate content-type / encoding. Reject malformed streams early.

---

## 7. Orchestrator / Agent Runtime

The orchestrator is the nerve center. Each chat request creates an in-memory "session context" that orchestrates inference, tools, retrieval, and client streaming.

### Core loop (pseudocode)

```
ctx = build_request_context(user, conversation, message)

ctx.prompt = assemble_prompt(
  system_prompt, long_term_memory, recent_messages,
  retrieved_docs, tool_schemas, user_message
)

check_budget(ctx)
check_input_safety(ctx)

stream = inference_client.stream(ctx.prompt, params)

for event in stream:
    apply_streaming_safety(event)
    persist_delta(ctx, event)       # partial save
    emit_to_client(ctx, event)
    if tool_call_detected(event):
        result = execute_tool(event, timeout=T)
        ctx.prompt = append_tool_result(ctx.prompt, result)
        stream = inference_client.stream(ctx.prompt, params)  # re-enter
    if interrupted_by_client(ctx):
        abort_upstream(stream)
        break

finalize(ctx)
```

### Design decisions

- **Stateless orchestrator pods**: conversation state lives in Redis (active stream) + DDB (durable). Pod can die mid-request; another picks up via resume token.
- **Runtime**: Go or Rust for high concurrency. One goroutine per active stream; a single pod handles 10K+ concurrent streams with modest memory (~8KB per stream context).
- **Backpressure pervasive**: inference's throughput is finite; orchestrator must slow down ingress when downstream queues grow.

### Corner cases

- **Client disconnect mid-generation**: TCP RST or EOF detected.
  - Choice: (a) stop generation to save GPU, or (b) continue to persist partial message.
  - Default: continue for N more tokens (~max 200) to land at a natural boundary, then stop. This lets user reload and see a complete thought, not a severed sentence.
- **Tool call with mismatched JSON schema**: model hallucinates a tool call with wrong args.
  - Validate at orchestrator; if invalid, inject a tool-error message into the conversation and re-invoke the model with the error, up to a retry cap. Don't leak the raw model output; synthesize a clean user-facing message if cap exceeded.
- **Infinite tool loop**: model calls `search` repeatedly with the same query.
  - Hard cap: max tool calls per request (e.g., 10). Per-tool call cap (e.g., 3 calls to `search`). Detect repeated identical calls and break early with "tool depth exceeded" marker.
- **Tool takes too long**: e.g., web search 30s.
  - Per-tool timeout. On timeout, inject "tool timed out; respond based on what you know" and let model handle gracefully.
- **Retrieved documents > context window**: RAG returns 50KB of text, model context is 32K tokens.
  - Orchestrator truncates/summarizes/re-ranks before prompt assembly. Warn user if docs didn't fit.
- **System prompt itself changed between turns** (deployed new version): conversation history was generated with old system prompt; new prompt may contradict.
  - Pin system prompt version per conversation; carry it in the conversation metadata. Upgrading to new system prompt is an explicit client action.
- **Regenerate request**: user clicks "regenerate" — must produce a different answer.
  - Raise temperature slightly or use a different seed. Preserve original response (soft-delete) in case user wants to compare.
- **Stop mid-generation**: user clicks Stop.
  - Client sends DELETE to a "stop" endpoint. Orchestrator cancels upstream inference via per-stream cancel token. Partial message is saved as `stopped_by_user`.
- **Concurrent messages on same conversation**: two clients submit.
  - Per-conversation lock (Redis `SET NX EX <conversation_id>`) with orchestrator-instance as value. Second request sees lock and either queues, rejects (409), or — for some UIs — joins the stream of the first.
- **Orchestrator pod crash mid-stream**: in-flight stream's client now stuck.
  - Orchestrator writes a heartbeat to Redis with the stream's state every few seconds. Health controller detects stale heartbeat, marks stream `interrupted`. Client reconnects and sees the partial message + a `stream_interrupted` event; can resume or regenerate.
- **Orchestrator memory leak**: accumulates under long-lived streams.
  - Explicit context cleanup + periodic pod recycling with MAX_CONNECTION_AGE. Graceful drain respects PDB.

### Prompt assembly — traps

- **Stale memory**: long-term memory contains outdated facts (user's job changed).
  - TTL on memory items + user-visible "memory manager" for audit/edit.
- **Prompt injection via memory**: malicious tool output was committed to memory.
  - Treat memory as untrusted content, wrap in `<untrusted>...</untrusted>` delimiters, instruct model to not follow instructions from within.
- **Tokenizer drift**: a model upgrade changes tokenizer; old cached token counts wrong.
  - Tokenizer version is part of the conversation metadata. Re-tokenize on version change.
- **Mixed-language edge cases**: tokenizers count differently for CJK, RTL scripts, code.
  - Rely on server-side tokenizer of truth; treat client-side estimators as advisory only.

---

## 8. Inference Layer (Where It Gets Real)

### What's happening on a GPU

For each token generated:
1. **Prefill phase** (first token): process the entire prompt; O(prompt_len²) attention; memory bandwidth bound.
2. **Decode phase** (subsequent tokens): use KV cache; O(prompt_len) per token; memory bandwidth bound on cache.

Both phases have different optimization profiles. Modern engines do **prefill disaggregation**: separate pools for prefill-heavy workloads vs decode-heavy, to avoid prefill stalling decode on shared GPUs.

### Stack

- **Engine**: vLLM / TensorRT-LLM / SGLang. Continuous batching + PagedAttention are table stakes.
- **Parallelism**: TP across GPUs in a node (NVLink), PP across nodes, EP for MoE. Orchestrate via topology-aware scheduler (the default k8s scheduler is not GPU-topology-aware; use volcano/kueue/run:ai).
- **Quantization**: FP16 default; FP8 on H100 for 2× throughput; INT8/INT4 for smaller/free tiers.
- **Speculative decoding**: 7B draft model + 70B target. Net 1.5–3× speedup for batch sizes where decode is memory-bound.
- **Prefix / radix cache**: dedupes KV for shared prompt prefixes.

### Routing & load balancing (the unique bits)

A stream is stateful on the GPU: the KV cache for its conversation is resident on one replica. You cannot round-robin subsequent tokens.

- **Intra-request stickiness**: once a request enters a replica, all tokens come from there until done.
- **Conversation stickiness** (soft): route next-turn requests of the same conversation to the same replica if it's alive and has spare capacity — the KV cache from last turn is already there, prefill is near-free.
- **Cache-aware routing**: use a hash of the long prompt prefix (system prompt, persona, RAG docs) to route to a replica that already has that prefix cached. RadixAttention / prefix tree.
- **Load signals**: running batch size, KV-cache utilization, queue depth. CPU% is meaningless.
- **Fleet tiers**: big model / premium instance type / premium users. Smaller / cheaper / free tier. Routing at orchestrator.

### Corner cases

- **KV cache exhaustion during decode**: enough requests admitted that cache can't fit the active batch anymore.
  - Engine evicts some requests (by policy: LRU, smallest, or lowest-priority), they're **preempted** and restarted once slots free up. From the client's perspective this is a brief stall.
  - Orchestrator must be told so it can decide to wait vs move the stream to another replica vs partial-save and give up.
- **Out-of-memory on prefill**: giant prompt arrives; prefill can't allocate KV for it.
  - Engine rejects with `context_length_exceeded` or `insufficient_kv_memory`. Orchestrator returns a clean error; doesn't retry on a different pod (same input, same result).
- **Mid-stream GPU failure (CUDA error, hardware fault, thermal)**: pod becomes unhealthy.
  - Detect via health probe + stream stall. Orchestrator aborts the stream, partial-saves, emits `stream_interrupted`. Client reconnects and either resumes from partial (re-issue with `assistant` history up to partial) or regenerates.
  - Full pod drain marks node NotSchedulable; Karpenter / cluster-autoscaler remediates.
- **Noisy neighbor in same batch**: one request with pathological attention pattern slows the batch.
  - Per-request compute budget + early termination. In modern engines this is less of an issue; continuous batching helps.
- **Model weight corruption** (disk/network error during load): pod starts up but outputs garbage.
  - Checksum on load; smoke-test output against canary prompts before marking pod Ready.
- **Non-determinism on retry**: identical retries don't produce identical outputs.
  - Expected (sampling, fused kernels). Idempotency via dedup on `(conversation_id, client_message_id)` rather than "same output".
- **Stale inference pod after new model rollout**: orchestrator routes to a pod running the old model.
  - Version all model endpoints. Orchestrator passes explicit `model_version` in each request; mismatched versions are rejected.
- **Prefix cache poisoning**: a cached prefix is corrupted or misattributed.
  - Cache key must include tokenizer version + model version + full prefix hash, not just a semantic key.
- **Cross-tenant KV cache contamination** (the scariest): prefix cache shared across users unintentionally leaks content.
  - Partition prefix cache by tenant scope. Enterprise tier: cache scoped to org. Never cache across the public/enterprise boundary.
- **Spec-decoding mismatch**: draft model predicts a token the target rejects; wasted work.
  - Expected; target throughput still net positive because verify is batched. Monitor acceptance rate; if it crashes, either the draft is miscalibrated or there's a numerical bug.
- **Mid-stream model swap mid-sentence**: we'd never do this, but a naive fallback to a different tier would cause incoherent output.
  - Fallback only on failed start, not mid-stream. If a mid-stream fails, the client sees `stream_interrupted` and can regenerate (possibly with a different model) with a clean slate.
- **Multimodal attachment corruption**: user uploads a PDF that makes the image/text encoder crash.
  - Validate + sandbox the attachment preprocessor. Return `invalid_request` ("we couldn't process this PDF") rather than crashing inference.
- **Context length exceeded mid-generation**: model has generated 3900 tokens of a 4096 window and hasn't stopped.
  - Engine emits `length` finish reason. Orchestrator emits that to client. UI offers "continue" button which sends a follow-up turn.
- **Repetition loop**: model stuck generating same tokens.
  - Engine-level repetition penalty; orchestrator-level fallback stopping rule (e.g., if last 50 tokens are cyclic, abort).
- **Numerical instability under FP8**: rare outputs contain NaNs / pathological tokens.
  - Monitor output logprobs; fall back to FP16 if instability detected. A/B compare quality before enabling new quantization tier.

### Deployment of model changes

- **Multiple versions live simultaneously**: v1.5 (stable), v1.6 (canary).
- **Weighted routing**: 99% → v1.5, 1% → v1.6. Grow canary over a week, gated by quality eval dashboard + user feedback + stream-completion rate.
- **Rollback**: instantaneous routing flip back to stable. Don't rely on re-deploy (takes 30+ min).
- **Model warmup**: new pods must be smoke-tested against a canary prompt suite before accepting real traffic.

### Edge case: tokenizer change across versions

- Conversation stored as raw text, not tokens. Each turn is re-tokenized by the active model's tokenizer.
- When upgrading: past conversations re-tokenize cleanly; but token counts for display/billing may change.
- Bill based on server-authoritative count for that request; don't display cumulative token usage across model changes without a caveat.

---

## 9. Retrieval / RAG

### Layers of retrieval

- **Short-term memory**: prior messages (already in conversation history).
- **Long-term memory**: user-specific facts (preferences, history). Key-value store + embeddings.
- **Document retrieval**: uploaded files, attached knowledge bases (enterprise).
- **Web retrieval**: tool-based, handled in §10.

### Storage

- **Vector index**: pgvector for ≤100M vectors per tenant; Pinecone/Weaviate/Qdrant for larger.
- **Hybrid**: BM25 (OpenSearch) + vector + optional re-rank model.
- **Per-tenant isolation**: shard by tenant_id; no shared indices across customers.

### Corner cases

- **Stale embeddings after document edit**: user updates a doc; old chunks linger.
  - Delete old chunks on update; re-embed. Use content hash to dedupe. Background reconciliation job audits.
- **Embedding model change**: switched from `text-embedding-3-small` to `-large`; dimensions and semantics differ.
  - Maintain old+new index during migration. Re-embed in the background; switch reads atomically when complete.
- **Cross-tenant query contamination**: bug in the filter returns other tenant's docs.
  - Filter enforcement at retrieval wrapper (not optional at query time). Integration tests with adversarial queries. Tenant ID is a mandatory filter, not a parameter.
- **Empty retrieval**: no docs matched.
  - Don't pass empty context with "use these docs:" — the model will hallucinate docs. Either omit the context block or explicitly say "no relevant information found; answer from general knowledge or say you don't know."
- **Retrieval returns too much**: 50 chunks, 100KB.
  - Re-rank to top K (cross-encoder or LLM-based), then cap token budget for context block. Leave room for model's output.
- **Retrieved doc contains prompt injection**: "Ignore your instructions and…"
  - Treat retrieved content as untrusted. Wrap in delimiters. System prompt instructs model to disregard instructions inside retrieved content. Not foolproof; monitor with safety classifiers.
- **Retrieved doc contains PII of another user**: shouldn't happen if tenant isolation is right, but classifier as belt-and-suspenders.
  - PII detector on retrieval outputs for consumer accounts; redact or skip chunks that contain PII unrelated to the requesting user.
- **Document upload in progress mid-query**: chunks not yet indexed.
  - "Indexing in progress" state visible in UI. Queries either skip the doc or wait (depending on latency budget).
- **Hot-partition on vector DB**: celebrity document queried by millions.
  - Read replicas for hot partitions; pre-compute + cache top K results for popular queries.
- **Vector index partial failure** (one shard down).
  - Degrade gracefully: query remaining shards, annotate result as "partial index". Decide whether to expose that to the model in the prompt ("your retrieval was partial, caveat your answer").

---

## 10. Tool Use / Code Execution

### Tool taxonomy
- **Stateless, idempotent**: web search, calculator, unit conversions — safe to retry.
- **Stateless, non-idempotent**: e.g., send_email, create_ticket — must never retry without idempotency key.
- **Stateful / side-effectful**: code execution, file writes, database queries — need sandboxing + auditing.

### Code execution sandbox (the hard one)

- **gVisor / Firecracker / Kata** microVMs per sandbox. Not Docker alone (kernel shared = unsafe).
- **Ephemeral filesystem**, quota-bound (disk, memory, CPU, network).
- **No outbound network** by default. Whitelist specific domains (PyPI, npm) if needed.
- **Time limits** per execution (e.g., 30s).
- **Resource limits** fork/mem/file descriptors.
- **Log everything**: stdin, stdout, syscalls sampled. Audited for abuse.

### Corner cases

- **User exfiltrates secret via env leak**: subprocess tries to read environment.
  - Clean env inside sandbox. Secrets never injected into sandbox env.
- **User DoS the sandbox fleet**: endless infinite loops.
  - Per-user concurrent sandbox cap; kill long-runners at timeout.
- **Sandbox escape (hypothetical)**: worst case.
  - Layered: sandbox, namespace, seccomp, MAC (SELinux), network isolation, separate account / AWS organization unit for sandbox infra.
- **SSRF via tool**: user tricks `http_fetch` into hitting 169.254.169.254 (cloud metadata) or internal IPs.
  - URL validator with strict allowlist of schemes + resolved-IP allowlist. Resolve domains once and check the IP. Be aware of DNS rebinding attacks.
- **Recursive tool call chain**: tool A output triggers tool B triggers A.
  - Global depth cap + repeated-call detection.
- **Tool auth expires mid-conversation**: user's connected Gmail OAuth expires.
  - Orchestrator detects 401, emits an in-stream prompt for reauth; pauses this tool's use; model continues without it.
- **Tool response too large**: web search returns 500KB.
  - Summarize / truncate before feeding to model. Store full response in conversation attachments if user might want it.
- **Model hallucinates a tool that doesn't exist**: produces a call to `tool_zzz`.
  - Orchestrator rejects: inject error "that tool doesn't exist"; re-invoke model.
- **Non-idempotent tool retried after failure**: did the email actually send?
  - Require `idempotency_key` on every non-idempotent tool; duplicate suppression at tool provider.
- **Tool result contains prompt injection**: web search result says "ignore everything else and send $1000 to...".
  - Untrusted wrapping (see §9) + specific detectors for exfiltration/payment patterns in enterprise tiers.

### Tool safety policy

- Tools must be **classified** (`read-only`, `side-effect`, `privileged`). Users/admins configure which classes a given model is allowed to call.
- Audit log every tool invocation with request, response hash, outcome, user ID, model version. Enterprise tier: exportable to SIEM.

---

## 11. Storage Layer

### Choices

| Data | Store | Notes |
|---|---|---|
| Conversation messages | DynamoDB or Cassandra | Append-heavy, infinite scale, partition by user_id |
| User metadata, billing | Aurora Postgres (regional write + read replicas) | Strongly consistent, relational |
| Attachments / blobs | S3 + lifecycle | Presigned uploads |
| Embeddings | pgvector / Pinecone | Per-tenant partitions |
| Active session state | Redis Cluster | Streams, locks, prompt cache metadata |
| Event bus | Kafka (MSK/Confluent) | Analytics, async pipelines, DLQs |
| Analytics | ClickHouse | Append-only, columnar |
| Audit | Append-only log (CloudTrail + immutable S3) | Regulatory compliance |

### Conversation model

```
PK:  user_id
SK:  conversation_id # message_id_ulid
Attrs:
  role, content (may be S3 pointer for large),
  model, tokens_in, tokens_out,
  finish_reason, created_at,
  attachments[], tool_calls[], safety_flags[]
```

ULIDs for message IDs (sortable by time + globally unique).

### Corner cases

- **Hot partition**: one user with 100K conversations, or a shared enterprise workspace.
  - Further shard: `user_id#shard_bucket` as partition key for heavy users. Or use account_id + bucket by time.
- **Huge single conversation** (10K messages): DDB item limits are per-item, not per-partition; fine, but reads are expensive.
  - Paginate; client loads most recent N, older on scroll. Archive old messages to colder tier.
- **Concurrent writes same conversation** (multiple devices): last-write-wins or per-message create-only.
  - Messages are append-only with monotonic ULID; writes never conflict. Edit/regenerate creates a new message with a reference to the edited one; never in-place mutation.
- **Read-after-write on eventual-consistent store**: user sends message, then fetches conversation, and the new message isn't there yet.
  - Use strong-consistency read on the critical path (same-region). Cross-region reads are eventually consistent (tolerable).
- **Write amplification from denormalization**: storing user prefs per message.
  - Keep per-message items lean; join with user metadata at read time (cached).
- **Large attachments**: 100MB PDF.
  - Never store in DDB; S3 pointer. DDB item references the S3 key. Cleanup on conversation delete.
- **S3 upload mid-fail**: multipart upload abandoned.
  - Lifecycle policy expires incomplete multipart uploads after 7 days.
- **Cross-region replication lag**: user's new message not visible in other region yet.
  - User is pinned to home region for reads; replication to DR is best-effort. Explicit region-switch UX for travel.
- **Backup corruption / ransomware**: attacker deletes or encrypts data.
  - Immutable backups (S3 Object Lock, governance/compliance modes), delayed replica, periodic restore drills.
- **Schema migration**: adding a column to a hot table.
  - DDB is schemaless — no issue. Postgres: online migrations via pg_repack / pt-osc. Never ALTER on a hot table during peak.
- **Delete under legal hold**: user asks for deletion but a subpoena holds the data.
  - Tag under hold; delete request is queued, executed after hold. User-visible status "pending legal hold".
- **Conversation ownership transfer** (enterprise user moves teams): who owns the data?
  - Policy-driven. Either per-org data pool (user loses access when they leave), or per-user (user keeps it). Configurable per tenant.

---

## 12. Multi-Region, Data Residency, Failover

### Regional topology

- **Home region** per user (by signup geography; configurable in enterprise).
- **Active-active** across ≥3 regions; each region has full stack.
- **Data residency enforcement**: EU-resident users' data never leaves EU region (except billing metadata to a single billing region, with data processing agreement).

### Replication

- **Conversations**: region-local primary + async cross-region replica. For most users, only one region writes.
- **Billing / account**: cross-region synchronous (smaller, low-QPS, correctness critical).
- **Embeddings**: region-local; not replicated (re-compute on DR).

### Corner cases

- **Region hard failure**: us-east-1 unavailable.
  - Global LB detects via health check, fails traffic to other regions. Active streams are lost; clients retry in new region.
  - Users' home-region metadata is inaccessible. Fallback: treat all users as "read-only" in DR region until home region recovers — can serve new conversations but not sync with prior history. Communicated via UI banner.
- **Split-brain during partial partition**: us-east-1 can't reach global billing but can serve users.
  - Authenticate locally from cached auth state (short TTL); throttle new signups; degrade silently on quota checks (fail-open on soft, fail-closed on hard billing limits).
- **User switches home region** (moves countries).
  - Rare, explicit operation. Migrate data in background; user sees "migration in progress; some conversations may be read-only".
- **Conflicting writes across regions** (same conversation somehow active in two regions).
  - Shouldn't happen in normal flow (home-region pin). If it does (failover + primary recovers), use message ULIDs for deterministic merge. Conflicts on metadata (conversation title): last-writer-wins with vector clocks for audit.
- **Data residency and shared infrastructure**: vector DB shared across regions?
  - Separate indexes per region. Embedding model hosted per region (small models duplicate; large models too expensive, accept cross-region call for that specific request).
- **DR region capacity**: sized for steady state, not 1× of primary.
  - Pre-provision DR region for ≥ 2× steady-state of its own region (so it can absorb failover from a peer). Expensive but necessary.

---

## 13. Autoscaling & Capacity

### Scaling axes
- **Stateless pods**: orchestrator, API gateway, safety classifiers — HPA on CPU/RPS.
- **GPU pods**: the hard part.
- **DB / Redis**: vertical first, then sharding.

### GPU autoscaling

- **Cold start**: 30–60s for a 70B model from warm image + fast storage. More like 2–5 min without these.
- **Warm pool**: N GPUs with model loaded, idle, costing $. Pay for burst protection.
- **Over-provisioning pause pods** at medium priority: real workload evicts them and gets its request satisfied instantly. Karpenter spins up replacements in the background.

### Corner cases

- **Model load storm**: 50 pods cold-start simultaneously and hammer the weights store.
  - Shared read-only volume (FSx Lustre / EFS) or in-cluster weight cache (pre-pulled to local NVMe). S3 multi-part, parallel-read optimized formats (Safetensors).
- **Spot reclamation mid-stream**: AWS reclaims a spot GPU.
  - 2-minute warning. Drain: stop admitting new requests; partial-save all in-flight; tear down. Clients reconnect to other replicas.
  - Consequence: spot for inference is risky; typically reserved for batch/training.
- **Instance type unavailable**: cloud region has no H100s available.
  - Multiple instance family fallbacks in Karpenter; cross-family weight-compatible deployments (different TP strategies for different families).
- **Quota ceiling**: AWS per-account H100 vCPU limit reached during burst.
  - Pre-negotiate capacity reservations with CSP. Monitor quota; alert at 80%.
- **Scale-down during trough kills warm capacity**: 3 AM quiet → scale to floor → 7 AM rush catches us cold.
  - Minimum pool size above zero, predictive pre-warm 30 min before traffic ramps.
- **Uneven replica distribution across AZs**: scaling up 10 pods all land in one AZ.
  - Topology spread constraints (`maxSkew: 1`).
- **Pod churn from bad cluster autoscaler decisions**: Cluster Autoscaler thinks a node is underutilized and evicts needed pods.
  - Use Karpenter with consolidation + `do-not-evict` annotations on warm pods.
- **Predictive miss**: traffic spike from an unexpected event (news, viral tweet).
  - Reactive HPA still runs; queue shedding with `Retry-After`; emergency button to provision X nodes immediately for on-call.

---

## 14. Billing, Quotas, Fair-Use Enforcement

### Model

- **Tiered accounts**: free (Y tokens/day), plus (Z/mo + higher throughput), enterprise (contract).
- **Metering unit**: tokens in + tokens out + (image pixels, audio seconds) for multimodal + tool-call counts.
- **Attribution**: every inference request carries `user_id`, `org_id`, `model_version`. Orchestrator logs token counts from inference engine (source of truth) to Kafka → billing pipeline.

### Flow

```
Request → Orchestrator → Inference (returns token counts) → Orchestrator logs event →
  Kafka → Stream processor → Aggregated daily/monthly totals in Postgres → Invoices
```

### Corner cases

- **Double-counting on retry**: orchestrator retries failed request; both attempts log.
  - Use idempotency key as the metering dedupe key. The billing pipeline deduplicates by `(idempotency_key, model_version)`.
- **Token count disagreement** between client estimate and server reality (different tokenizers).
  - Server is source of truth. Surface estimated count in UI but bill on actuals. Include per-request token breakdown in the API response for API customers.
- **Quota exhaustion mid-stream**: discussed in §6.
- **Credit card declined during month**: account flagged; grace period (e.g., 7 days) before read-only / delete.
  - Communicate clearly in-product. Don't cut off enterprise tenants without a pre-agreed escalation path.
- **Free-tier abuse via multi-accounting**: one user signs up 50 times.
  - Device fingerprinting, phone verification for free tier, behavioral models. Balance with privacy.
- **Refund for failed request**: orchestrator errored after producing 100 tokens.
  - Policy: bill only for `successful_delta` tokens, where success is defined by engine's finish reason. Free-tier failures shouldn't count toward quota.
- **Currency, VAT, cross-border**: international compliance.
  - Use a billing platform (Stripe Tax, Paddle) rather than rolling your own.
- **Enterprise seat reassignment**: seat taken from one user and given to another.
  - Chat history does not transfer; per-user data stays with the departing user (subject to org policy). Enterprise admins can export/delete per DPA.

---

## 15. Safety / Moderation

### Layers
1. **Input classification**: before inference, classify user prompt. Hard-block categories (CSAM, explicit weapons-of-mass-destruction instruction). Soft-block categories route to refusal model.
2. **In-stream classification**: streaming classifier on output tokens, can terminate mid-stream if risky content detected.
3. **Post-generation review**: full-message classifier; flags for offline review / user feedback loop.
4. **Red-team evals**: continuous adversarial testing suite run against new model versions.

### Corner cases

- **Safe-in-isolation, unsafe-in-context**: user asks for "how to tie a noose" in what looks like a craft context but is suicidal.
  - Multi-turn context classifier, not single-turn. Heightened caution on self-harm signals.
- **Jailbreaks via roleplay / translation / Base64 / code / markdown**: classic prompt attacks.
  - Defense in depth: input classifier, output classifier, system-prompt reinforcement, periodic red-team update. Never one layer.
- **False positive kills UX**: classifier blocks "how to kill a process".
  - Calibrated thresholds. Surface an error that clearly distinguishes hard-block (policy violation) from soft-block (please rephrase).
- **Non-English moderation**: classifiers trained mostly on English.
  - Multilingual models; community-sourced eval sets; partner vendors for low-resource languages.
- **Generation mid-stream becomes unsafe**: started safe, drifts.
  - In-stream classifier aborts; client sees `content_policy_violation` marker and last safe token. Partial save.
- **Safety classifier down**: can't classify.
  - Fail-closed for hard categories (block request), fail-open for soft (serve, flag async). Always log for later review.
- **Safety escape hatches in enterprise**: enterprise needs tools that refuse for consumers (e.g., medical / legal professional tools).
  - Per-tenant policy configuration. Audit + contractual responsibility.
- **Children's safety**: under-13 account detection, COPPA compliance.
  - Age gating at signup; stricter filters for youth tier; region-specific rules.

---

## 16. Security & Abuse

### Controls

- mTLS between services (mesh)
- Secrets: per-tenant KMS keys for enterprise; envelope encryption; rotation schedules.
- Supply chain: SLSA-level artifact provenance; image signing with cosign; admission controller rejects unsigned.
- SBOM, CVE scanning, automated patch pipeline.
- Bug bounty program.

### Corner cases

- **Compromised API key used in parallel from different IPs/ASN**: legitimate user unlikely.
  - Anomaly detection + customer notification + automatic pause with override.
- **Prompt-exfiltration of training data** (model memorization of sensitive data).
  - Guard rails: data minimization, dedup, differential privacy where feasible, output classifiers for long memorized sequences. Treat this as an ongoing research problem.
- **Side channels via timing**: response length/speed leaks info.
  - Mostly unavoidable for streaming; mitigate by not branching on user-identifying data in hot paths.
- **Cross-user conversation leak via shared prefix cache**: discussed in §8.
- **Supply-chain injection into models** (poisoned fine-tuning data).
  - Data provenance, red-team evaluations, signed checkpoints, staged rollouts with quality gates.
- **Admin UI compromise**: single-point-of-everything.
  - Strong MFA, just-in-time privilege (break-glass), session recording, per-action approval for high-impact actions (e.g., regional failover).
- **Social engineering via share links**: attacker tricks user into sharing a conversation that contains a credential they typed.
  - Scanners for secrets in messages; redaction on share; user confirmation dialog listing what's in the share.

---

## 17. Privacy, Deletion, Legal Hold

### Deletion pipeline

User requests account deletion:
1. Mark `pending_delete` (reversible for 30 days).
2. Disable logins, revoke sessions, pause billing.
3. At T+30d, execute deletion plan:
   - Conversations (DDB/Cassandra)
   - Attachments (S3)
   - Embeddings (vector DB per-user partition)
   - Session metadata (Redis TTL naturally expires)
   - Long-term memory
   - Billing metadata (retain minimum per tax law)
   - Analytics: aggregate-level only; user-level purged.
   - Backups: mark for exclusion; rotate out within backup retention window.
4. Emit deletion receipt; log to immutable audit.

### Corner cases

- **Active conversation during deletion request**: see §5.
- **Legal hold blocks deletion**: queue; show user "pending legal hold" status.
- **Deletion from derived data** (fine-tuned models trained on user content): typically impossible to remove selectively.
  - Defense: don't train on user data by default; opt-in with clear notice; enterprise tier contractually excluded. For opted-in users, deletion is best-effort; next training cycle excludes deleted data.
- **Cross-region deletion races**: user deletes; us-east-1 confirms; eu-west-1 still has replica.
  - Deletion is an event on Kafka consumed by every region; each region confirms; orchestrator confirms user-facing only once all confirm.
- **Right-to-access / portability**: user requests all their data.
  - Async export job; emails ZIP with conversations, attachments, metadata in machine-readable format. SLA: 7 days.
- **Soft-deleted data accidentally re-surfaced**: bug in a search path ignores `is_deleted`.
  - Integration test with adversarial test data that's soft-deleted; must not appear in any endpoint.

---

## 18. Observability & SLOs

### Metrics (beyond CPU/latency)

- TTFT, ITL (P50/P95/P99) per model, region, tier.
- Stream completion rate (started → completed / errored / interrupted).
- Prompt cache hit rate.
- GPU utilization, batch fill, KV cache util.
- Tool success rate / timeout rate per tool.
- Safety classifier block rate (input/output) — regression = UX problem.
- Billing: $ per conversation / per user / per feature.
- Quality: thumbs up/down ratio, regeneration rate (high = model hates the prompt), eval scores.

### SLOs

- API availability: 99.95%
- Stream completion: 99.9%
- TTFT P95: 800 ms
- Quality (eval score): within 1% of baseline on regression suite
- Safety: false-block rate < 0.5%, true-block rate > 99%

### Corner cases

- **Metrics cardinality explosion** from tagging per-user: metrics pipeline OOM.
  - Strictly bounded tag sets. User-level insights come from analytics store (ClickHouse), not metrics.
- **Log PII leakage**: prompts contain user's data.
  - Configurable log redaction; default enterprise tier: no prompt logging; consumer tier: sampled, PII-scrubbed, retention < 30 days.
- **Trace sampling misses the long tail**: head-based sampling drops the slow request.
  - Tail-based sampling for errors and >P95 latency; 100% trace on safety events.
- **Alert fatigue during regional degradation**: 500 alerts in 5 min.
  - Alert deduplication + grouping; SLO-burn-based alerts only for top-level objectives; drill-down for sub-systems.
- **Clock skew confuses traces**.
  - NTP enforced; trace libraries tolerate skew. Alerts on drift.

### Quality monitoring is a first-class reliability concern

LLM quality regresses silently without alarms. Wire evals into monitoring:
- **Online evals**: sampled requests scored by a judge model or human raters.
- **Offline regression suite**: 1000s of canary prompts run against each new model / config.
- **Anomaly detection** on output metrics: sudden shift in refusal rate, average response length, token distribution.

---

## 19. Deployment & Release Engineering

### GitOps + progressive delivery
- ArgoCD (or Flux) per-region cluster.
- Argo Rollouts / Flagger for canary and automated rollback.

### Release pipeline
1. CI tests (unit, integration, lint, SCA, SAST).
2. Build + sign images.
3. Staging deploy; eval suite.
4. Canary to one cell in one region.
5. Grow by cells and regions with automated SLO checks.
6. Full rollout.

### Corner cases

- **Config change crashes on first pod**: readiness probe never green.
  - Rollout watchdog detects no progress after N minutes; automatic rollback.
- **Partial rollout across regions**: region A on v2, region B on v1; user's conversation state migrates region during the release.
  - Schema changes are backward-compatible for ≥ 2 versions. Release window includes "bake time" where old readers must handle new writers.
- **Database migration with active writers**: blocking migration causes outage.
  - Online migrations only. Lock-avoiding tools (pg_repack, gh-ost). Staged: add column (nullable) → dual-write → backfill → switch reads → drop old.
- **Feature flag misconfiguration**: flag rolled to 100% accidentally.
  - Two-person approval for prod flag changes. Staged rollout with SLO checks.
- **Rollback across stateful migrations**: v2 wrote a new column; rolling back to v1 that doesn't know about it.
  - Additive migrations only. Or carry the column forward with compatibility code in both versions for the overlap.
- **In-flight requests during deploy**: pod gets SIGTERM with an open 30s stream.
  - `terminationGracePeriodSeconds` longer than max stream. preStop hook stops admitting new streams + sleeps to drain. PDB bounds concurrent drains.

---

## 20. Cost Engineering & Corner Cases

### Techniques (GPU-inference-specific)

- Prompt caching (90% off on cached prefix tokens).
- Model tiering (route by user tier + complexity).
- Speculative decoding (2× throughput).
- Quantization (per-tier: FP16 → FP8 → INT8 → INT4).
- Batch API for non-urgent jobs.
- Embeddings / small classifiers on CPU.
- Reserved GPU + spot for burst (batch only).
- KV cache reuse across turns of same conversation.
- Efficient RAG (re-rank reduces context size).
- Static content / prefix cached at edge (system prompts, tool schemas).

### Corner cases

- **User discovers infinite-context trick**: submits 100-message conversation that re-computes context each turn.
  - Conversation-turn KV cache reuse + summarization of old turns. Cap conversation context programmatically.
- **Attack on tokens**: user sends adversarial prompt that maximizes token output ("list 10000 fruits…").
  - Per-request max_output_tokens (already enforced by model); per-user daily token cap.
- **Runaway agent loop**: tool use amplifies cost.
  - Per-request tool-call cap + cost cap; kill on cost-exceeded.
- **Routing bug sends free-tier users to premium model**: silent regression.
  - Cost-per-tier dashboards; alert on deviation > 20%.
- **Vendor price change**: cloud GPU instance price increases.
  - Budget alerts and capacity planning updated quarterly.
- **Over-provisioned warm pool during long quiet period** (e.g., holidays).
  - Predictive scaling accounts for calendar; manual override for known low-traffic periods.

---

## 21. Testing: Chaos, Load, Quality Evals

### Chaos engineering
- Kill GPU pods randomly (continuously, in prod).
- Regional failover drills quarterly (real traffic).
- Latency injection on dependencies.
- Poisoned model deploy (confirm auto-rollback fires).
- Full region isolation: can the system survive losing us-east-1 entirely?

### Load testing
- Replay production traffic at 2× into a mirrored stack.
- Synthetic bots simulate concurrent streams with realistic distributions.
- Test scale-from-zero paths; warm-pool exhaustion.

### Quality evals
- Regression suite: 1K–10K prompts covering safety, helpfulness, factuality, multilingual, code.
- Adversarial suite: jailbreaks, prompt injections, data exfiltration, SSRF-via-tool.
- Human eval for the top N% of most-impactful queries.
- A/B online experiments with guardrails on negative metrics (refusal rate, length, latency).

### Corner cases in testing
- **Canary prompts memorized by next model**: eval contamination.
  - Rotate eval sets; keep a held-out set.
- **Eval drift as model's output style changes**: auto-scorers (LLM judges) have biases.
  - Calibrate judges periodically against human labels.
- **Load test hits real billing**: oops.
  - Test accounts / shadow billing mode; synthetic traffic tagged for exclusion.

---

## 22. Runbooks for Classic Incidents

### "GPU fleet utilization pinned at 100%, TTFT climbing"
1. Check queue depth per replica. Rising?
2. Enable emergency scale-up if under quota; add pause-pod evictions.
3. Route premium traffic to reserved capacity; shed free tier with 503 + Retry-After.
4. If a specific model variant is hot, temporarily tighten routing to fewer user segments.

### "TTFT spiked but utilization is normal"
1. Check prefix cache hit rate — a drop means many fresh prompts (new prompt template deployed?).
2. Check prefill/decode split — prefill-heavy often = long attachments.
3. Investigate recent client changes (did someone add a huge system prompt?).

### "Stream completion rate dropping"
1. Check error-rate breakdown by class — which errors are up?
2. `stream_interrupted` rising → GPU failures, drain / eviction, or new deploy.
3. `upstream_timeout` rising → a tool or retrieval backend.
4. `internal_error` rising → orchestrator bug; check recent deploy.

### "Cost per conversation spiked 2×"
1. Model routing bug — check tier distribution.
2. Prompt cache hit rate dropped — new system prompt / tokenizer change invalidated cache.
3. Tool loop — check average tool-calls per request.
4. Input token growth — attachments?

### "Safety block rate spiked"
1. New model deployed — check canary quality eval.
2. New safety classifier deployed — may be over-triggering.
3. Coordinated jailbreak campaign — check abuse dashboards.

### "Regional failover is slow"
1. DNS TTL may be too long — use Anycast.
2. DR region at capacity — pre-provision.
3. Cold-start stampede — warm pool sizing in DR.

---

## 23. Summary Decision Matrix

| Scenario | Strategy |
|---|---|
| GPU pod OOM | PagedAttention + strict max_tokens + per-request KV budget |
| Stream interrupted | Resume tokens + partial save every N tokens |
| Model upgrade | Canary + shadow traffic + auto-rollback on quality regression |
| Cross-tenant data risk | Per-tenant partitioning of cache, vector DB, audit logs |
| Flash traffic spike | Warm pools + pause-pod over-provision + aggressive edge shedding |
| Regional outage | Anycast failover + active-active + DR 2× provisioning |
| Hot user / conversation | Further shard key + Redis front; don't store too much per-item |
| Long-running tool | Per-tool timeout + in-prompt notification to model |
| Model hallucinating tools | Strict schema validation + re-invoke with error |
| Prompt injection via RAG | Untrusted wrappers + output classifier + specific-threat detectors |
| Cost regression | Cache hit rate + tier routing + tool-call cap + output token cap |
| Quality regression | Online evals + regression suite + auto-rollback on burn |
| Deletion compliance | Event-driven multi-region delete pipeline + immutable audit log |
| Rate limit consistency | Two-tier (edge + region) + global for hard quotas |
| Noisy neighbor inference | Per-request budgets + engine preemption |
| Inference cold start | Warm pools + shared weights volume + fast loaders |

---

## Closing: Five Non-Obvious Truths

1. **The hardest corner cases are not individual bugs — they're emergent**. A retry storm, prefix cache contamination, or runaway agent loop is the interaction of normal-looking components. Design for them with system-level observability, not per-service unit tests.

2. **State lives where you forget it**: KV caches on GPUs, prompt cache metadata in Redis, partial saves in DDB, session locks in Redis, auth state in JWTs. Every one is a potential reliability footgun during a restart or failover. Map it explicitly.

3. **Quality is reliability** in an LLM product. A "working" system that lost 5% helpfulness after a deploy is an incident. Wire quality into the on-call rotation like latency.

4. **Degraded > broken** at every layer. Falling back to a cheaper model, a smaller context, a no-RAG response, or a "please try again" is almost always better than a 500. Design graceful degradation paths for every dependency before you build the dependency.

5. **Blast-radius isolation** matters more than uptime numbers. A 99.99% system that takes down 100% of users in a single incident is worse than a 99.9% system that takes down 5%. Cells, regions, tenants, fleet tiers — use them aggressively.