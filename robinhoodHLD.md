# Robinhood — Staff-Level High-Level Design for a Retail Stock Trading Platform

## Table of Contents

1. [Requirements & Scale](#1-requirements--scale)
2. [Back-of-Envelope Estimation](#2-back-of-envelope-estimation)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Core API Design](#4-core-api-design)
5. [Data Models](#5-data-models)
6. [Market Data Pipeline](#6-market-data-pipeline)
7. [Order Management System (OMS)](#7-order-management-system-oms)
8. [Order Routing, PFOF & Broker Integration](#8-order-routing-pfof--broker-integration)
9. [Execution, Fills & Trade Settlement](#9-execution-fills--trade-settlement)
10. [Portfolio & Positions Service](#10-portfolio--positions-service)
11. [Ledger, Wallet & Payments](#11-ledger-wallet--payments)
12. [Risk, Margin & Regulatory Controls](#12-risk-margin--regulatory-controls)
13. [Corporate Actions Handling](#13-corporate-actions-handling)
14. [Real-Time Price Push (WebSocket Fanout)](#14-real-time-price-push-websocket-fanout)
15. [Notifications](#15-notifications)
16. [Database & Storage Architecture](#16-database--storage-architecture)
17. [Caching Architecture](#17-caching-architecture)
18. [Scalability Deep Dive](#18-scalability-deep-dive)
19. [Reliability & Fault Tolerance](#19-reliability--fault-tolerance)
20. [Multi-Region & Disaster Recovery](#20-multi-region--disaster-recovery)
21. [Security & Auth](#21-security--auth)
22. [Observability & Operational Excellence](#22-observability--operational-excellence)
23. [Corner Cases & Hard Problems](#23-corner-cases--hard-problems)
24. [Key Trade-offs](#24-key-trade-offs)

---

## 1. Requirements & Scale

### Functional Requirements

**User / Account**
- Sign up, KYC (identity verification via Jumio/Onfido/Socure), link bank account (Plaid), fund brokerage account via ACH/wire/instant.
- View account summary: cash balance, buying power, day/total portfolio value, P&L.

**Market Data**
- Real-time quotes (Level 1 bid/ask/last), candles (1m/5m/1h/1d), historical data, news, fundamentals.
- Watchlists, tickers search, symbol detail pages.

**Trading**
- Place orders: Market, Limit, Stop, Stop-Limit, Trailing Stop, for Equities / Options / Crypto / ETF.
- Time-in-force: Day, GTC (Good-Till-Cancelled, capped at 90 days), IOC, FOK, Extended Hours.
- Cancel / replace / modify orders.
- Fractional shares (buy $10 of AAPL regardless of share price).
- Dividend reinvestment (DRIP).

**Portfolio**
- Positions, cost basis (FIFO / LIFO / Specific Lot), realized & unrealized P&L, tax lots.
- Corporate actions: splits, dividends, mergers, spin-offs applied automatically.

**Compliance & Risk**
- Pattern Day Trader (PDT) enforcement, Reg T margin, options approval levels, 1099 tax docs.
- Trade confirms, monthly statements, account histories (retained 7 years minimum per SEC Rule 17a-4).

**Notifications**
- Order fills, price alerts, margin calls, dividends credited, account events.

### Non-Functional Requirements

| Property | Target |
|---|---|
| Availability (trading hours) | 99.995% (≤ ~1 min/market-day) |
| Order submission latency | p50 < 50 ms, p99 < 200 ms end-to-end from client → OMS ack |
| Market data fanout latency | p99 < 100 ms from exchange tick → client render |
| Order durability | **Zero data loss** once ACK'd to client — sync-replicated WAL + journal |
| Consistency | **Strong** for cash, positions, orders, ledger; **eventual** for analytics, watchlists, news |
| Regulatory | SEC Rule 15c3-5 (market access), FINRA, Reg SHO, MSRB, SIPC coverage |
| Retention | 7 years for trade records (17a-4), immutable WORM storage |
| Recovery Time Objective (RTO) | < 2 minutes for trading path |
| Recovery Point Objective (RPO) | 0 for committed orders (sync replication) |

### Out of Scope (mentioned but not deep-dived)

- Retirement accounts (IRA, Roth) — same core flow with tax wrappers.
- Options Greeks computation engine.
- Robo-advisor / auto-investing strategies.
- Tax document generation pipeline (1099-B/DIV/INT).

---

## 2. Back-of-Envelope Estimation

```
Users:
  Total funded accounts:            25 million
  Monthly Active:                   15 million
  Daily Active (DAU):               5 million
  Peak concurrent logged-in:        2 million (market open, 9:30 AM ET)

Trading volume:
  Orders placed per day:            ~20 million (peaks during earnings / volatility)
  Orders per second (market hrs):   20M / 23,400s ≈ 850 orders/sec avg
  Peak orders/sec:                  ~50,000/sec (9:30 AM open, GME-style events)
  Fills per day:                    ~30 million (one order may produce multiple fills)

Market data:
  Symbols tracked:                  ~12,000 US equities + 3,000 options chains expanded → ~1M contracts
  Ticks/sec from SIP/exchange feeds: 5–10 million ticks/sec (peak); Robinhood filters to actives
  WebSocket connections:            2M concurrent at peak → distributed across fanout layer

Storage (hot path):
  Orders / day: 20M × ~2 KB         ≈ 40 GB/day, 15 TB/year
  Fills / day:  30M × ~1.5 KB       ≈ 45 GB/day
  Ledger entries: 2× fills (double-entry) ≈ 60M/day, 90 GB/day
  Market data raw ticks:            ~50 TB/day (downsampled to candles for historical ~500 GB/day)
  7-year retention obligation:      orders+fills+ledger ≈ 500 TB (WORM / S3 Glacier)

Bandwidth:
  Outbound WebSocket price push:    2M clients × ~2 KB/sec filtered ticks = 4 GB/sec ≈ 32 Gbps egress
  Compressed (delta + binary):      ~8 Gbps realistic

Connections:
  Concurrent WebSockets:            2–3M peak (fractions of DAU are streaming)
  New order REST QPS:               50k/sec peak
```

Key takeaway: **the bottleneck isn't user count — it's burst write load on OMS and real-time fanout of market data.**

---

## 3. High-Level Architecture

```
                     ┌─────────────────────────────────────┐
                     │        Mobile / Web clients         │
                     └──────┬────────────────────┬─────────┘
                            │ HTTPS              │ WSS (wss://)
                    ┌───────▼────────┐   ┌───────▼─────────┐
                    │  API Gateway   │   │ WebSocket Edge  │
                    │ (Envoy/Kong)   │   │  (sticky/hash)  │
                    │  authN, rate   │   └───────┬─────────┘
                    │  limit, TLS    │           │
                    └───┬────────────┘           │
                        │                        │
        ┌───────────────┼────────────────────────┼──────────────────────┐
        │               │                        │                      │
    ┌───▼───┐   ┌───────▼─────┐   ┌──────────────▼──────┐   ┌───────────▼───────┐
    │Account│   │  OMS        │   │  Market Data Fanout │   │  Portfolio Svc    │
    │ / KYC │   │ (pre-trade  │   │  (quote/tick relay) │   │ (positions, P&L)  │
    │       │   │  risk +     │   │                     │   │                   │
    │       │   │  order      │   └──────────▲──────────┘   └───────┬───────────┘
    │       │   │  lifecycle) │              │                      │
    └───┬───┘   └───┬─────────┘              │                      │
        │           │ (Kafka: orders, fills) │ (Kafka: ticks)       │
        │           │                        │                      │
        │     ┌─────▼─────┐         ┌────────┴────────┐              │
        │     │ Order     │         │ Market Data     │              │
        │     │ Router /  │         │ Ingestor        │              │
        │     │ SOR       │         │ (NYSE, NASDAQ,  │              │
        │     └─────┬─────┘         │  SIP, IEX, dark)│              │
        │           │               └────────┬────────┘              │
        │           │ FIX 4.4 / OUCH         │ UDP multicast /        │
        │           ▼                        │ binary feeds           │
        │     ┌───────────┐                  ▼                        │
        │     │Exchanges/ │            ┌──────────┐                   │
        │     │Market     │            │ Kinesis/ │                   │
        │     │Makers     │            │ Kafka    │                   │
        │     │(Citadel,  │            └──────────┘                   │
        │     │Virtu,…)   │                                           │
        │     └─────┬─────┘                                           │
        │           │ Fills via FIX                                   │
        │           ▼                                                 │
        │     ┌──────────────┐                                        │
        │     │ Trade Capture│─────┐                                  │
        │     │ / Clearing   │     │                                  │
        │     │ (DTCC/NSCC)  │     │ fill events                      │
        │     └──────────────┘     │                                  │
        │                          ▼                                  │
        │                   ┌──────────────┐       updates            │
        │                   │   Ledger     ├─────────────────────────▶│
        │                   │  (double     │                          │
        │                   │   entry)     │                          │
        │                   └──────┬───────┘                          │
        │                          │                                  │
        │                   ┌──────▼──────┐                           │
        │                   │ Postgres    │◀── (CDC: Debezium) ──┐    │
        │                   │ (primary    │                      │    │
        │                   │  of record) │                      │    │
        │                   └─────────────┘                      │    │
        │                                                        │    │
        │                            ┌───────────────────────────┴─┐  │
        │                            │ Kafka (change log / events) │  │
        │                            └──────┬──────────────────────┘  │
        │                                   │                         │
        │              ┌────────────┬───────┴──────┬─────────┐        │
        │              ▼            ▼              ▼         ▼        │
        │        ┌─────────┐  ┌───────────┐  ┌──────────┐  ┌──────┐   │
        └───────▶│Auth/KYC │  │Risk Engine│  │Notif Svc │  │Search│   │
                 │(Cognito/│  │(PDT, Reg T│  │(FCM/APNS/│  │(Elas-│   │
                 │Auth0)   │  │, margin)  │  │SMS/email)│  │search│   │
                 └─────────┘  └───────────┘  └──────────┘  └──────┘   │
                                                                      │
                                     ┌────────────────────────────────┘
                                     ▼
                              ┌─────────────┐
                              │ Analytics   │
                              │ (Snowflake/ │
                              │ Druid / S3) │
                              └─────────────┘
```

**Plane separation (core architectural principle):**

1. **Trading plane** — synchronous, ultra-reliable, strict budgets. OMS, Risk, Order Router, Ledger.
2. **Market-data plane** — async, high-throughput, best-effort delivery with sequencing. Ingestion → Fanout.
3. **User/app plane** — standard web service patterns. Profile, watchlists, news, UI BFF.
4. **Back-office plane** — nightly batch, reconciliation, regulatory reporting, tax docs.

Failures in planes 3 and 4 must **never** degrade plane 1.

---

## 4. Core API Design

### REST (mobile / web, over HTTPS)

```
POST   /v1/accounts                          Create account (KYC starts)
POST   /v1/accounts/{id}/deposit             ACH/instant deposit
GET    /v1/accounts/{id}/portfolio           Positions + P&L snapshot
GET    /v1/quotes?symbols=AAPL,TSLA          Level-1 snapshot (REST)
GET    /v1/historicals/{symbol}?interval=5m  OHLCV candles

POST   /v1/orders                            Place order (Idempotency-Key header required)
DELETE /v1/orders/{id}                       Cancel
PATCH  /v1/orders/{id}                       Modify (price / qty)
GET    /v1/orders?status=filled              List orders

GET    /v1/options/chains/{symbol}           Options chain
POST   /v1/options/orders                    Options multi-leg order
```

**Idempotency:** every mutating call (place order, deposit) requires `Idempotency-Key` header; OMS dedupes for 24h via Redis. 
A retried order with the same key returns the original order's state — it does not create a duplicate.

### Place-Order request (annotated)

json
POST /v1/orders
Idempotency-Key: 9f14ab-...
```
{
  "account_id": "acct_abc",
  "symbol": "AAPL",
  "side": "buy",            // buy | sell | sell_short | buy_to_cover
  "type": "limit",          // market | limit | stop | stop_limit | trailing_stop
  "qty": "10",              // decimal for fractional
  "notional": null,         // or "qty": null, "notional": "500.00"
  "limit_price": "170.25",
  "stop_price": null,
  "tif": "gtc",             // day | gtc | ioc | fok
  "extended_hours": false,
  "client_order_id": "..."
}
```

### WebSocket (real-time)

```
wss://ws.broker.com/v1/stream
> subscribe  { "type":"quotes", "symbols":["AAPL","TSLA"] }
> subscribe  { "type":"orders", "account_id":"acct_abc" }
> subscribe  { "type":"trades", "symbols":["SPY"] }
< { "channel":"quotes", "symbol":"AAPL", "bid":170.24, "ask":170.26, "ts":... }
< { "channel":"orders", "order_id":"ord_1","status":"filled","avg_price":170.25,"filled_qty":"10" }
```

Sequencing: every message has a monotonically increasing `seq` per channel. Client reconnect sends last-seen seq → server replays from buffer (60 s ring) or sends snapshot + catchup.

---

## 5. Data Models

### accounts (Postgres, strong consistency)
```
id                uuid PK
user_id           uuid FK
type              enum(cash, margin, ira)
status            enum(pending_kyc, active, restricted, closed)
cash_balance      numeric(18,4)
buying_power      numeric(18,4)
margin_used       numeric(18,4)
day_trades_count  int  -- rolling 5 biz-days for PDT
options_level     int  -- 0–3
created_at        timestamptz
```

### orders (sharded by account_id)
```
id                uuid PK
account_id        uuid
client_order_id   text   -- client-supplied (idempotency)
symbol            text
side              enum
type              enum
qty               numeric(18,8)       -- fractional support
notional          numeric(18,4)
limit_price       numeric(18,4)
stop_price        numeric(18,4)
tif               enum
status            enum(new, pending_risk, routed, partially_filled, filled, cancelled, rejected, expired)
filled_qty        numeric(18,8)
avg_fill_price    numeric(18,4)
created_at        timestamptz
updated_at        timestamptz
venue             text                -- exchange / MM that executed
parent_order_id   uuid                -- for child orders / replaces
```
Status transitions are an explicit FSM with allowed transitions; illegal transitions rejected at DB via trigger + application validation.

### fills (append-only, partitioned by trade_date)
```
id                uuid PK
order_id          uuid
account_id        uuid
symbol            text
side              enum
qty               numeric(18,8)
price             numeric(18,4)
venue             text
liquidity_flag    enum(added, removed)
execution_ts      timestamptz
settlement_date   date                -- T+1 post-2024
```

### positions (materialized, rebuilt from fills + corporate_actions)
```
account_id  symbol  qty  avg_cost  last_updated
```
Primary source of truth is the append-only `fills` + `corporate_actions` log. Positions table is a materialized view, 
rebuildable at any time — critical for audit and "we disagree with the user's balance" incidents.

### ledger (double-entry, append-only, WORM)
```
id          uuid PK
txn_id      uuid              -- groups debit/credit pair(s)
account_id  uuid
asset       text              -- 'USD' | 'AAPL' | 'BTC'
amount      numeric(18,8)     -- signed: + credit, - debit
type        enum(deposit, withdrawal, buy, sell, fee, dividend, interest, corp_action, adjustment)
reference   text              -- order_id, fill_id, etc.
created_at  timestamptz
```
**Invariant**: `SUM(amount) over a txn_id == 0` — every txn balances. Daily job asserts global ledger invariants per asset.

### tax_lots (per (account, symbol), FIFO/LIFO/SpecID)
```
id          uuid
account_id  uuid
symbol      text
qty         numeric
cost_basis  numeric
acquired_at date
closed      bool
```
Tax lots are created on buys and consumed on sells according to user-selected lot method. Every closed lot produces a realized-P&L ledger entry.

---

## 6. Market Data Pipeline

### Upstream sources
- **SIP (Securities Information Processor)** for consolidated NBBO (National Best Bid/Offer).
- **Direct exchange feeds** (NYSE Integrated, NASDAQ TotalView, CBOE, IEX DEEP) for lower-latency and depth.
- **OPRA** for consolidated options data.
- **Crypto venues** (Coinbase, FTX-successors, internal liquidity) over WS/FIX.
- **News** (Benzinga, Reuters) over HTTP/push.

All feeds arrive on dedicated multicast (inside a co-lo / AWS Direct Connect). Ingestor servers are pinned NUMA + SR-IOV network, 
running C++/Rust decoders with kernel-bypass (DPDK / Solarflare) for lowest jitter.

### Pipeline stages

```
Exchange  →  Ingestor  →  Normalizer  →  Sequencer  →  Kafka (partitioned by symbol)
                                             │
                                             ├──▶ Fanout (WebSocket)
                                             ├──▶ Candle Builder → time-series DB (Druid / InfluxDB)
                                             ├──▶ Alert Engine (price alerts trigger)
                                             └──▶ Historical archive (S3 parquet)
```

- **Sequencer** assigns a monotonic `seq` per symbol so downstream can detect gaps.
- **Dedup** across SIP + direct feeds: direct feed wins on tiebreak, SIP is failover.
- **Slow consumers** never block producers — Kafka bounded retention + unbounded disk; fanout consumers that fall behind are disconnected, 
  not buffered in memory.

### Candle aggregation
- Streaming aggregation in Flink: rolling 1-minute windows → emit on close → stored in Druid for fast historical queries.
- 1m candles rolled up nightly into 5m, 1h, 1d.

---

## 7. Order Management System (OMS)

The OMS is the beating heart — it owns the canonical order state machine and is the only component allowed to mutate order rows.

### Architecture

Two-tier:

1. **Stateless front (`oms-api`)** — accepts HTTP, does syntactic validation, assigns `order_id`, writes to idempotency cache, then calls core.
2. **Stateful core (`oms-core`)** — partitioned by `account_id` hash. Each partition is a **single-writer** service instance owning a Raft-replicated WAL. Within a partition, orders for an account are processed serially (so buying-power checks are consistent).

```
          oms-api (stateless, 100s of pods)
                         │
           consistent-hash by account_id
                         │
         ┌───────┬───────┼───────┬───────┐
         ▼       ▼       ▼       ▼       ▼
      oms-core oms-core oms-core …  (Raft group per shard, leader + 2 followers)
         │       │       │
         └───────┴───────┴─── WAL → Postgres (sync commit) → Kafka (fills, orders topics)
```

### Submission flow

```
1.  Client POST /v1/orders  (Idempotency-Key)
2.  oms-api → Redis: SETNX idem_key ord_xyz (24h TTL)
       - if exists, return cached response
3.  oms-api → oms-core (shard for account_id)
4.  oms-core:
     a. Load account state from in-memory cache (replayed from WAL on startup)
     b. Pre-trade risk checks (see §12):
         - Buying power / cash available
         - Position limits
         - PDT (Pattern Day Trader) gate
         - Reg T margin
         - SSR (Short Sale Restriction), circuit breakers, LULD
         - Blacklist / restricted securities
         - Options approval level
     c. If pass: assign status=pending_risk → persist to WAL (Raft replicated, fsync)
     d. On WAL commit (quorum ack) → return 201 to client
     e. Emit event to Kafka topic `orders.new` for Order Router
5.  Async: Order Router picks up, routes to venue (§8)
6.  Fill events come back → OMS updates order state → Ledger entry created → client WS push
```

Client sees ack only **after** WAL quorum commit → zero-loss guarantee for accepted orders.

### Order state machine

```
       ┌─────┐                    ┌──────────┐
       │ new │ ──pre-trade fail──▶│ rejected │ (terminal)
       └──┬──┘                    └──────────┘
          ▼
   ┌──────────────┐                ┌───────────┐
   │ pending_risk │ ──routing────▶ │  routed   │
   └──────────────┘                └─────┬─────┘
                                         ▼ (fills arrive)
                                ┌──────────────────┐
                                │ partially_filled │
                                └───────────┬──────┘
                                            ▼
                     ┌──────────┐  ┌─────────────┐  ┌────────┐
                     │ filled   │  │  cancelled  │  │expired │  (all terminal)
                     └──────────┘  └─────────────┘  └────────┘
```

Every transition is an event emitted to Kafka `orders.events` (partition key = order_id) so every downstream consumer sees the same canonical sequence.

---

## 8. Order Routing, PFOF & Broker Integration

### Smart Order Router (SOR)

SOR decides **where** an order goes. Inputs: order type, size, symbol characteristics, venue fees/rebates, NBBO, historical fill quality.

**Retail flow (typical Robinhood path):** most marketable equity orders are routed to wholesale market-makers (Citadel Securities, Virtu, G1X, Two Sigma) via **Payment For Order Flow (PFOF)**. Market makers internalize against their own book, offering price improvement vs NBBO, and rebate a fraction of the spread to the broker.

**Non-marketable limit orders** go to exchanges (NYSE, NASDAQ, IEX, EDGX) or dark pools to earn maker rebates.

**Crypto** routes to internal matching against partner liquidity or external venues.

### Protocols
- **FIX 4.4 / FIX 5.0 SP2** for equity, options, futures orders — `NewOrderSingle` (35=D), `OrderCancelRequest` (35=F), `ExecutionReport` (35=8).
- **OUCH** / **ITCH** for NASDAQ binary (lower latency).
- **REST / WS** for crypto venues.

### Reliability patterns on the FIX leg
- Persistent FIX session per venue with `MsgSeqNum` resync on reconnect.
- Every outbound `NewOrderSingle` is logged before send; on reconnect, unacked messages replay.
- Dual FIX sessions (primary/backup) per major venue, with automatic failover.
- Heartbeat every 30 s; if 2 heartbeats missed, force re-session.

### Reg NMS / best execution
- Order Protection Rule (Rule 611) forbids executing at a worse price than a protected quote — SOR routes or re-routes if NBBO changes intra-flight.
- **ISO (Intermarket Sweep Order)** used to execute large size against multiple venues simultaneously.

---

## 9. Execution, Fills & Trade Settlement

### Fill processing

Fills arrive asynchronously from venues via `ExecutionReport`. A single order may have 1..N partial fills across multiple venues.

```
Fill received by FIX gateway
     │
     ▼
Write to fills table (idempotent via exec_id)
     │
     ▼
Emit Kafka event `fills.v1` (partition = account_id)
     │
     ├──▶ OMS consumer: update order filled_qty, avg_fill_price, status
     ├──▶ Ledger consumer: write double-entry (debit cash, credit position)
     ├──▶ Positions consumer: update positions materialized view
     ├──▶ Tax-lot consumer: open / close lots
     └──▶ Notification consumer: push "Filled 10 AAPL @ $170.25"
```

Each consumer has its own offset; an outage in one does not block others. All consumers are **idempotent** (use exec_id as dedup key).

### T+1 settlement (US equities, post May 2024)
- Trade date (T): fill happens, positions updated immediately ("for display").
- Settlement (T+1): cash actually moves via DTCC/NSCC. Until settled, funds are **unsettled** and restricted from withdrawal / can trigger Good Faith Violations if re-used.
- Instant Deposits / Gold users get advance credit from broker's own balance sheet; unfunded users see a hold.

### Clearing / DTCC integration
- End-of-day batch: submit trade files to NSCC (Continuous Net Settlement).
- Reconciliation nightly against DTCC confirms; any mismatch triggers an ops ticket and manual break resolution before market open.

---

## 10. Portfolio & Positions Service

### Read model

Positions and P&L are **read-heavy** (every user refresh hits them). The authoritative log is `fills + corporate_actions`; positions are a denormalized materialized view.

- On fill event → update Postgres positions row + write to Redis (`positions:{account_id}:{symbol}` → qty, avg_cost).
- Unrealized P&L = `qty × (current_mark - avg_cost)` computed on-read using cached quote.
- Realized P&L aggregated from closed tax lots.

### Rebuild path
Positions can be fully rebuilt from scratch at any time by replaying the fills topic from earliest. Useful for:
- Recovering from a data corruption bug.
- Applying a retroactive corporate action.
- Bootstrapping a new replica.

### Hot-path read
- Portfolio summary is cached in Redis (TTL 5 s during market hours, longer after-hours).
- Tick updates don't write the cache; they publish to WS. A reconnect triggers a fresh read-through.

---

## 11. Ledger, Wallet & Payments

### Double-entry ledger
Every economic event in the system becomes one or more balanced ledger entries. This is the canonical source of truth for money and assets.

Example — Buy 10 AAPL @ $170.25 with $2 commission equivalent cost:
```
txn_id=t1
  (acct=alice, asset=USD,  amount=-1702.50, type=buy,  ref=fill_id)
  (acct=alice, asset=AAPL, amount=+10,      type=buy,  ref=fill_id)
  (acct=alice, asset=USD,  amount=-2.00,    type=fee,  ref=fill_id)
  (acct=fees , asset=USD,  amount=+2.00,    type=fee,  ref=fill_id)
SUM = 0 ✓  (when netted against asset dimensions; invariants checked per asset)
```

### Why double-entry (rather than "subtract from balance")
- **Auditability**: every change is traceable to a source event.
- **Reversibility**: fixing a mistake = issuing a compensating entry, never a destructive update.
- **Reconciliation**: daily closing balance = opening + sum(entries); any drift fails loud.
- **Regulatory**: required effectively by 17a-4 WORM retention and financial audit.

### Deposits / withdrawals
- **ACH**: 1–3 business days to clear; instant credit up to a risk-adjusted limit; clawback path if ACH returns (R01, R08, etc.).
- **Instant Deposits**: broker front the cash; risk engine sets per-user limits.
- **Wire**: same-day.
- **Crypto**: on-chain deposit with N-confirmation rule per asset (BTC 2, ETH 12, etc.).

### Sweep / interest
Uncollateralized cash is swept overnight to partner banks (FDIC-insured program) — another ledger transfer, reversed each morning before open.

---

## 12. Risk, Margin & Regulatory Controls

Risk checks run synchronously **before** accepting an order. Budget: < 10 ms p99 for the entire suite.

### Pre-trade risk checklist (in order)

| Check | What |
|---|---|
| Account status | active (not restricted / closed / frozen for KYC review) |
| Tradeable asset | symbol not on broker's restricted list; not halted/delisted |
| Market hours | respect RTH / ETH depending on `extended_hours` flag |
| LULD / halt | Limit Up Limit Down bands; reject orders outside band |
| SSR | Short Sale Restriction — uptick rule after 10% drop |
| Circuit breakers | Market-wide Level 1/2/3 halts → reject non-GTC orders |
| Buying power | `cash + margin_available ≥ order notional + fees` |
| Position limit | per-symbol exposure cap (risk-based) |
| PDT | ≥ 4 day-trades in 5 rolling business days & account < $25k → reject day-trading orders |
| Reg T | initial margin: 50% equity / 50% loan on marginable securities |
| Options approval | level 0 (no options), 1 (covered calls / cash secured puts), 2 (long options), 3 (spreads), 4 (naked) |
| Suitability | volatile / leveraged ETF acknowledgments for new users |
| Fat-finger | reject if notional > user's historical 99th percentile × 10, or if price is >5% off NBBO for market orders |
| Wash trade | same user self-crossing within short window |
| Sanctions / OFAC | screen order against restricted lists (crypto especially) |

### Day-trade counter (PDT)
Rolling window. Each buy-then-sell (or sell-then-buy for shorts) of the **same symbol on the same day** = 1 day-trade. On the 4th in 5 biz days for an account with equity < $25k, lock the account into closing-only mode for 90 days.

### Margin call flow
- Intraday mark-to-market of positions runs every 1 min (per shard, in-memory).
- If equity < maintenance margin (typically 25%), issue margin call: user has to deposit or liquidate by T+2 (or sooner for severe calls).
- Automated liquidation is the broker's backstop if user doesn't comply — sell biggest-loss position first (or most liquid).

### 15c3-5 "market access rule"
Broker is **legally responsible** for pre-trade controls. Hence risk checks cannot be bypassed, cannot be disabled per-user, and must be independently audited. Risk engine emits a trace log for every rejected order with the specific rule that fired — retained 7 years.

---

## 13. Corporate Actions Handling

Corporate actions are scheduled events published by NYSE's DTCC CA Web / exchanges. The broker ingests them daily and applies effects on the ex-date.

### Types
| Action | Effect |
|---|---|
| Cash dividend | Credit cash = qty × div/share on pay date; tax-lot adjustment for qualified div |
| Stock dividend / split | Multiply position qty by ratio; divide avg_cost by ratio; adjust open orders' prices/qty |
| Reverse split | Qty / ratio; cost × ratio; handle fractional residuals (cash-in-lieu) |
| Spin-off | Create new position per distribution ratio; allocate cost basis between parent & spun entity |
| Merger (cash) | Close position, credit cash |
| Merger (stock) | Swap qty of old → new at ratio |
| Rights / warrants | Grant instrument, user can exercise |
| Symbol change | Update symbol mapping; preserve positions + open orders |
| Delisting | Halt trading in symbol; manual review for residual positions |

### Processing
- Nightly job reads CA feed, generates `pending_corporate_action` records.
- On ex-date premarket, applier job:
  1. Freezes affected accounts' trading in the symbol.
  2. Replays each account's position delta → writes ledger entries of type `corp_action`.
  3. Adjusts open GTC orders (split-adjust their prices/qty to keep intent).
  4. Unfreezes.
- All operations inside a single transactional unit per account.

---

## 14. Real-Time Price Push (WebSocket Fanout)

### Fanout tier

Millions of concurrent WS connections can't fit on one server. Typical pattern:
- **WS edge** layer: 100–500 stateless edge pods, each holding 10–20k connections (tune per NIC / CPU).
- **Router / fanout** layer: subscribes to Kafka tick topics, maintains `symbol → set(edge_pods)` mapping, pushes ticks to relevant edges only.
- **Edge** holds `conn → set(symbols)` and filters incoming broadcasts.

```
                   Kafka(ticks, partitioned by symbol)
                               │
                ┌──────────────┼──────────────┐
                ▼              ▼              ▼
            Fanout-1       Fanout-2       Fanout-N     (each consumes subset of partitions)
                │              │              │
         ┌──────┴──────┐       …        ┌─────┴──────┐
         ▼             ▼                ▼            ▼
      Edge-pods  (hold user WS conns, apply user filter by symbol)
         │             │
         ▼             ▼
       Clients       Clients
```

### Connection routing
- Clients connect via DNS round-robin → NLB → Edge pod (with consistent hash on user_id for stickiness, so reconnects land on the same pod and replay is cheap).
- Inside Edge: per-user connection holds (subs, last_seq, backpressure queue).

### Backpressure
- Each WS conn has a bounded send buffer (e.g. 2 MB).
- On slow client: drop to **coalesced quote mode** — keep only the latest bid/ask per symbol, drop intermediate ticks.
- If buffer stays full > 5 s: disconnect; client reconnects with snapshot+catchup.

### Scaling
- Horizontal: add more edges; LB hash redistributes.
- Hot symbol (e.g. GME during a squeeze): partition that symbol across multiple Kafka partitions by **hash(symbol || bucket)**, fanout replicas consume each partition, edges dedupe on `(symbol, seq)`.

---

## 15. Notifications

Cross-channel delivery: push (FCM/APNS), SMS (Twilio), email (SES), in-app.

Event sources:
- Order events (filled, cancelled, rejected, expired).
- Price alerts (user-defined thresholds).
- Account events (margin call, dividend credited, deposit settled).
- Regulatory / account restrictions.

Architecture:
```
Kafka events → Notification Orchestrator → user preferences + dedup + rate-limit
                                         → Channel dispatchers (push/sms/email/in-app)
```
Critical rules:
- **Once per event** (dedup by event_id across retries).
- **Priority queues**: margin call > order fill > price alert > marketing.
- **Do-not-disturb** / quiet hours honored for non-urgent.
- Delivery receipts fed back to analytics.

---

## 16. Database & Storage Architecture

### Polyglot persistence — right store per workload

| Component | Store | Why |
|---|---|---|
| Accounts, orders, fills, ledger, positions | **Postgres** (sharded by account_id, HA via Patroni or Aurora) | ACID, transactional integrity, mature ops |
| Tick archive (raw) | **S3 Parquet** partitioned by date/symbol | cheap, queryable by Athena/Presto |
| Candles / historical OHLCV | **Druid** (or TimescaleDB / InfluxDB) | fast time-series rollups |
| Real-time quotes cache | **Redis Cluster** | µs reads, ephemeral |
| Sessions, idempotency, rate-limits | **Redis** | TTL, atomic ops |
| Watchlists, preferences, news | **DynamoDB** (or Cassandra) | key-value, regional, high QPS |
| Search (symbols, companies, news) | **Elasticsearch** | full-text, autocomplete |
| Event bus / CDC | **Kafka** (MSK) | durability, replay, partitioning |
| Analytics / reporting | **Snowflake** / **BigQuery** | columnar, BI / data-science |
| Regulatory WORM retention | **S3 Object Lock** (compliance mode) | 17a-4 requires immutable 7-year |

### Sharding of orders/fills
- Shard key: `account_id` — natural locality, all of a user's data co-located.
- Global index for `order_id` → shard via a thin mapping table (or encode shard in the ID).
- Cross-shard reads (e.g., regulatory queries by symbol across all accounts) go through an analytics replica (ETL'd to Snowflake).

### Hot storage tiering
- Last 90 days of orders/fills in hot Postgres shards.
- 90 days – 2 years in cheap "warm" Postgres (fewer indexes, slower disks).
- > 2 years: Parquet on S3 (queryable via Athena).

---

## 17. Caching Architecture

| Layer | Cache | Contents | TTL / Invalidation |
|---|---|---|---|
| Client | LocalStorage / SQLite | UI state, last-seen quotes | app-managed |
| CDN | CloudFront / Fastly | Static assets, public quotes snapshot for unauthenticated visits | 5–15 s for quotes; hours for statics |
| Edge cache (API gateway) | Redis | Session tokens, rate-limit counters | TTL-based |
| Quote cache | Redis Cluster | Latest NBBO per symbol | updated by market-data fanout, no TTL (overwrite on tick) |
| Positions / portfolio | Redis | `positions:{account}:{symbol}` | invalidated on fill event |
| Reference data (fundamentals, company info) | Redis + in-process LRU | rarely changes | TTL 1h |

Invalidation pattern: **CDC from Postgres (Debezium) → Kafka → cache invalidator** for strongly-consistent caches (positions, orders). Never cache what you can't invalidate.

---

## 18. Scalability Deep Dive

### Vertical axes of load

1. **Orders / sec** → OMS sharded by account_id; hot shards detected and re-split online.
2. **Market data ticks / sec** → Ingestor kernel-bypass; Kafka partitioned by symbol (hot symbols hashed into multiple partitions as mentioned in §14).
3. **Concurrent WS** → Edge tier horizontally scaled; connections load-balanced by hash(user_id).
4. **Read QPS (portfolio, quotes)** → Redis + read replicas.
5. **Storage** → tiered hot/warm/cold.

### Shard rebalancing
Shards addressed by consistent-hash-with-bounded-loads. New shards added = virtual-node remap — **only** accounts landing on the new vnodes migrate; the rest are untouched. Migration is:
1. Dual-write to old and new shard.
2. Backfill historical data.
3. Flip reads to new shard.
4. Stop dual-write.

### Hot-account handling
A single whale account generating sustained order flow can saturate its shard's single-writer. Mitigations:
- Rate limit per account (with generous burst).
- Split sub-accounts internally for the largest users.
- The OMS partition is CPU-bound on WAL fsync; use NVMe + group-commit.

### Black-swan event scaling (e.g., meme-stock squeeze)
- **Auto-scaler**: CPU/QPS triggers on OMS, Risk, Fanout — scale horizontally in minutes.
- **Pre-scaled headroom**: during market hours, run at ~40% utilization, not 80%.
- **Shed load gracefully**: if queue depths exceed thresholds, start returning `503 overloaded` on non-critical APIs (news, analytics) to preserve the trading path.
- **Symbol-level throttle**: per-symbol order rate caps can be tightened dynamically (Robinhood used this during GME Jan 2021 — controversial but legally mandated when clearinghouse collateral requirements spike).

---

## 19. Reliability & Fault Tolerance

### OMS: the no-loss path

- **Raft replication (3 replicas)** per OMS partition.
- Acknowledge client **only after quorum fsync** of the WAL.
- Postgres as the persistent store uses synchronous replication to a standby; standby in another AZ.
- If leader dies mid-request: follower takes over; requests in-flight but not quorum-committed **fail client-visibly** (client retries with same Idempotency-Key — safe).

### Idempotency everywhere
- Every mutating API requires `Idempotency-Key`.
- Every Kafka consumer uses event_id dedupe (Redis SET with TTL) or relies on DB unique constraint.

### Circuit breakers & bulkheads
- Between OMS and venue FIX gateways: circuit breakers per-venue so a sick venue doesn't starve others.
- Risk engine, ledger, and OMS run in separate pods / processes — crashing the ledger writer can't crash OMS.

### Graceful degradation
| Dependency down | Behavior |
|---|---|
| Market data feed primary | Fail over to secondary; if both down, show stale quotes with "delayed" banner, reject market orders (only limit with user confirmation) |
| Risk engine | OMS refuses new orders (fail closed); existing routed orders continue |
| Notifications | Non-blocking; drop messages rather than back-pressure trading |
| Analytics / Snowflake | Invisible to trading |
| Portfolio service | Return cached snapshot with "last updated" timestamp; trading unaffected |

### Chaos / game days
- Quarterly exercises: kill an AZ, kill OMS leader, corrupt a Kafka partition, force FIX disconnect.
- Validate RTO/RPO targets.

---

## 20. Multi-Region & Disaster Recovery

### Region topology
- **Active-Active** isn't feasible for the order path (single-writer per account for consistency).
- **Active-Standby** with a **warm** standby region:
  - Primary: us-east-1 (near NYSE's Mahwah / NASDAQ's Carteret datacenters — market data proximity).
  - Standby: us-west-2, receiving async Postgres replication (logical or physical) + mirrored Kafka via MirrorMaker.
- User-facing reads can serve from either region (read-local); writes always go to primary.

### Failover
- Orchestrated, not automatic — the cost of a false positive (accidentally failing over when primary is healthy) is high for financial systems.
- Runbook target: < 15 min to failover.
- DNS TTL for API endpoints kept low (30 s) for fast cutover.

### Within-region HA
- 3 AZs, everything replicated across at least 2.
- Zonal outage → degrade to remaining AZs; capacity provisioned N+1.

### RPO considerations
- Cross-region replication is async → RPO ~5–30 s typical.
- Critical ledger entries: also written to S3 cross-region bucket synchronously as belt-and-suspenders WORM backup.

---

## 21. Security & Auth

### Authentication
- Email/password + mandatory 2FA (TOTP preferred; SMS fallback discouraged for fin-tech due to SIM-swap risk).
- Biometrics (FaceID/TouchID) on mobile.
- Session tokens (JWT short-lived + refresh rotation) stored in Keychain / Android Keystore.
- Suspicious-device flow (new device → email/SMS challenge before first order).

### Authorization
- Scopes on tokens: `read:profile`, `read:portfolio`, `trade:equity`, `trade:options`, `withdraw`.
- Withdraw flow always requires step-up auth regardless of session freshness.

### Infra security
- All services in private VPC; only the API gateway and WS edge in public subnets.
- mTLS between services (SPIFFE/SPIRE or istio-based).
- Secrets in KMS-backed vault; no plaintext secrets in pods.
- Customer PII (SSN, DOB) encrypted column-level (envelope encryption) in Postgres; KMS keys per environment.

### Regulatory security
- **SOC 2 Type II**, **PCI DSS** (for card flows), **NIST 800-53** controls.
- Annual penetration test; bug bounty program.
- SIPC coverage discloses protections ($500k securities, $250k cash).

### Fraud & account takeover
- Device fingerprinting, impossible-travel detection, anomalous withdrawal patterns → step-up auth or hold.
- New bank link → 24h hold before funds usable for withdrawal.
- Crypto withdrawal: whitelist addresses with 24h cool-down after adding.

---

## 22. Observability & Operational Excellence

### Three pillars

**Metrics (Prometheus / Datadog)**
- RED per service (Rate, Errors, Duration).
- Business KPIs: orders/sec, fills/sec, rejection rate, average price improvement vs NBBO, margin call count, WS subscribers.

**Logs (structured JSON, shipped to Splunk/ELK)**
- Correlated with `trace_id` end-to-end.
- Sensitive fields redacted at source (PII, card numbers).

**Traces (OpenTelemetry → Tempo/Jaeger)**
- Every order has a trace from client → api → OMS → Risk → Router → Venue → Fill → Ledger.
- Sampling: 100% of errors + slow orders, 1% normal traffic.

### SLOs & error budgets
- API availability (market hours): 99.995% / quarter.
- Order ack latency: p99 < 200 ms.
- Market data: p99 freshness < 100 ms.
- Budget burn alerts at 50% / 75% / 100% of monthly budget.

### Reconciliation
- **Nightly**: position reconciliation with DTCC, cash reconciliation with bank partners, ledger invariant checks (sum(debits) + sum(credits) == 0 per asset).
- **Breaks** generate tickets before market open; unresolved breaks block market-open readiness.

### Runbooks & oncall
- Clear severity matrix: SEV-1 (trading halted) to SEV-4 (cosmetic).
- Regulatory incident path: any event affecting customer orders may require FINRA disclosure within 30 days.

---

## 23. Corner Cases & Hard Problems

### 1. Duplicate order from retry
Client retries POST /orders after timeout; without idempotency this could double the user's position.
**Solution**: `Idempotency-Key` header → Redis `SETNX` with the first response cached. Client safely retries; server returns identical response.

### 2. OMS leader fails mid-commit
Request is WAL-written on leader but not yet replicated.
**Solution**: Raft majority required before ack; if leader dies before quorum, followers don't apply that entry, client retry re-places the order — safe.

### 3. Split-brain during network partition
Old leader thinks it's still leader; new leader elected.
**Solution**: leases with short TTLs + fencing tokens; stale leader's writes rejected by Postgres (monotonic epoch).

### 4. Fill arrives for an unknown order (venue/state mismatch)
OMS lost the order reference (e.g., bug, race).
**Solution**: "unknown order" exception queue → manual review; fills are sacred, never dropped. Ledger posts to a suspense account until resolved.

### 5. Double-fill from venue
Venue sends two ExecutionReports with the same exec_id (or different but semantically identical).
**Solution**: `exec_id` is unique per venue per day → idempotency at fill-writer; also alert on double-execution patterns to catch venue bugs.

### 6. Partial fill at a better price than NBBO
Legitimate (internalizer improves), but need to verify vs protected quote.
**Solution**: best-ex auditing retains NBBO at time of receipt; reconciliation confirms.

### 7. Market halt during an open order
LULD halt fires on AAPL while a user has an open limit.
**Solution**: Order stays open (not cancelled) but no routing during halt; resume on reopening auction. GTC orders may be cancelled per exchange rules (e.g., LULD auctions sometimes cancel resting orders).

### 8. Corporate action on an open GTC order
AAPL splits 4:1 while user has a GTC limit @ $400.
**Solution**: split-adjust the order to 4× qty @ $100 (proportional) before market opens on ex-date.

### 9. ACH return after purchase
User deposits $1000 via ACH, immediately buys TSLA, then ACH returns as NSF 3 days later.
**Solution**: Instant Deposits are risk-limited; if return happens, broker absorbs the exposure for small amounts, attempts clawback, or locks account. Ledger entry reverses deposit; deficit becomes a negative cash balance recoverable from future activity.

### 10. User runs out of cash mid-order (fractional)
Order specifies $500 notional; by time it fills, price moved and user only has $498.
**Solution**: reserve buying power at submit time (hold the funds); release unused on cancel/partial-fill.

### 11. PDT evasion by switching accounts
User opens multiple accounts to bypass 4-day-trades-in-5-days.
**Solution**: PDT counted at SSN / tax-ID level, aggregated across all accounts.

### 12. Options exercise / assignment
ITM short call gets assigned overnight.
**Solution**: nightly OCC exercise processing creates fills at strike price, updates positions; margin implications computed before next-day open.

### 13. Symbol change (e.g., Facebook → Meta, FB → META)
Open orders, history, watchlists, tax lots all reference the old symbol.
**Solution**: symbol alias table + explicit migration job on the effective date; historical records keep the old symbol but render with a dual label.

### 14. Clock skew between services
Order timestamps, risk checks, and exchange timestamps must align.
**Solution**: PTP (Precision Time Protocol) or chrony with strict drift alarms; reject orders with > 100 ms clock skew vs NTP stratum-1.

### 15. "Flash crash" backpressure
Market moves 5% in 30 seconds; order flow spikes 100×.
**Solution**: load-shed low-priority APIs; per-symbol rate limits at order router; pre-provisioned headroom; auto-scale on queue depth, not just CPU.

### 16. Clearinghouse margin call on the broker itself
DTCC raises VaR-based collateral demand (GME event, Jan 2021) — broker may temporarily restrict new purchases in that symbol to avoid violating net-capital rule (SEC 15c3-1).
**Solution**: symbol-level "buy restriction" toggle in OMS, surfaced in the UI; legally mandated disclosures; post-incident reporting.

### 17. Stale market data on network partition
Edge can't reach market-data fanout.
**Solution**: stale-quote detection (freshness SLO violated) → clients see "quotes delayed" banner; OMS refuses market orders in the affected region (user-visible error, not silent bad fills).

### 18. User disputes a trade
"I didn't place that order."
**Solution**: full audit trail — user_agent, IP, device_id, trace_id, WAL entry, idempotency key, 2FA log — surfaced to support; SAR filed if fraud suspected.

### 19. Time-in-force edge cases
GTC order left open 90 days hits expiration.
**Solution**: scheduled TTL sweeper transitions orders to `expired` nightly; emits user notification.

### 20. Cross-day cutover / after-hours
Order placed 3:59:59 PM for DAY tif; does it expire at 4:00:00?
**Solution**: explicit session model — DAY tif means "active during the session defined by this order's venue, respecting extended hours if requested." Test carefully around DST transitions.

---

## 24. Key Trade-offs

| Decision | Chosen | Alternative | Why |
|---|---|---|---|
| Order state | Strongly consistent (Raft + Postgres sync) | Eventual via Kafka-only | Money — can't lose orders; complexity acceptable |
| Order routing | PFOF to wholesalers | Self-clearing / direct-to-exchange | Retail economics; accept the best-ex scrutiny burden |
| Positions | Materialized view rebuilt from fills | Authoritative in-place updates | Rebuildability wins for audit & bug recovery |
| Ledger | Double-entry, immutable | Single-entry balance updates | Required for audit, prevents corruption, standard in finance |
| Multi-region | Active-Standby | Active-Active | Single-writer-per-account is simpler than CRDTs for money |
| Market data | Kernel-bypass + binary | Generic HTTP feeds | Latency is a competitive differentiator |
| Database | Sharded Postgres | NewSQL (Spanner/CockroachDB) | Postgres maturity, ops familiarity; sharding complexity is known-manageable |
| WS fanout | Custom edge + Kafka | Vendor (Pusher/Ably) | Scale and cost at our level; tight control over protocol |
| Idempotency | Client-supplied key + server dedup | Server-generated only | Survives client retries across network boundaries |
| Risk checks | Synchronous, blocking | Async with post-trade | 15c3-5 legally requires pre-trade |

---

## Appendix A — Regulatory Summary

- **SEC Rule 15c3-5** — Market Access: broker must have pre-trade risk controls; cannot be bypassed.
- **SEC Rule 15c3-1** — Net Capital: broker must maintain minimum net capital; drives clearinghouse collateral posting.
- **SEC Rule 17a-4** — WORM retention of records 6–7 years (varies by record type).
- **FINRA 4210** — Margin requirements.
- **FINRA 4370** — Business continuity planning.
- **Reg T** — Initial margin 50%; maintenance 25% (FINRA minimum; brokers often higher).
- **Reg NMS Rule 611** — Order Protection Rule.
- **Reg SHO** — Short sale locate requirement; SSR (circuit-breaker uptick rule).
- **Dodd-Frank §984** — best execution.
- **SIPC** — insurance $500k securities / $250k cash (not an investment performance guarantee).
- **Pattern Day Trader** — FINRA rule: ≥ 4 day trades / 5 biz days & equity < $25k → restricted.
- **KYC / AML / BSA** — identity verification + suspicious activity reporting.
- **1099-B / 1099-DIV / 1099-INT** — annual tax reporting.

## Appendix B — Service inventory

Trading plane:
- `oms-api`, `oms-core`, `order-router`, `fix-gateway-{venue}`, `risk-engine`, `ledger-writer`, `positions-svc`, `tax-lot-svc`.

Market-data plane:
- `md-ingestor`, `md-normalizer`, `md-sequencer`, `candle-builder`, `ws-fanout`, `ws-edge`, `quote-cache`.

User plane:
- `auth-svc`, `kyc-svc`, `account-svc`, `bank-link-svc`, `notification-svc`, `search-svc`, `watchlist-svc`, `news-svc`, `bff-web`, `bff-mobile`.

Back-office plane:
- `reconciler`, `corp-actions-processor`, `clearing-exporter`, `tax-doc-generator`, `regulatory-reporter`, `analytics-etl`.

Infra / shared:
- Kafka (MSK), Postgres (Aurora), Redis Cluster, Elasticsearch, Druid, Snowflake, S3, KMS, VPC, NLB, CDN.