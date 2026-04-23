# Global Traffic Routing Across Multi-AZ & Multi-Region
## Comprehensive Design for Amazon-Scale E-Commerce

## Table of Contents

1. [Requirements & Scale](#1-requirements--scale)
2. [Routing Architecture Overview](#2-routing-architecture-overview)
3. [DNS Layer — The First Routing Decision](#3-dns-layer--the-first-routing-decision)
4. [Edge Layer — CDN & Global Accelerator](#4-edge-layer--cdn--global-accelerator)
5. [Load Balancing — The Three-Tier Stack](#5-load-balancing--the-three-tier-stack)
6. [Availability Zone Architecture](#6-availability-zone-architecture)
7. [Multi-Region Architecture](#7-multi-region-architecture)
8. [Cell-Based Architecture & Blast Radius Isolation](#8-cell-based-architecture--blast-radius-isolation)
9. [Data Layer Routing](#9-data-layer-routing)
10. [Service Mesh & Service-to-Service Routing](#10-service-mesh--service-to-service-routing)
11. [Health Checking at Every Layer](#11-health-checking-at-every-layer)
12. [Failover Strategies](#12-failover-strategies)
13. [Traffic Shifting & Deployment Routing](#13-traffic-shifting--deployment-routing)
14. [Rate Limiting & Admission Control](#14-rate-limiting--admission-control)
15. [Concrete Configurations](#15-concrete-configurations)
16. [Observability & Traffic Visibility](#16-observability--traffic-visibility)
17. [Corner Cases & Hard Problems](#17-corner-cases--hard-problems)

---

## 1. Requirements & Scale

### Traffic Numbers (Amazon-Scale)

```
Global daily traffic:
  Page views/day:        ~10 billion
  Orders/day:            ~60 million (peak: ~100M during Prime Day)
  Peak requests/sec:     ~1,000,000 (aggregate across all services)
  API calls/sec:         ~50,000,000 (service-to-service included)

Geographic distribution:
  North America:         ~40% of traffic
  Europe:                ~25%
  Asia-Pacific:          ~20%
  Rest of world:         ~15%

Infrastructure:
  Regions:               10+ (us-east-1, us-west-2, eu-west-1, ap-northeast-1, ...)
  AZs per region:        3-6
  Total AZs:             ~40-60
  Edge locations (CDN):  400+
  Microservices:         ~1,000+
  Total instances:       ~1,000,000+ containers/VMs
```

### Routing Requirements

- **Latency**: Route users to the closest healthy region/AZ (< 50ms to first byte)
- **Availability**: Survive AZ failure (zero downtime), region failure (< 60s)
- **Blast Radius**: A single failure must not affect more than ~5% of users
- **Consistency**: Writes must route to the correct shard/cell/region for data locality
- **Stickiness**: User sessions must be routed to consistent backends during a session
- **Deployments**: Support canary, blue-green, and traffic-shift deployments without dropping requests
- **Compliance**: Certain data must stay within geographic boundaries (GDPR — EU data stays in EU)

---

## 2. Routing Architecture Overview

```
        User (browser / mobile app)
                    │
                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ LAYER 1: DNS (Route 53)                                             │
│  Decision: Which REGION / edge to send the user to?                 │
│  Inputs: user geo-IP, health checks, latency probes, routing policy │
│  Output: IP address of nearest healthy edge / regional LB           │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│ LAYER 2: EDGE (CloudFront / Global Accelerator)                     │
│  Decision: Serve from cache (CDN), or route to origin region?       │
│  Inputs: cache status, URL pattern, request headers                 │
│  Output: cached response, or forwarded to regional entry point      │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│ LAYER 3: REGIONAL LOAD BALANCER (NLB → ALB)                         │
│  Decision: Which AZ, which target group, which cell?                │
│  Inputs: AZ health, target health, routing rules, headers           │
│  Output: specific backend pod/instance in a healthy AZ              │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│ LAYER 4: SERVICE MESH (Envoy / App Mesh)                            │
│  Decision: Which upstream service instance? Retry? Circuit break?   │
│  Inputs: service health, retry budget, circuit breaker state        │
│  Output: specific upstream pod, or fallback/error                   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│ LAYER 5: DATA ROUTING (application-level)                           │
│  Decision: Which database shard, which cache node, which queue?     │
│  Inputs: shard key (user_id, order_id), cell affinity, consistency  │
│  Output: specific database primary/replica, cache cluster           │
└─────────────────────────────────────────────────────────────────────┘

Each layer makes an independent routing decision.
Each layer has its own health checking.
Each layer can independently fail over.
Failure at one layer does not propagate upward (isolation).
```

---

## 3. DNS Layer — The First Routing Decision

### Route 53 Routing Policies

```
User types amazon.com → DNS resolution starts.
This is the FIRST and most important routing decision.
It determines which region (and thus which continent) handles the request.

Route 53 offers multiple routing policies — Amazon uses them in combination:

┌────────────────────┬──────────────────────────────────────────────────────┐
│ Policy             │ How It Works                                         │
├────────────────────┼──────────────────────────────────────────────────────┤
│ Latency-based      │ Route to the region with lowest measured latency    │
│                    │ from the user's resolver IP. Route 53 maintains     │
│                    │ a global latency map (updated continuously).        │
│                    │ User in Mumbai → ap-south-1 (Mumbai region)         │
│                    │ User in NYC → us-east-1 (Virginia)                  │
├────────────────────┼──────────────────────────────────────────────────────┤
│ Geolocation        │ Route based on user's geographic location.          │
│                    │ Used for compliance: EU users → eu-west-1           │
│                    │ Overrides latency when regulatory requirements      │
│                    │ dictate data residency.                             │
├────────────────────┼──────────────────────────────────────────────────────┤
│ Failover           │ Primary/secondary records. If primary health check  │
│                    │ fails → return secondary IP.                        │
│                    │ Primary: us-east-1 ALB. Secondary: us-west-2 ALB.   │
├────────────────────┼──────────────────────────────────────────────────────┤
│ Weighted           │ Distribute traffic by percentage.                   │
│                    │ us-east-1: weight=70, us-west-2: weight=30          │
│                    │ Used for gradual region migration or cost balancing. │
├────────────────────┼──────────────────────────────────────────────────────┤
│ Multivalue answer  │ Return multiple healthy IPs. Client picks one.      │
│                    │ Provides client-side load balancing.                │
└────────────────────┴──────────────────────────────────────────────────────┘
```

### Amazon's Actual DNS Strategy (Layered)

```
amazon.com
  │
  ▼
CNAME → www.amazon.com
  │
  ▼
ALIAS → d123456.cloudfront.net  (CloudFront distribution)
  │
  │  CloudFront resolves to nearest edge location (400+ PoPs)
  │  This is latency-based by default (CloudFront's anycast routing)
  │
  ▼
Edge location serves static content from cache.
For dynamic content (API calls, checkout, cart):
  │
  ▼
CloudFront origin is configured as:
  Regional ALB with Route 53 ALIAS:
    api.amazon.internal
      │
      ├── Latency routing:
      │   us-east-1-alb.amazon.internal  (Virginia)  ← US East users
      │   us-west-2-alb.amazon.internal  (Oregon)    ← US West users
      │   eu-west-1-alb.amazon.internal  (Ireland)   ← EU users
      │   ap-south-1-alb.amazon.internal (Mumbai)    ← India users
      │
      └── Each has a health check.
          If us-east-1 health check fails → Route 53 removes it
          → traffic shifts to us-west-2 (next lowest latency for US East users)
```

### DNS TTL Strategy

```
Critical trade-off: Low TTL = fast failover but more DNS lookups.
                    High TTL = cached everywhere but slow failover.

Amazon's approach (per record type):

┌──────────────────────────────┬───────┬──────────────────────────────────┐
│ Record                       │ TTL   │ Why                              │
├──────────────────────────────┼───────┼──────────────────────────────────┤
│ api.amazon.com (dynamic)     │ 60s   │ Fast failover (< 60s to drain). │
│                              │       │ Acceptable DNS query rate.       │
├──────────────────────────────┼───────┼──────────────────────────────────┤
│ static.amazon.com (CDN)      │ 3600s │ Never changes (CloudFront CNAME).│
│                              │       │ CDN itself handles failover.     │
├──────────────────────────────┼───────┼──────────────────────────────────┤
│ Health check interval        │ 10s   │ Route 53 probes every 10 seconds.│
│                              │       │ 3 consecutive failures = unhealthy│
│                              │       │ = removed from rotation in 30s.  │
├──────────────────────────────┼───────┼──────────────────────────────────┤
│ Failover total time:         │       │ Detection: 30s + DNS TTL: 60s    │
│                              │       │ = ~90 seconds worst case.        │
│                              │       │ In practice: ~30-60s (many       │
│                              │       │ clients re-resolve before TTL).  │
└──────────────────────────────┴───────┴──────────────────────────────────┘
```

---

## 4. Edge Layer — CDN & Global Accelerator

### CloudFront (Static + Dynamic Content)

```
CloudFront serves TWO fundamentally different workloads:

1. Static content (images, JS, CSS, product images):
   Cache-Control: max-age=86400, public
   Hit rate: ~95%
   Origin: S3 bucket (per-region, cross-region replicated)
   
2. Dynamic content (API responses, personalized pages):
   Cache-Control: no-store (or short TTL per endpoint)
   Hit rate: 0-30% depending on endpoint
   Origin: Regional ALB
   CloudFront here is NOT a cache — it's a TCP optimization:
     - TLS termination at edge (saves 1 RTT to user)
     - Persistent connections to origin (connection reuse)
     - HTTP/2 multiplexing to edge
     - Smart origin selection (if multi-origin configured)
```

### Global Accelerator (For Latency-Critical Dynamic APIs)

```
Problem with CloudFront for dynamic APIs:
  CloudFront routes to origin via the public internet.
  Internet routing is "best effort" — packets may take suboptimal paths.
  Frankfurt user → CloudFront edge → (public internet, 6 hops) → us-east-1 origin
  Latency: ~120ms

Global Accelerator:
  Routes traffic onto AWS's private backbone network from the nearest edge.
  Frankfurt user → GA edge → (AWS backbone, 2 hops) → us-east-1 origin
  Latency: ~80ms (30% lower)

  ┌──────────────────────────────────────────────────────┐
  │             Global Accelerator                       │
  │                                                      │
  │  Anycast IPs: 2 static IPs (advertised from all      │
  │               AWS edge locations globally)           │
  │                                                      │
  │  User connects to nearest edge (BGP anycast).        │
  │  Traffic enters AWS backbone immediately.            │
  │  Routed to configured endpoint (ALB, NLB, EC2, EIP)  │
  │  in the healthiest, closest region.                  │
  │                                                      │
  │  Failover: < 30 seconds (health check + routing      │
  │  update on AWS backbone — no DNS TTL dependency)     │
  └──────────────────────────────────────────────────────┘

When to use which:
  - Product pages, search results → CloudFront (cacheable, high hit rate)
  - Cart, checkout, payment APIs → Global Accelerator (dynamic, latency-critical)
  - WebSocket / real-time updates → Global Accelerator (persistent connections)
```

---

## 5. Load Balancing — The Three-Tier Stack

### The Full Path

```
        Internet
           │
           ▼
  ┌──────────────────┐
  │ L3/L4: NLB       │  Network Load Balancer
  │ (or ECMP/Anycast)│  
  │                  │  Routes by: IP, port, protocol
  │                  │  Terminates: TCP (or passes through)
  │                  │  Speed: Millions of connections/sec
  │                  │  AZ-aware: yes (cross-zone LB)
  └────────┬─────────┘
           │
  ┌────────▼─────────┐
  │ L7: ALB          │  Application Load Balancer
  │                  │
  │  Routes by:      │  Host header (api.amazon.com vs www.amazon.com)
  │    Path          │  /api/cart/* → cart-service target group
  │    Headers       │  X-Cell-ID: cell-42 → cell-42 target group
  │    Query string  │  ?version=v2 → canary target group
  │    HTTP method   │  POST vs GET → different target groups
  │                  │
  │  Terminates: TLS │  (SSL offload)
  │  Features:       │  WAF integration, request tracing, sticky sessions
  └────────┬─────────┘
           │
  ┌────────▼─────────┐
  │ Envoy sidecar    │  Service Mesh (per-pod)
  │ (L7 proxy)       │
  │                  │  Routes by: service name, headers, retry policy
  │                  │  Features: circuit breaking, retry budgets,
  │                  │            outlier detection, canary splits
  └──────────────────┘
```

### ALB Routing Rules — Concrete Configuration

```yaml
# ALB Listener Rule Configuration (conceptual — maps to AWS ALB rules)

rules:
  # Rule 1: Cart service
  - priority: 10
    conditions:
      - field: path-pattern
        values: ["/api/cart/*"]
    actions:
      - type: forward
        target_group: cart-service-tg
        stickiness:
          enabled: true
          duration: 3600  # 1 hour session stickiness

  # Rule 2: Checkout (latency-critical, separate scaling)
  - priority: 20
    conditions:
      - field: path-pattern
        values: ["/api/checkout/*"]
    actions:
      - type: forward
        target_group: checkout-service-tg

  # Rule 3: Search (read-heavy, can tolerate stale)
  - priority: 30
    conditions:
      - field: path-pattern
        values: ["/api/search/*"]
    actions:
      - type: forward
        target_group: search-service-tg

  # Rule 4: Canary deployment — 5% of traffic to v2
  - priority: 40
    conditions:
      - field: path-pattern
        values: ["/api/product/*"]
      - field: http-header
        name: "X-Canary"
        values: ["true"]
    actions:
      - type: forward
        target_group: product-service-v2-tg

  # Rule 5: Default
  - priority: 999
    conditions:
      - field: path-pattern
        values: ["/*"]
    actions:
      - type: forward
        target_groups:
          - arn: product-service-v1-tg
            weight: 95
          - arn: product-service-v2-tg
            weight: 5

  # Rule 6: Maintenance mode (activated during planned failover)
  - priority: 1  # highest priority when active
    conditions:
      - field: path-pattern
        values: ["/*"]
    actions:
      - type: fixed-response
        status_code: 503
        content_type: "application/json"
        body: '{"error":"Service temporarily unavailable. Please retry."}'
    # Activated by changing rule state — normally disabled
```

### Cross-Zone Load Balancing

```
3-AZ region: AZ-A (40% capacity), AZ-B (30%), AZ-C (30%)

Without cross-zone LB:
  Each AZ's LB node distributes only to targets in its own AZ.
  AZ-A LB → AZ-A targets (40% of targets get 33% of traffic = underloaded)
  AZ-C LB → AZ-C targets (30% of targets get 33% of traffic = overloaded)
  
  Uneven load distribution. AZ-C targets may be overwhelmed.

With cross-zone LB (enabled on ALB by default):
  Each LB node distributes to ALL healthy targets across ALL AZs.
  Load is even regardless of target distribution per AZ.
  
  Trade-off: Cross-AZ data transfer cost ($0.01/GB).
  At Amazon's scale: significant cost. But correctness > cost.

Amazon's nuance:
  - External-facing ALBs: cross-zone ENABLED (user experience > cost)
  - Internal service-to-service: PREFER same-AZ (latency + cost)
    but ALLOW cross-AZ when same-AZ targets are unhealthy
```

---

## 6. Availability Zone Architecture

### What IS an Availability Zone

```
A region (e.g., us-east-1) contains multiple Availability Zones (AZs):

  ┌─────────────────── us-east-1 Region ──────────────────────┐
  │                                                           │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
  │  │    AZ-A      │  │    AZ-B      │  │    AZ-C      │    │
  │  │  (us-east-1a)│  │ (us-east-1b) │  │ (us-east-1c) │    │
  │  │              │  │              │  │              │    │
  │  │ Data center  │  │ Data center  │  │ Data center  │    │
  │  │ cluster 1    │  │ cluster 1    │  │ cluster 1    │    │
  │  │              │  │              │  │              │    │
  │  │ Power: indep.│  │ Power: indep.│  │ Power: indep.│    │
  │  │ Network: indep│  │ Network: indep│  │ Network: indep│   │
  │  │ Cooling: indep│  │ Cooling: indep│  │ Cooling: indep│   │
  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘    │
  │         │                 │                 │             │
  │         └─────────────────┼─────────────────┘             │
  │                           │                               │
  │              High-bandwidth, low-latency                  │
  │              interconnect (< 2ms RTT between AZs)         │
  │              (typically dark fiber, redundant paths)       │
  │                                                            │
  └────────────────────────────────────────────────────────────┘

Key properties:
  - AZs are physically separated (different buildings, different blocks, sometimes miles apart)
  - Independent power, cooling, networking
  - Connected by dedicated high-bandwidth links (not public internet)
  - An AZ failure (power outage, flooding, fire) does NOT affect other AZs
  - Cross-AZ latency: < 2ms RTT (effectively "local" for application logic)
```

### AZ-Aware Deployment Pattern

```
Every service is deployed IDENTICALLY across all AZs:

  ┌──────────────────────────────────────────────────────────┐
  │                    cart-service                          │
  │                                                          │
  │  AZ-A:  10 pods (auto-scaled)                            │
  │  AZ-B:  10 pods (auto-scaled)                            │
  │  AZ-C:  10 pods (auto-scaled)                            │
  │                                                          │
  │  Total: 30 pods                                          │
  │  Each AZ can handle 50% of total traffic (N+1 design)    │
  │  If AZ-A dies: AZ-B + AZ-C absorb the load (15 + 15)     │
  │  This means we're running at ~67% utilization normally   │
  │  (33% headroom for AZ failure)                           │
  └──────────────────────────────────────────────────────────┘

N+1 AZ design:
  3 AZs, but sized so any 2 can handle full load.
  
  Normal: each AZ handles 33% of traffic.
  AZ failure: each surviving AZ handles 50% of traffic.
  
  Capacity = (total_traffic × 1.5) / num_AZs
  
  This 50% overhead is the COST of AZ fault tolerance.
  Amazon considers it non-negotiable.
```

---

## 7. Multi-Region Architecture

### Active-Active vs Active-Passive

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Amazon's Approach: REGIONAL ACTIVE-ACTIVE        │
│                        (with cell-based write routing)                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────┐          ┌─────────────────────┐              │
│  │   us-east-1         │          │   eu-west-1         │              │
│  │   (Virginia)        │          │   (Ireland)         │              │
│  │                     │          │                     │              │
│  │  ┌───────────────┐  │          │  ┌───────────────┐  │              │
│  │  │ All services  │  │          │  │ All services  │  │              │
│  │  │ (cart, order, │  │          │  │ (cart, order, │  │              │
│  │  │  search, etc) │  │          │  │  search, etc) │  │              │
│  │  └───────────────┘  │          │  └───────────────┘  │              │
│  │  ┌───────────────┐  │          │  ┌───────────────┐  │              │
│  │  │ DB primaries  │  │  async   │  │ DB primaries  │  │              │
│  │  │ for US users  │──┼──repl───→│  │ for EU users  │  │              │
│  │  └───────────────┘  │          │  └───────────────┘  │              │
│  │                     │          │                     │              │
│  │  Handles:           │          │  Handles:           │              │
│  │  - US user reads    │          │  - EU user reads    │              │
│  │  - US user writes   │          │  - EU user writes   │              │
│  │  - EU user reads    │          │  - US user reads    │              │
│  │    (from replica)   │          │    (from replica)   │              │
│  └─────────────────────┘          └─────────────────────┘              │
│                                                                         │
│  Critical rule: **WRITES are routed to the user's HOME REGION**.            │
│  User 12345 is homed in us-east-1. Even if they're traveling in EU:     │
│    - Reads: served from eu-west-1 (local read replica, fast)            │
│    - Writes: routed to us-east-1 (where their data primary lives)       │
│    - Write latency: +80ms cross-Atlantic (acceptable for cart/order)    │
│                                                                         │
│  This avoids cross-region write conflicts entirely.                      │
│  No conflict resolution. No CRDT. No last-writer-wins.                   │
│  Strong consistency for writes within the home region.                   │
└──────────────────────────────────────────────────────────────────────────┘
```

### Region Selection for a New User

```
When a user creates an account:

1. DNS tells us the user is in Germany → eu-west-1 is nearest region.
2. User record created in eu-west-1 (this becomes their HOME REGION).
3. user_id encoded with region hint: user_id = EU_00000012345
   (or metadata stored: user_id → home_region mapping in global lookup).
4. All future writes for this user are routed to eu-west-1.
5. Read replicas in all other regions serve reads for this user.

For 500M users:
  us-east-1:     200M users homed (40%)
  eu-west-1:     125M users homed (25%)
  ap-northeast-1: 50M users homed (10%)
  ap-south-1:     50M users homed (10%)
  Other regions:  75M users homed (15%)
```

---

## 8. Cell-Based Architecture & Blast Radius Isolation

### Why Cells

```
Problem with standard multi-AZ:
  All users in a region share the same service fleet and database.
  A bad deployment to the "cart" service affects ALL users in that region.
  Region has 200M users → 200M affected.
  
  Blast radius: 100% of region's users = 40% of global users.

Cell-based architecture:
  Divide each region into independent CELLS.
  Each cell handles ~5% of the region's users.
  A bad deployment or failure in one cell affects only 5% of regional users.
  
  Blast radius: 5% of region = 2% of global users.
```

### Cell Architecture

```
  ┌──────────────────── us-east-1 Region ─────────────────────────────┐
  │                                                                    │
  │  ┌──────── Cell 1 ────────┐   ┌──────── Cell 2 ────────┐        │
  │  │                        │   │                        │        │
  │  │  Users: 0-4.99%        │   │  Users: 5-9.99%        │        │
  │  │  (10M users)           │   │  (10M users)           │        │
  │  │                        │   │                        │        │
  │  │  ┌────────────────┐    │   │  ┌────────────────┐    │        │
  │  │  │ Cart Service   │    │   │  │ Cart Service   │    │        │
  │  │  │ Order Service  │    │   │  │ Order Service  │    │        │
  │  │  │ User Service   │    │   │  │ User Service   │    │        │
  │  │  │ Payment Service│    │   │  │ Payment Service│    │        │
  │  │  └────────────────┘    │   │  └────────────────┘    │        │
  │  │  ┌────────────────┐    │   │  ┌────────────────┐    │        │
  │  │  │ DB shard group │    │   │  │ DB shard group │    │        │
  │  │  │ Redis cluster  │    │   │  │ Redis cluster  │    │        │
  │  │  │ Kafka partition│    │   │  │ Kafka partition│    │        │
  │  │  └────────────────┘    │   │  └────────────────┘    │        │
  │  │                        │   │                        │        │
  │  └────────────────────────┘   └────────────────────────┘        │
  │                                                                    │
  │  ... Cell 3, Cell 4, ..., Cell 20                                 │
  │  (20 cells per region, each handling ~5% of regional users)       │
  │                                                                    │
  │  ┌──────── Shared Services (NOT per-cell) ─────────────────┐      │
  │  │  Search (read-only, shared Elasticsearch cluster)       │      │
  │  │  Product Catalog (read-only, shared cache)              │      │
  │  │  CDN / Static Content (shared)                          │      │
  │  │  DNS / Edge / Load Balancer (shared)                    │      │
  │  └────────────────────────────────────────────────────────┘      │
  │                                                                    │
  └────────────────────────────────────────────────────────────────────┘

Cell assignment:
  cell_id = hash(user_id) % 20
  
  User 12345 → hash → cell 7
  ALL requests from user 12345 are routed to cell 7.
  Cell 7 has its own cart, order, payment instances and DB shards.
  
  If cell 7 has a bad deployment:
  - 10M users affected (5% of region)
  - Other 190M users in the region: zero impact
  - Rollback cell 7's deployment → 10M users restored
```

### Cell Routing

```
How does the ALB know which cell to route to?

Option 1: **Header-based routing (after auth)**
  1. User authenticates → auth service returns JWT with cell_id embedded
  2. JWT: { user_id: 12345, cell_id: 7, region: "us-east-1", ... }
  3. ALB rule:
       condition: header "X-Cell-ID" = "7"
       action: forward to target-group-cell-7

Option 2: Cookie-based routing
  1. First request → **routing service computes cell_id from user_id**
  2. Sets cookie: Set-Cookie: cell=7; Path=/; Secure; HttpOnly
  3. Subsequent requests: ALB reads cookie, routes to cell-7 target group

Option 3: Path-based routing (simple but requires URL rewriting)
  1. Edge/API gateway rewrites URL:
     /api/cart/items → /cell-7/api/cart/items
  2. ALB path rule: /cell-7/* → target-group-cell-7

Amazon uses a combination: cell ID is determined at the edge and propagated
as a header through all downstream services. Service mesh (Envoy) uses
the header for all service-to-service routing within the cell.
```

---

## 9. Data Layer Routing

### Database Shard-to-Cell Mapping

```
Each cell owns a range of database shards:

  Cell 1:  shards 0-15    (user_id hash ∈ [0, 15])
  Cell 2:  shards 16-31   (user_id hash ∈ [16, 31])
  ...
  Cell 20: shards 304-319  (user_id hash ∈ [304, 319])

  Total: 320 logical shards distributed across 20 cells.

Database routing in application code:

  func getDBConnection(userID int64) *sql.DB {
      shardID := hash(userID) % 320
      cellID := shardID / 16  // 16 shards per cell
      
      // Return the primary for writes, replica for reads
      return dbPool.GetShard(cellID, shardID)
  }

Shard placement across AZs (within a cell):

  Cell 7, Shard 112:
    Primary:     AZ-A
    Sync replica: AZ-B
    Async replica: AZ-C (read-only)
    
  Cell 7, Shard 113:
    Primary:     AZ-B  ← spread primaries across AZs
    Sync replica: AZ-C
    Async replica: AZ-A

  Primaries are deliberately spread across AZs.
  If AZ-A goes down: some shards lose their primary (failover to sync replica)
  but NOT ALL shards. Only ~33% of shards need failover.
```

### Read/Write Routing

```
┌──────────────────────────────────────────────────────────┐
│              Application-Level Read/Write Split          │
│                                                          │
│  WRITES:                                                 │
│    Always routed to the PRIMARY for the user's shard.    │
│    Primary is in the user's home region.                 │
│    If user is traveling: write crosses regions           │
│    (accepted latency penalty for correctness).           │
│                                                          │
│  READS (strong consistency required):                    │
│    Order status, payment confirmation, cart after add    │
│    → Route to PRIMARY (same as write)                    │
│    → Ensures read-your-writes consistency                │
│                                                          │
│  READS (eventual consistency acceptable):                │
│    Product catalog, search results, recommendations      │
│    → Route to NEAREST READ REPLICA                       │
│    → May be in same AZ (fastest) or same region          │
│    → Stale by 10-100ms (acceptable for these use cases)  │
│                                                          │
│  READS (after a recent write — "read-your-writes"):      │
│    User adds item to cart → immediately reads cart       │
│    → Problem: read from replica may not have the write   │
│    → Solution: "sticky to primary for 5 seconds"         │
│      **After a write, subsequent reads from the same user  │
│      are routed to primary for 5 seconds**.                │
│      After 5 seconds: safe to read from replica.         │
│      Implemented via: cookie/header flag set on write    │
│      response, checked by routing layer.                 │
└──────────────────────────────────────────────────────────┘
```

---

## 10. Service Mesh & Service-to-Service Routing

### Envoy Sidecar Configuration

```yaml
# Envoy sidecar config for cart-service (simplified)

static_resources:
  clusters:
    # Upstream: order-service
    - name: order-service
      type: EDS  # Endpoint Discovery Service (from control plane)
      lb_policy: ROUND_ROBIN
      
      # AZ-aware routing: prefer same-AZ endpoints
      common_lb_config:
        zone_aware_lb_config:
          routing_enabled:
            value: 100  # 100% zone-aware
          min_cluster_size: 3
      
      # Health checking
      health_checks:
        - timeout: 2s
          interval: 5s
          unhealthy_threshold: 3
          healthy_threshold: 2
          http_health_check:
            path: "/health"
      
      # Circuit breaker
      circuit_breakers:
        thresholds:
          max_connections: 1000
          max_pending_requests: 100
          max_requests: 1500
          max_retries: 3
      
      # Outlier detection (eject unhealthy endpoints)
      outlier_detection:
        consecutive_5xx: 5
        interval: 10s
        base_ejection_time: 30s
        max_ejection_percent: 30  # never eject more than 30% of endpoints

    # Upstream: payment-service (cell-aware routing)
    - name: payment-service
      type: EDS
      lb_policy: ROUND_ROBIN
      
      # Cell affinity: route to same cell
      metadata_match:
        filter_metadata:
          envoy.lb:
            cell_id: "7"  # dynamically set from request header
```

### Retry Policy

```yaml
# Retry policy for order-service calls
route_config:
  virtual_hosts:
    - name: order-service
      routes:
        - match:
            prefix: "/"
          route:
            cluster: order-service
            timeout: 5s
            retry_policy:
              retry_on: "5xx,reset,connect-failure,retriable-4xx"
              num_retries: 2
              per_try_timeout: 2s
              retry_back_off:
                base_interval: 0.1s
                max_interval: 1s
              # Retry budget: max 20% of requests can be retries
              # Prevents retry storms (100% of requests failing → 300% load from retries)
              retry_budget:
                budget_percent: 20
                min_retry_concurrency: 3
```

---

## 11. Health Checking at Every Layer

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    Health Checks at Every Layer                          │
│                                                            │             │
│  Layer          │ What checks      │ Interval │ Threshold  │ Action      │
│─────────────────┼──────────────────┼──────────┼────────────┼─────────────│
│  DNS (Route 53) │ Region endpoint  │ 10s      │ 3 failures │ Remove      │
│                 │ (TCP or HTTP to  │          │ = 30s      │ region from │
│                 │  ALB /health)    │          │            │ DNS rotation│
│─────────────────┼──────────────────┼──────────┼────────────┼─────────────│
│  ALB            │ Each target pod  │ 10s      │ 2 failures │ Stop routing│
│                 │ (HTTP GET        │          │ = 20s      │ to target.  │
│                 │  /health/ready)  │          │            │ Drain conns.│
│─────────────────┼──────────────────┼──────────┼────────────┼─────────────│
│  Envoy sidecar  │ Each upstream    │ 5s       │ 3 failures │ Eject from  │
│                 │ endpoint         │          │ = 15s      │ LB pool     │
│                 │ (HTTP /health)   │          │            │ for 30s     │
│─────────────────┼──────────────────┼──────────┼────────────┼─────────────│
│  Kubernetes     │ Liveness probe   │ 10s      │ 3 failures │ Kill and    │
│  (kubelet)      │ (HTTP/TCP/exec)  │          │            │ restart pod │
│                 │ Readiness probe  │ 5s       │ 1 failure  │ Remove from │
│                 │ (HTTP /ready)    │          │            │ service     │
│                 │                  │          │            │ endpoints   │
│─────────────────┼──────────────────┼──────────┼────────────┼─────────────│
│  Database       │ Primary check    │ 1s       │ 3 failures │ Promote     │
│  (Patroni /     │ (pg_isready +    │          │ = 3s       │ sync standby│
│   RDS Multi-AZ) │ replication lag) │          │            │             │
│─────────────────┼──────────────────┼──────────┼────────────┼─────────────│
│  Redis Sentinel │ Primary check    │ 1s       │ quorum     │ Promote     │
│                 │ (PING + role)    │          │ (2 of 3    │ replica     │
│                 │                  │          │  sentinels)│             │
└──────────────────────────────────────────────────────────────────────────┘
```

### Deep vs Shallow Health Checks

```
Shallow health check:
  GET /health → 200 OK
  Tells you: process is running, HTTP stack is up.
  Doesn't tell you: can it actually serve requests? DB connected? Cache reachable?

Deep health check:
  GET /health/ready
  Checks: DB connection pool (has available connections)
          Redis connection (PING returns PONG)
          Kafka producer (last successful send < 30s ago)
          Downstream service (circuit breaker NOT open)
  → 200 if ALL pass, 503 if ANY fail

When to use which:
  Kubernetes LIVENESS:    SHALLOW (just "is the process alive?")
                          If deep, a DB outage kills all pods → worse
  Kubernetes READINESS:   DEEP (can it serve traffic?)
                          Failing readiness = no traffic routed, but pod stays alive
  ALB health check:       DEEP (should this target receive traffic?)
  Route 53 health check:  SEMI-DEEP (can the region's entry point respond?)
                          NOT too deep — don't fail a whole region because one DB shard is slow
```

---

## 12. Failover Strategies

### AZ Failure Playbook

```
Scenario: AZ-A in us-east-1 loses power.

t=0s      AZ-A goes dark. All instances in AZ-A unreachable.

t=0-5s    ALB health checks: AZ-A targets fail health checks (instant for TCP).
          ALB stops routing NEW requests to AZ-A targets.
          In-flight requests to AZ-A: timeout (connection reset).
          
t=5-20s   Client retries: timed-out requests are retried by:
          - Client-side retry (mobile/web app)
          - ALB retry (if configured)
          - Envoy sidecar retry
          Retried requests land on AZ-B or AZ-C targets (healthy).

t=10-30s  Database failover:
          Shards with primaries in AZ-A: sync replicas in AZ-B/C promoted.
          **Patroni/RDS** performs automatic failover.
          Write-capable again within 10-30 seconds.
          
t=5-60s   Auto-scaling:
          AZ-B and AZ-C now receiving 50% each (up from 33%).
          HPA detects increased CPU/request rate.
          Scales up pods in AZ-B and AZ-C.
          But: N+1 design means existing capacity handles it WITHOUT new pods.
          Auto-scaling provides extra headroom.

t=60s     Full recovery.
          - No DNS change needed (ALB absorbs AZ failure internally)
          - No manual intervention
          - Users experienced: 5-20 seconds of elevated errors for
            requests that were mid-flight to AZ-A, then full recovery.

Impact:
  - Zero downtime for reads (read replicas in other AZs)
  - 10-30 seconds of write failures for ~33% of shards (those with primary in AZ-A)
  - Retries absorb most of the impact — user may not even notice
```

### Region Failure Playbook

```
Scenario: Entire us-east-1 region becomes unreachable.
          (Power grid failure, major network partition, etc.)

This is the WORST CASE. Here's the playbook:

t=0s      us-east-1 unreachable.

t=0-30s   Route 53 health checks: us-east-1 endpoints fail.
          3 consecutive failures at 10s interval = 30s to detect.

t=30s     Route 53 removes us-east-1 from **DNS rotation.**
          DNS TTL: 60s. Some clients still have cached us-east-1 IPs.
          
t=30-90s  Traffic draining:
          - Clients with stale DNS: requests fail → client-side retry with DNS re-resolve
          - New DNS resolutions: return us-west-2 (next lowest latency for US users)
          - Mobile apps: built-in fallback IP list (hardcoded backup)

t=60-120s  ~90% of traffic has migrated to us-west-2.
           us-west-2 receives 2x normal load (its own users + failed-over US-East users).
           Pre-provisioned for this (region-level N+1 design).

DATA IMPLICATIONS:
  us-east-1 was the HOME REGION for 200M users.
  Their data primaries are in us-east-1 (unreachable).
  
  Reads: us-west-2 has async replicas of us-east-1 data.
         Replicas may be 100-500ms behind.
         Product browsing, search, recommendations: work fine (eventually consistent).
         
  Writes: CANNOT write to us-east-1 primaries.
    Option A (chosen): Fail writes gracefully.
      Cart additions: buffered locally on client, synced when region recovers.
      Orders: blocked. Show: "Ordering temporarily unavailable in your region."
      This is Amazon's approach — they'd rather fail orders than create inconsistency.
      
    Option B (alternative): Promote us-west-2 replicas to primaries.
      Enables writes but risks data divergence when us-east-1 recovers.
      Only done if us-east-1 recovery ETA > 4 hours (manual decision).

t=varies  us-east-1 recovers.
          - DNS adds us-east-1 back (gradual: 10% → 30% → 50% → 100%)
          - Data reconciliation: any writes that went to us-west-2 during failover
            must be replicated back to us-east-1 (conflict resolution needed)
          - Gradual traffic shift over 30-60 minutes (not instantaneous)
```

---

## 13. Traffic Shifting & Deployment Routing

### Canary Deployments

```
Deploy new version of cart-service to 5% of traffic:

  ┌──────────────────────────────────────────┐
  │              ALB                         │
  │                                          │
  │  Rule: /api/cart/*                       │
  │  Action: weighted forward                │
  │    cart-service-v1-tg: weight = 95       │
  │    cart-service-v2-tg: weight = 5        │
  │                                          │
  └──────────────────────────────────────────┘
  
  Canary progression (automated by deployment pipeline):
  
  Stage 1:  5% to v2 for 10 minutes.
            Monitor: error rate, latency p99, business metrics (conversion rate).
            If error rate > baseline + 0.5%: auto-rollback.
  
  Stage 2:  25% to v2 for 10 minutes.
            Same monitoring criteria.
  
  Stage 3:  50% to v2 for 15 minutes.
            Now also check: downstream service impact (order-service error rate).
  
  Stage 4:  100% to v2.
            Old v1 target group kept alive for 30 minutes (instant rollback).
  
  Total deployment time: ~45 minutes for a safe global rollout.
```

### Cell-Based Deployments (Even Safer)

```
Instead of canary by traffic percentage, deploy to ONE CELL first:

  Step 1: Deploy v2 to Cell 1 (5% of regional users).
          Monitor for 30 minutes.
          
  Step 2: Deploy v2 to Cells 2-5 (25% of regional users).
          Monitor for 15 minutes.
          
  Step 3: Deploy v2 to all cells in us-east-1.
          Monitor for 15 minutes.
          
  Step 4: Deploy v2 to eu-west-1 (repeat cell-by-cell).
  
  Step 5: Deploy v2 to all regions.

  If Cell 1 shows problems:
  - Only 5% of regional users affected
  - Rollback Cell 1 in < 60 seconds
  - Other 95% never saw the bad code
  
  This is how Amazon deploys to production: one cell at a time,
  with automatic rollback triggers at each stage.
```

---

## 14. Rate Limiting & Admission Control

### Per-Layer Rate Limiting

```
Layer 1: CDN / Edge (coarse-grained)
  Per-IP: 1000 requests/minute (blocks DDoS, aggressive scrapers)
  Implementation: CloudFront + WAF rate-based rules

Layer 2: API Gateway (medium-grained)
  Per-user: 100 requests/minute (authenticated users)
  Per-API: /api/checkout/* limited to 10 requests/minute per user
  Implementation: API Gateway throttling + Redis sliding window

Layer 3: Service-level (fine-grained)
  Per-cell: max 10K requests/sec to checkout-service (prevents cell overload)
  Implementation: Envoy rate limit filter + external rate limit service (Redis-backed)

Layer 4: Database (protection of last resort)
  Per-shard: connection pool limit = 100 connections per service
  Total connections to a shard: 5 services × 100 = 500 connections
  PostgreSQL max_connections: 600 (headroom for admin)
  Implementation: PgBouncer connection pooler

Admission control (load shedding):
  When a service is at 90% CPU:
    - Return 503 for 10% of lowest-priority requests
    - Priority: checkout > cart > search > recommendations > analytics
    - Implementation: Envoy priority-based load shedding
```

---

## 15. Concrete Configurations

### Route 53 Configuration

```json
{
  "HostedZone": "api.amazon.com",
  "Records": [
    {
      "Name": "api.amazon.com",
      "Type": "A",
      "AliasTarget": {
        "DNSName": "api-us-east-1.amazon.internal",
        "EvaluateTargetHealth": true
      },
      "SetIdentifier": "us-east-1",
      "Region": "us-east-1",
      "RoutingPolicy": "Latency",
      "HealthCheckId": "hc-us-east-1-api"
    },
    {
      "Name": "api.amazon.com",
      "Type": "A",
      "AliasTarget": {
        "DNSName": "api-us-west-2.amazon.internal",
        "EvaluateTargetHealth": true
      },
      "SetIdentifier": "us-west-2",
      "Region": "us-west-2",
      "RoutingPolicy": "Latency",
      "HealthCheckId": "hc-us-west-2-api"
    },
    {
      "Name": "api.amazon.com",
      "Type": "A",
      "AliasTarget": {
        "DNSName": "api-eu-west-1.amazon.internal",
        "EvaluateTargetHealth": true
      },
      "SetIdentifier": "eu-west-1",
      "Region": "eu-west-1",
      "RoutingPolicy": "Latency",
      "HealthCheckId": "hc-eu-west-1-api"
    }
  ],
  "HealthChecks": [
    {
      "Id": "hc-us-east-1-api",
      "Config": {
        "Type": "HTTPS",
        "FullyQualifiedDomainName": "api-us-east-1.amazon.internal",
        "Port": 443,
        "ResourcePath": "/health/region",
        "RequestInterval": 10,
        "FailureThreshold": 3,
        "Regions": ["us-east-1", "us-west-2", "eu-west-1"]
      }
    }
  ]
}
```

### Kubernetes Service & HPA

```yaml
# cart-service deployment (per cell, per AZ)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cart-service
  namespace: cell-7
  labels:
    app: cart-service
    cell: "7"
spec:
  replicas: 10  # baseline per AZ
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 2
  template:
    metadata:
      labels:
        app: cart-service
        cell: "7"
    spec:
      topologySpreadConstraints:
        # Spread pods evenly across AZs
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: cart-service
              cell: "7"
      containers:
        - name: cart-service
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2000m"
              memory: "2Gi"
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 1
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cart-service-hpa
  namespace: cell-7
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cart-service
  minReplicas: 10   # minimum per cell (N+1 AZ design)
  maxReplicas: 50   # handle Prime Day spike
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60  # scale at 60% (headroom for AZ failure)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "200"  # scale at 200 RPS per pod
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30   # wait 30s before scaling up
      policies:
        - type: Percent
          value: 50        # scale up by max 50% at a time
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # wait 5 min before scaling down
      policies:
        - type: Percent
          value: 10        # scale down by max 10% at a time
          periodSeconds: 60
```

### PostgreSQL Connection Pool (PgBouncer)

```ini
; pgbouncer.ini for a shard within a cell

[databases]
shard_112 = host=shard-112-primary.db.internal port=5432 dbname=ecommerce
shard_112_ro = host=shard-112-replica.db.internal port=5432 dbname=ecommerce

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0

; Transaction pooling: connection returned to pool after each transaction
; (not after each session — much more efficient)
pool_mode = transaction

; Per-database limits
default_pool_size = 50      ; 50 connections per service per shard
min_pool_size = 10          ; keep 10 warm connections
reserve_pool_size = 10      ; emergency overflow
reserve_pool_timeout = 3    ; wait 3s before using reserve pool

max_client_conn = 2000      ; max inbound connections from services
max_db_connections = 100    ; max outbound to PostgreSQL (across all pools)

; Timeouts
server_idle_timeout = 300   ; close idle server connections after 5 min
client_idle_timeout = 60    ; close idle client connections after 1 min
query_timeout = 30          ; kill queries running > 30 seconds
```

---

## 16. Observability & Traffic Visibility

### Traffic Flow Dashboard

```
┌─────────────────────────────────────────────────────────────────────┐
│               GLOBAL TRAFFIC ROUTING — LIVE VIEW                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  DNS ROUTING                         REGIONAL DISTRIBUTION          │
│  ├─ Total resolutions/sec: 2.1M     ├─ us-east-1:  412K req/sec   │
│  ├─ Latency-routed: 85%             ├─ us-west-2:  188K req/sec   │
│  ├─ Geo-routed (GDPR): 12%         ├─ eu-west-1:  245K req/sec   │
│  └─ Failover-routed: 3%             ├─ ap-south-1: 112K req/sec   │
│                                      └─ ap-ne-1:    58K req/sec    │
│  CDN HIT RATE                                                       │
│  ├─ Static content: 96.2%           AZ HEALTH (us-east-1)          │
│  ├─ API responses: 12.8%            ├─ AZ-A: ✓  142K req/sec      │
│  └─ Total bandwidth: 4.2 Tbps       ├─ AZ-B: ✓  138K req/sec      │
│                                      └─ AZ-C: ✓  132K req/sec      │
│  CELL STATUS (us-east-1)                                            │
│  ├─ Cell 1:  ✓  err: 0.02%         ALB HEALTH                     │
│  ├─ Cell 2:  ✓  err: 0.01%         ├─ Healthy targets: 12,847     │
│  ├─ Cell 3:  ⚠  err: 0.15%         ├─ Unhealthy targets: 3        │
│  │  (canary deploy in progress)     ├─ Draining targets: 12        │
│  ├─ Cell 4:  ✓  err: 0.01%         └─ Total target groups: 240    │
│  └─ ...                                                             │
│                                                                     │
│  ALERTS                                                             │
│  └─ ⚠ Cell 3 error rate elevated (canary v2.14 — auto-rollback    │
│       triggers at 0.5%, currently 0.15%, monitoring)                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 17. Corner Cases & Hard Problems

### 1. DNS Caching Beyond Your Control

```
Problem: You set TTL=60s. But:
  - Some ISP resolvers enforce minimum TTL = 300s (override your setting)
  - Java's InetAddress caches DNS FOREVER by default (JVM setting)
  - Mobile OS DNS caches survive app restart
  - Corporate proxies cache aggressively

  You fail over us-east-1 → us-west-2. Route 53 updated. TTL expired.
  But 5-10% of users still have cached us-east-1 IPs.
  These users send requests to a dead region for 5-15 minutes.

Mitigation layers:
  1. Client-side retry with re-resolve:
     Mobile app: on 3 consecutive connection errors, flush DNS cache, re-resolve.
     
  2. Global Accelerator as primary entry point:
     GA uses anycast (BGP routing, not DNS). Failover is at the network layer.
     No DNS TTL dependency. Failover in < 30 seconds.
     
  3. Multi-IP DNS response:
     Return IPs from BOTH us-east-1 AND us-west-2.
     Client tries first IP; if timeout → try second IP.
     "Happy Eyeballs" algorithm (RFC 8305) does this automatically.
     
  4. JVM configuration (for service-to-service):
     networkaddress.cache.ttl=30 in java.security
     Ensures JVM re-resolves every 30 seconds.
```

### 2. Thundering Herd on Failover Recovery

```
Problem: AZ-A was down for 20 minutes. Traffic shifted to AZ-B and AZ-C.
  AZ-A recovers. ALB adds AZ-A targets back to rotation.
  
  Immediately: AZ-A targets get 33% of traffic.
  But: AZ-A pods just restarted. Caches are cold. Connection pools empty.
  Cold pods get slammed with warm-cache-rate traffic → overload → crash → repeat.

Solution: Gradual re-introduction.

  1. Readiness probe on AZ-A pods:
     Don't report ready until:
     - Connection pools warmed (DB connections established)
     - Local caches loaded (application-level warm-up)
     - Health check endpoint returns 200 after self-test
     
  2. ALB slow-start mode:
     ALB slow start: new targets receive linearly increasing traffic over 
     a configurable ramp-up period (e.g., 120 seconds).
     t=0:   target gets 1% of traffic
     t=30s: target gets 25%
     t=60s: target gets 50%
     t=120s: target gets 100% (full share)
     
  3. Traffic weight (manual):
     Operator adjusts ALB target group weights:
     AZ-A: 10% → 20% → 33% over 15 minutes (controlled ramp).
```

### 3. Split-Brain DNS (Region Appears Dead But Is Only Partitioned)

```
Problem: Network partition between AWS regions and Route 53 health checkers.
  us-east-1 is actually healthy and serving users.
  But Route 53 health checkers (which run FROM specific locations) can't reach it.
  Route 53 marks us-east-1 as unhealthy → removes from DNS.
  
  Result: users who already had us-east-1 IPs continue working fine.
  New DNS lookups get us-west-2 → those users work fine too.
  But: us-west-2 is now receiving double traffic unexpectedly.
  
  When partition heals: Route 53 adds us-east-1 back.
  Some users on us-east-1, some on us-west-2 → inconsistent writes.

Mitigation:
  1. Route 53 health checkers run from MULTIPLE locations (at least 3 regions).
     Endpoint marked unhealthy only if >70% of checker locations agree.
     Single network partition between one checker and the endpoint
     doesn't trigger failover.
     
  2. Health check from INSIDE the region (CloudWatch alarm → Route 53):
     Each region has an internal canary that checks its OWN health.
     Publishes to a global CloudWatch metric.
     Route 53 health check reads this metric (not direct probe).
     This is immune to network partitions between Route 53 and the region.
     
  3. Minimum healthy threshold:
     Route 53 never removes the LAST healthy region.
     If all regions appear unhealthy (likely a monitoring issue, not real outage):
     → keep all in DNS (better to send traffic to a "possibly healthy" region
        than to send traffic nowhere).
```

### 4. Cross-Region Consistency During Failover

```
Problem: User in us-east-1 adds item to cart (writes to us-east-1 primary).
  1 second later: us-east-1 goes down.
  User retries (now routed to us-west-2).
  Reads cart from us-west-2 replica.
  
  Async replication lag was 200ms. The cart-add happened 800ms before failover.
  The write IS on the us-west-2 replica. User sees their cart. OK.
  
  But what if the cart-add happened 50ms before failover?
  Async replication lag: 200ms. The write has NOT reached us-west-2 yet.
  User sees: empty cart. "I just added that item!"

  The 200ms window between "write committed on primary" and
  "write visible on cross-region replica" is the danger zone.

Mitigation:
  1. Accept the inconsistency window (pragmatic):
     Most cross-region failovers take 30-90 seconds.
     After 90 seconds, replica has caught up.
     Tell user: "Your recent changes may take a moment to appear."
     
  2. Write-ahead to client:
     When user adds item to cart, client stores it locally immediately.
     Client displays local cart + server cart merged.
     After failover, server cart may be stale by a few writes.
     Client-local cart fills the gap.
     When server catches up: client reconciles (remove local-only items
     that now appear in server cart).
     
  3. Multi-region write buffer:
     Cart writes go to BOTH us-east-1 primary AND us-west-2 write buffer.
     us-west-2 write buffer is a Redis queue (not DB).
     On failover: us-west-2 replays the buffer into its promoted primary.
     Guarantees zero write loss at the cost of double-write latency.
```

### 5. Cascading Failure Across Cells

```
Problem: Cell 7 fails. Its 10M users retry requests.
  Retries are routed to... Cell 7 (because cell affinity is based on user_id hash).
  Cell 7 is down → retries fail → more retries → no improvement.

  Meanwhile: Cell 7's database shard receives no writes (down).
  When Cell 7 recovers: 10M users worth of queued retries hit simultaneously.
  Cell 7 overloads again → fails again → repeat.

  The cell-based design contains the INITIAL failure,
  but retry behavior can still cause problems.

Solution:
  1. Cell overflow routing:
     If Cell 7 is unhealthy for > 30 seconds:
     Route Cell 7's users to Cell 8 (overflow cell).
     Cell 8 can serve reads from Cell 7's replica DB.
     Writes: queued or blocked (cannot write to Cell 7's primary).
     
  2. Retry with jitter and backoff:
     Client retry: wait 1s, 2s, 4s, 8s (exponential backoff with jitter).
     Envoy retry budget: max 20% of requests can be retries.
     This prevents the retry storm when Cell 7 recovers.
     
  3. Cold start protection on recovery:
     Cell 7 comes back → slow-start mode:
     t=0:   accept 10% of normal traffic
     t=60s: accept 30%
     t=120s: accept 60%
     t=180s: accept 100%
     
     Queued retries are drained gradually, not all at once.
```

### 6. AZ-Imbalanced Deployment Causes Partial Outage

```
Problem: Deploy new version of order-service.
  Rolling update hits AZ-A first (Kubernetes default: no AZ preference).
  All AZ-A pods are cycling (terminating old, starting new).
  For 30 seconds: AZ-A has 0 ready pods for order-service.
  
  ALB routes all order-service traffic to AZ-B and AZ-C.
  AZ-B and AZ-C now at 150% load → elevated latency → some timeouts.
  Not a full outage, but degraded for everyone in the region.

Solution: AZ-aware rolling update.
  Deploy to one AZ at a time, verify health, then move to next AZ.
  
  Configuration (Kubernetes + Argo Rollouts):
    Step 1: Update AZ-A pods (maxUnavailable: 25% of AZ-A pods)
    Step 2: Wait for AZ-A pods healthy + 60 seconds bake time
    Step 3: Update AZ-B pods
    Step 4: Wait for AZ-B pods healthy + 60 seconds bake time
    Step 5: Update AZ-C pods
    
  At no point is more than 25% of one AZ's capacity unavailable.
  Total capacity never drops below ~92%.
```

### 7. Payment Gateway Routing During Region Failover

```
Problem: Stripe integration is configured per-region.
  us-east-1: Stripe API key scoped to US processing
  eu-west-1: Stripe API key scoped to EU processing (for SCA compliance)
  
  US user is failed over to eu-west-1.
  eu-west-1 uses EU Stripe key → payment processed as EU transaction.
  May trigger: different fraud rules, different card network routing,
  different regulatory treatment (SCA vs no SCA).

Solution: Payment routing is by USER HOME REGION, not SERVING REGION.

  PaymentService:
    1. Determine user's home region from user profile (or user_id → region mapping)
    2. Use the Stripe API key for the HOME region, not the serving region
    3. Call Stripe from the serving region, but with the home region's credentials
    
  class PaymentRouter:
      def charge(user_id, amount, payment_method):
          home_region = user_service.get_home_region(user_id)
          stripe_key = config.get_stripe_key(home_region)
          
          # Call Stripe with home region's key
          # Stripe routes to correct processing region
          return stripe.charge(
              api_key=stripe_key,
              amount=amount,
              payment_method=payment_method,
              metadata={"home_region": home_region}
          )

  The HTTP call to Stripe might cross regions (eu-west-1 → Stripe US endpoint),
  adding ~50ms latency. Acceptable for payment processing (user expects 1-3s anyway).
```

### 8. Database Shard Rebalancing During Live Traffic

```
Problem: Cell 3 has grown disproportionately (popular product category).
  Cell 3's database shards are at 85% capacity.
  Need to move some shards from Cell 3 to Cell 12 (underutilized).

  Shard 48 is being moved. During migration:
  - Reads can go to either old or new location
  - Writes must go to exactly one location (no split-brain)

Migration steps (zero-downtime):

  Phase 1: Dual-read, single-write (on old location)
    - Set up logical replication: old shard → new shard location
    - Routing: writes → old, reads → old (no change yet)
    - Wait for replica to catch up (lag < 100ms)
    
  Phase 2: Dual-read, still single-write (on old)
    - Routing: reads → both old and new (shadow reads to verify consistency)
    - Compare results: any mismatches = replication lag issue, wait longer
    
  Phase 3: Switchover (the critical moment)
    - Pause writes to shard 48 for 200ms (hold in application queue)
    - Wait for replication to fully catch up (should be < 200ms)
    - Update routing: writes → new location, reads → new location
    - Release held writes (now go to new location)
    
    User impact: 200ms pause on writes to shard 48.
    Affected users: ~0.3% of cell 3's users (1 shard out of 16).
    Duration: 200ms. Not noticeable.
    
  Phase 4: Cleanup
    - Remove old shard replica after 24 hours (safety net for rollback)
    - Update cell → shard mapping in configuration service
```

---

## Summary: Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| DNS routing | Route 53 latency-based + failover + geo (layered) | Latency for performance, failover for HA, geo for compliance |
| DNS TTL | 60s for dynamic endpoints | Balance: fast failover (~90s) without excessive DNS query load |
| Edge entry | CloudFront for cacheable, Global Accelerator for dynamic | CDN for cache hits; GA for TCP optimization on private backbone |
| Load balancing | NLB → ALB → Envoy (3-tier) | Each layer: different routing granularity, different failover speed |
| AZ design | N+1 (any 2 of 3 AZs handle full load) | Survive AZ failure with zero customer impact |
| Multi-region | Active-active reads, single-primary writes per user | Avoids cross-region write conflicts; strong consistency for writes |
| Cell architecture | 20 cells/region, ~5% of users per cell | Blast radius capped at 2% of global users per failure |
| Cell routing | Header-based (X-Cell-ID from auth token) | Stateless, no affinity cookies, works across all LB layers |
| Data routing | Shard-to-cell affinity, primary spread across AZs | Write locality within cell; AZ failure affects ~33% of shards (not all) |
| Read/write split | "Sticky to primary for 5s after write" | Read-your-writes consistency without always hitting primary |
| Health checking | Deep (readiness) + Shallow (liveness), different per layer | Readiness removes from traffic; liveness kills pod. Don't conflate. |
| Failover speed | AZ: ~5-20s, Region: ~30-90s | AZ via ALB (instant drain); Region via DNS (TTL-bound) |
| Deployments | Cell-by-cell with auto-rollback | Blast radius of bad deploy = 1 cell = 2% of users |
| Retry policy | Exponential backoff + jitter + 20% retry budget | Prevents retry storms that turn partial failures into full outages |



Structure (17 Sections)

┌──────────────────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   
│         Section          │                                                               Key Content                                                               │   
├────────Section ───────│── ─────────────────────────────────────────────────────────────Key Content  ───────────────────────────────────────────────────────────────┤
│ Requirements & Scale  ┼  ─ 1M+ peak req/sec, 10+ regions, 40-60 AZs, 1000+ microservices, 400+ edge PoPs                                                           │   
├───────────────────────│─1M+ peak req/sec, 10+ regions, 40-6─ AZs, 1000+ microservices, 400+ edg─ PoPs   ───────────────────────────────────────────────────────────┤   
│ Routing Architecture  ┼  ─ 5-layer routing stack diagram: DNS → Edge (CDN/GA) → Regional LB (NLB→ALB) → Service Mesh (Envoy) → Data Routing (shard/cell). Each     │   
│ Overview              │ 5-layer routing stack diagrtm: DNS → Edge (CDN/GA) → Regional LB (NLB→ALB) → Service M sh (Envoy) → Data Routing (shard/cell). Each layer  │   
└───────────────────────│─independent with its─own health checking.         ─────────────────────────────────────────────────────────────────────────────────────────┘   
├                      ┼─  ─                                                                                                                                         ┤   
──│─DNS Layer────────────│ Route 53 policy stack: latency-based (default) + geolocation (GDPR compliance) + failover (HA) + weighted (migration). TTL strategy per ────│───
│                      │ record type. Actual resolution chain: amazon.com → CloudFront → regional ALB via latency routing.                                           │   
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤───
│ Edge Layer           │ CloudFront for static (96% hit rate) + dynamic (TCP optimization, not caching). Global Accelerator for latency-critical APIs (AWS private   │   
│                      │ backbone, 30% lower latency than public internet). When to use which.                                                                       │   
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ 3-Tier Load          │ NLB (L4, millions of connections) → ALB (L7, path/header/weighted routing) → Envoy sidecar (service mesh, retries, circuit breaking). Full  │   
│ Balancing            │ ALB routing rule config: path-based, header-based, weighted canary, maintenance mode. Cross-zone LB trade-offs.                             │   
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ AZ Architecture      │ What an AZ physically is. N+1 AZ design (3 AZs, sized so any 2 handle full load = 50% headroom). topologySpreadConstraints to guarantee     │   
│                      │ even pod distribution.                                                                                                                      │   
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Multi-Region         │ Active-active reads, single-primary writes per user's home region. Region selection at account creation. Write routing: always to home      │   
│                      │ region primary (even if user is traveling). Read routing: nearest replica. Cross-region write penalty: +80ms (acceptable).                  │   
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Cell-Based           │ 20 cells per region, each handling 5% of users. Independent service fleet + DB shards + Redis + Kafka per cell. Cell routing via X-Cell-ID  │   
│ Architecture         │ header from auth token. Shared services (search, catalog, CDN) outside cells. Blast radius: 1 cell failure = 2% of global users.            │   
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Data Layer Routing   │ Shard-to-cell mapping (320 shards across 20 cells). Primary spread across AZs within a cell. Read/write split with "sticky to primary for   │   
│                      │ 5s after write" for read-your-writes consistency.                                                                                           │   
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Service Mesh         │ Full Envoy sidecar config: AZ-aware routing, cell-affinity metadata matching, circuit breakers, outlier detection (eject unhealthy after 5  │   
│                      │ consecutive 5xx), retry policy with 20% retry budget to prevent retry storms.                                                               │   
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Health Checking      │ Health checks at every layer (Route 53 10s, ALB 10s, Envoy 5s, K8s probes, DB 1s, Redis Sentinel). Deep vs shallow checks: liveness =       │   
│                      │ shallow (don't kill pods because DB is slow), readiness = deep (stop routing traffic).                                                      │   
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Failover Strategies  │ AZ failure playbook (5-60s, no DNS change, ALB absorbs internally). Region failure playbook (30-90s, DNS-driven, reads from replica, writes │   
│                      │  blocked or buffered). Gradual recovery with traffic ramp.                                                                                  │   
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Traffic Shifting     │ Canary via ALB weighted target groups (5% → 25% → 50% → 100% over 45 min). Cell-based deployments (deploy to 1 cell, monitor, expand).      │   
│                      │ Auto-rollback triggers.                                                                                                                     │   
├──────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Rate Limiting        │ 4-layer rate limiting: CDN/edge (per-IP) → API Gateway (per-user) → Service (per-cell) → Database (connection pool). Priority-based load    │   
│                      │ shedding (checkout > cart > search > recommendations).                                                                                      │   
Key Content: Health checks at every layer (Route 53 10s, ALB 10s, Envoy 5s, K8s probes, DB 1s, Redis Sentinel). Deep vs shallow checks: liveness = shallow (don't kill
pods because DB is slow), readiness = deep (stop routing traffic).                                                                                                     
────────────────────────────────────────                                                                                                                                 
Section: Failover Strategies                                                                                                                                          
Key Content: AZ failure playbook (5-60s, no DNS change, ALB absorbs internally). Region failure playbook (30-90s, DNS-driven, reads from replica, writes blocked or      
buffered). Gradual recovery with traffic ramp.                                                                                                                         
────────────────────────────────────────                                                                                                                              
Section: Traffic Shifting                                                                                                                                                
Key Content: Canary via ALB weighted target groups (5% → 25% → 50% → 100% over 45 min). Cell-based deployments (deploy to 1 cell, monitor, expand). Auto-rollback        
triggers.
────────────────────────────────────────                                                                                                                                 
Section: Rate Limiting                                                                                                                                                 
Key Content: 4-layer rate limiting: CDN/edge (per-IP) → API Gateway (per-user) → Service (per-cell) → Database (connection pool). Priority-based load shedding (checkout
>
    cart > search > recommendations).
────────────────────────────────────────
Section: Concrete Configs
Key Content: Full Route 53 JSON, Kubernetes Deployment + HPA YAML (topologySpreadConstraints, readiness/liveness probes, scale-up/down behavior), PgBouncer INI
(transaction pooling, connection limits).
────────────────────────────────────────
Section: Observability
Key Content: Global traffic routing dashboard: DNS resolution rates, regional distribution, AZ health, cell status, ALB target health, canary monitoring.
────────────────────────────────────────
Section: Corner Cases
Key Content: 8 hard problems: (1) DNS caching beyond your control (ISP overrides, JVM forever-cache) → GA anycast + happy eyeballs + JVM config. (2) Thundering herd on
failover recovery → ALB slow-start + gradual re-introduction. (3) Split-brain DNS (healthy region appears dead to health checkers) → multi-location checkers + internal

    canary. (4) Cross-region consistency during failover (200ms replication gap) → client-side write-ahead + write buffer. (5) Cascading failure across cells (retry loops)

    → overflow routing + retry budget + cold-start protection. (6) AZ-imbalanced deployment → AZ-aware rolling update (one AZ at a time). (7) Payment routing during region

    failover → route by user's home region key, not serving region. (8) Shard rebalancing under live traffic → dual-read/single-write phases with 200ms write pause at
    switchover.
