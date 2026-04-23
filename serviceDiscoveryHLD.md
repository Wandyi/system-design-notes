# Service Discovery in Multi-AZ, Multi-Region E-Commerce
## Comprehensive Design for Amazon-Scale Infrastructure

## Table of Contents

1. [The Problem Service Discovery Solves](#1-the-problem-service-discovery-solves)
2. [Scale & Constraints](#2-scale--constraints)
3. [Discovery Patterns — Taxonomy](#3-discovery-patterns--taxonomy)
4. [Service Registry Architecture](#4-service-registry-architecture)
5. [Registration Lifecycle](#5-registration-lifecycle)
6. [Health Checking Deep Dive](#6-health-checking-deep-dive)
7. [AZ-Aware Discovery](#7-az-aware-discovery)
8. [Multi-Region Discovery](#8-multi-region-discovery)
9. [Cell-Aware Discovery](#9-cell-aware-discovery)
10. [Service Mesh & Control Plane (Envoy xDS)](#10-service-mesh--control-plane-envoy-xds)
11. [DNS-Based Discovery (AWS Cloud Map)](#11-dns-based-discovery-aws-cloud-map)
12. [Load Balancing Integration](#12-load-balancing-integration)
13. [Version-Aware Routing & Traffic Splitting](#13-version-aware-routing--traffic-splitting)
14. [Graceful Shutdown & Connection Draining](#14-graceful-shutdown--connection-draining)
15. [Configuration & Concrete Examples](#15-configuration--concrete-examples)
16. [Scalability Deep Dive](#16-scalability-deep-dive)
17. [Reliability & Fault Tolerance Deep Dive](#17-reliability--fault-tolerance-deep-dive)
18. [Observability](#18-observability)
19. [Corner Cases & Hard Problems](#19-corner-cases--hard-problems)

---

## 1. The Problem Service Discovery Solves

### Why Static Configuration Fails at Scale

```
Small system (10 services, 3 instances each):
  order-service talks to payment-service.
  Config file: payment-service.host = 10.0.1.42:8080
  Works fine. Admin updates the IP if it changes.

Amazon-scale (1,000+ services, 1,000,000+ instances):
  order-service talks to payment-service.
  payment-service has 5,000 instances across 3 AZs in 5 regions.
  Instances are created and destroyed every second (auto-scaling, deployments, crashes).
  
  Static config file with 5,000 IPs?
  - Stale within minutes (auto-scaling adds/removes instances)
  - Wrong after every deployment (new pods get new IPs)
  - Deadly after an AZ failure (1,700 IPs suddenly unreachable)
  - Impossible to maintain across 1,000 services
  
Service discovery is the answer: a dynamic, real-time registry of
"what instances of service X are alive, where are they, and how do I reach them?"
```

### What Must Be Resolved at Query Time

```
When order-service wants to call payment-service, discovery must answer:

  1. WHAT instances exist?
     → payment-service has instances at 10.0.1.42, 10.0.2.18, 10.0.3.91, ...

  2. WHERE are they?
     → 10.0.1.42 is in us-east-1a (same AZ as the caller — prefer this)
     → 10.0.2.18 is in us-east-1b (same region, different AZ — second choice)
     → 10.0.3.91 is in eu-west-1a (different region — only if needed)

  3. WHICH are healthy?
     → 10.0.1.42 is healthy (last health check passed 2s ago)
     → 10.0.2.18 is unhealthy (last 3 health checks failed)
     → 10.0.3.91 is healthy

  4. WHAT version / metadata?
     → 10.0.1.42 is running v2.14 (canary)
     → 10.0.3.91 is running v2.13 (stable)
     → Caller wants stable → skip 10.0.1.42

  5. WHAT protocol / port?
     → gRPC on port 9090 (not HTTP on 8080)
     → mTLS required

All of this resolved in < 1ms, millions of times per second, with
zero single points of failure.
```

---

## 2. Scale & Constraints

```
Amazon-scale service discovery numbers:

  Services:                    ~1,000-2,000 distinct service types
  Total instances:             ~1,000,000+
  Instances per service (avg): ~500 (range: 3 to 50,000)
  Instance churn:              ~10,000 registrations/deregistrations per minute
                               (deployments, auto-scaling, spot terminations)
  
  Discovery queries:
    Every service-to-service call resolves an endpoint.
    50M API calls/sec (internal, service-to-service)
    Each call needs endpoint resolution.
    
    With caching (10s cache at sidecar): 50M / 10 = 5M cache misses/sec
    Without caching: 50M resolution queries/sec (impossible to centralize)

  Registry state:
    1M instances × ~500 bytes metadata = 500 MB total registry state
    Small enough to fit in memory on any single node.
    The problem is not storage — it's CONSISTENCY and AVAILABILITY at this query rate.

  Health checks:
    1M instances × 1 health check every 10 seconds = 100,000 health checks/sec
    Distributed health checking is mandatory (centralized checker can't keep up).
```

---

## 3. Discovery Patterns — Taxonomy

### Pattern 1: Client-Side Discovery

```
  ┌─────────────┐         ┌──────────────┐
  │ order-service│────────→│   Registry   │  1. Query registry for payment-service endpoints
  │ (caller)    │←────────│   (Consul/   │  2. Registry returns: [10.0.1.42, 10.0.3.91]
  │             │         │    Eureka)   │
  └──────┬──────┘         └──────────────┘
         │
         │  3. Client picks one (round-robin, least-connections, etc.)
         │
         ▼
  ┌──────────────┐
  │ payment-svc  │
  │ 10.0.1.42    │
  └──────────────┘

  Pros: No intermediate proxy (lower latency). Client has full control over LB algorithm.
  Cons: Every service must implement discovery client library. Language-specific.
  
  Used by: Netflix (Eureka + Ribbon), early Spring Cloud
```

### Pattern 2: Server-Side Discovery (Load Balancer)

```
  ┌─────────────┐         ┌──────────────┐        ┌──────────────┐
  │ order-service│────────→│  Load        │───────→│ payment-svc  │
  │ (caller)    │         │  Balancer    │        │ 10.0.1.42    │
  └─────────────┘         │  (ALB/NLB)  │        └──────────────┘
                          │             │        ┌──────────────┐
                          │  Resolves + │───────→│ payment-svc  │
                          │  routes     │        │ 10.0.3.91    │
                          └──────────────┘        └──────────────┘

  Caller just calls: http://payment-service-lb.internal:8080
  LB handles discovery + routing + health checking.
  
  Pros: Simple for callers (just a hostname). No client library needed.
  Cons: Extra network hop (LB in the path). LB is a scaling bottleneck.
        All traffic funnels through LB fleet.
  
  Used by: AWS ALB/NLB, Kubernetes Services (kube-proxy)
```

### Pattern 3: DNS-Based Discovery

```
  ┌─────────────┐  DNS query                  ┌──────────────┐
  │ order-service│─────────────────────────────→│ DNS Server   │
  │ (caller)    │  "payment-service.internal"  │ (Route53 /   │
  │             │←─────────────────────────────│  CoreDNS /   │
  │             │  A: 10.0.1.42, 10.0.3.91    │  Cloud Map)  │
  └──────┬──────┘                              └──────────────┘
         │
         │  Client picks one IP, connects directly.
         ▼
  ┌──────────────┐
  │ payment-svc  │
  │ 10.0.1.42    │
  └──────────────┘

  Pros: Universal (every language has DNS). No special library.
  Cons: DNS TTL causes stale entries. DNS doesn't support rich metadata
        (health, version, AZ). Limited load balancing (random from IP list).
  
  Used by: AWS Cloud Map, Consul DNS interface, Kubernetes (CoreDNS)
```

### Pattern 4: Service Mesh (Sidecar Proxy) — THE MODERN ANSWER

```
  ┌──────────────────────────────────────────────────────────────┐
  │  Pod                                                         │
  │  ┌─────────────┐     ┌───────────────┐                     │
  │  │ order-service│────→│ Envoy Sidecar │──── routes to ──→   │
  │  │ (app code)  │     │ (proxy)       │                     │
  │  │             │     │               │                     │
  │  │ Calls:      │     │ Intercepts all│                     │
  │  │ payment-svc │     │ outbound calls│                     │
  │  │ :8080       │     │               │                     │
  │  └─────────────┘     │ Resolves via  │                     │
  │                      │ control plane │                     │
  │                      │ (xDS API)     │                     │
  │                      └───────┬───────┘                     │
  └──────────────────────────────┼──────────────────────────────┘
                                 │
                   ┌─────────────▼──────────────┐
                   │  Control Plane              │
                   │  (Istio / App Mesh /        │
                   │   Consul Connect)           │
                   │                             │
                   │  Pushes endpoint updates    │
                   │  to all sidecars in real-   │
                   │  time via gRPC streaming.   │
                   │                             │
                   │  Sources:                   │
                   │  - Kubernetes API (pod IPs) │
                   │  - Consul catalog           │
                   │  - Cloud Map                │
                   └─────────────────────────────┘

  App code calls payment-service:8080 on localhost.
  Envoy sidecar intercepts (via iptables redirect).
  Envoy resolves payment-service using its local endpoint table
  (pushed from control plane, cached in-memory, always fresh).
  Envoy picks a healthy endpoint, applies retry/circuit-breaking policy, routes.

  Pros: App is completely unaware of discovery. Any language. Rich routing
        (AZ-aware, version-aware, canary, circuit breaking, retries, mTLS).
  Cons: Sidecar adds ~1ms latency per hop. Control plane is a critical dependency.
  
  Used by: Amazon (internally), Istio (Google/Lyft), Linkerd, AWS App Mesh.
  This is the standard at Amazon scale.
```

### Amazon's Actual Approach: Layered

```
Amazon doesn't use ONE pattern — they layer multiple:

  External traffic (user → Amazon):
    DNS (Route 53) → CloudFront → ALB → Pod
    Server-side discovery (ALB is the resolver)

  Internal traffic (service → service, within a region):
    Service Mesh (Envoy sidecar with internal control plane)
    + DNS as fallback (internal Route 53 / Cloud Map)
    Client-side discovery via sidecar (Pattern 4)

  Cross-region traffic (service in us-east → service in eu-west):
    DNS-based discovery (internal Route 53 with region-aware records)
    + Global Accelerator for latency-sensitive cross-region calls
```

---

## 4. Service Registry Architecture

### Registry Internals

```
The registry is the source of truth for "what's running where."
It must be:
  - Highly available (if registry is down, no service can find any other service)
  - Consistent enough (no stale entries persisting for minutes)
  - Fast (resolve in < 1ms)
  - Scalable (handle 5M+ queries/sec)

Registry data model (per instance):

  {
    "service": "payment-service",
    "instance_id": "i-abc123",
    "address": "10.0.1.42",
    "port": 9090,
    "protocol": "grpc",
    "health": "HEALTHY",
    "az": "us-east-1a",
    "region": "us-east-1",
    "cell": "cell-7",
    "version": "v2.13",
    "metadata": {
      "canary": false,
      "weight": 100,
      "capabilities": ["credit-card", "paypal", "upi"]
    },
    "registered_at": "2026-04-21T10:00:00Z",
    "last_heartbeat": "2026-04-21T10:05:32Z",
    "ttl": 30  // seconds; deregistered if no heartbeat within TTL
  }
```

### Registry Technology Options

```
┌────────────────┬───────────────┬────────────────┬──────────────────────────────┐
│ Technology     │ Consistency   │ Discovery      │ Characteristics              │
│                │ Model         │ Protocol       │                              │
├────────────────┼───────────────┼────────────────┼──────────────────────────────┤
│ Consul         │ CP (Raft)     │ HTTP API +     │ Multi-DC native, health      │
│ (HashiCorp)    │               │ DNS + gRPC     │ checking built-in, KV store. │
│                │               │                │ 5-node cluster per DC.       │
├────────────────┼───────────────┼────────────────┼──────────────────────────────┤
│ etcd           │ CP (Raft)     │ gRPC + watch   │ Kubernetes native. Used by   │
│ (CNCF)         │               │                │ K8s API server for all state.│
│                │               │                │ 3-5 nodes per cluster.       │
├────────────────┼───────────────┼────────────────┼──────────────────────────────┤
│ Eureka         │ AP (eventual) │ HTTP REST      │ Netflix. Favors availability │
│ (Netflix)      │               │                │ over consistency. Peer-to-   │
│                │               │                │ peer replication.            │
├────────────────┼───────────────┼────────────────┼──────────────────────────────┤
│ AWS Cloud Map  │ Eventually    │ DNS + HTTP API │ Managed service. Integrates  │
│ (AWS)          │ consistent    │ (AWS SDK)      │ with Route 53, ECS, EKS.     │
│                │               │                │ No cluster to manage.        │
├────────────────┼───────────────┼────────────────┼──────────────────────────────┤
│ Kubernetes API │ CP (etcd)     │ K8s Service +  │ Built-in to K8s. Service     │
│ (K8s native)   │               │ CoreDNS        │ endpoints auto-updated.      │
│                │               │                │ Only within-cluster.         │
├────────────────┼───────────────┼────────────────┼──────────────────────────────┤
│ Envoy xDS      │ Eventual      │ gRPC streaming │ Not a registry itself —      │
│ (control plane)│ (push-based)  │ (ADS/EDS/CDS)  │ reads from a registry and    │
│                │               │                │ pushes to sidecars.          │
└────────────────┴───────────────┴────────────────┴──────────────────────────────┘

Amazon's internal approach:
  - Custom internal service registry (similar to Consul but built in-house)
  - Backed by a replicated data store (similar to etcd/Raft)
  - Exposed via DNS (for simple callers) and gRPC (for Envoy sidecars)
  - Per-region registry clusters (not one global registry)
  - Cross-region: DNS-based (Route 53 private hosted zones)
```

### Registry Topology (Per Region)

```
                    ┌────────────────────── us-east-1 ──────────────────────┐
                    │                                                       │
                    │  ┌──────────────────────────────────────────────┐     │
                    │  │          Registry Cluster (Raft)             │     │
                    │  │                                              │     │
                    │  │  ┌────────┐  ┌────────┐  ┌────────┐          │     │
                    │  │  │ Node 1 │  │ Node 2 │  │ Node 3 │          │     │
                    │  │  │ (AZ-A) │  │ (AZ-B) │  │ (AZ-C) │         │      │
                    │  │  │ LEADER │  │FOLLOWER│  │FOLLOWER│         │      │
                    │  │  └───┬────┘  └───┬────┘  └───┬────┘         │      │
                    │  │      │           │           │              │      │
                    │  │      └───────────┼───────────┘              │      │
                    │  │                  │ Raft consensus            │     │
                    │  └──────────────────┼───────────────────────────┘     │
                    │                     │                                 │
                    │         ┌───────────▼────────────┐                    │
                    │         │   Read Replicas         │                   │
                    │         │   (per-AZ, for queries) │                   │
                    │         │                        │                    │
                    │         │  AZ-A: 3 read replicas │                    │
                    │         │  AZ-B: 3 read replicas │                    │
                    │         │  AZ-C: 3 read replicas │                    │
                    │         │                        │                    │
                    │         │  Total read capacity:  │                    │
                    │         │  9 replicas × 500K QPS │                    │
                    │         │  = 4.5M queries/sec    │                    │
                    │         └────────────────────────┘                    │
                    │                                                        │
                    └────────────────────────────────────────────────────────┘

Writes (registrations, deregistrations): → Raft leader → replicated to followers
  Write throughput: ~10,000/sec (sufficient for instance churn)

Reads (endpoint resolution): → any read replica (local AZ preferred)
  Read throughput: 4.5M/sec (handles all service-to-service resolution)
```

---

## 5. Registration Lifecycle

### Instance Registration Flow

```
Pod starts up → registration sequence:

  t=0s     Kubernetes scheduler places pod in AZ-A.
           Pod IP: 10.0.1.42
           
  t=0-3s   Container starts. Application initializing.
           NOT registered yet. NOT receiving traffic.
           
  t=3-5s   Application completes initialization:
           - DB connection pool established
           - Caches warmed
           - Feature flags loaded
           
  t=5s     Application starts HTTP server on :8080
           Health endpoint /health/ready returns 200.
           
  t=5s     Kubernetes readiness probe succeeds.
           Kubelet adds pod IP to Service Endpoints.
           
  t=5-6s   Control plane (Istio / App Mesh) detects new endpoint.
           Pushes EDS update to ALL Envoy sidecars in the mesh:
           
           "payment-service now includes 10.0.1.42:9090 in us-east-1a"
           
  t=6s     Instance is DISCOVERABLE.
           Other services' sidecars now include 10.0.1.42 in their LB pool.
           Traffic starts flowing to the new instance.
           
  Total: ~6 seconds from pod start to receiving traffic.
  
  Alternatively (self-registration pattern):
  
  t=5s     Application calls registry API directly:
           PUT /v1/agent/service/register
           Body: { service: "payment-service", address: "10.0.1.42", port: 9090,
                   check: { http: "http://10.0.1.42:8080/health", interval: "10s" } }
           
           Registry stores the entry, starts health checking.
```

### Heartbeat / TTL-Based Liveness

```
After registration, the instance must prove it's still alive.

Two approaches:

Approach 1: Registry actively probes the instance (Consul default)
  Registry sends HTTP GET /health to 10.0.1.42:8080 every 10 seconds.
  3 consecutive failures → mark unhealthy → remove from discovery results.
  
  Problem at scale: 1M instances × 1 probe/10s = 100K probes/sec from registry.
  Registry cluster must sustain 100K outbound HTTP calls/sec.
  Distributed: each registry node probes its own subset of instances.

Approach 2: Instance sends heartbeat to registry (Eureka default)
  Instance sends PUT /heartbeat to registry every 30 seconds.
  If registry doesn't receive heartbeat within TTL (90s) → deregistered.
  
  Problem: If instance crashes, 90 seconds of stale entry.
  Mitigation: Shorten TTL to 30s, heartbeat every 10s.
  3 missed heartbeats = 30s to detect death.

Approach 3: Sidecar reports (Kubernetes / Service Mesh)
  Kubelet runs readiness probe locally (no network call to registry).
  If probe fails → kubelet removes pod from Endpoints immediately.
  Control plane pushes update to all sidecars within 1-2 seconds.
  
  Detection time: probe interval (5s) + propagation (1-2s) = ~7 seconds.
  No registry polling needed. Scales to any number of instances.

Amazon uses Approach 3 for Kubernetes-based services.
Approach 1 (Consul-style) for legacy EC2-based services.
```

---

## 6. Health Checking Deep Dive

### Multi-Level Health Checks

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Health Check Hierarchy                              │
│                                                                         │
│  Level 1: PROCESS ALIVE (Liveness)                                      │
│    Check: Can I open a TCP connection to port 8080?                     │
│    Failure means: Process crashed or OOM-killed.                        │
│    Action: Restart container (Kubernetes liveness probe).               │
│    Interval: 10s. Threshold: 3 failures.                                │
│    DO NOT check downstream dependencies here.                           │
│    If DB is down and liveness check fails → all pods restart            │
│    → now you have no pods AND no DB = worse.                            │
│                                                                         │
│  Level 2: READY TO SERVE (Readiness)                                    │
│    Check: GET /health/ready → checks:                                   │
│      ✓ DB connection pool has available connections                     │
│      ✓ Redis is reachable                                               │
│      ✓ Feature flags loaded                                             │
│      ✓ Warmup complete                                                  │
│    Failure means: Instance is alive but can't serve traffic.            │
│    Action: Remove from Service Endpoints (no traffic routed).           │
│    Pod stays running. May recover on its own.                           │
│    Interval: 5s. Threshold: 1 failure.                                  │
│                                                                         │
│  Level 3: DEEP HEALTH (Application-level)                               │
│    Check: GET /health/deep → tests actual request path:                 │
│      ✓ Can execute a trivial DB query (SELECT 1)                        │
│      ✓ Can read from cache                                              │
│      ✓ Downstream services reachable (circuit breakers not all open)    │
│    Used by: Registry (Consul) or control plane for routing decisions.   │
│    Not used for pod lifecycle — only for discovery/routing.             │
│    Interval: 10s. Threshold: 3 failures = marked unhealthy in registry. │
│                                                                         │
│  Level 4: SYNTHETIC TRANSACTIONS (End-to-end)                           │
│    Check: External canary makes a real API call every 30 seconds.       │
│      Place a test order, verify it completes, cancel it.                │
│    Failure means: The SERVICE is broken (even if individual pods are    │
│      "healthy" — e.g., a bug that causes 500 on all /checkout calls).   │
│    Used by: Route 53 health check → region-level failover.              │
│    Interval: 30s.                                                       │
│                                                                         │
│  Each level catches different failure modes.                            │
│  Level 1 catches crashes.                                               │
│  Level 2 catches initialization failures.                               │
│  Level 3 catches dependency outages.                                    │
│  Level 4 catches business logic bugs.                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Health Check Response Contract

```json
// GET /health/ready (Level 2)
// HTTP 200 = healthy, HTTP 503 = unhealthy

// Healthy response:
{
  "status": "UP",
  "checks": {
    "database": { "status": "UP", "latency_ms": 2 },
    "redis": { "status": "UP", "latency_ms": 0.3 },
    "feature_flags": { "status": "UP", "loaded_at": "2026-04-21T10:00:00Z" }
  },
  "uptime_seconds": 3600,
  "version": "v2.13"
}

// Unhealthy response (HTTP 503):
{
  "status": "DOWN",
  "checks": {
    "database": { "status": "DOWN", "error": "connection pool exhausted" },
    "redis": { "status": "UP", "latency_ms": 0.3 },
    "feature_flags": { "status": "UP" }
  }
}
```

---

## 7. AZ-Aware Discovery

### The Core Principle: Prefer Same-AZ

```
Cross-AZ call: ~1-2ms extra RTT + $0.01/GB data transfer cost.
Same-AZ call:  ~0.1ms RTT + $0 data transfer cost.

At 50M internal API calls/sec:
  If all cross-AZ: 50M × 1.5ms extra = 75,000 CPU-seconds/sec wasted on latency
  And: 50M × 1 KB avg response × $0.01/GB = $500/sec = $43M/year in data transfer

Same-AZ preference: saves latency AND millions of dollars per year.
```

### AZ-Aware Resolution

```
order-service (in us-east-1a) calls payment-service:

  Step 1: Envoy sidecar has the full endpoint list for payment-service:
    [
      { addr: "10.0.1.42:9090", az: "us-east-1a", health: "HEALTHY" },  ← same AZ
      { addr: "10.0.1.43:9090", az: "us-east-1a", health: "HEALTHY" },  ← same AZ
      { addr: "10.0.2.18:9090", az: "us-east-1b", health: "HEALTHY" },
      { addr: "10.0.2.19:9090", az: "us-east-1b", health: "HEALTHY" },
      { addr: "10.0.3.91:9090", az: "us-east-1c", health: "HEALTHY" },
      { addr: "10.0.3.92:9090", az: "us-east-1c", health: "HEALTHY" },
    ]

  Step 2: Envoy zone-aware routing:
    If same-AZ endpoints are healthy AND have capacity:
      → Route 100% to same-AZ endpoints (10.0.1.42, 10.0.1.43)
    
    If same-AZ endpoints are overloaded (> 80% CPU or high error rate):
      → Spill to other AZs: 60% same-AZ, 20% AZ-B, 20% AZ-C
    
    If same-AZ endpoints are ALL unhealthy:
      → Route to other AZs: 50% AZ-B, 50% AZ-C
      → Log alert: "payment-service has no healthy endpoints in us-east-1a"

  Step 3: Envoy's zone-aware config:

    clusters:
      - name: payment-service
        lb_policy: ROUND_ROBIN
        common_lb_config:
          zone_aware_lb_config:
            routing_enabled:
              value: 100        # enable zone-aware routing
            min_cluster_size: 6  # need at least 6 endpoints total
                                # (below this, zone-awareness is disabled
                                #  to avoid uneven distribution with few endpoints)
```

### AZ Failure: Discovery Response

```
Scenario: us-east-1a loses power.

t=0s    All pods in AZ-A stop sending heartbeats / fail health checks.
        
t=5-7s  Kubelet readiness probes fail → pods removed from Endpoints.
        Control plane detects: all AZ-A endpoints gone.
        
t=7-8s  Control plane pushes EDS update to ALL sidecars:
        payment-service endpoints: [AZ-B and AZ-C only]
        
t=8s    All callers in AZ-B and AZ-C automatically route to local AZ.
        Callers that were in AZ-A are also dead → no issue.
        
No DNS change. No routing table change. No human intervention.
Discovery update propagated to all sidecars in < 3 seconds.
Much faster than DNS-based failover (60s+ TTL).
```

---

## 8. Multi-Region Discovery

### Per-Region Registries (Federated Model)

```
**Each region has its OWN independent registry cluster.**
Registries do NOT replicate across regions (too slow, too fragile).

  ┌─────────────────┐           ┌─────────────────┐
  │  us-east-1      │           │  eu-west-1       │
  │  Registry Cluster│          │  Registry Cluster│
  │                  │          │                  │
  │  Knows about:    │          │  Knows about:    │
  │  - All instances │          │  - All instances │
  │    in us-east-1  │          │    in eu-west-1  │
  │  - Does NOT know │          │  - Does NOT know │
  │    about eu-west │          │    about us-east │
  └──────────────────┘          └──────────────────┘

Cross-region discovery uses DNS (Route 53 private hosted zones):

  payment-service.us-east-1.internal  → [us-east-1 endpoint IPs]
  payment-service.eu-west-1.internal  → [eu-west-1 endpoint IPs]
  payment-service.internal            → latency-based routing to nearest region

When a service in us-east-1 needs to call payment-service in eu-west-1 specifically:
  It resolves: payment-service.eu-west-1.internal
  DNS returns: eu-west-1 ALB / NLB IP (not individual pod IPs)
  The **regional LB handles intra-region routing to specific pods.**

Why not expose individual pod IPs cross-region?
  - Pod IPs are private (VPC-scoped, not routable cross-region without peering)
  - Cross-region calls go through NLB/ALB (TLS termination, health checking)
  - Keeps cross-region path simple: DNS → LB → pod
```

### Cross-Region Service Call Flow

```
order-service (us-east-1) needs to call fraud-service (eu-west-1):

  1. order-service Envoy sidecar:
     Resolves fraud-service → checks local (us-east-1) endpoints first.
     No local endpoints (fraud-service only runs in eu-west-1).
     
  2. Envoy falls back to DNS:
     Resolves fraud-service.eu-west-1.internal
     → Returns eu-west-1 NLB IP: 52.18.xx.xx
     
  3. Envoy connects to eu-west-1 NLB over VPC peering / Transit Gateway.
     Cross-region latency: ~80ms RTT (us-east → eu-west).
     
  4. eu-west-1 NLB → eu-west-1 ALB → fraud-service pod in eu-west-1.
     eu-west-1 internal routing is AZ-aware (prefer same AZ within eu-west-1).
     
  Total latency: ~80ms (network) + ~5ms (eu-west processing) = ~85ms.

Configuration (Envoy):
  clusters:
    - name: fraud-service
      type: STRICT_DNS    # resolve via DNS (not EDS, because cross-region)
      dns_lookup_family: V4_ONLY
      load_assignment:
        cluster_name: fraud-service
        endpoints:
          - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: fraud-service.eu-west-1.internal
                    port_value: 443
      transport_socket:
        name: envoy.transport_sockets.tls
        typed_config:
          sni: fraud-service.eu-west-1.internal
```

---

## 9. Cell-Aware Discovery

```
Within a region, services are further divided into cells.
Cell 7's order-service should talk to cell 7's payment-service (data locality).

How cell affinity is enforced in discovery:

  1. Each pod is labeled with its cell ID:
     metadata:
       labels:
         app: payment-service
         cell: "7"

  2. Control plane creates per-cell subsets:
     payment-service endpoints:
       subset "cell-7": [10.0.1.42, 10.0.1.43]
       subset "cell-8": [10.0.2.18, 10.0.2.19]
       ...

  3. Envoy sidecar for order-service (cell 7) is configured:
     route:
       cluster: payment-service
       metadata_match:
         cell: "7"
     
     → Only routes to payment-service instances in cell 7.

  4. If cell 7's payment-service has NO healthy endpoints:
     Fallback to any cell's payment-service (better than errors).
     Envoy config:
       retry_policy:
         retry_on: "5xx"
       # On retry, Envoy can try a different subset (different cell)

This is enforced at the mesh level, transparent to application code.
order-service just calls "payment-service" — Envoy handles cell affinity.
```

---

## 10. Service Mesh & Control Plane (Envoy xDS)

### xDS Protocol (How Envoy Learns About Endpoints)

```
xDS = "x Discovery Service" — a family of gRPC streaming APIs:

  ┌─────────────────────────────────────────────────────────────────┐
  │  xDS API    │ What it provides                                  │
  ├─────────────┼───────────────────────────────────────────────────┤
  │  EDS        │ Endpoint Discovery Service                        │
  │             │ "What IPs/ports exist for this service?"           │
  │             │ + AZ, health, weight, metadata per endpoint        │
  │             │ This is the core service discovery API.            │
  ├─────────────┼───────────────────────────────────────────────────┤
  │  CDS        │ **Cluster Discovery Service**                      │
  │             │ "What services (clusters) exist?"                  │
  │             │ + timeout, circuit breaker, LB policy per service  │
  ├─────────────┼───────────────────────────────────────────────────┤
  │  RDS        │ Route Discovery Service                           │
  │             │ "How should I route requests?"                    │
  │             │ + path matching, header matching, traffic splits  │
  ├─────────────┼───────────────────────────────────────────────────┤
  │  LDS        │ Listener Discovery Service                        │
  │             │ "What ports should I listen on?"                  │
  │             │ + TLS config, filter chains                       │
  └─────────────┴───────────────────────────────────────────────────┘

Push-based (streaming), not pull-based:
  Envoy opens a long-lived gRPC stream to the control plane.
  Control plane pushes updates whenever endpoints change.
  No polling. No TTL expiry. Updates arrive in 1-2 seconds.

  Contrast with DNS-based discovery:
    DNS: poll every TTL (60s). Up to 60s stale. No metadata.
    xDS: pushed instantly. < 2s stale. Rich metadata (AZ, version, health).
```

### Control Plane Architecture

```
  ┌────────────────────────────────────────────────────────────┐
  │                    Control Plane                             │
  │                                                            │
  │  ┌──────────────┐    ┌──────────────┐   ┌──────────────┐  │
  │  │  Kubernetes   │   │   Consul     │   │  Cloud Map   │  │
  │  │  API Server   │   │   Catalog    │   │  (AWS)       │  │
  │  │  (watches     │   │  (watches    │   │  (watches    │  │
  │  │   Endpoints)  │   │   services)  │   │   instances) │  │
  │  └──────┬───────┘    └──────┬───────┘   └──────┬───────┘  │
  │         │                   │                   │          │
  │         └───────────────────┼───────────────────┘          │
  │                             │                              │
  │                    ┌────────▼────────┐                     │
  │                    │   Aggregator    │                     │
  │                    │   (merges all   │                     │
  │                    │    sources into │                     │
  │                    │    unified      │                     │
  │                    │    endpoint     │                     │
  │                    │    list)        │                     │
  │                    └────────┬────────┘                     │
  │                             │                              │
  │                    ┌────────▼────────┐                     │
  │                    │  xDS Server     │                     │
  │                    │  (gRPC stream   │                     │
  │                    │   to all Envoys)│                     │
  │                    │                 │                     │
  │                    │  Connected      │                     │
  │                    │  sidecars:      │                     │
  │                    │  ~100,000       │                     │
  │                    │  (per CP node)  │                     │
  │                    └─────────────────┘                     │
  │                                                            │
  │  Scaled:                                                   │
  │  10 control plane nodes per region                         │
  │  Each handles 100K sidecar connections                     │
  │  Total: 1M sidecars supported (= 1M service instances)     │
  └────────────────────────────────────────────────────────────┘
```

---

## 11. DNS-Based Discovery (AWS Cloud Map)

```
For simpler services or cross-platform (non-K8s) services:

AWS Cloud Map creates DNS records in Route 53:

  Namespace: ecommerce.internal
  Service: payment-service.ecommerce.internal
  
  Instances:
    A record: payment-service.ecommerce.internal
    → 10.0.1.42 (AZ-A, healthy)
    → 10.0.2.18 (AZ-B, healthy)
    → 10.0.3.91 (AZ-C, healthy)
    
    SRV record (includes port):
    → 10.0.1.42:9090 priority=10 weight=100
    → 10.0.2.18:9090 priority=10 weight=100
    → 10.0.3.91:9090 priority=10 weight=100

  Health checks: Cloud Map integrates with Route 53 health checks.
  Unhealthy instances automatically removed from DNS responses.
  
  TTL: 10 seconds (low, for fast updates).
  
  Caller resolves:
    dig payment-service.ecommerce.internal
    → Gets 2-3 healthy IPs. Picks one. Connects directly.

Limitations vs service mesh:
  - No AZ-awareness (caller doesn't know which IP is in which AZ)
  - No rich metadata (version, cell, canary flag)
  - DNS TTL delay (up to 10 seconds stale)
  - No circuit breaking or retries at discovery layer
  
Good for: Cross-platform services, legacy services, simple architectures.
Not sufficient for: Amazon-scale cell-aware, AZ-aware, version-aware routing.
```

---

## 12. Load Balancing Integration

### How Discovery Feeds Load Balancing

```
Discovery answers "WHICH endpoints exist?"
Load balancing answers "WHICH endpoint to send THIS request to?"

They work together:

  Discovery provides: [A (AZ-A, healthy), B (AZ-B, healthy), C (AZ-B, unhealthy)]
  LB filters:          [A (AZ-A, healthy), B (AZ-B, healthy)]  (remove unhealthy)
  LB selects:          A (same AZ, preferred)

LB algorithms used in service discovery:

  Round Robin:
    A → B → A → B → A → B
    Simple. Even distribution. No awareness of endpoint load.
    
  Weighted Round Robin:
    A (weight=100), B (weight=50)
    A → A → B → A → A → B
    Useful for canary (v2 gets weight=5, v1 gets weight=95).
    
  Least Connections:
    Route to endpoint with fewest active connections.
    Better for uneven request durations (long + short requests).
    Envoy tracks active connection count per endpoint.
    
  P2C (Power of Two Choices) — Envoy's default for high-performance:
    Pick 2 random endpoints. Route to the one with fewer active requests.
    Near-optimal load distribution without global state.
    O(1) decision time. No hot-path lock.
    
  Consistent Hashing:
    hash(request_key) → endpoint.
    Same key always routes to same endpoint (cache locality).
    Used for: session affinity, cache services.
```

---

## 13. Version-Aware Routing & Traffic Splitting

```
Deploy v2.14 of payment-service to 5% of traffic:

  Control plane configuration:
    service: payment-service
    subsets:
      - name: stable
        labels:
          version: v2.13
      - name: canary
        labels:
          version: v2.14
    traffic_policy:
      - subset: stable
        weight: 95
      - subset: canary
        weight: 5

  Envoy receives this via RDS (Route Discovery Service):
    When routing to payment-service:
      95% of requests → endpoints labeled version=v2.13
      5% of requests → endpoints labeled version=v2.14

  Canary selection is CONSISTENT per user session:
    Hash(user_id) % 100 < 5 → canary
    Same user always goes to same version within a session.
    No "flickering" between versions.

  Monitoring: compare error rates between subsets.
    If canary error rate > 2x stable: auto-rollback (set canary weight to 0).
```

---

## 14. Graceful Shutdown & Connection Draining

### The Problem

```
Pod is being terminated (deployment update, scale-down, preemption).

Without graceful shutdown:
  t=0: Pod receives SIGTERM.
  t=0: Pod killed immediately.
  t=0: 50 in-flight requests get TCP RST (connection reset).
  t=0: 50 users see errors.
  t=0-5s: Discovery still has old IP. New requests routed to dead pod. More errors.

With graceful shutdown (correct implementation):

  t=0:    Pod receives SIGTERM.
  
  t=0s:   Application STOPS accepting NEW requests.
          **Readiness probe starts returning 503.**
          
  t=0-1s: Kubelet sees failed readiness probe.
          Removes pod from Service Endpoints.
          Control plane pushes EDS update: remove this endpoint.
          Envoy sidecars stop routing new requests to this pod.
          
  t=0-30s: Application finishes processing in-flight requests.
           Drains all active connections.
           Closes DB connections, flushes buffers.
           
  t=30s:   Application exits cleanly (exit code 0).
           Kubernetes confirms pod terminated.
           
  Zero errors. Zero dropped requests.

Configuration:

  spec:
    terminationGracePeriodSeconds: 45  # K8s waits up to 45s before SIGKILL
    containers:
      - name: payment-service
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]
              # Sleep 5s: gives time for Endpoints update to propagate
              # to all Envoy sidecars before the app starts draining.
              # Without this sleep: sidecars may still route to this pod
              # for 1-2 seconds after it stops accepting connections.
```

---

## 15. Configuration & Concrete Examples

### Consul Service Registration

```json
{
  "service": {
    "name": "payment-service",
    "id": "payment-service-i-abc123",
    "address": "10.0.1.42",
    "port": 9090,
    "tags": ["v2.13", "cell-7", "grpc"],
    "meta": {
      "az": "us-east-1a",
      "region": "us-east-1",
      "cell": "7",
      "version": "v2.13",
      "canary": "false",
      "protocol": "grpc"
    },
    "checks": [
      {
        "name": "gRPC health",
        "grpc": "10.0.1.42:9090",
        "grpc_use_tls": true,
        "interval": "10s",
        "timeout": "3s",
        "deregister_critical_service_after": "60s"
      },
      {
        "name": "HTTP readiness",
        "http": "http://10.0.1.42:8080/health/ready",
        "interval": "5s",
        "timeout": "2s"
      }
    ],
    "weights": {
      "passing": 100,
      "warning": 50
    }
  }
}
```

### Kubernetes Service + EndpointSlice

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: cell-7
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
spec:
  selector:
    app: payment-service
    cell: "7"
  ports:
    - name: grpc
      port: 9090
      targetPort: 9090
      protocol: TCP
  type: ClusterIP  # internal only; Envoy handles external routing

---
# Kubernetes auto-creates EndpointSlices:
# (you don't write this — K8s generates it from pod readiness)
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: payment-service-abc12
  namespace: cell-7
  labels:
    kubernetes.io/service-name: payment-service
addressType: IPv4
ports:
  - name: grpc
    port: 9090
    protocol: TCP
endpoints:
  - addresses: ["10.0.1.42"]
    conditions:
      ready: true
      serving: true
    zone: "us-east-1a"
    nodeName: "ip-10-0-1-100.ec2.internal"
  - addresses: ["10.0.2.18"]
    conditions:
      ready: true
      serving: true
    zone: "us-east-1b"
    nodeName: "ip-10-0-2-200.ec2.internal"
```

---

## 16. Scalability Deep Dive

### Registry Scaling

```
Registry read path (endpoint resolution):
  5M queries/sec (after sidecar caching)
  
  Approach 1: Sidecar caching eliminates most queries.
    Envoy caches endpoint list. Updated via push (xDS stream).
    Local resolution: ~1 microsecond (in-memory lookup).
    No query to registry at all for 99.9% of calls.
    Registry only receives: xDS connection setup + update pushes.
    
  Approach 2: Read replicas for non-mesh clients (DNS queries).
    9 read replicas per region → 4.5M DNS queries/sec capacity.
    Each replica: full copy of registry state (~500MB in-memory).

Registry write path (registrations, heartbeats):
  10,000 writes/sec (instance churn).
  
  Raft consensus: 10K writes/sec is well within Raft capacity
  (etcd handles 10K-30K writes/sec on moderate hardware).
  
  If needed: shard the registry by service name.
    Services A-M → registry cluster 1
    Services N-Z → registry cluster 2
    But rarely needed — 10K writes/sec is comfortable for a single cluster.
```

### xDS Push Scaling

```
1M Envoy sidecars, each connected to the control plane.

Problem: when payment-service adds 1 endpoint, all 1M sidecars
  need to be notified (they might all call payment-service).
  
Optimization: Incremental EDS.
  Don't push the FULL endpoint list (5,000 endpoints × 500 bytes = 2.5MB).
  Push only the DELTA: "added 10.0.1.99:9090 to payment-service" (~100 bytes).
  
  1M sidecars × 100 bytes = 100MB per push.
  10 CP nodes → 10MB per node per push.
  At 1 push/sec: 10MB/s per CP node → trivial.

Further optimization: Not all sidecars call all services.
  order-service only calls: payment-service, inventory-service, user-service (3 services).
  It only subscribes to EDS for those 3 services.
  It does NOT receive updates for the other 997 services.
  
  Control plane tracks subscriptions:
    sidecar-abc123 watches: [payment-service, inventory-service, user-service]
  
  When payment-service changes: push only to sidecars subscribed to payment-service.
  Typically: 50K-200K sidecars per service (not 1M).
```

---

## 17. Reliability & Fault Tolerance Deep Dive

### What Happens When the Registry Goes Down

```
Scenario: The entire registry cluster in us-east-1 becomes unreachable.

Impact analysis:

  1. NEW service registrations: FAIL.
     Newly starting pods cannot register.
     But: existing pods already running are unaffected.
     
  2. Endpoint resolution (Envoy sidecars): UNAFFECTED.
     Envoy has the endpoint list cached in-memory.
     It was pushed via xDS stream.
     If the xDS stream disconnects, Envoy KEEPS its last known configuration.
     
     This is critical: Envoy is designed to survive control plane outages.
     "Last known good" config serves traffic indefinitely.
     
  3. Health checking: PARTIALLY AFFECTED.
     Registry-driven health checks stop (registry is down).
     But: Envoy's local outlier detection continues working.
     Envoy detects unhealthy endpoints via real traffic (5xx responses, timeouts).
     Removes them from its local LB pool.
     
  4. DNS-based discovery: AFFECTED.
     DNS entries are served from Route 53 (managed, separate from registry).
     Cloud Map may not update DNS records (can't read from registry).
     Stale DNS entries served until registry recovers.
     
  5. New deployments: AFFECTED.
     New pods register but discovery doesn't update.
     Old pods being terminated are removed from Kubernetes Endpoints (K8s API is separate).
     But Envoy sidecars may not get the update (xDS stream is down).
     Stale config: Envoy may route to terminated pods → connection refused → retry to another.

Recovery:
  Registry cluster recovers (Raft leader election, typically < 30s).
  xDS streams reconnect. Full state sync pushed to all sidecars.
  New registrations processed. Health checks resume.
  
Blast radius during outage:
  If outage is < 5 minutes: virtually zero user impact.
  Envoy's cached config handles everything.
  Only new pods starting during the outage have delayed discovery.
```

### Split-Brain Registry

```
Problem: Network partition splits the 3-node Raft cluster.
  Node 1 (AZ-A): LEADER, isolated from nodes 2 and 3.
  Nodes 2, 3 (AZ-B, AZ-C): form new quorum, elect new LEADER.

  Now two leaders exist.

  Node 1 (old leader):
    Accepts writes from AZ-A services → succeeds locally.
    Cannot replicate → writes are NOT committed (Raft requires majority).
    Node 1 eventually steps down (Raft: leader without quorum self-demotes).
    Any writes accepted during the partition are ROLLED BACK.
    
  Nodes 2-3 (new leader):
    Accepts writes from AZ-B and AZ-C services → committed (has majority).
    These writes are durable.

  When partition heals:
    Node 1 discovers nodes 2-3 have a higher term.
    Node 1 becomes follower. Replays committed log from new leader.
    Any uncommitted local writes on Node 1 are discarded.
    
  Impact on discovery:
    AZ-A services temporarily can't register (old leader can't commit).
    AZ-B and AZ-C services: unaffected.
    Existing discovery (Envoy caches): unaffected in all AZs.
    
  This is why Raft-based registries are preferred:
    Split-brain is automatically resolved. No manual intervention.
    Availability may be temporarily reduced in the minority partition.
    But correctness (no conflicting registrations) is always preserved.
```

---

## 18. Observability

### Discovery Health Dashboard

```
┌─────────────────────────────────────────────────────────────────────┐
│               SERVICE DISCOVERY — HEALTH                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  REGISTRY CLUSTER                 SIDECARS                          │
│  ├─ Leader: node-2 (AZ-B)  ✓    ├─ Connected: 312,487 / 314,000     │
│  ├─ Followers: 2/2 healthy  ✓   ├─ Disconnected: 1,513 (0.48%)      │
│  ├─ Raft commit index: 48.2M    │   (mostly draining pods)          │
│  ├─ Write latency: p50=1ms      ├─ Config version lag: p50=0s       │
│  │                  p99=8ms     │                      p99=1.2s     │
│  └─ Read replicas: 9/9 healthy  └─ Stale configs: 47 (0.015%)       │
│                                                                     │
│  SERVICE ENDPOINTS                                                  │
│  ├─ Total registered: 314,000                                       │
│  ├─ Healthy: 312,102 (99.4%)                                        │
│  ├─ Unhealthy: 1,898 (0.6%)                                         │
│  ├─ Churn rate: 142/min (normal for deployments)                    │
│  └─ Services with 0 healthy: 0  ✓                                   │
│                                                                     │
│  AZ DISTRIBUTION                                                    │
│  ├─ us-east-1a: 104,821 (33.4%) ✓                                   │
│  ├─ us-east-1b: 104,312 (33.2%) ✓                                   │
│  └─ us-east-1c: 104,867 (33.4%) ✓                                   │
│                                                                     │
│  ALERTS                                                             │
│  └─ ✓ All nominal                                                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 19. Corner Cases & Hard Problems

### 1. Stale Endpoint After Sudden Crash (No Graceful Shutdown)

```
Problem: A pod is OOM-killed. No SIGTERM. No deregistration.
  The pod's IP (10.0.1.42) is gone instantly.
  But the registry still has it registered (last heartbeat 5s ago, TTL 30s).
  For the next 25 seconds, discovery returns a dead endpoint.
  Callers get connection refused.

Detection timeline:
  t=0:    Pod OOM-killed. Gone.
  t=5s:   Kubelet detects pod is dead. Removes from Endpoints.
  t=5-7s: Control plane pushes EDS update. Sidecars remove endpoint.
  
  Window of errors: 5-7 seconds.

Mitigation (minimizing the window):
  1. Envoy outlier detection: after 3 consecutive connection failures to 10.0.1.42,
     Envoy ejects it from its local LB pool (within seconds, without waiting for
     control plane update). Configured:
     
     outlier_detection:
       consecutive_gateway_errors: 3   # 3 TCP failures = eject
       interval: 5s
       base_ejection_time: 15s
       
     First caller gets error → retry to different endpoint (succeeds).
     Second and third callers: same.
     After 3 failures: Envoy ejects 10.0.1.42. No more errors.
     
  2. Client-side retry: Envoy retries on connection failures.
     First attempt: 10.0.1.42 → connection refused.
     Retry: 10.0.2.18 → success.
     User never sees an error.
     
  Net effect: 1-3 requests fail and are transparently retried.
  Zero user-visible errors despite a sudden pod crash.
```

### 2. Container IP Reuse (AKA the "Phantom Endpoint" Problem)

```
Problem: Pod A (payment-service) at 10.0.1.42 dies.
  30 seconds later, Pod B (order-service) starts and gets IP 10.0.1.42.
  (Kubernetes recycles IPs from the pod CIDR.)

  If the registry still has the STALE entry:
    payment-service → 10.0.1.42
  
  Caller tries to reach payment-service at 10.0.1.42.
  They connect to order-service instead (wrong service!).
  
  The request may fail (wrong API) or — worse — succeed with wrong data.

Why this happens:
  - Pod CIDR is finite (e.g., /16 = 65,536 IPs)
  - With high pod churn, IPs are reused within minutes
  - Registry deregistration lag: 5-30 seconds after pod death
  - New pod registration: immediate

Mitigation:
  1. Readiness-based registration: Pod B isn't added to Endpoints
     until its readiness probe passes. By that time, Pod A's entry
     is already removed (Kubelet removed it when Pod A died).
     The overlap window is extremely small (< 1 second).
     
  2. **mTLS with service identity**: Even if Envoy routes to the wrong IP,
     TLS handshake includes the service certificate.
     Envoy expects: cert CN = "payment-service"
     Pod B presents: cert CN = "order-service"
     TLS handshake fails → connection rejected → retry to correct endpoint.
     
     With mTLS, IP reuse CANNOT cause cross-service routing. Ever.
     This is the definitive solution.
     
  3. Kubernetes EndpointSlice includes pod UID:
     Even if IP is reused, the UID is different.
     Envoy can distinguish: same IP, different pod = different endpoint.
```

### 3. Registry Stampede After Cluster Restart

```
Problem: Registry cluster restarts (upgrade, failure recovery).
  All 1M Envoy sidecars lost their xDS connection.
  All 1M reconnect simultaneously.
  Each requests a FULL state dump (all endpoint data for their subscriptions).
  
  1M full syncs × ~50KB each = 50 GB of data to serve in seconds.
  Registry cluster overwhelmed on restart.

Mitigation:
  1. Sidecar reconnect jitter:
     On disconnect, Envoy waits a random delay (0-30 seconds) before reconnecting.
     1M reconnects spread over 30 seconds = ~33K/sec (manageable).
     
     Envoy config:
       initial_backoff: 1s
       max_backoff: 30s
       # Automatic jitter built into Envoy's reconnect logic
     
  2. Incremental state sync:
     Instead of full dump, sidecars send their last known config version.
     Registry sends only changes since that version.
     If sidecar was disconnected for < 5 minutes: delta is tiny.
     Full dump only if sidecar was disconnected for > 5 minutes (rare).
     
  3. Read replica shielding:
     xDS connections go to read replicas, not the Raft leader.
     Read replicas can be scaled horizontally (add more during restart).
     Leader focuses on accepting registrations and Raft consensus.
```

### 4. Flapping Service (Rapid Healthy ↔ Unhealthy Oscillation)

```
Problem: payment-service pod is borderline unhealthy.
  Health check: passes → fails → passes → fails (every 5 seconds).
  
  Each state change triggers:
  - Endpoint added/removed from EDS
  - Push to all subscribed sidecars (~100K pushes)
  - Sidecars update their LB pool
  
  Flapping at 5-second interval: 12 pushes/minute × 100K sidecars
  = 1.2M push messages/minute for ONE flapping pod.
  Multiply by 10 flapping pods = 12M pushes/minute.
  Control plane drowns in spurious updates.

Mitigation:
  1. Hysteresis (damping):
     Don't change health status on a single probe result.
     Require N consecutive failures to mark unhealthy.
     Require M consecutive successes to mark healthy again.
     
     unhealthy_threshold: 3  (3 failures to go unhealthy)
     healthy_threshold: 3    (3 successes to go healthy again)
     
     This means: 3 × 5s = 15 seconds of sustained failure before removal.
     Prevents oscillation from causing churn.
     
  2. Outlier detection (traffic-based, not probe-based):
     Envoy ejects endpoints based on ACTUAL error rate, not just health probes.
     If 10.0.1.42 has 30% error rate while others have 1%:
     → Eject it for 30 seconds. If it's still bad after re-introduction: eject for 60s.
     Exponential ejection time prevents flapping.
     
  3. Control plane dedup:
     Control plane batches EDS updates.
     If an endpoint flips healthy → unhealthy → healthy within 5 seconds:
     Net change = 0. Don't push any update.
```

### 5. Zero-Downtime Deployment with Discovery Lag

```
Problem: Rolling update of payment-service from v2.13 to v2.14.
  Kubernetes terminates a v2.13 pod. Starts a v2.14 pod.
  
  Between termination and registration:
  - Old pod removed from Endpoints (immediate)
  - New pod not yet in Endpoints (readiness probe hasn't passed)
  - For a brief window: fewer total endpoints
  
  If deployment rolls too fast (many pods cycling simultaneously):
  - Significant capacity drop
  - Remaining pods overloaded

Mitigation:
  1. maxUnavailable: 1 (only 1 pod cycling at a time)
     Ensures at most 1/N capacity reduction.
     For 30 pods: 1/30 = 3.3% capacity reduction. Negligible.
     
  2. minReadySeconds: 30
     Kubernetes waits 30 seconds after a pod is ready before
     proceeding to roll the next pod. Ensures the new pod
     is ACTUALLY handling traffic (not just passing readiness probe).
     
  3. PodDisruptionBudget:
     Prevents Kubernetes from terminating more than N pods simultaneously
     (even during node drains, cluster upgrades, etc.).
     
     apiVersion: policy/v1
     kind: PodDisruptionBudget
     metadata:
       name: payment-service-pdb
     spec:
       maxUnavailable: 1
       selector:
         matchLabels:
           app: payment-service
```

### 6. Discovery in a Multi-Cluster Federation

```
Problem: Payment-service runs in 3 Kubernetes clusters:
  Cluster A (us-east-1), Cluster B (us-west-2), Cluster C (eu-west-1).
  
  Each cluster has its own Kubernetes API / CoreDNS / Endpoints.
  Cluster A knows about its own payment-service pods but NOT about Cluster B's.

  order-service in Cluster A wants to call payment-service.
  It should prefer local (Cluster A) endpoints.
  But if Cluster A's payment-service is down: route to Cluster B.

Solution: Federated discovery layer on top of per-cluster discovery.

  Option A: Service mesh federation (Istio multi-cluster)
    Each cluster's control plane discovers its own endpoints.
    Control planes exchange endpoint lists over a federation API.
    Each sidecar knows about endpoints in ALL clusters.
    Routing priority: local cluster > same region > other region.
    
  Option B: DNS-based federation (simpler)
    Each cluster registers its service LB in Cloud Map:
      payment-service.us-east-1.internal → Cluster A NLB
      payment-service.us-west-2.internal → Cluster B NLB
    
    Global DNS:
      payment-service.internal → latency-routed to nearest cluster's NLB
    
    Cross-cluster calls go through NLB (not direct pod-to-pod).
    Extra hop but simpler to operate.
    
  Amazon likely uses a combination: mesh within a cluster, DNS between clusters.
```

### 7. Discovery During Mass Auto-Scaling Event (Prime Day)

```
Problem: Prime Day starts. Auto-scaler spins up 10,000 new pods in 2 minutes.
  Each pod registers in the registry. Each triggers an EDS push to all sidecars.
  
  10,000 registrations × 1 push each × 100K subscribed sidecars
  = 1 billion push messages in 2 minutes = 8.3M pushes/sec.

Mitigation:
  1. Batch EDS updates:
     Control plane doesn't push on EVERY registration.
     Batches updates every 1-2 seconds.
     10,000 pods in 2 minutes = ~83 pods/sec.
     At 1-second batch: each push includes ~83 new endpoints.
     120 pushes × 100K sidecars = 12M pushes in 2 minutes (manageable).
     
  2. Incremental (delta) EDS:
     Each push includes only the DIFF (+83 new endpoints), not the full list.
     Push size: ~83 × 100 bytes = 8.3 KB per push.
     12M pushes × 8.3 KB = ~100 GB total push volume over 2 minutes.
     Across 10 CP nodes: 10 GB per node over 2 minutes = 83 MB/s (comfortable).
     
  3. Progressive rollout of scaling:
     Don't add all 10,000 pods to the LB pool at once.
     Readiness probes ensure pods are healthy before they appear.
     Pods start at different times (scheduler takes time) → natural stagger.
```

---

## Summary: Key Design Decisions

| Decision | Choice | Why |
|---|---|---|
| Primary discovery pattern | Service mesh (Envoy sidecar + xDS push) | AZ-aware, cell-aware, version-aware, sub-second updates, app-agnostic |
| Fallback discovery | DNS (Cloud Map / Route 53) | Universal, works for non-mesh and cross-region services |
| Registry consistency | Raft (CP) for writes, eventually consistent read replicas | Correct registrations (no split-brain); fast reads |
| Health checking | 4-level hierarchy (liveness → readiness → deep → synthetic) | Each level catches different failure modes |
| AZ preference | Envoy zone-aware routing (100% same-AZ when healthy) | Saves latency + $43M/year in cross-AZ data transfer at Amazon scale |
| Cross-region | DNS-based (per-region registry, no cross-region replication) | Regions are independent; DNS handles cross-region resolution |
| Cell affinity | Envoy metadata-match on cell label | Transparent to app code; enforced at mesh layer |
| Stale endpoint protection | Envoy outlier detection + retry + mTLS identity | Handles sudden crash: eject after 3 failures, retry transparently, prevent cross-service IP reuse confusion |
| Graceful shutdown | preStop hook (sleep 5s) + readiness fail + drain | Zero dropped requests during deployments |
| Control plane scaling | Push-based (xDS stream), batched + incremental deltas | 1M sidecars with < 100 MB/s push bandwidth |
| Registry failure | Envoy keeps last-known config indefinitely | Discovery works even if registry is down for hours |
| Flapping prevention | Hysteresis (3 consecutive failures to mark unhealthy) + control plane dedup | Prevents churn storms from borderline pods |