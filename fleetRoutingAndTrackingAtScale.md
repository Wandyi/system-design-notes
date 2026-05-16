# Fleet Routing, Tracking, and Optimization at Scale

A staff-level system design for backend services that track and execute hundreds of thousands of concurrent routes in real time, with reporting and optimization recommendations for vehicles, drivers, and route choice. Built for the operational reality of last-mile, mid-mile, and field-service fleets — where the routing problem is NP-hard, the GPS feed never stops, drivers don't appreciate being re-routed every minute, and customers have time windows that don't move.

---

## 1. Problem Framing

### 1.1 What this system is and isn't

It is the backend that:
- Plans routes for fleets of vehicles given orders, capacities, time windows, and driver constraints.
- Assigns those routes to drivers and dispatches them.
- Tracks live execution: GPS, route adherence, ETAs, geofence events.
- Detects disruptions (traffic, breakdown, cancellation, exception) and re-plans.
- Reports operational KPIs in real time and historically.
- Suggests optimizations: vehicle/driver utilization, route shape, schedule shifts.

It is not:
- A consumer-facing rideshare matching engine (different problem — instantaneous one-to-one match with surge pricing). Many primitives overlap; the optimization shape doesn't.
- A turn-by-turn navigation app. We integrate with HERE / Mapbox / Google Directions for the polyline; we don't compute turn instructions.
- A warehouse management system. We start where the parcel leaves the dock.
- A finance/billing system. We feed it operational events; it owns the money.

### 1.2 Functional requirements

- **Plan** routes for an arbitrary fleet against a daily order book. Daily plans for tens of thousands of vehicles, completed within minutes.
- **Re-plan** continuously as live events arrive: new orders, cancellations, traffic, breakdowns, weather. Sub-minute response on disruptions.
- **Track** every active route: GPS at 5–30s cadence, geofence events (entered/exited stop), stop completions, exceptions.
- **Dispatch** to driver apps with low-latency push and reliable delivery — drivers can lose connectivity for hours.
- **Compute ETAs** continuously, surfaced to dispatch, customer-facing tracking pages, and partner systems.
- **Detect deviations** from plan: late, off-route, prolonged stop, missed window.
- **Reports**: real-time ops dashboard, historical analytics, driver scorecards, customer SLA reports, cost-per-stop economics.
- **Suggestions**: surface to ops planners optimizations that would improve utilization next week, not just optimize today.

### 1.3 Non-functional requirements

- **Scale:** 200k concurrent active routes, 5M GPS pings/min steady state, 30M peak. 50M stops/day. 100k vehicles, 200k drivers across all tenants.
- **Latency:** GPS-to-dashboard < 2s p99. Re-plan on disruption < 60s p99. Initial plan for a 10k-stop cluster < 5min p99.
- **Availability:** 99.95% for tracking and dispatch. 99.9% for planning. Tracking must outlive planning — fleets in motion can't pause for our outage.
- **Durability:** RPO 0 for committed orders and completed stops. RPO < 5s for in-flight GPS.
- **Geographic spread:** multi-region, with data residency for EU and India tenants.
- **Cost ceiling:** infrastructure cost per stop measured in cents, not dollars. Optimization compute amortized across shippers.

### 1.4 Capacity envelope

Anchor numbers used through the rest of the design:

| Metric | Steady | Peak |
|---|---|---|
| Active vehicles | 100k | 150k |
| GPS pings/sec | 5k | 50k (event spike) |
| New orders/sec | 200 | 5k (campaign drop) |
| Re-plan triggers/sec | 50 | 1k |
| Stop completions/sec | 500 | 5k |
| ETA recomputes/sec | 10k | 100k |
| Concurrent routes | 200k | 300k |

50M stops/day × 200B avg payload per event × 5 events per stop ≈ 50GB/day of operational events (compressed). Storage is not the constraint; latency, geospatial query QPS, and optimization wall clock are.

---

## 2. Domain Model

### 2.1 Core entities

```
Tenant ─owns→ Fleet ─owns→ Vehicles, Drivers
Tenant ─owns→ Orders ─bundle into→ Stops
Plan ─assigns→ Route(driver, vehicle, [stops in order, with windows])
Route ─executes-as→ Trip ─emits→ TripEvents (GPS, geofence, stop, exception)
Plan ─is-a-snapshot-of→ optimization run, immutable, versioned
Suggestion ─is-a→ delta against current plan with projected impact
```

- **Order** — a customer-level intent: pickup at A, deliver to B, time windows, weight/volume, special handling.
- **Stop** — a single physical action at one location (pickup or delivery). A multi-leg order produces multiple stops.
- **Route** — an ordered sequence of stops assigned to one driver+vehicle for one shift.
- **Trip** — the *execution* of a route. The route is the plan; the trip is what actually happens.
- **Plan** — a versioned, immutable bundle of routes for a fleet for a planning window (today, the next 4 hours, the next week).
- **Driver / Vehicle** — bounded by hours-of-service rules, vehicle capacity, vehicle type compatibility, certifications, home depot.
- **Suggestion** — a proposed *change* to current state with projected impact: "swap orders X and Y between routes A and B; saves 14 miles, finishes 22 min earlier."

### 2.2 Lifecycle of a stop (illustrative)

```
   created → planned → assigned → enroute → arrived → completed
                                       ↘                   ↗
                                          delayed → exception
                                                       ↓
                                                   replanned
```

Every transition is an event written to the trip log. The state is a derived projection — same durable-execution discipline as the agent platform document, applied here to logistics.

### 2.3 Why model the route and trip separately

A common mistake is to mutate the route in place as execution diverges. Done that way, you lose the plan-vs-actual comparison that powers most reporting (on-time rate, planned vs actual miles, lateness root cause).

Plan is immutable per version; the live trip carries deviations. Re-plans produce new plan versions. Reports query both.

---

