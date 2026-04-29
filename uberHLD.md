# Uber — Comprehensive High-Level Design

## Table of Contents

1. [Requirements & Scale](#1-requirements--scale)
2. [Back-of-Envelope Estimation](#2-back-of-envelope-estimation)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Core API Design](#4-core-api-design)
5. [Data Models](#5-data-models)
6. [Real-Time Location Service](#6-real-time-location-service)
7. [Geospatial Indexing Deep Dive](#7-geospatial-indexing-deep-dive)
8. [Driver Matching & Dispatch System](#8-driver-matching--dispatch-system)
9. [Trip Lifecycle & State Machine](#9-trip-lifecycle--state-machine)
10. [Pricing Engine (Surge & Fare Estimation)](#10-pricing-engine-surge--fare-estimation)
11. [ETA Computation & Routing](#11-eta-computation--routing)
12. [Payments & Financial Ledger](#12-payments--financial-ledger)
13. [Notifications & Real-Time Communication](#13-notifications--real-time-communication)
14. [Database & Storage Architecture](#14-database--storage-architecture)
15. [Caching Architecture](#15-caching-architecture)
16. [Scalability Deep Dive](#16-scalability-deep-dive)
17. [Reliability & Fault Tolerance Deep Dive](#17-reliability--fault-tolerance-deep-dive)
18. [Observability & Operational Excellence](#18-observability--operational-excellence)
19. [Corner Cases & Hard Problems](#19-corner-cases--hard-problems)

---

## 1. Requirements & Scale

### Functional Requirements

- **Rider**: Request a ride (pickup location, destination), see ETA, track driver in real-time, rate driver, pay
- **Driver**: Go online/offline, accept/reject ride requests, navigate to pickup/destination, see earnings
- **Matching**: Find nearest available driver for a rider, optimize for ETA and fairness
- **Pricing**: Dynamic surge pricing based on supply/demand, upfront fare estimate
- **Trip**: Full lifecycle from request → match → pickup → in-progress → completed → payment
- **Payments**: Charge rider, pay driver, handle tips, promo codes, split fares
- **Real-Time Tracking**: Both rider and driver see live map with vehicle position
- **Ratings & Feedback**: Bidirectional rating (rider ↔ driver)
- **Ride Types**: UberX, Pool/Share, Comfort, Black, XL, etc.
- **Scheduled Rides**: Book a ride for a future time

### Non-Functional Requirements

- **Availability**: 99.99% — Uber is critical infrastructure in many cities; downtime strands riders
- **Latency**: Match a driver in < 10 seconds (p95), location updates processed in < 1 second
- **Consistency**: Strong consistency for trip state and payments; eventual for location and ETA
- **Scale**: Global — 10,000+ cities, millions of concurrent drivers
- **Real-Time**: Driver positions updated every 4 seconds, shown to rider in near-real-time

### Out of Scope

- Uber Eats (food delivery) — different matching/routing model
- Freight, autonomous vehicles
- Driver onboarding / background check pipeline
- In-app chat message history (mention but don't deep-dive)

---

## 2. Back-of-Envelope Estimation

```
Users:
  Monthly Active Riders:         130 million
  Monthly Active Drivers:        5 million
  Daily Active Riders:           ~25 million
  Daily Active Drivers:          ~5 million (most drive daily or near-daily)
  Peak Concurrent Drivers:       ~3 million (active on the road)
  Peak Concurrent Riders:        ~2 million (in a ride or requesting)

Trips:
  Trips per day:                 ~25 million
  Trips per second (average):    25M / 86400 ≈ 290/sec
  Peak trips per second:         ~1,000/sec (Friday/Saturday night, NYE)
  Average trip duration:         15 minutes
  Concurrent active trips:       25M × (15/1440) ≈ 260,000

Location Updates:
  Active drivers sending GPS:    3 million
  Update frequency:              every 4 seconds
  Location writes/sec:           3M / 4 = 750,000 writes/sec
  Peak (rush hour):              ~1.2 million writes/sec

  Each update: { driver_id, lat, lng, heading, speed, timestamp } ≈ 100 bytes
  Throughput: 750K × 100 bytes = 75 MB/sec sustained, ~120 MB/sec peak

Matching:
  Ride requests/sec:             ~290/sec average, ~1,000/sec peak
  Each request: find nearest drivers within 3-5 km
    → geospatial query: "return 10 nearest available drivers to (lat, lng)"
    → 1,000 queries/sec × ~20ms each = 20 CPU-seconds/sec of geo-index work

Storage:
  Trip records:                  25M/day × 2 KB = 50 GB/day
  Location history:              750K/sec × 100 bytes × 86400 = 6.5 TB/day (raw)
  GPS traces per trip:           15 min × 1 update/4 sec = 225 points × 25M trips = 5.6B points/day
  Financial transactions:        25M trips × ~3 records each = 75M rows/day
```

**Key Insight**: The system is write-heavy for location data (750K writes/sec), read-heavy for matching/ETA (geospatial queries every request), and requires strong consistency for trip state and financial transactions. This mix of workloads demands purpose-built storage per concern.

---

## 3. High-Level Architecture

```
          ┌──────────────┐          ┌──────────────┐
          │  Rider App   │          │  Driver App  │
          └──────┬───────┘          └──────┬───────┘
                 │                         │
                 │  HTTPS/WSS              │  HTTPS/WSS (GPS stream)
                 │                         │
          ┌──────▼─────────────────────────▼─────┐
          │           API Gateway / LB           │
          │     (geo-routed to nearest cell)     │
          └──┬─────┬────┬──────┬──────┬──────┬───┘
             │     │    │      │      │      │
       ┌─────▼─┐ ┌─▼───┐│ ┌────▼───┐ ┌▼─────┐│
       │ Ride  │ │Trip ││ │Pricing │ │Pay-  ││
       │Request│ │Svc  ││ │Engine  │ │ments ││
       │ Svc   │ │     ││ │        │ │ Svc  ││
       └───┬───┘ └──┬──┘│ └────┬───┘ └──┬───┘│
           │        │   │      │        │    │
     ┌─────▼────────▼───▼──────▼────────▼────▼───┐
     │              Event Bus (Kafka)            │
     └──┬────────┬────────┬───────┬────────┬─────┘
        │        │        │       │        │
  ┌─────▼──┐ ┌───▼────┐ ┌─▼──────┐│   ┌────▼─────┐
  │Location│ │Dispatch│ │ ETA /  ││   │Analytics │
  │Service │ │/Match  │ │Routing ││   │Pipeline  │
  │        │ │Service │ │Service ││   │(Spark/   │
  │        │ │        │ │        ││   │ Flink)   │
  └───┬────┘ └───┬────┘ └───┬────┘│   └──────────┘
      │          │          │     │
  ┌───▼────┐ ┌───▼──────────▼─────▼─────────────────┐
  │GeoIndex│ │       Database Layer                 │
  │(In-Mem │ │                                      │
  │S2/Quad)│ │  PostgreSQL   Cassandra    Redis     │
  │        │ │  (trips,      (location    (caches,  │
  │        │ │   users,       history,    sessions, │
  │        │ │   payments)    analytics)  geo-index)│
  └────────┘ └──────────────────────────────────────┘
                          │
                ┌─────────▼──────────┐
                │  Object Storage    │
                │  (S3 — receipts,   │
                │   map tiles, ML    │
                │   model artifacts) │
                └────────────────────┘
```

### Service Decomposition

| Service | Responsibility | State / Data Store |
|---|---|---|
| **Ride Request Service** | Validate ride request, fare estimate, find ride type options | Stateless (calls Pricing + Dispatch) |
| **Dispatch / Matching** | Find nearest available drivers, assign to rider | In-memory geo-index + Redis |
| **Trip Service** | Trip state machine (request→match→pickup→drop→complete) | PostgreSQL (sharded by trip_id) |
| **Location Service** | Ingest GPS updates, maintain real-time driver positions | In-memory geo-index + Kafka → Cassandra |
| **Pricing Engine** | Surge calculation, fare estimation, fare finalization | Redis (surge cache) + PostgreSQL (rate cards) |
| **ETA / Routing** | Road-network routing, ETA calculation, navigation | Graph engine (OSRM/Valhalla) + caching |
| **Payments Service** | Charge rider, pay driver, handle refunds, ledger | PostgreSQL (double-entry ledger) |
| **Notification Service** | Push notifications, SMS, in-app real-time events | APNs/FCM + WebSocket/SSE |
| **User Service** | Rider/driver profiles, ratings, preferences | PostgreSQL (sharded by user_id) |

---

## 4. Core API Design

### Rider APIs

```
POST   /v1/ride/estimate
  Body: { pickup: {lat, lng}, destination: {lat, lng}, ride_type?: "uberx" }
  → 200 {
      ride_options: [
        { ride_type: "uberx",    fare_estimate: {min: 12, max: 16, currency: "USD"},
          eta_minutes: 4,  surge_multiplier: 1.0 },
        { ride_type: "comfort",  fare_estimate: {min: 18, max: 24, currency: "USD"},
          eta_minutes: 6,  surge_multiplier: 1.2 },
        { ride_type: "pool",     fare_estimate: {min: 8, max: 11, currency: "USD"},
          eta_minutes: 7,  surge_multiplier: 1.0 }
      ],
      fare_id: "fare_abc123",   ← opaque token locking in this estimate for 5 minutes
      expires_at: "..."
    }

POST   /v1/ride/request
  Body: { fare_id: "fare_abc123", ride_type: "uberx", payment_method_id: "pm_xyz",
          pickup: {lat, lng, address}, destination: {lat, lng, address} }
  → 201 { trip_id: "trip_001", status: "MATCHING", estimated_pickup_eta: 240 }

GET    /v1/ride/{trip_id}/status
  → 200 { trip_id, status, driver: { name, photo, rating, vehicle, lat, lng },
          pickup_eta, route_polyline, fare_estimate }

POST   /v1/ride/{trip_id}/cancel
  Body: { reason?: "changed_plans" }
  → 200 { cancellation_fee: 5.00, refund_amount: 0 }

POST   /v1/ride/{trip_id}/rate
  Body: { rating: 5, tip_amount?: 3.00, feedback?: "great ride" }
  → 200 { rated: true }
```

### Driver APIs

```
POST   /v1/driver/status
  Body: { status: "ONLINE" | "OFFLINE", location: {lat, lng} }
  → 200 { status: "ONLINE" }

POST   /v1/driver/location
  Body: { lat, lng, heading, speed, accuracy, timestamp }
  → 204
  (Called every 4 seconds when driver is online.
   In practice, batched: client buffers 3-5 updates, sends in one request.)

POST   /v1/driver/trip/{trip_id}/accept
  → 200 { trip_id, rider: { name, rating, pickup_location },
          route_to_pickup_polyline, estimated_pickup_minutes }

POST   /v1/driver/trip/{trip_id}/reject
  → 200 { acknowledged: true }
  (Trip re-dispatched to next nearest driver.)

POST   /v1/driver/trip/{trip_id}/arrive
  → 200 { status: "DRIVER_ARRIVED", wait_timer_started: true }

POST   /v1/driver/trip/{trip_id}/start
  → 200 { status: "IN_PROGRESS", route_to_destination_polyline }

POST   /v1/driver/trip/{trip_id}/complete
  Body: { final_location: {lat, lng}, odometer_end?: 12345 }
  → 200 { fare: { base: 3.00, distance: 8.40, time: 4.20, surge: 0,
                   tolls: 2.50, total: 18.10, rider_charged: 18.10,
                   driver_payout: 14.48, uber_fee: 3.62 } }
```

### Internal / Service-to-Service

```
POST   /internal/dispatch/find-drivers
  Body: { pickup: {lat, lng}, radius_meters: 5000, ride_type: "uberx",
          max_results: 10, exclude_driver_ids: [...] }
  → 200 { drivers: [ { driver_id, lat, lng, eta_seconds, rating, heading }, ... ] }

POST   /internal/pricing/calculate-fare
  Body: { trip_id, route_distance_meters, route_duration_seconds,
          surge_multiplier, ride_type, city_id }
  → 200 { fare_breakdown: {...}, total: 18.10 }

POST   /internal/pricing/surge
  Body: { city_id, geohash_prefix: "9q8yy" }
  → 200 { surge_multiplier: 1.8, demand_score: 0.9, supply_score: 0.3 }
```

---

## 5. Data Models

### PostgreSQL — User Service (sharded by user_id)

```sql
CREATE TABLE riders (
    rider_id        BIGINT PRIMARY KEY,     -- snowflake ID
    name            VARCHAR(150),
    email           VARCHAR(255) UNIQUE,
    phone           VARCHAR(20) UNIQUE,
    rating          NUMERIC(3,2) DEFAULT 5.00,
    total_trips     INT DEFAULT 0,
    payment_methods JSONB,                   -- [{ id, type, last4, default }]
    home_location   GEOGRAPHY(POINT, 4326),
    work_location   GEOGRAPHY(POINT, 4326),
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE drivers (
    driver_id       BIGINT PRIMARY KEY,
    name            VARCHAR(150),
    email           VARCHAR(255) UNIQUE,
    phone           VARCHAR(20) UNIQUE,
    rating          NUMERIC(3,2) DEFAULT 5.00,
    total_trips     INT DEFAULT 0,
    vehicle_id      BIGINT REFERENCES vehicles(vehicle_id),
    license_number  VARCHAR(30),
    city_id         INT,
    status          VARCHAR(20) DEFAULT 'OFFLINE',  -- ONLINE, OFFLINE, ON_TRIP
    ride_types      TEXT[],                          -- {'uberx','comfort','xl'}
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE vehicles (
    vehicle_id      BIGINT PRIMARY KEY,
    driver_id       BIGINT,
    make            VARCHAR(50),
    model           VARCHAR(50),
    year            INT,
    color           VARCHAR(30),
    license_plate   VARCHAR(20),
    capacity        INT DEFAULT 4
);
```

### PostgreSQL — Trip Service (sharded by trip_id)

```sql
CREATE TABLE trips (
    trip_id             BIGINT PRIMARY KEY,
    rider_id            BIGINT NOT NULL,
    driver_id           BIGINT,                      -- NULL until matched
    status              VARCHAR(30) NOT NULL,         -- see state machine (Section 9)
    ride_type           VARCHAR(20) NOT NULL,
    pickup_location     GEOGRAPHY(POINT, 4326),
    pickup_address      TEXT,
    destination         GEOGRAPHY(POINT, 4326),
    destination_address TEXT,
    route_polyline      TEXT,                         -- encoded polyline
    distance_meters     INT,
    duration_seconds    INT,
    surge_multiplier    NUMERIC(3,2) DEFAULT 1.00,
    fare_id             VARCHAR(64),                  -- reference to locked fare estimate
    city_id             INT,
    requested_at        TIMESTAMPTZ,
    matched_at          TIMESTAMPTZ,
    pickup_at           TIMESTAMPTZ,
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    cancelled_at        TIMESTAMPTZ,
    cancelled_by        VARCHAR(10),                  -- 'rider' or 'driver'
    created_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_trips_rider ON trips (rider_id, created_at DESC);
CREATE INDEX idx_trips_driver ON trips (driver_id, created_at DESC);
CREATE INDEX idx_trips_status ON trips (status) WHERE status IN ('MATCHING','DRIVER_ASSIGNED','IN_PROGRESS');
```

### PostgreSQL — Payments Ledger (sharded by trip_id)

```sql
-- Double-entry accounting ledger
CREATE TABLE ledger_entries (
    entry_id        BIGINT PRIMARY KEY,
    trip_id         BIGINT NOT NULL,
    account_type    VARCHAR(20) NOT NULL,    -- 'rider_charge', 'driver_payout',
                                             -- 'uber_commission', 'promo', 'tip'
    amount          NUMERIC(12,2) NOT NULL,  -- positive = credit, negative = debit
    currency        VARCHAR(3) DEFAULT 'USD',
    status          VARCHAR(20) DEFAULT 'PENDING',  -- PENDING, SETTLED, FAILED, REFUNDED
    payment_method  VARCHAR(50),
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_ledger_trip ON ledger_entries (trip_id);

-- Invariant: SUM(amount) for a trip = 0 (double-entry always balances)
-- rider_charge(−18.10) + driver_payout(+14.48) + uber_commission(+3.62) = 0
```

### Cassandra — Location History (write-heavy: 750K/sec)

```sql
CREATE TABLE driver_location_history (
    driver_id    BIGINT,
    day          DATE,                -- partition per driver per day
    timestamp    TIMESTAMP,
    lat          DOUBLE,
    lng          DOUBLE,
    heading      SMALLINT,
    speed        FLOAT,
    accuracy     FLOAT,
    trip_id      BIGINT,              -- NULL if not on a trip
    PRIMARY KEY ((driver_id, day), timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC)
  AND default_time_to_live = 2592000;  -- 30-day TTL
```

### Redis — Real-Time State

```
Driver Status:
  Key:    driver:status:{driver_id}
  Type:   Hash
  Fields: { status, lat, lng, heading, speed, last_update, trip_id, ride_types }
  TTL:    300s (driver must heartbeat every 4s; 300s = dead-man switch)

Active Trip Lookup:
  Key:    trip:active:{trip_id}
  Type:   Hash
  Fields: { rider_id, driver_id, status, pickup_lat, pickup_lng, dest_lat, dest_lng }
  TTL:    3600s

Surge Cache:
  Key:    surge:{city_id}:{geohash_4}
  Type:   String (JSON)
  Value:  { multiplier: 1.8, computed_at: timestamp }
  TTL:    120s (recomputed every 2 minutes)

Fare Lock:
  Key:    fare:{fare_id}
  Type:   Hash
  Fields: { rider_id, ride_type, estimate_min, estimate_max, surge, pickup, dest }
  TTL:    300s (fare estimate valid for 5 minutes)
```

---

## 6. Real-Time Location Service

### The Problem

3 million active drivers each sending GPS coordinates every 4 seconds. That's 750,000 writes per second, each of which must:

1. Update the driver's current position in the in-memory geo-index (for matching)
2. Be written to durable storage (for trip tracking and analytics)
3. Be pushed to any rider currently tracking this driver on the map

### Architecture

```
   Driver App                                  Rider App
   (sends every 4s)                            (receives updates)
       │                                            ▲
       ▼                                            │
  ┌──────────────┐                          ┌───────┴───────┐
  │  Location    │  publish to Kafka        │  Notification │
  │  Ingestion   │───────────────────┐      │  Service      │
  │  (stateless) │                   │      │  (WebSocket/  │
  └──────┬───────┘                   │      │   SSE push)   │
         │                           │      └───────▲───────┘
         │ update geo-index          │              │
         ▼                           │              │
  ┌──────────────┐                   │      ┌───────┴───────┐
  │  GeoIndex    │                   │      │  Trip Tracker  │
  │  (in-memory  │                   │      │  (subscribes   │
  │  per-cell    │                   │      │   to driver    │
  │  S2/Quadtree)│                   │      │   locations    │
  └──────────────┘                   │      │   for active   │
                                     │      │   trips)       │
                              ┌──────▼──┐   └───────▲───────┘
                              │  Kafka  │           │
                              │ (topic: │───────────┘
                              │  driver-│
                              │  location)
                              └────┬────┘
                                   │
                            ┌──────▼───────┐
                            │  Cassandra   │
                            │  (location   │
                            │   history)   │
                            └──────────────┘
```

### Why In-Memory Geo-Index (Not a Database Query)

```
Option A: Query PostgreSQL for nearest drivers
  SELECT driver_id, location
  FROM active_drivers
  WHERE ST_DWithin(location, ST_MakePoint(-73.98, 40.75)::geography, 5000)
  ORDER BY ST_Distance(location, ST_MakePoint(-73.98, 40.75)::geography)
  LIMIT 10;

  Problems:
  - 750K location UPDATEs/sec on the table (MVCC bloat, vacuum death)
  - Spatial index rebuild on every update
  - Query latency: 50-100ms (too slow for matching)
  - DB becomes the bottleneck

Option B: In-memory geo-index
  - Maintained in application memory on the Location Service
  - Updated 750K times/sec (in-memory hash map update = nanoseconds)
  - Queried by Dispatch Service via gRPC (< 5ms per query)
  - No disk I/O, no WAL, no vacuum
  - Horizontally partitioned by geography (city or S2 cell range)
```

### Location Update Flow (Detail)

```
Driver phone sends: { driver_id: 42, lat: 40.7589, lng: -73.9851,
                       heading: 270, speed: 12.5, accuracy: 8.0,
                       timestamp: 1713620000 }

Step 1: Location Ingestion Service (stateless, behind LB)
  - Validate payload (bounds check: lat ∈ [-90,90], lng ∈ [-180,180])
  - Check driver auth token
  - Reject if accuracy > 100m (GPS too noisy)
  - Reject if timestamp > 30s old (stale update from buffering)

Step 2: Write to Kafka (topic: driver-locations, key: driver_id)
  - Key = driver_id ensures all updates for one driver go to same partition
  - Ordering guaranteed within a partition → no out-of-order updates

Step 3: Geo-Index Updater (consumes from Kafka)
  - Update in-memory geo-index:
      geoIndex.Update(driverID=42, lat=40.7589, lng=-73.9851)
  - This removes the driver from their old S2 cell and inserts into the new one
  - Also updates Redis driver status hash:
      HSET driver:status:42 lat 40.7589 lng -73.9851 last_update 1713620000

Step 4: Trip Tracker (consumes from same Kafka topic)
  - If driver 42 is on an active trip:
      Push location to rider's WebSocket channel
  - If not on a trip: ignore (no one is watching)

Step 5: Location History Writer (consumes from same Kafka topic)
  - Batch-write to Cassandra: 1 row per GPS update
  - Used for: trip route reconstruction, driver analytics, dispute resolution
```

---

## 7. Geospatial Indexing Deep Dive

### The Core Problem

Given a GPS coordinate (rider's pickup location), find the 10 nearest available drivers within 5 km, sorted by ETA. This query runs ~1,000 times/sec at peak, with the index being mutated 750,000 times/sec.

### Approach: S2 Geometry + In-Memory Hash Map

Uber uses Google's S2 Geometry library. S2 divides the Earth's surface into hierarchical cells:

```
S2 Cell Hierarchy:

  Level 0:  6 faces of the cube (covers entire hemisphere)
  Level 1:  ~7,800 km per side
  ...
  Level 12: ~1.27 km per side     ← used for coarse city-level indexing
  Level 14: ~320 m per side       ← used for driver lookup
  Level 16: ~80 m per side        ← used for precise pickup matching
  ...
  Level 30: ~1 cm per side

Each cell has a unique 64-bit ID.
A GPS coordinate maps to exactly one cell at any given level.
Cells at the same level tile the Earth perfectly (no overlap, no gaps).
```

### Data Structure

```
type GeoIndex struct {
    mu    sync.RWMutex
    
    // cellToDrivers: S2 cell ID (level 14) → set of driver IDs in that cell
    // Level 14 cells are ~320m × 320m → typically 0-20 drivers per cell
    cellToDrivers map[s2.CellID]map[int64]struct{}
    
    // driverToCell: driver ID → current S2 cell ID
    // Used to remove driver from old cell on position update
    driverToCell  map[int64]s2.CellID
    
    // driverInfo: driver ID → current position + metadata
    driverInfo    map[int64]*DriverLocation
}

type DriverLocation struct {
    DriverID  int64
    Lat       float64
    Lng       float64
    Heading   float64
    Speed     float64
    Status    string    // "AVAILABLE", "ON_TRIP", "OFFLINE"
    RideTypes []string  // ["uberx", "comfort"]
    UpdatedAt int64
}
```

### Update Operation (750K/sec)

```
func (g *GeoIndex) Update(loc *DriverLocation) {
    newCell := s2.CellIDFromLatLng(s2.LatLngFromDegrees(loc.Lat, loc.Lng)).Parent(14)

    g.mu.Lock()
    defer g.mu.Unlock()

    // Remove from old cell
    if oldCell, exists := g.driverToCell[loc.DriverID]; exists {
        if oldCell != newCell {
            delete(g.cellToDrivers[oldCell], loc.DriverID)
            if len(g.cellToDrivers[oldCell]) == 0 {
                delete(g.cellToDrivers, oldCell) // free empty cells
            }
        }
    }

    // Insert into new cell
    if g.cellToDrivers[newCell] == nil {
        g.cellToDrivers[newCell] = make(map[int64]struct{})
    }
    g.cellToDrivers[newCell][loc.DriverID] = struct{}{}
    g.driverToCell[loc.DriverID] = newCell
    g.driverInfo[loc.DriverID] = loc
}

// Performance: ~200ns per update (hash map insert/delete)
// At 750K updates/sec: 750K × 200ns = 150ms of CPU time per second
// A single core can handle the entire write load with room to spare
```

### Nearest-Driver Query

```
func (g *GeoIndex) FindNearest(lat, lng float64, radiusMeters float64,
    rideType string, limit int) []*DriverLocation {

    center := s2.PointFromLatLng(s2.LatLngFromDegrees(lat, lng))

    // S2 RegionCoverer: finds all level-14 cells that cover the search radius
    cap := s2.CapFromCenterAngle(center, s2.Angle(radiusMeters/earthRadiusMeters))
    coverer := s2.RegionCoverer{MaxLevel: 14, MinLevel: 14, MaxCells: 100}
    covering := coverer.Covering(s2.Region(cap))

    g.mu.RLock()
    defer g.mu.RUnlock()

    // Collect all drivers in covered cells
    var candidates []*DriverLocation
    for _, cellID := range covering {
        for driverID := range g.cellToDrivers[cellID] {
            info := g.driverInfo[driverID]
            if info.Status != "AVAILABLE" {
                continue
            }
            if !containsRideType(info.RideTypes, rideType) {
                continue
            }
            candidates = append(candidates, info)
        }
    }

    // Sort by distance to pickup point
    sort.Slice(candidates, func(i, j int) bool {
        di := haversine(lat, lng, candidates[i].Lat, candidates[i].Lng)
        dj := haversine(lat, lng, candidates[j].Lat, candidates[j].Lng)
        return di < dj
    })

    if len(candidates) > limit {
        candidates = candidates[:limit]
    }
    return candidates
}

// Performance:
// 5 km radius at level 14 → covers ~50-80 cells
// 50 cells × ~10 drivers each = 500 candidates to sort
// Sort 500 items: ~5µs
// Total query time: ~20-50µs (microseconds, not milliseconds)
// At 1000 queries/sec: 1000 × 50µs = 50ms of CPU/sec (trivial)
```

### Why S2 Cells Over Other Options

```
┌──────────────────┬──────────────┬──────────────────────────────────────┐
│ Approach         │ Update Cost  │ Query Cost    │ Issues               │
├──────────────────┼──────────────┼───────────────┼──────────────────────┤
│ S2 Cells (Uber's │ O(1) hash    │ O(cells × k)  │ Best all-around.     │
│ actual choice)   │ map update   │ ~50µs         │ Uniform cell sizes.  │
│                  │ ~200ns       │               │ Hierarchical.        │
├──────────────────┼──────────────┼───────────────┼──────────────────────┤
│ Geohash          │ O(1) prefix  │ O(cells × k)  │ Non-uniform cell     │
│                  │ update       │ ~50µs         │ sizes near poles.    │
│                  │              │               │ Edge discontinuities.│
├──────────────────┼──────────────┼───────────────┼──────────────────────┤
│ Quadtree         │ O(log n)    │ O(log n + k)  │ Tree rebalancing     │
│                  │ tree insert │ ~100µs        │ under high write     │
│                  │ ~500ns      │               │ rate. Pointer chasing│
│                  │             │               │ = cache unfriendly.  │
├──────────────────┼──────────────┼───────────────┼──────────────────────┤
│ R-tree           │ O(log n)     │ O(log n + k)  │ **Rebalancing is       │
│ (PostGIS uses)   │ + rebalance  │ ~200µs        │ expensive on updates.**│
│                  │ ~2µs         │               │ Designed for read-   │
│                  │              │               │ heavy, not 750K w/s. │
├──────────────────┼──────────────┼───────────────┼──────────────────────┤
│ Brute force      │ O(1) update  │ O(n) scan     │ n = 3M drivers →     │
│ (scan all)       │              │ ~100ms        │ 100ms per query.     │
│                  │              │               │ Doesn't scale.       │
└──────────────────┴──────────────┴───────────────┴──────────────────────┘
```

### Partitioning the Geo-Index Across Machines

```
A single machine can hold all 3M drivers in memory:
  3M × 200 bytes per DriverLocation = 600 MB (fits in RAM easily)

But for fault tolerance and locality, partition by city/region:

  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
  │ Location Service│  │ Location Service│  │ Location Service │
  │ Instance A      │  │ Instance B      │  │ Instance C       │
  │                 │  │                 │  │                  │
  │ Cities:         │  │ Cities:         │  │ Cities:          │
  │ New York        │  │ San Francisco   │  │ London           │
  │ Boston          │  │ Los Angeles     │  │ Paris            │
  │ Philadelphia    │  │ Seattle         │  │ Berlin           │
  │                 │  │                 │  │                  │
  │ Drivers: ~400K  │  │ Drivers: ~300K  │  │ Drivers: ~200K   │
  │ Memory: ~80 MB  │  │ Memory: ~60 MB  │  │ Memory: ~40 MB   │
  └─────────────────┘  └─────────────────┘  └─────────────────┘

Routing:
  Rider requests from NYC → API Gateway routes to Instance A
  Based on: rider's pickup lat/lng → city lookup → instance mapping
  
Redundancy:
  Each instance has a hot standby consuming from the same Kafka partition.
  If Instance A dies, standby takes over in < 5 seconds.
  It rebuilds the geo-index from Kafka replay (last ~30 seconds of data).
```

---

## 8. Driver Matching & Dispatch System

### Matching Algorithm

```
Rider requests UberX from (40.7589, -73.9851):

Step 1: Find 10 nearest available UberX drivers
  → GeoIndex.FindNearest(40.7589, -73.9851, radius=5000m, type="uberx", limit=10)
  → Returns: [D1 (0.5km), D2 (0.8km), D3 (1.2km), ..., D10 (4.8km)]

Step 2: Compute ETA for each candidate
  For each driver: call ETA Service with (driver_location → pickup_location)
  → D1: 3 min, D2: 4 min, D3: 3 min (D3 is on a faster road despite being farther)

Step 3: Score candidates
  score = w1 × (1/ETA)              -- prefer shorter ETA
        + w2 × driver_rating         -- prefer higher-rated drivers
        + w3 × acceptance_rate        -- prefer drivers who don't reject
        - w4 × driver_idle_time       -- prioritize drivers waiting longest (fairness)
        + w5 × heading_alignment      -- prefer drivers already headed toward pickup

Step 4: Send offer to top-scored driver
  → D3 selected (best score: short ETA despite longer distance, high rating, heading toward pickup)

Step 5: Driver has 15 seconds to accept or reject
  → If D3 accepts: trip moves to DRIVER_ASSIGNED
  → If D3 rejects or times out: try D1 (next best), then D2, etc.
  → If all 10 reject: expand radius to 8 km, repeat from Step 1
```

### Dispatch State Machine

```
                          ride request arrives
                                  │
                                  ▼
                          ┌───────────────┐
                          │   MATCHING    │
                          └───────┬───────┘
                                  │
                    ┌─────────────┼─────────────┐
                    │             │             │
              best driver    no drivers     rider cancels
              found          in radius
                    │             │             │
                    ▼             ▼             ▼
            ┌──────────────┐ ┌───────────┐ ┌───────────┐
            │ OFFER_SENT   │ │ EXPAND    │ │ CANCELLED │
            │ (to driver)  │ │ RADIUS    │ └───────────┘
            └──────┬───────┘ │ (retry)   │
                   │         └─────┬─────┘
          ┌────────┼────────┐      │
          │        │        │      │ still no drivers
       accepted  rejected  timeout │
          │        │        │      ▼
          │        └────┬───┘  ┌───────────────┐
          │             │      │ NO_DRIVERS    │
          │             ▼      │ (notify rider)│
          │      try next      └───────────────┘
          │      driver
          │
          ▼
   ┌──────────────────┐
   │ DRIVER_ASSIGNED  │
   └──────────────────┘
```

### Preventing Double-Dispatch (Critical Concurrency Issue)

```
Problem:
  Driver D1 is the nearest to both Rider A and Rider B.
  Both ride requests arrive at the same time.
  Without locking, both dispatchers send an offer to D1.
  D1 accepts both → assigned to two rides simultaneously.

Solution: Optimistic locking on driver status

  Step 1: Read driver status from Redis:
    HGET driver:status:D1 → status: "AVAILABLE"

  Step 2: Atomically claim the driver using CAS (compare-and-swap):
    -- Lua script executed atomically in Redis
    EVAL "
      local status = redis.call('HGET', KEYS[1], 'status')
      if status == 'AVAILABLE' then
        redis.call('HSET', KEYS[1], 'status', 'OFFER_PENDING')
        redis.call('HSET', KEYS[1], 'pending_trip_id', ARGV[1])
        redis.call('EXPIRE', KEYS[1], 30)  -- 15s offer timeout + buffer
        return 1
      end
      return 0
    " 1 driver:status:D1 trip_001

  Step 3: If CAS returns 1 → we claimed D1, send offer
           If CAS returns 0 → D1 already claimed, try next driver

  This guarantees exactly one trip can claim a driver at a time.
  No distributed lock needed — Redis single-threaded execution is the lock.
```

---

## 9. Trip Lifecycle & State Machine

```
┌───────────┐  rider requests   ┌───────────┐  driver matched ┌─────────────────┐
│  (start)  │─────────────────→ │ MATCHING  │────────────────→│ DRIVER_ASSIGNED │
└───────────┘                   └─────┬─────┘                 └────────┬────────┘
                                      │                                │
                                rider cancels                   driver arrives
                                      │                                │
                                      ▼                                ▼
                               ┌──────────────┐              ┌─────────────────┐
                               │  CANCELLED   │              │ DRIVER_ARRIVED  │
                               │  (by rider)  │              └────────┬────────┘
                               └──────────────┘                       │
                                                              driver starts trip
                                                                      │
                                                                      ▼
                                                              ┌───────────────┐
                                                              │  IN_PROGRESS  │
                                                              └───────┬───────┘
                                                                      │
                                                              driver completes
                                                                      │
                                                                      ▼
                                                              ┌───────────────┐
                                                              │   COMPLETED   │
                                                              └───────┬───────┘
                                                                      │
                                                              fare calculated,
                                                              payment processed
                                                                      │
                                                                      ▼
                                                              ┌───────────────┐
                                                              │    SETTLED    │
                                                              └───────────────┘

Additional transitions:
  DRIVER_ASSIGNED → CANCELLED (rider cancels → cancellation fee if > 2 min)
  DRIVER_ASSIGNED → CANCELLED (driver cancels → no fee, re-dispatch)
  DRIVER_ARRIVED  → CANCELLED (rider no-show after 5 min wait → no-show fee)
  IN_PROGRESS     → CANCELLED (extremely rare: emergency, safety issue)
```

### Trip State Persistence

```
Every state transition is:
  1. Written to PostgreSQL (source of truth) with optimistic locking:

     UPDATE trips
     SET status = 'IN_PROGRESS', started_at = NOW(), version = version + 1
     WHERE trip_id = 123 AND status = 'DRIVER_ARRIVED' AND version = 5;

     If affected_rows = 0 → concurrent modification → retry with fresh read

  2. Published to Kafka: { trip_id, old_status, new_status, timestamp, actor }
     Consumers: notification service, analytics, pricing engine, etc.

  3. Updated in Redis: HSET trip:active:123 status IN_PROGRESS
     For fast status lookups (rider polling "where is my driver?")
```

---

## 10. Pricing Engine (Surge & Fare Estimation)

### Surge Pricing

```
Surge is computed per geographic area, updated every 1-2 minutes.

Input signals:
  - Demand: ride requests in this area in the last 5 minutes
  - Supply: available drivers in this area right now
  - Historical: typical demand for this area/time (day of week, time of day)
  - Events: concerts, sports games, holidays (pre-loaded from events DB)

Computation:
  For each geohash level-4 cell (~20km × 20km, subdivided to level-6 for precision):

    demand_score = current_requests / historical_avg_requests
    supply_score = current_available_drivers / historical_avg_drivers
    
    imbalance = demand_score / supply_score
    
    if imbalance < 1.0:   surge = 1.0x (no surge)
    if imbalance 1.0-1.5: surge = 1.0-1.3x
    if imbalance 1.5-2.0: surge = 1.3-1.6x
    if imbalance 2.0-3.0: surge = 1.6-2.0x
    if imbalance > 3.0:   surge = 2.0-3.0x (capped)
    
    Additional smoothing:
    - Surge changes are damped (max ±0.3x per 2-minute cycle)
    - Surge is smoothed across adjacent cells (no sharp boundaries)
    - Minimum 3 data points to trigger surge (avoid noise from 1 request)

Storage:
  Computed surge values written to Redis:
    SET surge:{city_id}:{geohash6} '{"multiplier":1.8,"updated_at":...}' EX 120

  Read on every fare estimate request: O(1) Redis GET
```

### Fare Estimation & Finalization

```
Fare Estimate (before ride):
  1. Rider enters pickup and destination
  2. ETA Service computes route: distance=8.5km, duration=18min
  3. Look up rate card for city + ride type:
       base_fare: $2.50, per_km: $1.20, per_min: $0.25, minimum_fare: $7.00
  4. Compute: base + (8.5 × $1.20) + (18 × $0.25) = $2.50 + $10.20 + $4.50 = $17.20
  5. Apply surge: $17.20 × 1.0 = $17.20
  6. Add tolls (from routing engine): $2.50
  7. Estimated fare: $19.70

  Fare estimate is LOCKED for 5 minutes (fare_id stored in Redis).
  Rider sees this upfront price. If they request within 5 min, this price is honored.
  This prevents "surge spiked 2x while I was deciding" rage.

Fare Finalization (after ride):
  1. Trip completes. Actual distance: 9.1 km (driver took a detour). Actual time: 22 min.
  2. Compare actual vs estimated:
       - If actual ≤ 120% of estimate → charge estimated (upfront) price
       - If actual > 120% of estimate → rider pays lower of actual and estimate
         (Uber absorbs the difference — driver inefficiency shouldn't penalize rider)
       - If actual < 80% of estimate → rider pays actual
         (shorter route found or traffic was light)
  3. Final fare: $19.70 (upfront price honored)
  4. Breakdown:
       rider_charged: -$19.70
       uber_commission: +$4.93 (25%)
       driver_payout: +$14.77
       (SUM = $0.00 — ledger balances)
```

---

## 11. ETA Computation & Routing

### Architecture

```
ETA computation is one of the hottest paths:
  - Called on every fare estimate (~290/sec)
  - Called for every candidate driver during matching (10 drivers × 290 = 2,900/sec)
  - Called for in-trip navigation updates (260K active trips polling every 30s = 8,700/sec)
  Total: ~12,000 ETA computations/sec average, ~40,000/sec peak

The routing engine must be FAST (< 20ms per query) and ACCURATE.
```

```
┌───────────────────────────────────────────────────────────┐
│                   ETA / Routing Service                    │
│                                                            │
│  ┌──────────────────┐    ┌─────────────────────┐           │
│  │  Road Network    │    │  Real-Time Traffic  │           │
│  │  Graph           │    │  Overlay            │           │
│  │  (OSRM/Valhalla) │    │  (edge weights      │           │
│  │                  │    │  updated every 2min │           │
│  │  Nodes: ~500M    │    │ From: GPS traces of │           │
│  │  Edges: ~1B      │    │ active Uber drivers │           │
│  │  In-memory: ~30GB│    │ + third-party feeds │           │
│  └────────┬─────────┘    └──────────┬──────────┘           │
│           │                         │                       │
│           └────────────┬────────────┘                       │
│                        │                                    │
│                        ▼                                    │
│           ┌────────────────────────┐                        │
│           │  Contraction           │                        │
│           │  Hierarchies (CH)      │                        │
│           │  + A* Search           │                        │
│           │                        │                        │
│           │  Precomputed shortcuts │                        │
│           │  reduce A* search space│                        │
│           │  from O(n) to O(√n)    │                        │
│           │                        │                        │
│           │  Result: 1-5ms per     │                        │
│           │  point-to-point route  │                        │
│           └────────────────────────┘                        │
│                                                             │
│  ETA = route_time + pickup_time + historical_adjustment     │
│                                                             │
│  ML model adjusts ETA based on:                             │
│  - Time of day, day of week                                 │
│  - Weather conditions                                       │
│  - Historically, how long does pickup take in this area?    │
│  - Driver's current speed and heading                       │
└───────────────────────────────────────────────────────────┘
```

### Uber's Real-Time Traffic Advantage

```
Uber has millions of GPS-equipped cars driving on roads right now.
Each GPS update includes speed.
This is the world's largest real-time traffic sensor network.

Every 2 minutes:
  1. Aggregate GPS traces from all drivers on each road segment
  2. Compute average speed per segment:
       segment_123: 45 km/h (based on 12 Uber cars that traversed it in last 5 min)
       segment_456: 8 km/h (congested)
  3. Update edge weights in the road graph:
       edge_123.weight = segment_123.distance / 45 km/h = 0.8 minutes
       edge_456.weight = segment_456.distance / 8 km/h = 4.5 minutes
  4. Routing queries automatically use updated weights

This gives Uber more accurate ETAs than Google Maps in areas with high Uber density,
because Google relies on Android phone GPS (passive) while Uber has active, continuous
high-frequency GPS from vehicles actually driving the roads.
```

---

## 12. Payments & Financial Ledger

### Architecture

```
         Trip Completed
              │
              ▼
    ┌──────────────────┐
    │  Fare Calculator │  Computes final fare from actual route
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │  Payment         │
    │  Orchestrator    │
    │                  │
    │  1. Charge rider │──→ Payment Gateway (Stripe/Braintree/local)
    │  2. Record ledger│──→ PostgreSQL (double-entry)
    │  3. Queue driver │──→ Driver Payout Queue (weekly batch)
    │     payout       │
    │  4. Apply promo  │──→ Promo Service (deduct from promo pool)
    │  5. Record tip   │──→ Ledger (100% goes to driver)
    └──────────────────┘

    Critical invariant:
      Rider is charged EXACTLY once.
      Driver is paid EXACTLY once.
      Ledger always balances to zero.
```

### Double-Entry Ledger

```
Every financial event creates BALANCED entries:

Trip #1001 ($19.70, 25% commission, no promo, $3 tip):

  ┌──────────────┬────────────┬──────────┐
  │ account_type │ amount     │ status   │
  ├──────────────┼────────────┼──────────┤
  │ rider_charge │ -19.70     │ SETTLED  │  ← rider's credit card charged
  │ uber_revenue │ +4.93      │ SETTLED  │  ← Uber's 25% commission
  │ driver_earn  │ +14.77     │ PENDING  │  ← driver payout queued
  │ rider_tip    │ -3.00      │ SETTLED  │  ← tip charged to rider
  │ driver_tip   │ +3.00      │ PENDING  │  ← tip goes 100% to driver
  ├──────────────┼────────────┼──────────┤
  │ SUM          │  0.00      │          │  ← always zero
  └──────────────┴────────────┴──────────┘

  This is auditable, reversible (refund = opposite entries), and guaranteed consistent.
```

### Idempotent Payment Charging

```
Problem: Network failure between Uber and Stripe.
  Uber sends charge request → timeout → did it charge or not?
  If Uber retries → rider potentially double-charged.

Solution: Idempotency key per trip payment.

  Every payment attempt includes:
    X-Idempotency-Key: "trip_1001_rider_charge_v1"

  Stripe/Braintree guarantees:
    First call with this key → processes the charge → stores result
    Subsequent calls with same key → returns stored result (no new charge)

  If Uber crashes after sending but before recording:
    On restart, retry with same idempotency key → gets the same result
    No double-charge possible.
```

---

## 13. Notifications & Real-Time Communication

### Push Channels

```
┌─────────────────────────┬─────────────────────────────────────┐
│ Event                   │ Delivery Method                     │
├─────────────────────────┼─────────────────────────────────────┤
│ Driver matched          │ Push notif (APNs/FCM) + in-app      │
│ Driver arriving (ETA)   │ In-app only (WebSocket/SSE)         │
│ Driver arrived          │ Push + SMS (critical — rider might  │
│                         │ not be looking at phone)            │
│ Trip started            │ In-app only                         │
│ Trip completed + fare   │ Push + in-app + email receipt       │
│ Surge alert             │ In-app only (when rider opens)      │
│ Promo code              │ Push + email                        │
│ Driver location updates │ WebSocket/SSE stream (every 4s)     │
│ Payment failed          │ Push + SMS + email                  │
└─────────────────────────┴─────────────────────────────────────┘
```

### Real-Time Driver Tracking (Rider's Map)

```
Rider opens app and is tracking driver:

Option A: Polling (simple, wasteful)
  Client polls GET /v1/ride/{trip_id}/status every 4 seconds
  Problem: 260K active trips × 1 poll/4sec = 65K requests/sec
  Each request: full HTTP round trip, TLS handshake overhead, stateless lookup

Option B: Server-Sent Events (SSE) — what Uber largely uses
  Client opens: GET /v1/ride/{trip_id}/stream  (Accept: text/event-stream)
  Server holds connection open, pushes updates:

    data: {"driver_lat":40.7589,"driver_lng":-73.9851,"eta_seconds":180}

    data: {"driver_lat":40.7591,"driver_lng":-73.9848,"eta_seconds":170}

  Advantages:
    - Single HTTP connection (no repeated handshakes)
    - Server pushes only when there's new data
    - Works through proxies/CDNs that support HTTP streaming
    - Auto-reconnect built into EventSource browser API
    - Simpler than WebSocket (unidirectional is sufficient)

Backend:
  Trip Tracker Service subscribes to Kafka topic for driver location updates.
  Filters for trip_id's driver.
  Pushes to rider's SSE connection.

  260K active trips = 260K concurrent SSE connections.
  Each connection: ~20 KB memory on the server.
  Total: 260K × 20 KB = 5.2 GB — easily handled by a fleet of 50 servers.
```

---

## 14. Database & Storage Architecture

### Sharding Strategy

```
┌───────────────────┬────────────────┬────────────────────────────────────┐
│ Data              │ Shard Key      │ Why                                │
├───────────────────┼────────────────┼────────────────────────────────────┤
│ Riders            │ rider_id       │ Profile lookups by ID              │
├───────────────────┼────────────────┼────────────────────────────────────┤
│ Drivers           │ driver_id      │ Profile lookups by ID              │
├───────────────────┼────────────────┼────────────────────────────────────┤
│ Trips             │ trip_id        │ Even distribution; trip lookup     │
│                   │                │ is by trip_id 99% of the time      │
├───────────────────┼────────────────┼────────────────────────────────────┤
│ Trip history      │ rider_id       │ "My past rides" needs all trips    │
│ (secondary index) │                │ for one rider co-located           │
├───────────────────┼────────────────┼────────────────────────────────────┤
│ Ledger entries    │ trip_id        │ All financial records for a trip   │
│                   │                │ must be on same shard (atomicity)  │
├───────────────────┼────────────────┼────────────────────────────────────┤
│ Location history  │ (driver_id,    │ Cassandra partition per driver     │
│                   │  day)          │ per day. ~225 rows per partition   │
│                   │                │ (15 min trip × 15 updates/min)     │
├───────────────────┼────────────────┼────────────────────────────────────┤
│ Surge data        │ (city_id,      │ Redis. Small enough to fit in one  │
│                   │  geohash)      │ cluster per city.                  │
└───────────────────┴────────────────┴────────────────────────────────────┘
```

### Database Topology

```
                    PostgreSQL (Trips Service)
                    
  ┌───────────────────────────────────────────────────┐
  │                 Shard Group 1                     │
  │                                                   │
  │   Primary (us-east-1a)                            │
  │      ├── Sync Standby (us-east-1b)     ← HA       │
  │      ├── Async Replica (us-west-2a)    ← DR       │
  │      └── Read Replica (us-east-1c)     ← reads    │
  │                                                   │
  │   Shard range: trip_id % 128 ∈ [0, 15]            │
  └───────────────────────────────────────────────────┘
  × 8 shard groups = 128 logical shards across 32 physical hosts

                    Cassandra (Location History)

  ┌────────────────────────────────────────────────────┐
  │   Ring: 12 nodes per datacenter, 2 datacenters     │
  │   Replication factor: 3 (local) + 2 (remote DC)    │
  │   Consistency: LOCAL_QUORUM for writes             │
  │   Total: 24 nodes                                  │
  │                                                    │
  │   750K writes/sec ÷ 24 nodes = ~31K writes/node    │
  │   Well within Cassandra's ~50K writes/node ceiling │
  └────────────────────────────────────────────────────┘
```

---

## 15. Caching Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       Cache Layers                               │
│                                                                  │
│  Layer 0: Client App Cache                                       │
│    - Map tiles cached on device (offline support)                │
│    - Last fare estimate, last ride details                       │
│    - Driver/rider profile photos                                 │
│                                                                  │
│  Layer 1: CDN (static content only)                              │
│    - Map tiles, profile photos, app assets                       │
│    - Not used for dynamic data (locations, ETAs)                 │
│                                                                  │
│  Layer 2: Redis Cluster (~5 TB total)                            │
│    ┌─────────────────────────────────────────────────────────┐   │
│    │  Driver Status Cache    │  1.5 TB                       │   │
│    │  3M drivers × 500B each │  TTL: 300s (heartbeat-based)  │   │
│    ├─────────────────────────┼───────────────────────────────┤   │
│    │  Active Trip Cache      │  500 GB                       │   │
│    │  260K trips × ~2 KB     │  TTL: 3600s                   │   │
│    ├─────────────────────────┼───────────────────────────────┤   │
│    │  Surge Cache            │  50 GB                        │   │
│    │  Per city + geohash     │  TTL: 120s                    │   │
│    ├─────────────────────────┼───────────────────────────────┤   │
│    │  Fare Locks             │  100 GB                       │   │
│    │  Active fare estimates  │  TTL: 300s                    │   │
│    ├─────────────────────────┼───────────────────────────────┤   │
│    │  Rate Card Cache        │  10 GB                        │   │
│    │  Per city × ride type   │  TTL: 3600s                   │   │
│    ├─────────────────────────┼───────────────────────────────┤   │
│    │  ETA Cache              │  200 GB                       │   │
│    │  (origin_cell, dest_cell) → ETA                         │   │
│    │  Coarse-grained to allow reuse. TTL: 60s                │   │
│    ├─────────────────────────┼───────────────────────────────┤   │
│    │  User Session Cache     │  200 GB                       │   │
│    │  Auth tokens, prefs     │  TTL: 86400s                  │   │
│    └─────────────────────────┴───────────────────────────────┘   │
│                                                                  │
│  Layer 3: In-Memory Application Cache                            │
│    - Geo-index (driver positions): 600 MB per instance           │
│    - Road network graph: 30 GB per ETA instance                  │
│    - Rate cards: 10 MB per instance                              │
│    These are NOT Redis — they're in-process for zero-latency     │
│    access on the hottest paths.                                  │
│                                                                  │
│  Layer 4: Database (PostgreSQL, Cassandra)                       │
│    Only reached for:                                             │
│    - Trip state mutations (writes always go to DB)               │
│    - Historical queries (past rides)                             │
│    - Ledger queries (financial audit)                            │
│    - Cache misses on cold data                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 16. Scalability Deep Dive

### Uber's Ringpop: Consistent Hashing for Stateful Services

```
Problem: The geo-index is stateful (in-memory). You can't just load-balance
randomly across instances — a driver's location must consistently route to
the same instance so the index stays accurate.

Solution: Consistent hash ring (Uber open-sourced this as "Ringpop").

                         ┌───────────────┐
                    ┌───→│  Instance A   │───┐
                    │    │  cities: NYC, │   │
                    │    │  BOS, PHL     │   │
                    │    └───────────────┘   │
            ┌───────┴───┐              ┌─────┴─────┐
            │ Hash Ring  │             │ Instance B │
            │            │             │ cities: SF,│
            │  key =     │             │ LA, SEA    │
            │  city_id   │             └─────┬─────┘
            └───────┬───┘                    │
                    │    ┌───────────────┐   │
                    └───→│  Instance C    │←─┘
                         │  cities: LDN, │
                         │  PAR, BER     │
                         └───────────────┘

When a request arrives:
  1. Extract city_id from the driver's lat/lng
  2. Hash city_id to find responsible instance on the ring
  3. Route request to that instance

When an instance dies:
  1. Its key range is automatically assigned to the next instance on the ring
  2. That instance rebuilds the geo-index from Kafka replay (last 60s of updates)
  3. During rebuild (~5-10 seconds): matching queries for those cities return
     slightly stale data (drivers positions up to 60s old)
  4. Within 60 seconds: fully caught up, no user-visible impact

When adding capacity:
  1. New instance joins the ring, takes ownership of some key ranges
  2. Existing instances stop processing those ranges
  3. New instance builds index from Kafka (no data migration needed)
```

### Handling NYE / Super Bowl Spike

```
Normal day:          25M trips, 290/sec average
NYE midnight:        ~10x surge in specific cities
                     Trip requests: 3,000-5,000/sec in NYC alone

Pre-scaling strategy:
  1. Predicted capacity planning (2 weeks before)
     - Analyze last year's NYE traffic patterns per city
     - Pre-provision 3x normal capacity for top 20 cities

  2. Auto-scaling triggers
     - Matching latency > 5s (p95) → scale dispatch workers
     - Kafka consumer lag > 100K → scale consumers
     - Redis memory > 80% → add read replicas

  3. Graceful degradation during overload
     - Disable Explore/Nearby features (non-critical)
     - Increase surge aggressively (reduces demand = reduces load)
     - Extend driver offer timeout from 15s to 20s (reduce re-dispatches)
     - Pool/Share rides disabled (simplifies matching)
     - Scheduled rides deprioritized

  4. Regional isolation
     - NYC spike doesn't affect San Francisco
     - Each city's services scale independently
     - Cross-city blast radius = zero
```

### Location Write Scaling Path

```
Current: 750K writes/sec (Kafka + Cassandra)

If Uber grows to 10M concurrent drivers (3x current):
  → 2.5M writes/sec

Kafka scaling:
  topic: driver-locations, 256 partitions
  Each partition: 2.5M / 256 ≈ 10K messages/sec (well within Kafka's ceiling)
  Brokers: 24 brokers, each handling ~10 partitions = 100K msgs/sec per broker
  If needed: add partitions + brokers (Kafka scales linearly)

Cassandra scaling:
  2.5M writes/sec ÷ 50K writes/node = 50 nodes needed
  Currently 24 → add 26 nodes
  Cassandra: add nodes to the ring, data automatically rebalances
  No downtime, no schema changes, no application changes

Geo-index scaling:
  10M drivers × 200 bytes = 2 GB in-memory
  Still fits on a single machine
  Partition by city (already done) → each instance holds ~100K-500K drivers
  20-30 instances total (with 2x redundancy = 40-60 instances)
```

---

## 17. Reliability & Fault Tolerance Deep Dive

### Cell-Based Architecture

```
Uber uses geographic cells. Each city (or group of small cities) is an independent cell.

    ┌──────────────────────────────────────────────────────────────┐
    │                     Global Services                          │
    │  User Authentication, Payment Gateway Proxy, Configuration   │
    └───────────┬──────────────────────────────┬───────────────────┘
                │                              │
    ┌───────────▼────────────┐    ┌────────────▼───────────────┐
    │     Cell: US-East      │    │      Cell: US-West         │
    │                        │    │                            │
    │  Cities: NYC, BOS,     │    │  Cities: SF, LA, SEA,      │
    │  PHL, DC, MIA, ATL     │    │  DEN, PHX, PDX             │
    │                        │    │                            │
    │  ┌──────────────────┐  │    │  ┌──────────────────┐      │
    │  │ All services     │  │    │  │ All services     │      │
    │  │ (dispatch, trip, │  │    │  │ (dispatch, trip, │      │
    │  │  pricing, ETA,   │  │    │  │  pricing, ETA,   │      │
    │  │  notifications)  │  │    │  │  notifications)  │      │
    │  └──────────────────┘  │    │  └──────────────────┘      │
    │  ┌──────────────────┐  │    │  ┌──────────────────┐      │
    │  │ DB (PG + C*)     │  │    │  │ DB (PG + C*)     │      │
    │  │ Redis, Kafka     │  │    │  │ Redis, Kafka     │      │
    │  │ Geo-Index        │  │    │  │ Geo-Index        │      │
    │  └──────────────────┘  │    │  └──────────────────┘      │
    └────────────────────────┘    └────────────────────────────┘

Key property: A ride in NYC uses ZERO resources from the US-West cell.
  NYC outage → SF continues perfectly.
  
Cross-cell traffic (rare):
  - A rider in NYC books a scheduled ride for when they land in SF
    → Request routed to US-West cell for matching/dispatch
  - Payment processing goes through global payment gateway
    (but the gateway itself is multi-region active-active)
```

### Failure Scenarios

```
┌──────────────────────────┬──────────────────────────────────────────────────────┐
│ Failure                  │ Impact & Mitigation                                  │
├──────────────────────────┼──────────────────────────────────────────────────────┤
│ Single Location Service  │ Ringpop reassigns its key range to neighbor.          │
│ instance crashes         │ Neighbor rebuilds geo-index from Kafka (< 10s).       │
│                          │ During rebuild: matching uses slightly stale data.     │
│                          │ Riders see ETA fluctuate by ±30 seconds. Invisible.  │
├──────────────────────────┼──────────────────────────────────────────────────────┤
│ Redis cluster node down  │ Redis Cluster auto-failover (< 5s).                   │
│                          │ During gap: driver status lookups fail.               │
│                          │ Dispatch falls back to geo-index only (no status      │
│                          │ filter) → may offer trip to busy driver → driver      │
│                          │ rejects → re-dispatch. Slightly slower matching.      │
├──────────────────────────┼──────────────────────────────────────────────────────┤
│ PostgreSQL shard failover│ Primary dies → sync standby promotes (10-30s).        │
│ (trips DB)               │ During failover:                                      │
│                          │  - Trip state writes fail → retry with backoff        │
│                          │  - Active trips frozen (no state transitions)         │
│                          │  - Riders see "updating trip status..." for 30s       │
│                          │  - NO trips lost (WAL replicated to standby)          │
│                          │ After promotion: all in-flight retries succeed.       │
├──────────────────────────┼───────────────────────────────────────────────────────┤
│ Kafka broker failure     │ Partition leadership moves to ISR follower.           │
│                          │ Producers retry (idempotent). Consumers rebalance.    │
│                          │ Location updates delayed by 2-5 seconds.              │
│                          │ No data loss (replication factor = 3).                │
├──────────────────────────┼──────────────────────────────────────────────────────┤
│ ETA Service overloaded   │ Circuit breaker opens.                                │
│                          │ Fall back to straight-line distance × city average     │
│                          │ speed. ETA accuracy drops from ±1 min to ±3 min.      │
│                          │ Matching still works, just less optimal driver choice.│
├──────────────────────────┼──────────────────────────────────────────────────────┤
│ Payment gateway down     │ Trip completes normally. Payment queued.               │
│ (Stripe/Braintree)       │ Driver sees "payment pending" instead of fare.        │
│                          │ Background job retries every 5 min for up to 24h.     │
│                          │ Rider charged when gateway recovers.                  │
│                          │ Driver paid on normal weekly schedule regardless.     │
├──────────────────────────┼──────────────────────────────────────────────────────┤
│ Entire cell failure      │ Global LB stops routing to dead cell.                 │
│ (regional outage)        │ Riders in affected cities see "service unavailable."  │
│                          │ No failover to other cells — rides are local.         │
│                          │ Recovery: cell comes back online, geo-index rebuilds  │
│                          │ from Kafka, active trips resume from last known state.│
│                          │ Rides in other cells: ZERO impact.                    │
├──────────────────────────┼──────────────────────────────────────────────────────┤
│ DNS failure (global)     │ CDN and app both have hardcoded fallback IPs.         │
│                          │ Mobile apps cache DNS aggressively (TTL 300s).        │
│                          │ Impact limited to new connections during outage.      │
└──────────────────────────┴──────────────────────────────────────────────────────┘
```

### Consistency Guarantees by Subsystem

```
┌───────────────────┬────────────────┬────────────────────────────────────┐
│ Subsystem         │ Consistency    │ Why                                │
├───────────────────┼────────────────┼────────────────────────────────────┤
│ Trip State        │ Strong         │ Cannot have rider see "matching"   │
│                   │ (linearizable) │ while driver sees "in progress."   │
│                   │                │ PostgreSQL with sync replication.  │
├───────────────────┼────────────────┼────────────────────────────────────┤
│ Payments / Ledger │ Strong         │ Financial data must be exact.      │
│                   │ (serializable) │ Double-entry must always balance.  │
├───────────────────┼────────────────┼────────────────────────────────────┤
│ Driver Location   │ Eventual       │ Position is inherently stale by    │
│                   │ (< 4 seconds)  │ the time it arrives (driver moved).│
│                   │                │ Exact consistency is meaningless.  │
├───────────────────┼────────────────┼────────────────────────────────────┤
│ Surge Pricing     │ Eventual       │ Surge changes gradually. A 2-min   │
│                   │ (< 2 minutes)  │ stale surge is fine. Over-precise  │
│                   │                │ surge causes user confusion anyway.│
├───────────────────┼────────────────┼────────────────────────────────────┤
│ ETA               │ Eventual       │ ETA is a prediction. Being 30s     │
│                   │ (best effort)  │ wrong is inherent, not a bug.      │
├───────────────────┼────────────────┼────────────────────────────────────┤
│ Driver Status     │ Eventual       │ "AVAILABLE" in Redis vs "ON_TRIP"  │
│ (Redis)           │ (< 1 second)   │ in trip DB → dispatch CAS catches  │
│                   │                │ conflicts. Brief inconsistency is  │
│                   │                │ handled by retry, not prevention.  │
├───────────────────┼────────────────┼────────────────────────────────────┤
│ Rider Rating      │ Eventual       │ Rating updated after trip. No      │
│                   │                │ real-time requirement.             │
└───────────────────┴────────────────┴────────────────────────────────────┘
```

---

## 18. Observability & Operational Excellence

### Key Metrics Dashboard

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Uber Operations Dashboard                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Business Metrics (real-time):                                      │
│  ├─ Completed trips/min:         420 ▲ (+5% vs last week)           │
│  ├─ Active trips right now:      261,000                            │
│  ├─ Online drivers:              3.1M                               │
│  ├─ Avg match time:              8.2s (p95: 14.1s)                 │
│  ├─ Avg ETA accuracy:            ±1.2 min                          │
│  └─ Cancellation rate:           4.2%                               │
│                                                                     │
│  Infrastructure Metrics:                                            │
│  ├─ Location ingestion rate:     748K/sec ✓                         │
│  ├─ Kafka consumer lag:          12K msgs (< 30s behind) ✓          │
│  ├─ Geo-index query latency:     p50: 0.3ms  p99: 2.1ms ✓         │
│  ├─ Trip DB write latency:       p50: 4ms    p99: 22ms ✓          │
│  ├─ Payment success rate:        99.7% ✓                           │
│  ├─ Redis hit rate:              99.2% ✓                           │
│  └─ SSE connection count:        258K ✓                             │
│                                                                     │
│  Alerts (active):                                                   │
│  ├─ ⚠ Surge > 2.5x in Manhattan (normal for Friday 6PM)            │
│  └─ ⚠ ETA service p99 > 50ms in London (investigating)             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Distributed Tracing: Anatomy of a Ride Request

```
Trace: ride_request_a1b2c3

[Rider App]  0ms
  └─ POST /v1/ride/request  (1)
      │
[API Gateway]  2ms
  └─ Auth + rate limit check  (2)
      │
[Ride Request Service]  5ms
  ├─ Validate fare_id (Redis GET)  1ms  (3)
  ├─ Call Dispatch Service  (4)
  │    │
  │  [Dispatch Service]  8ms
  │    ├─ GeoIndex.FindNearest (gRPC to Location Service)  2ms  (5)
  │    │    └─ [Location Service]  query 62 S2 cells, found 8 drivers  (6)
  │    │
  │    ├─ ETA for 8 candidates (batch gRPC to ETA Service)  12ms  (7)
  │    │    └─ [ETA Service]  8 parallel route computations  (8)
  │    │
  │    ├─ Score + rank candidates  1ms  (9)
  │    │
  │    └─ Claim top driver (Redis CAS)  1ms  (10)
  │         └─ Driver D3 claimed successfully
  │
  ├─ Create trip record (PostgreSQL INSERT)  4ms  (11)
  ├─ Publish trip_created event (Kafka)  1ms  (12)
  └─ Return { trip_id, status: MATCHING }  (13)

[Notification Service]  +50ms (async, from Kafka)
  └─ Push to Driver D3: "New ride request"  (14)
  └─ Push to Rider: "Finding your driver..."  (15)

Total synchronous latency: 35ms (well within 200ms budget)
```

---

## 19. Corner Cases & Hard Problems

### 1. GPS Drift / Spoofing

```
Problem: Driver's phone reports a GPS position that's wrong.
  - GPS drift: Phone says driver is 200m away, actually right outside.
  - GPS spoofing: Fraud — driver fakes location to get rides in surge zone.

Detection:
  1. Speed check: If driver "teleported" > 200 km/h between two updates → flag
  2. Heading consistency: GPS heading should match bearing between consecutive points
  3. Road snapping: Project GPS onto road network. If point is consistently
     off-road (building, river) → GPS is drifting
  4. Trip anomaly: Trip route deviates significantly from expected road route
     → flag for review

Mitigation:
  - ETA calculation uses road-snapped position, not raw GPS
  - Matching uses road-snapped position (driver shown on correct street)
  - Spoofing: ML model flags drivers with anomalous position patterns
    → account suspended pending review
```

### 2. Driver Accepts Then Goes Offline (Ghost Trip)

```
Problem:
  1. Driver D1 accepts trip for Rider R1
  2. D1's phone dies (battery, crash, airplane mode)
  3. Rider waits at pickup. No driver coming. No updates.

Detection:
  - Driver heartbeat (location update) expected every 4 seconds
  - If no update for 30 seconds: driver presumed offline

Resolution:
  1. t+0s:    D1 accepts trip. Status: DRIVER_ASSIGNED.
  2. t+4s:    Last location update from D1.
  3. t+34s:   No heartbeat for 30s. System triggers:
                - Trip status → DRIVER_UNREACHABLE
                - Send SMS to D1: "Are you still on the way?"
  4. t+60s:   Still no response. Auto-cancel trip. Re-dispatch:
                - Remove D1 from available pool
                - Find next nearest driver for R1
                - Push notification to R1: "Your driver changed. New ETA: 6 min"
  5. t+120s:  New driver D2 assigned. R1's experience: 2-minute delay total.

  D1 is not penalized unless this happens repeatedly (pattern = deactivation risk).
```

### 3. Surge During System Outage (Pricing Integrity)

```
Problem: Surge computation service goes down.
  All surge cache entries expire (TTL = 120s).
  Now: no surge data available.

  Option A: Default to surge = 1.0x (no surge)
    Risk: Massive demand, no supply incentive. 30-minute wait times.
          Drivers have no reason to come to high-demand areas.

  Option B: Use last known surge values (stale by >2 min)
    Risk: Demand pattern may have shifted. Surge in wrong areas.
          But better than nothing.

  Option C: Hard-coded "emergency surge" profile per city
    Pre-computed: "If surge service is down during 5-8 PM on a Friday,
    use these surge values for these zones."
    Based on historical averages.

Uber's approach: Option B + Option C as fallback.
  1. Cache TTL extended from 120s to 600s on surge service failure detection
  2. Stale surge values served for up to 10 minutes
  3. If still down after 10 min: switch to historical surge profiles
  4. Rider sees: "Prices may be slightly inaccurate. Updated prices coming soon."
```

### 4. Two Riders Request Same Driver Simultaneously

```
Problem: Covered in Section 8 (Redis CAS), but what about the edge case:
  - Rider A's request claims Driver D1 via CAS → success
  - D1 receives push notification → "Accept?"
  - D1's phone is in a tunnel → notification never arrives
  - After 15s timeout → trip offered to D2
  - D1 exits tunnel → receives stale notification → taps "Accept"

  Now D1 thinks they have a trip, but it was already re-dispatched.

Solution: Version check on accept.
  Driver taps "Accept" → app sends:
    POST /v1/driver/trip/123/accept
    Body: { offer_version: 1 }

  Server checks:
    Current offer for trip 123 is version 2 (sent to D2).
    D1's accept is for version 1 → REJECTED.
    Response: 409 Conflict { message: "This trip was reassigned." }

  D1 sees: "Trip no longer available" → returns to idle.
```

### 5. Split-Brain Driver Location (Network Partition)

```
Problem: Network partition between geo-index instances.
  Instance A (primary for NYC) can't communicate with Instance B (standby).
  
  Driver D1 sends updates:
  - Some updates reach Instance A (through one network path)
  - Some updates reach Instance B (through another path)
  
  Instance A thinks D1 is at (40.75, -73.98)
  Instance B thinks D1 is at (40.76, -73.97)
  
  If Instance A fails and B takes over: D1's position jumps.

Mitigation:
  1. Kafka is the source of truth, not the geo-index.
     Both instances consume from the same Kafka partition.
     Same partition = same order = eventual convergence.
     
  2. During partition: one instance may lag.
     Kafka consumer health check detects lag > 30s → instance declares itself unhealthy.
     Ring routes around it.
     
  3. Driver updates include monotonic sequence number.
     Index ignores updates with sequence < last_seen_sequence (reject out-of-order).
     
  4. Worst case: driver position off by a few seconds of movement.
     For matching: ±100 meters doesn't meaningfully affect which driver is "nearest."
     ETA service re-computes from latest position anyway.
```

### 6. Payment Fails After Trip Completes (Financial Integrity)

```
Problem: Trip completes. Fare = $25. Rider's credit card declines.
  Driver already drove. Uber owes the driver.

Resolution:
  1. First retry: Attempt charge again (maybe transient decline)
  2. If declined: Try rider's other payment methods on file
  3. If no valid payment method:
     - Rider's account flagged. Cannot request new rides until balance paid.
     - Push notification: "Payment failed. Update your payment method."
     - Email: invoice with payment link
     - After 7 days: escalate to collections
  
  4. Driver payout: Driver is ALWAYS paid regardless of rider payment failure.
     Uber absorbs the loss (bad debt).
     This is critical: drivers must trust they'll be paid, or they'll leave the platform.
  
  5. Scale of bad debt: ~0.1-0.5% of trips → factored into Uber's take rate (commission).
```

### 7. Mass Event Ending (Concert / Game — 50K People, 20 Drivers)

```
Problem: Concert ends at 11 PM. 50,000 people pull out phones simultaneously.
  2,000 ride requests in 60 seconds. 20 available drivers nearby.

What happens:
  1. t+0s:  2,000 requests arrive. Surge engine: demand = 2000, supply = 20.
            Surge → 3.0x (maximum cap).
  
  2. t+5s:  Only 20 drivers can be matched immediately.
            1,980 riders in MATCHING state (waiting).
  
  3. t+10s: Surge draws drivers from surrounding areas (incentive works).
            50 additional drivers heading toward venue.
  
  4. t+30s: 70 drivers now available. Match 70 more riders.
            Many riders seeing "5+ minute wait" cancel → reduces demand.
  
  5. t+60s: Surge dropping to 2.5x as supply catches up.
  
  6. t+5min: 200 drivers now in area. Surge dropping to 1.8x.
            1,500 riders served or cancelled.
  
  7. t+15min: Back to equilibrium. Surge at 1.0x.

System engineering for this scenario:
  - Queue management: Ride requests don't all hit dispatch simultaneously.
    They're queued with FIFO + priority (Uber Pass members get priority).
  - Expanding radius over time: Start with 2km radius, expand to 5km, 10km.
  - Batch matching: Instead of matching one-at-a-time, batch every 5 seconds
    and run assignment optimization across all pending requests and available drivers.
    This produces globally better matching than greedy per-request matching.
  - Predictive pre-positioning: Uber knows when concerts end (partnerships + historical data).
    30 minutes before event ends: send push to nearby drivers:
    "High demand expected at [venue] in 30 min. Head there for surge earnings."
```

### 8. Clock Skew Between Services

```
Problem: Trip state machine depends on timestamps.
  - matched_at must be after requested_at
  - started_at must be after matched_at
  - completed_at must be after started_at
  
  If Dispatch Service's clock is 2 seconds behind Trip Service:
    matched_at (12:00:00.000) < requested_at (12:00:01.500)
    This violates the invariant and breaks analytics.

Solution:
  1. NTP sync: All servers synchronized via NTP with < 10ms drift.
  2. Logical clocks: Trip state machine uses server-assigned timestamps,
     not client timestamps. The Trip Service assigns all timestamps
     from its own clock when it processes the state transition.
  3. Monotonic guarantee: Each state transition checks
     new_timestamp > MAX(previous_timestamps). If violated due to clock
     skew, use MAX + 1ms.
  4. Driver/rider app timestamps are NEVER used for state transitions.
     They're stored as client_reported_at for debugging only.
```

---

## Summary: Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Driver location index | In-memory S2 cell hash map | 750K writes/sec impossible with any DB; sub-microsecond queries |
| Location ingestion | Kafka → parallel consumers | Decouples ingestion from processing; replay for recovery |
| Trip state | PostgreSQL (sync replication) | Financial correctness requires strong consistency |
| Location history | Cassandra (LOCAL_QUORUM) | 750K writes/sec, time-series, 30-day TTL |
| Driver matching | Geo-index + ETA scoring + Redis CAS | Fast spatial lookup, optimal assignment, no double-dispatch |
| Pricing/Surge | Pre-computed per geohash, Redis cached | Must be fast (on every fare estimate) and tolerant of staleness |
| ETA/Routing | Contraction Hierarchies + live traffic | 1-5ms per route query; Uber's own GPS fleet for traffic accuracy |
| Payments | Double-entry ledger + idempotency keys | Auditable, reversible, exactly-once charging |
| Real-time tracking | Kafka → SSE push to rider | Low-latency, single connection, unidirectional is sufficient |
| Fault isolation | Cell-based (per city/region) | NYC outage doesn't affect SF; independent scaling |
| Service discovery | Ringpop (consistent hashing) | Stateful geo-index needs consistent routing; auto-failover |
| Consistency model | Strong for trips/payments; eventual for location/surge | Match consistency to business impact of staleness |

┌──────────────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│       Section        │                                                                 Key Content                                                                 │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Requirements & Scale │ 130M MAU, 25M trips/day, 3M concurrent drivers, 99.99% availability target                                                                  │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Back-of-Envelope     │ 750K location writes/sec, 1K trip requests/sec peak, 6.5 TB/day GPS data, 260K concurrent active trips                                      │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Architecture Diagram │ Full service decomposition — 9 services with data store mapping, Kafka event bus, geo-index layer                                           │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ API Design           │ Rider APIs (estimate, request, cancel, rate), Driver APIs (status, location, accept/reject/arrive/start/complete), internal                 │
│                      │ dispatch/pricing APIs                                                                                                                       │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Data Models          │ PostgreSQL schemas (riders, drivers, trips, double-entry ledger), Cassandra (location history with TTL), Redis structures (driver status,   │
│                      │ surge cache, fare locks, active trips)                                                                                                      │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Real-Time Location   │ 750K writes/sec pipeline: Driver → Ingestion → Kafka → Geo-Index + Cassandra + Trip Tracker → Rider SSE push. Why in-memory beats any       │
│ Service              │ database.                                                                                                                                   │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Geospatial Indexing  │ S2 Geometry deep dive: cell hierarchy, hash map data structure, Update at 200ns, FindNearest at 50µs. Comparison vs                         │
│                      │ Geohash/Quadtree/R-tree. Partitioning by city.                                                                                              │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Driver Matching &    │ 5-step algorithm (find → ETA → score → offer → retry), dispatch state machine, Redis CAS to prevent double-dispatch (Lua script for atomic  │
│ Dispatch             │ claim)                                                                                                                                      │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Trip State Machine   │ Full lifecycle diagram: MATCHING → DRIVER_ASSIGNED → ARRIVED → IN_PROGRESS → COMPLETED → SETTLED, with all cancellation edges. Optimistic   │
│                      │ locking on transitions.                                                                                                                     │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Pricing Engine       │ Surge computation (demand/supply per geohash, 2-min cycles, damping, smoothing), fare estimation with upfront price locking, fare           │
│                      │ finalization with 120% rule                                                                                                                 │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ ETA & Routing        │ Contraction Hierarchies for 1-5ms route queries, Uber's real-time traffic from its own GPS fleet (more accurate than Google Maps), 30 GB    │
│                      │ in-memory road graph                                                                                                                        │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Payments             │ Double-entry ledger (always balances to zero), idempotent charging via idempotency keys, driver always paid regardless of rider payment     │
│                      │ failure                                                                                                                                     │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Real-Time            │ SSE vs WebSocket vs polling comparison, 260K concurrent SSE connections for driver tracking, push/SMS/email per event type                  │
│ Communication        │                                                                                                                                             │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Database             │ Sharding per concern (trip_id for trips, driver_id+day for locations), PostgreSQL 128-shard topology, Cassandra 24-node ring                │
│ Architecture         │                                                                                                                                             │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Caching              │ 5 TB Redis across 7 purpose-specific caches + in-process caches (geo-index 600MB, road graph 30GB). Layered: client → CDN → Redis →         │
│                      │ in-memory → DB                                                                                                                              │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Scalability          │ Ringpop consistent hashing for stateful geo-index, NYE spike handling (pre-scaling + auto-scaling + graceful degradation), location write   │
│                      │ path scaling to 10M drivers                                                                                                                 │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Fault Tolerance      │ Cell-based architecture by city, 8 failure scenarios with detailed mitigations, consistency guarantees per subsystem (strong for            │
│                      │ trips/payments, eventual for location/surge)                                                                                                │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Observability        │ Ops dashboard metrics, full distributed trace of a ride request (35ms total, 15 spans)                                                      │
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                      │ 8 hard problems: GPS drift/spoofing, ghost trips (driver goes offline), surge during outage, simultaneous dispatch (offer versioning),      │
│ Corner Cases         │ split-brain location, payment failure after trip, mass event ending (50K people + batch matching + predictive pre-positioning), clock skew  │
│                      │ between services                                                                                                                            │
└──────────────────────┴─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘

create an agentic setup with calude code, which can address following issues..



1) Add an entry to the articles table in news_data database in postgres (postgres@localhost:5532).
2) Create a new table in the database called articles_view.
3) 