## 3. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              INGESTION PLANE                                 │
│  Driver mobile apps → Edge gateway (regional) → Kafka                        │
│  Vehicle telematics → Edge → Kafka                                           │
│  Tenant order APIs → Order ingestion service → Kafka                         │
│  Weather / traffic feeds → Adapters → Kafka                                  │
└──────────────────────────────────────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┼────────────────────────────────────────┐
│                          STREAM PROCESSING / STATEFUL                       │
│  ┌────────────────┐  ┌──────────────────┐  ┌──────────────────────────┐     │
│  │ Tracking SVC   │  │ Geofence engine  │  │  ETA service             │     │
│  │ (Flink, sharded│  │ (event matcher,  │  │  (per-vehicle ML model   │     │
│  │  by H3 cell)   │  │  H3-indexed)     │  │  + map directions cache) │     │
│  └────────────────┘  └──────────────────┘  └──────────────────────────┘     │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌────────────────┐     │
│  │ Map-matcher          │  │ Deviation detector   │  │ Disruption hub │     │
│  │ (HMM snap-to-road)   │  │ (route ↔ live diff)  │  │ (re-plan trig.)│     │
│  └──────────────────────┘  └──────────────────────┘  └────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┼────────────────────────────────────────┐
│                         CONTROL / EXECUTION PLANE                           │
│  ┌──────────────┐   ┌─────────────┐   ┌──────────────┐   ┌─────────────┐    │
│  │ Order SVC    │   │ Plan SVC    │   │ Dispatch SVC │   │ Driver API  │    │
│  │              │ ▶ │ (durable    │ ▶ │ (assignment, │ ▶ │ (push +     │    │
│  │              │   │  workflow)  │   │  notifs)     │   │  long-poll) │    │
│  └──────────────┘   └─────────────┘   └──────────────┘   └─────────────┘    │
│         │                  │                   │                            │
│         ▼                  ▼                   ▼                            │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │   - Optimization workers (VRP solvers, GPU-pooled)                  │    │
│  │   - Daily batch planner (full VRP)                                  │    │
│  │   - Continuous local-search worker (online re-opt)                  │    │
│  │   - Suggestion engine (offline what-if)                             │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┼────────────────────────────────────────┐
│                                DATA PLANE                                   │
│  Hot KV  : Redis Cluster (last-known location, ETA, in-flight route state)  │
│  Spatial : PostGIS / sharded Postgres + H3 cells indexed in Redis           │
│  TimeSer : Cassandra/ScyllaDB for GPS history and event log                 │
│  RDBMS   : Postgres (orders, plans, drivers, vehicles) with strong ACID     │
│  Lake    : S3 + Iceberg + Trino for batch analytics                         │
│  Search  : OpenSearch for ops UI (full-text on shipments, vehicles)         │
│  Map     : OSRM / Valhalla self-hosted for routing matrix; Mapbox for UI    │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┼─────────────────────────────────────────┐
│                      REPORTING / ANALYTICS / SUGGESTIONS                     │
│  Real-time OLAP : ClickHouse / Druid / Pinot — sub-second slice/dice         │
│  Batch analytics: Iceberg + Trino / Spark — historical, complex joins        │
│  ML platform    : feature store (Feast), training (Spark+TF), online (Triton)│
│  Suggestion eng : nightly + on-demand what-if VRP runs against shadow plans  │
└──────────────────────────────────────────────────────────────────────────────┘
```

The four planes are sized independently because their scaling shapes differ:
- Ingestion is QPS-bound (GPS firehose).
- Stream processing is keyed-state-bound.
- Optimization is CPU/GPU-bound and bursty.
- Reporting is storage-and-query-bound.

---

## 4. Real-Time Tracking

### 4.1 GPS ingestion

The hot path. If this stutters, every downstream consumer (dashboards, ETAs, customer pages, dispatchers) lies.

**Shape of the load.**
- 100k vehicles pinging every 10s → 10k pings/sec sustained. Peak shifts: morning rollouts in dense urban regions push 5x.
- Pings are small (~200 bytes). The challenge is QPS and stateful keying, not bandwidth.
- Drivers lose connectivity. Apps buffer locally, batch-flush on reconnect. The ingest must accept out-of-order pings up to several hours late.

**Pipeline.**
1. Driver app POSTs (or WebSocket-pushes) ping batches to **edge gateway** (regional, anycasted). Gateway authenticates, validates schema, stamps server time.
2. Gateway publishes to **Kafka** topic `gps.raw`, keyed by `vehicle_id`. Partitioned for ordering per vehicle.
3. **Tracking service** (Flink) consumes `gps.raw`. Per-vehicle keyed state holds last 60s of pings, last-known location, last-known route assignment.
4. Tracking service emits to:
   - `vehicle.location.latest` (compacted Kafka topic, also Redis hash) — last-known per vehicle.
   - `gps.cleaned` (denoised and time-corrected) → Cassandra time-series for history.
   - `gps.snapped` after map-matching → enables route-adherence checks.
5. Subscribers: ETA service, geofence engine, deviation detector, dashboards.

**Backpressure.**
- Each vehicle has a per-key rate limit at the edge: max 10 pings/sec. A misbehaving app can't poison the topic.
- Topic-level backpressure: if Flink lag exceeds threshold, edge gateway returns 429 with `Retry-After`. App buffers; tail-latency on tracking grows but the system stays correct.

**Out-of-order handling.**
- Pings carry device-time and server-time. Tracking service uses **event-time watermarking** (Flink): emits late pings into a side output reconciled within a lateness window (10 minutes).
- Late pings older than the lateness window are written to history but do not retroactively update the live state — they would mislead the live dashboard.

**Map matching.**
- Raw GPS is noisy: tunnels, urban canyons, sub-meter wander. Without snap-to-road, "off-route" alerts fire on every block.
- Hidden Markov Model over a time window (Newson-Krumm style) snaps the trajectory to the most likely sequence of road segments.
- Run as a separate Flink stage on a 30s window per vehicle. Result published to `gps.snapped`.
- Computationally non-trivial: budget 5-10ms per ping, parallelized by vehicle key. Cache the local road graph per Flink task slot.

### 4.2 Geofencing

Geofences detect: arrived at stop, departed from stop, entered restricted zone, left service area.

**Indexing.**
- All geofences indexed by **H3** cells at resolution 9 (~150m hexagons) for stops, resolution 7 (~5km) for service areas. H3 makes "what fences contain this point" an O(1) lookup against an in-memory map.
- Sharded by H3 cell prefix to keep per-shard memory bounded.

**Matching.**
- Per ping, lookup containing cells, check candidate fences, emit enter/exit events when state transitions.
- Hysteresis to suppress flapping at fence boundaries: require N consecutive pings inside before "entered."

**Why H3 over geohashes or quadtrees.**
- Hexagons have uniform neighbor distance — better for proximity queries.
- Multi-resolution makes hierarchical aggregation trivial (sum stops by parent cell).
- Used widely in industry (Uber, Lyft, DoorDash) so engineering recruiting and tooling are friendlier.

### 4.3 Last-known location and live state

- Redis hash `vehicle:<id>` with `lat`, `lng`, `heading`, `speed`, `last_seen`, `route_id`, `current_stop`, `eta_next`. Updated by Flink job from `vehicle.location.latest`.
- Read by dispatcher dashboards, customer tracking pages, partner APIs.
- Multi-region replication: vehicle's home region holds the authoritative copy; cross-region read replicas serve dashboards in other regions for global tenants.

### 4.4 ETA computation

ETAs are the single most-watched output of the system. Every dispatch decision, customer SLA, and driver coordination depends on them.

**ETA = remaining time on current segment + sum of segment times for remaining route**.

The naive approach (call directions API on every ping) costs millions per month and adds 200ms latency. The platform approach:

1. **Pre-compute the planned route polyline** at plan time. Each segment has a baseline duration from the routing engine (OSRM/Valhalla self-hosted with historical traffic).
2. **Per ping, project the vehicle's progress** onto the polyline (closest point projection on snapped GPS).
3. **Adjust the remaining time** by:
   - Live traffic delta on remaining segments (from a traffic data subscription).
   - Driver-specific speed factor (learned from history; some drivers consistently 5-10% faster).
   - Stop-service-time model (per-stop dwell time learned from historical similar stops).
4. **Republish ETA** at a controlled cadence — every ping for the *current* stop, every 60s for *future* stops. Avoid republishing every ETA on every ping; that floods dashboards.

**ETA as an ML problem.**
- Feature inputs: planned segment duration, time-of-day, day-of-week, weather, current traffic, driver, vehicle, stop type, neighborhood. Output: predicted remaining time.
- Trained nightly on completed trips. Online inference via Triton with sub-50ms p99.
- Fall back to the rule-based estimate if the model is unavailable; never serve a stale ETA without disclosure.

**Confidence intervals.**
- Customer-facing pages show "arriving by 3:47" — pessimistic 80th percentile, not the median. Showing the median means half of customers get a "missed" experience. Pessimism is a UX feature.
- Internal dispatcher views show median, p20, p80 so planners know the spread.

---

## 5. Route Planning and Optimization

This is the hard problem. The Vehicle Routing Problem (VRP) is NP-hard, real instances have soft constraints that pure solvers handle poorly, and every solution must be reproducible and explainable to a human ops team.

### 5.1 Problem variants we actually solve

- **CVRP** — capacitated VRP (vehicle volume/weight limits).
- **VRPTW** — VRP with time windows.
- **VRPPD** — pickup and delivery (bound pairs).
- **MDVRP** — multiple depots.
- **DVRP** — dynamic VRP (online insertions during the day).
- **HFVRP** — heterogeneous fleet (mixed vehicle types).
- Plus driver-hours-of-service, break rules, certification matching, customer preferences, route shape preferences (drivers don't like crisscrossing).

In practice, every customer wants their *specific* combination. The solver design must compose constraints, not hardcode a variant.

### 5.2 Solver architecture

We do not run one VRP solver. We run a **portfolio**:

1. **Insertion heuristics** (cheap, fast, online): "best insertion" inserts a new order into an existing route at the lowest-cost position. Sub-second for one order. Used for live dispatch of new orders mid-day.
2. **Local search** (Lin-Kernighan, 2-opt, or-opt swaps): improves an existing solution by small moves. Used for online re-optimization on disruptions.
3. **Metaheuristics** (simulated annealing, large neighborhood search, genetic algorithms): full optimization from cold or warm start. Minutes of wall time on the daily plan.
4. **MILP solvers** (Gurobi, CPLEX, OR-Tools CP-SAT): used for small high-value instances or for exact verification of heuristic results on a sample.
5. **Specialized solvers**: HGS (Hybrid Genetic Search) for CVRP, OR-Tools for VRPTW. Pick based on problem variant.

The dispatcher chooses the solver based on:
- Problem size (vehicles × stops).
- Time budget (30s for live re-opt, 5min for daily plan, hours for weekly what-if).
- Whether warm-starting from a prior solution.

### 5.3 Daily batch planning

Triggered: 3-4 hours before dispatch (typically 2am-4am for morning routes).

**Pipeline.**
1. **Order finalization cutoff.** Late orders are inserted online, not in batch.
2. **Cluster** orders by depot and geography (k-means on coordinates with depot affinity). Reduces a 10k-stop problem to many sub-problems of 200-500 stops, each tractable.
3. **Solve each cluster** in parallel on a worker pool. HGS/OR-Tools, 2-3 minutes per cluster, 20-50 clusters in flight.
4. **Cross-cluster repair**: sweep edges between adjacent clusters to fix obvious sub-optima (a stop that's clearly closer to a route in the next cluster).
5. **Validate** against business rules: every order assigned, every driver within hours, every vehicle within capacity.
6. **Publish plan v1** atomically. Plan service emits `plan.published` event.
7. Dispatch service picks up, schedules driver assignments for the day.

**Why cluster-then-solve, not single-solve.**
- VRP scales super-linearly. Solving 10k stops as one problem is impractical; solving 50 × 200 stops is.
- Trade-off: clusters can produce locally-optimal routes that miss global optima. The cross-cluster repair step recovers most of the gap.

**Reproducibility.**
- Every plan is versioned with: solver name+version, parameters, input snapshot hash, random seed. Re-running with the same inputs produces an identical plan.
- This matters for auditing ("why did I get this route?") and regression testing.

### 5.4 Online re-optimization

Triggered by:
- New order arrives (insertion).
- Order canceled (removal + tighten).
- Vehicle breakdown (rebalance affected stops to other vehicles).
- Major traffic incident on a planned route (re-route or re-sequence).
- Weather event affecting a region.
- Driver missed shift (rebalance).

**Constraints on online re-opt:**
- **Small change radius.** Drivers and customers hate seeing routes mutate every minute. Limit changes to: orders not yet started, future stops, future driver-vehicle pairs only. Already-assigned drivers see at most one swap per disruption.
- **Time budget.** 30 seconds. Use insertion + local-search, not full re-solve.
- **Stability heuristic.** Prefer the solution that minimally diverges from the existing plan even if slightly worse. A 2% miles increase that keeps every driver's first 5 stops the same beats a 5% miles savings that scrambles everyone.
- **Notification cost.** Each re-route notification to a driver has a real cost (annoyance, distraction). Bundle changes; don't send N messages where one will do.

### 5.5 Suggestion engine

Where the platform earns its keep beyond just "executing today's plan well."

**Daily suggestions** computed offline:
- "Vehicle V3 ran 60% utilized this week. Consolidating its routes with V7 would save $X/day."
- "Driver D14's average dwell time at customer C is 12 min vs fleet average 7 min. Either C has access issues or training opportunity."
- "Route shape on Tuesdays consistently produces deadhead miles to depot. A west-side mini-depot would amortize in 9 months."

**Weekly suggestions** from longer windows:
- "Demand on Wed AM in zone 3 is consistently underpredicted. Adding 1 vehicle reduces missed windows by 22%."
- "Driver scheduling: shifting 3 drivers to start 30min earlier saves $Y in late-window penalties."

**On-demand what-if**: ops planner asks "what if we add 5 vehicles?" or "what if we move depot to address X?" The suggestion engine runs a shadow VRP on historical data and returns projected impact with confidence intervals.

**How we surface suggestions.**
- Not as alerts. Suggestions don't page anyone.
- Weekly report to fleet managers, with one-click "apply" that creates a draft plan change for review.
- Track adoption rate per suggestion type. Suggestions ignored 20+ times in a row get demoted or deprecated — the model is overfitting or the recommendation isn't operationally feasible.

### 5.6 Constraints and the "soft constraint" reality

Real fleets violate constraints. A "hard" capacity constraint becomes "do whatever it takes to get this order out today, even if the truck is over by 3%." A staff-level system models this:

- **Hard constraints** that genuinely cannot be violated (vehicle weight limit if the regulator weighs trucks; driver hours per DOT).
- **Soft constraints** with penalty weights (preferred but not required: driver-customer matching, lunch break window, route shape).
- **Customer-overridable constraints** (exceeded with explicit dispatcher approval, logged for compliance).

The solver objective becomes weighted multi-objective: minimize miles + α·late_penalty + β·overtime_cost + γ·preference_violation, with α/β/γ tuned per tenant.

---

## 6. Dispatch and Execution

### 6.1 Driver-app communication

Drivers need:
- Current stop and next 3 stops (no need for full route — too much info, pickpocket risk).
- Live navigation to the next stop.
- Order details, customer contact, special instructions.
- Mark stop arrived / completed / exception.
- Photo/signature capture for proof of delivery.

The communication problem:
- Drivers go offline (parking garages, rural, tunnels). Sometimes for hours.
- Must work with intermittent 2-3G connectivity.
- Cannot depend on push notifications for critical state — must be pull-able.

**Design.**
- **Pull-first model.** The app polls a per-driver state endpoint at 30s cadence. State endpoint is a thin read off Redis; sub-50ms latency.
- **Push as optimization.** Push notification triggers an immediate poll, doesn't carry payload. If push fails, polling catches up within 30s.
- **Local persistence on the device.** Last known route, all stop details, are in local SQLite. Driver can complete stops offline; events queue, flushed on reconnect.
- **Long-poll for ack-style flows** (driver accepts assignment, dispatch waits for confirmation): 30s max, then fall through to next polling cycle.

**Idempotency.**
- Every event from the app carries a client-generated UUID. Server dedupes.
- This handles flaky reconnects where the app retries an event whose previous send did succeed.

**Versioned route handoff.**
- Server sends route with version. Re-plans bump version. App displays the new route only when it has fully received and acknowledged the new version. No partial state shown.

### 6.2 Dispatch service responsibilities

- Assigns drivers to vehicles to routes for a shift, respecting:
  - Driver skills/certifications (CDL, HAZMAT).
  - Vehicle compatibility (driver licensed for vehicle class).
  - Driver home depot, shift hours, lunch.
  - Driver preferences (some drivers own their routes by tenure).
- Notifies drivers (push + SMS fallback for critical).
- Receives accept/decline; reassigns if decline within window.
- Handles escalations: driver no-show, mid-shift dropout.

**Driver acceptance loop.**
- Notification fires 30 minutes pre-shift.
- Driver has 10 minutes to accept; reminder at 5 minutes.
- Decline or timeout → reassign. Hot pool of "extra board" drivers ranked by proximity, availability, prior performance.
- All decisions logged for accountability.

### 6.3 Stop execution

The stop is the unit of work. Every interesting metric ties back to stop events.

**Lifecycle.**
1. `stop.assigned` — route published with this stop in it.
2. `stop.enroute` — driver leaves prior stop heading here.
3. `stop.geofence_entered` — within 100m. Pre-arrival signal.
4. `stop.arrived` — driver presses "arrived" or geofence + low-speed for 60s (auto).
5. `stop.servicing` — between arrived and completed.
6. `stop.completed` — driver completes (with proof: photo, signature, scan).
7. (alternatives) `stop.failed`, `stop.skipped`, `stop.deferred`.

Each event carries timestamp, location, source (manual vs auto), proof artifacts. Persisted to event log; projected to OLAP store for reporting.

**Dwell time** (servicing → completed) is a key operational metric. Anomalies bubble up: unexpectedly long dwell signals customer access problems, undertrained driver, or fraud.

### 6.4 Disruption handling

A disruption is any event invalidating the current plan:

| Disruption | Detection | Response |
|---|---|---|
| Traffic incident | Traffic feed + ETA spike on routes through that segment | Re-route around if feasible; communicate revised ETA to dispatcher and customer |
| Vehicle breakdown | Driver reports + telematics (engine fault) | Pull remaining stops; reassign to nearby vehicles or re-pickup tomorrow; dispatch tow |
| Driver illness mid-route | Driver reports | Pull remaining stops; reassign; dispatch driver swap if same-day completion required |
| Customer cancellation | Customer API + ops UI | Remove stop from route; re-optimize remaining for tighter sequencing |
| New high-priority order | Order API | Insert into best route within constraints; if no fit, dispatch new vehicle or partner |
| Late order arrival to depot | Warehouse signal | Bump stop deeper into the day; if no slot fits, push to next day |
| Severe weather | Weather feed | Cancel routes in zone; communicate to customers; reschedule |
| Geofence anomaly (vehicle 100km off-route, no GPS for 1hr) | Detector | Page dispatch; check driver safety |

**The disruption hub** consumes signals, classifies them, decides response action, dispatches:
- to the **online re-opt worker** for routing changes;
- to **dispatch service** for driver communication;
- to **customer notification service** for "your delivery is now arriving by …";
- to **ops alerting** for human attention if the disruption is large enough.

The decision tree per disruption type is explicit (configurable per tenant). Adding a new disruption type means adding a row in the disruption registry, not a code change.

---

## 7. Geospatial Infrastructure

### 7.1 Sharding by H3

The fundamental geospatial choice: shard stateful services by H3 cell.

- **Tracking service:** keyed by `(tenant_id, vehicle_id)`. Per-vehicle state is local.
- **Geofence engine:** keyed by H3 cell. Each cell's fences live on one shard.
- **Spatial queries** ("what vehicles are within 5km of point X"): expand to neighboring H3 cells (k-ring) and fan out to those shards. K-ring of resolution 9 hexagons covers a few hundred meters per ring; for 5km, expand 3-4 rings.

### 7.2 Routing engine (turn-by-turn time/distance matrix)

The VRP solver consumes a **distance matrix** between every pair of stops in a problem. For 500 stops, that's 250k pair queries.

- **Self-hosted OSRM or Valhalla** with OpenStreetMap data, plus per-region traffic overlays from a paid feed.
- Pre-compute matrices in batch: a daily plan caches all pairwise times for tomorrow's stop list before the solver runs.
- Online queries (re-opt, ETA) hit a query API with sub-10ms p99 against an in-memory contracted graph.
- Per-region service: routing engine is regional (different map data, different traffic feeds). Cross-region routing for long-haul handled by a different model.

### 7.3 Traffic data integration

- Subscriptions to HERE / TomTom / Mapbox / INRIX for live traffic deltas.
- Updates segment-level free-flow-vs-current speed every 60-120s.
- Routing engine refreshes its weights; ETA service consumes deltas continuously.
- Historical traffic stored by hour-of-week tile; used in the absence of live feed and as a baseline.

### 7.4 Map data lifecycle

- OSM updates weekly; routing graphs rebuilt on a rolling schedule per region.
- New roads, closed roads, restrictions discovered via driver reports + automated detection (vehicles consistently can't traverse a planned segment).
- Map versioning: a route assignment is bound to a specific map version; replay always uses that version's graph.

---

## 8. Data Storage

### 8.1 What goes where

| Data | Store | Why |
|---|---|---|
| Last-known vehicle state | Redis | sub-ms reads, accept some loss tolerance |
| Active route in-flight state | Postgres + Redis cache | strong consistency on assignment + cache for read |
| Order book | Postgres | transactional integrity matters |
| Plans (versioned) | Postgres + S3 (large blobs) | metadata in RDBMS, full plan blob in S3 |
| GPS history (raw + snapped) | Cassandra / ScyllaDB | wide rows by `(vehicle, day)`, time-series |
| Trip event log | Cassandra | append-only, partition by `(trip_id)` |
| Geofence definitions | Postgres + Redis (H3-indexed) | low write rate, ultra-high read rate |
| OLAP for ops dashboards | ClickHouse / Druid | sub-second slice/dice |
| Long-term analytics | Iceberg on S3 + Trino | ad-hoc complex queries, ML training |
| Search (find shipments, vehicles) | OpenSearch | full-text + filter |
| Map graph | Static, in-memory per service | hot path for routing queries |

### 8.2 GPS retention

- Hot (Redis): last 5 min.
- Warm (Cassandra): 90 days.
- Cold (S3 Parquet): 7 years (compliance / claims).

90% of queries hit hot+warm. Cold is for legal/compliance retrievals, accessed infrequently.

### 8.3 Plan store

- Each plan is large (5-50MB JSON for a daily plan).
- Postgres holds the metadata row (id, version, tenant, status, summary stats, S3 URI).
- S3 holds the plan blob (compressed, immutable).
- Plans are immutable; mutations create new versions. Old versions retained 30 days for replay.

### 8.4 Multi-region and residency

- Tenant records pinned to home region.
- Cross-region replication for DR with bounded RPO.
- EU-pinned tenants: nothing leaves EU. Map data and routing engines in-region, GPS storage in-region, analytics warehouse in-region.
- Global tenants: multi-master with conflict-resolution rules per entity type. (Most logistics data is naturally region-local — a truck in Frankfurt rarely interacts with a truck in São Paulo.)

---

## 9. ML/AI Components

### 9.1 ETA model

Already covered. Trained nightly, served online with Triton, falls back to rule-based.

### 9.2 Demand forecasting

Predicts orders per zone per hour for tomorrow (and the next 7 days).

- Inputs: historical order volume, day-of-week, holidays, weather forecast, marketing calendar, external macro.
- Model: per-zone LightGBM or Prophet for baseline; per-tenant fine-tuning for high-value tenants.
- Output: feeds capacity planning (how many vehicles to dispatch tomorrow), suggestion engine (where to position vehicles preemptively).

### 9.3 Service-time / dwell-time model

Predicts how long the driver will spend at a stop. Critical for accurate ETAs and feasible plans.

- Per-stop features: stop type (residential, commercial, multi-tenant), historical dwell at this address, time-of-day, parking difficulty score, package count/weight.
- Output: mean + 80th percentile dwell time per planned stop.

### 9.4 Driver-route affinity

Some drivers are 20% faster on familiar routes. Model learns per-driver, per-zone speed factor and applies as a correction in ETAs and the planner.

Keeps drivers on familiar routes when possible (a soft constraint in the planner: penalty for assigning a driver to a zone they haven't visited in 30 days).

### 9.5 Anomaly detection

- Vehicle suddenly off-route by 5km+: classifier identifies legitimate detour (traffic) vs anomalous (theft, fraud, distress).
- Stops marked completed without GPS confirmation: scoring for fraud review.
- Driver dwell distribution drift: training opportunity or coaching trigger.

### 9.6 Route recommendation classifier

Among the top 5 candidate routes from the solver (each with similar cost), which is most likely to actually execute well? Trained on plan-vs-actual outcomes. Sometimes the second-best route on paper is the first-best in practice because of factors the solver didn't price (parking ease, customer access patterns, driver preference).

---

## 10. Reporting and Analytics

### 10.1 Real-time operations dashboard

Audience: dispatchers, ops managers.

- Live map: every vehicle plotted, color-coded by status (on-time, late, exception, idle).
- Per-driver: current stop, next stop, ETA, miles vs plan, completed/total.
- Aggregates: routes completed today, on-time rate, exceptions, vehicles down.
- Alerts: stale GPS, missed window, prolonged dwell.

**Architecture.**
- Frontend subscribes via WebSocket to a per-tenant fanout.
- Server reads from Redis (live state) and ClickHouse (rolling aggregates).
- Aggregates pre-computed in Flink → ClickHouse so the dashboard is sub-second even at 10k vehicles per tenant.

### 10.2 Customer SLA reports

- Per shipper: on-time rate by week, by route, by zone, by driver.
- Cost per stop trend.
- Exception breakdown (what caused misses).

Generated nightly into Iceberg; on-demand via Trino. For largest tenants, materialized views in ClickHouse for sub-second access.

### 10.3 Driver scorecards

- Dwell time vs peer median.
- Completion rate.
- Customer feedback aggregated.
- Safety: harsh braking, speeding, idle time (from telematics).

Not used punitively — surfaced to drivers and managers for coaching.

### 10.4 Cost economics

- Cost per mile, cost per stop, cost per delivered package.
- Driver cost (hourly + overtime), vehicle cost (depreciation + fuel + maintenance), platform cost.
- Used for:
  - Pricing decisions (does this lane / customer pay for itself?).
  - Capacity planning.
  - Negotiation with carriers / shippers.

### 10.5 Suggestion delivery

- Weekly "your fleet's optimization opportunities" report per tenant.
- Each suggestion: text rationale, projected impact ($/day, miles/day, late-rate delta), confidence, "apply as draft plan" button.
- Adoption tracking: did the customer apply it? Did the projected impact materialize?
- Adoption rate per suggestion class is the platform's product KPI.

---

## 11. Multi-Tenancy

### 11.1 Isolation

- **Data:** every store partitioned by tenant; cross-tenant reads mechanically blocked at the data layer (RLS in Postgres, per-tenant indexes elsewhere).
- **Compute:** per-tenant queues; fair-share scheduling at planning workers, dispatch workers, optimization GPUs. Cell isolation for top-N tenants.
- **Models:** per-tenant ML models for ETA / dwell / demand. Shared baseline + per-tenant fine-tuning.
- **Maps:** tenants in the same region share the routing graph (it's the same road network), but per-tenant routing preferences (avoid tolls, restrict to certain corridors) are personal.
- **Geofences:** per-tenant; some tenants have 100k geofences (every customer has one).

### 11.2 Per-tenant configuration

- Default solver parameters and constraint weights.
- SLA windows.
- Customer notification templates.
- Driver app branding.
- Tool/tracking permissions for partner integrations.
- Compliance flags (DOT for US carriers, Tachograph for EU).

### 11.3 Noisy neighbor

- A tenant with 5x volume must not delay another tenant's plans.
- Plan service runs per-tenant queues against a shared worker pool, with per-tenant concurrency caps.
- Top-1% tenants get dedicated planning capacity (cell isolation).

---

## 12. Cost Control

### 12.1 Where costs hide

- Map / routing API calls (if you don't self-host, this becomes the dominant cost).
- Cellular data on driver devices.
- Optimization compute (GPU/CPU minutes for solver runs).
- Storage and queries (GPS firehose accumulates fast).
- Notifications (SMS/push at scale).

### 12.2 Strategies

- **Self-host OSRM/Valhalla.** Replaces Google Directions API at 1-5% the cost. Loses some quality on traffic; supplement with paid traffic feed.
- **Cache routing matrices.** A new order rarely shifts the matrix meaningfully; cache pairwise times by `(origin, destination, hour-of-week)` with TTL.
- **Compress GPS aggressively.** Pings are highly compressible; columnar storage (Parquet, ClickHouse native) reduces 10x.
- **Sample old GPS.** After 30 days, downsample 10s pings to 1-min representative points; full fidelity preserved for incidents flagged for retention.
- **Adaptive ping frequency.** Vehicle stationary or slow-moving → 60s pings. Moving fast → 5s. Idle in depot → 5min. Driver app does this autonomously.
- **Solver budget caps.** Solver runs longer than budget → return current best. No solver runs forever.
- **Per-tenant cost dashboards** and budgets — same pattern as the agent platform's cost control.

### 12.3 Cost per stop economics

The platform's product KPI: keep infrastructure cost per stop in cents. At 50M stops/day:
- 1¢/stop = $5M/day infra spend (clearly too much).
- 0.1¢/stop = $50k/day = $18M/yr — manageable.
- 0.01¢/stop = $5k/day = $1.8M/yr — the target for a mature platform.

Optimization, infrastructure, and automation pull this down over time.

---

## 13. Fault Tolerance

### 13.1 Failure scenarios

| Failure | Detection | Response |
|---|---|---|
| Edge gateway region down | Health probe | Anycast routes drivers to next-nearest; DNS-level failover for partner APIs |
| Kafka cluster degradation | Lag and 5xx alerts | Producer-side circuit breaker; degrade to "in-memory only" tracking with reduced fidelity |
| Tracking service shard failure | Heartbeat | Repartition, replay last 5 min from `gps.raw` to rebuild keyed state |
| Postgres failover | Replica lag breach | Promote replica; brief write unavailability (~10s); writes retry transparently |
| Routing engine down (regional) | Health + p99 alarm | Fall back to cached matrices; planning continues with last-known traffic; ETAs degrade to plan-baseline |
| Map data corruption | Routing 5xx + plan validation failures | Roll back to previous map version; routing graph hot-swap |
| Solver pool exhausted | Queue depth | Reject re-opt with fallback "best-effort insertion only"; alert; autoscale |
| Driver app push provider down | Provider health | Fall back to SMS; pull cadence is the safety net |
| ML model serving down | Feature store lag + model 5xx | Fall back to baseline rule-based ETA / dwell models; alert |

### 13.2 Idempotency and exactly-once

- GPS pings: best-effort, dedupe on `(vehicle_id, device_time)` at the tracking service.
- Stop events: at-least-once from the app, deduped on `(trip_id, stop_id, event_type, client_uuid)`.
- Order assignments: exactly-once via Postgres transactional outbox into Kafka.
- Plan publishes: atomic — a plan either fully published or not at all.
- Driver notifications: deduped on `(driver_id, notification_id)`.

### 13.3 Tracking outlives planning

If the planning system is down, fleets must keep operating. Drivers see their last assigned route; live tracking continues; stop events flow; ETAs degrade gracefully. Re-plans queue up. When planning recovers, queued disruptions are processed.

This is the operational principle: tracking and dispatch are critical-path; planning is mission-important but tolerates minutes of downtime. The architecture reflects that — they share no failure domain.

### 13.4 Chaos drills

- Quarterly: kill a routing engine region, kill a tracking shard, kill the solver pool, kill Kafka.
- Validate: customer pages stay live, drivers complete shifts, plans recover within RTO.
- Drills uncover unspecified dependencies — every drill produces backlog tickets.

---

## 14. APIs

Sketch.

```
# Order management
POST   /v1/orders                          # create order
GET    /v1/orders/{id}                     # status
PATCH  /v1/orders/{id}                     # update (window, address)
DELETE /v1/orders/{id}                     # cancel

# Plans
POST   /v1/plans                           # request plan generation
GET    /v1/plans/{id}                      # full plan
GET    /v1/plans/{id}/versions             # version history
POST   /v1/plans/{id}/publish              # commit a draft plan

# Routes & trips
GET    /v1/routes/{id}                     # planned route
GET    /v1/trips/{id}                      # live execution
GET    /v1/trips/{id}/events?since=...     # event stream

# Tracking
GET    /v1/vehicles/{id}/location          # last known
GET    /v1/vehicles?bbox=...               # bounding-box query
WS     /v1/streams/tenant/{id}             # live updates for tenant

# Driver app
GET    /v1/me/route                        # my current route
POST   /v1/me/events                       # arrive/complete/exception
POST   /v1/me/location                     # ping batch (offline-capable)

# Suggestions
GET    /v1/suggestions?tenant=...          # current suggestions
POST   /v1/suggestions/{id}/apply          # create draft plan from suggestion

# Reports
GET    /v1/reports/sla?from=&to=           # SLA report
GET    /v1/reports/scorecards/{driver_id}  # driver scorecard
POST   /v1/reports/whatif                  # ad-hoc what-if
```

---

## 15. Walk-Throughs

### 15.1 A driver's day

- 04:00 — daily plan published. Route assigned to driver D, vehicle V, 35 stops, depot start.
- 04:10 — D's app polls, gets route v1. Acceptance prompt; D accepts.
- 06:00 — shift starts. D departs depot. GPS pings begin flowing.
- 06:15 — first stop's geofence entered. App auto-marks `arrived`. D completes; takes photo; emits `completed`.
- 06:45 — re-plan triggered: customer C cancelled their pickup; ops added a high-priority delivery 2km off the route. Online insertion runs in 8s; route v2 published. D's app receives v2 push, then poll-fetches; UI shows the change with a "1 stop added" banner and a 4-min ETA delta.
- 11:30 — D's vehicle telematics report engine fault. Disruption hub kicks: D's remaining 18 stops triaged. 12 reassigned to nearby V2 (with online re-opt of V2's route). 6 deferred to tomorrow's plan. Tow dispatched. D notified to wait at safe location.
- 14:00 — V2 completes the rescued stops. Dispatcher closes the disruption ticket.
- Evening — daily report shows: 35 planned stops, 30 completed today, 6 deferred, 1 cancelled. On-time rate 92%. Cause analysis: 1 traffic, 1 vehicle breakdown.

Every event is in the trip log, replayable, attributable.

### 15.2 A peak day

- 09:00 — flash sale launches. Order volume 5x baseline for 30 minutes.
- Order ingestion service applies per-tenant rate limits; excess orders queued.
- Online insertion solver ramps up to 50 workers (autoscale on queue depth).
- Half the new orders fit existing routes; half spawn extra-board vehicle dispatches.
- Customer-facing ETA pages: latency holds because live state reads come from Redis, sized for peak.
- Solver pool peaks at 90% utilization. Backpressure: orders take longer to be assigned; never lost.
- Aftermath: cost dashboard shows the day's compute spike; suggestion engine flags "consider scheduled extra-board for predicted flash sales."

### 15.3 A region failover

- 14:32 — Region US-West loses primary AZ. Tracking service in US-West degrades.
- 14:32:15 — health probes fail; traffic redirected to US-East via DNS / load-balancer failover.
- 14:32:30 — drivers' apps reconnect to US-East gateway. Some pings buffered locally are flushed; ingestion catches up within 60s.
- 14:33 — tracking service in US-East rehydrates state from Kafka replication (3s lag).
- 14:34 — dispatcher dashboards back to live. Customer pages show "last update X min ago" briefly.
- Operationally: zero stops missed, ETAs degraded by 1-2 min for the duration, no driver action required.

---

## 16. Trade-offs and Alternatives

### 16.1 Snap-to-road on every ping vs at query time

- **Chosen:** snap on a sliding window in stream processing, store snapped trajectory.
- **Rejected:** snap only when reporting needs it. Why: re-deriving trajectories at query time is expensive and inconsistent across consumers (each consumer might use a slightly different algorithm).

### 16.2 Single VRP solver vs solver portfolio

- **Chosen:** portfolio (insertion + local search + metaheuristic + optional MILP).
- **Rejected:** one solver. Why: VRP variants and time budgets differ widely across daily-batch vs online-disruption vs what-if. No solver is best at all.

### 16.3 Self-hosted routing engine vs SaaS directions API

- **Chosen:** self-host OSRM/Valhalla; SaaS for traffic only.
- **Rejected:** SaaS for everything. Why: cost. At 250M routing queries/day, SaaS pricing exceeds infra+ops cost of self-host by 50x. Quality gap closeable with paid traffic feed.

### 16.4 Per-tenant solver capacity vs shared

- **Chosen:** shared pool with per-tenant queues + cell isolation for top-N.
- **Rejected:** dedicated solver per tenant. Why: utilization. Most tenants need solver capacity in narrow windows; sharing amortizes. Top-N pay for dedicated to insulate from fairness debates.

### 16.5 Mutating route in place vs immutable plan + live trip

- **Chosen:** immutable versioned plan, separate live trip.
- **Rejected:** mutate in place. Why: every report becomes a temporal-difficulty puzzle. Plan vs actual is the foundation of fleet ops insight; you can't compute it if "plan" is constantly re-written.

### 16.6 Push vs poll for driver app

- **Chosen:** poll-first with push as optimization.
- **Rejected:** push as primary. Why: drivers go offline, push providers have outages, drivers ignore notifications. Polling is the floor that always works.

### 16.7 Real-time everything vs tiered freshness

- **Chosen:** tiered. Live state (last 60s) hot in Redis; rolling aggregates in ClickHouse refreshed seconds-ish; long-tail in Iceberg.
- **Rejected:** real-time OLAP for everything. Why: cost, and most queries don't need second-level freshness. Fleet manager looking at last week's on-time rate doesn't care if it's 5 minutes stale.

---

## 17. Build & Rollout Sequencing

### Phase 0 — Tracking foundation (weeks 1–4)
- Edge ingest, Kafka, basic tracking service, Redis live state.
- Driver app skeleton with GPS pinging and stop events.
- Single-region, single-tenant.

### Phase 1 — Plan & dispatch v1 (weeks 5–10)
- Order service, simple insertion-heuristic solver, plan service.
- Dispatch + driver acceptance loop.
- Live ops dashboard v1.

### Phase 2 — Real planning (weeks 11–18)
- HGS / OR-Tools daily batch planning.
- Cluster-then-solve pipeline.
- Geofences and stop event semantics finalized.
- ETA service v1 (rule-based).

### Phase 3 — Online re-optimization (weeks 19–26)
- Disruption hub.
- Online local-search solver.
- Stability heuristics for driver UX.
- Customer notifications.

### Phase 4 — Scale and multi-region (weeks 27–36)
- Multi-region, region failover.
- Cell isolation.
- Self-hosted routing engine.
- ML-based ETA.

### Phase 5 — Suggestions and analytics (weeks 37–48)
- Demand forecasting.
- Suggestion engine + adoption tracking.
- Driver scorecards.
- What-if API.

The pattern again: build the parts that fail loudly and visibly first (tracking, dispatch). Build the parts that delight slowly afterward (suggestions, ML). A platform that can't track its fleet doesn't earn the right to suggest optimizations.

---

## 18. Senior vs Staff Framing

A senior engineer designs this with one solver, one region, one tenant, real-time tracking, and gets a working last-mile system.

A staff engineer designs the system in this document and decides:
- We separate plan from trip — every reporting, audit, and SLA outcome depends on this discipline being foundational.
- We invest in self-hosted routing early — the cost trajectory of SaaS routing fails the unit economics test.
- We treat disruption handling as a first-class subsystem with its own registry and decision tree, not a pile of conditionals scattered across the dispatcher.
- Tracking outlives planning — the architecture reflects that drivers must finish their day even if our solver is down.
- Solver portfolio over single solver — the time-budget and problem-shape diversity demands it.
- Suggestion adoption rate is the platform's product KPI, not solver wall-clock or routing quality. The platform is judged by whether customers actually got better.

The staff lens treats fleet operations as a system that runs continuously, has economics, has a human workforce on the receiving end of every decision, and must remain explainable and auditable when something goes wrong on the road. The unit of work is "the fleet operated better this quarter than last," not "this PR shipped."