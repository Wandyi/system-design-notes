⏺ E-Commerce Platform at Amazon Scale — Comprehensive High-Level Design
                                                                                                                                                                                      
---                                                                                                                                                                                 
1. Scale Definition

Before designing anything, pin down the numbers. Every architectural choice flows from these.

┌──────────────────────────────────────────────────────────┐                                                                                                                        
│  STEADY STATE                                             │                                                                                                                       
│                                                           │                                                                                                                       
│  Registered users:         400 M                          │                                                                                                                       
│  Daily active users:       50 M                           │                                                                                                                       
│  Product catalog:          500 M SKUs (marketplace)       │                                                                                                                       
│  Active sellers:           3 M                            │                                                                                                                       
│  Daily page views:         2 B                            │
│  Orders per day:           20 M (~230/sec avg)            │                                                                                                                       
│  Peak orders per second:   ~5,000 (flash sales)           │                                                                                                                       
│  Search queries per sec:   100,000                        │                                                                                                                       
│  Cart operations per sec:  50,000                         │                                                                                                                       
│  Avg page latency target:  < 200ms (P50), < 500ms (P99)  │                                                                                                                        
│                                                           │                                                                                                                       
│  PEAK (Prime Day / Black Friday)                          │                                                                                                                       
│                                                           │                                                                                                                       
│  All steady-state numbers × 10-20x                        │                                                                                                                       
│  Orders per second:        50,000-100,000                 │                                                                                                                       
│  Page views per second:    500,000+                       │                                                                                                                       
│  Catalog reads per second: 5,000,000+                     │                                                                                                                       
│                                                           │                                                                                                                       
│  DATA VOLUMES                                             │                                                                                                                       
│                                                           │                                                                                                                       
│  Product images:           50 PB (multiple resolutions)   │
│  Order history:            100+ PB                        │                                                                                                                       
│  Event bus throughput:     10 TB/hr                       │
│  Log volume:               10 TB/hr                       │                                                                                                                       
│  ML training data:         500 TB                         │                                                                                                                       
│                                                           │                                                                                                                       
│  INFRASTRUCTURE                                           │                                                                                                                       
│                                                           │
│  Regions:                  6+ (US-E, US-W, EU, AP, etc.)  │
│  Availability zones / region: 3+                          │                                                                                                                       
│  Microservices:            2000+                          │                                                                                                                       
│  Deployments per day:      10,000+                        │                                                                                                                       
│  Server instances:         100,000+                       │                                                                                                                       
└──────────────────────────────────────────────────────────┘
                  
---                                                                                                                                                                                 
2. Guiding Architectural Principles

These aren't platitudes. Each one directly eliminates a class of production incidents.

1. CELL-BASED ISOLATION                                                                                                                                                             
   The system is partitioned into independent cells. A failure                                                                                                                      
   in cell 7 does not propagate to cell 8. This is the single                                                                                                                       
   most important reliability decision.

2. ASYNC BY DEFAULT, SYNC BY EXCEPTION                                                                                                                                              
   Services communicate through events. Synchronous calls only
   exist within the critical request path (page render, checkout).                                                                                                                  
   This decouples failure domains.

3. EVERY PAGE MUST RENDER                                                                                                                                                           
   If recommendations are down, show the product page without
   recommendations. If reviews are down, show the page without                                                                                                                      
   reviews. Only price and buy-button are hard dependencies.
   Everything else degrades gracefully.

4. DATA BELONGS TO THE SERVICE                                                                                                                                                      
   No shared databases. Each service owns its data store.
   Other services access data through APIs or consume events.                                                                                                                       
   This eliminates the "shared database as integration layer"
   anti-pattern that kills scalability.

5. PRE-COMPUTE OVER QUERY                                                                                                                                                           
   Don't compute at read time what you can compute at write time.                                                                                                                   
   Product detail pages, search results, and recommendation lists                                                                                                                   
   are pre-built and served from cache. Read path is a lookup,                                                                                                                      
   not a computation.

6. DESIGN FOR 10x, RE-ARCHITECT AT 100x                                                                                                                                             
   Every component should handle 10x current load with config
   changes (add nodes, increase limits). At 100x, expect a                                                                                                                          
   redesign of that component.

  ---                                                                                                                                                                                 
3. Global Infrastructure Layer

                            ┌──────────────┐
                            │  Route 53    │                                                                                                                                        
                            │  (DNS)       │
                            │              │                                                                                                                                        
                            │  Latency +   │                                                                                                                                        
                            │  health-based│
                            │  routing     │                                                                                                                                        
                            └──────┬───────┘                                                                                                                                        
                                   │
            ┌──────────────────────┼──────────────────────┐                                                                                                                         
            │                      │                      │
            ▼                      ▼                      ▼
   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                                                                                                                   
   │  CDN Edge    │      │  CDN Edge    │      │  CDN Edge    │
   │  (200+ PoPs) │      │  (200+ PoPs) │      │  (200+ PoPs) │                                                                                                                   
   │              │      │              │      │              │                                                                                                                   
   │  Static:     │      │  Static:     │      │  Static:     │                                                                                                                   
   │  images, JS, │      │  images, JS, │      │  images, JS, │                                                                                                                   
   │  CSS         │      │  CSS         │      │  CSS         │                                                                                                                   
   │              │      │              │      │              │
   │  Dynamic:    │      │  Dynamic:    │      │  Dynamic:    │                                                                                                                   
   │  edge compute│      │  edge compute│      │  edge compute│
   │  (A/B test,  │      │  (A/B test,  │      │  (A/B test,  │                                                                                                                   
   │   geo-redir, │      │   geo-redir, │      │   geo-redir, │                                                                                                                   
   │   auth token │      │   auth token │      │   auth token │                                                                                                                   
   │   validation)│      │   validation)│      │   validation)│                                                                                                                   
   └──────┬───────┘      └──────┬───────┘      └──────┬───────┘                                                                                                                   
   │                      │                      │                                                                                                                         
   ▼                      ▼                      ▼
   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                                                                                                                   
   │  REGION:     │      │  REGION:     │      │  REGION:     │
   │  US-EAST     │      │  EU-WEST     │      │  AP-SOUTH    │                                                                                                                   
   │              │      │              │      │              │                                                                                                                   
   │  3 AZs       │      │  3 AZs       │      │  3 AZs       │                                                                                                                   
   │  Active      │      │  Active      │      │  Active      │                                                                                                                   
   │  traffic     │      │  traffic     │      │  traffic     │
   └──────────────┘      └──────────────┘      └──────────────┘

   Active-Active: All regions serve live traffic.                                                                                                                                 
   Not active-passive. Not failover. Always on.

DNS Strategy

Global:     shop.example.com → Route 53 latency-based routing
→ nearest healthy region

Per-region:  us-east.shop.example.com (for explicit pinning)

Health checks: Route 53 health checks hit /health on each region's                                                                                                                  
edge load balancer. Unhealthy region is drained over
60 seconds (DNS TTL).

Failover time: ~60-90 seconds (DNS propagation + connection drain)

CDN Layer

STATIC ASSETS (images, JS, CSS):
• 50 PB of product images across multiple resolutions
• Images served from CDN edge, origin is S3                                                                                                                                       
• Cache-Control: public, max-age=31536000 (immutable, versioned URLs)                                                                                                             
• Image resizing at the edge (Lambda@Edge / CloudFlare Workers)                                                                                                                   
→ single source image, resize per device on first request, cache result

DYNAMIC CONTENT:                                                                                                                                                                    
• Product detail pages: edge-cached for 30-60s with stale-while-revalidate                                                                                                        
• Personalized content (cart, recommendations): not cached at CDN                                                                                                                 
→ passes through to origin, but responses are assembled from                                                                                                                    
cached fragments (ESI or client-side composition)

CACHE HIT RATE TARGET: 95%+ for static, 60-70% for dynamic                                                                                                                          
At 2B page views/day, a 95% hit rate means only 100M requests/day                                                                                                                   
reach origin. That's 1,157/sec — manageable.
                                                                                                                                                                                      
---                                                                                                                                                                                 
4. Region-Level Architecture

Every region is a complete, self-sufficient deployment. If all other regions vanish, one region can serve global traffic (at degraded latency but full functionality).

┌─────────────────────────────────────────────────────────────────────────┐
│  REGION: US-EAST                                                         │                                                                                                        
│                                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐     │
│  │  EDGE TIER                                                         │   │                                                                                                        
│  │                                                                    │   │
│  │  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐            │   │                                                                                                          
│  │  │  NLB / GLB   │   │  WAF         │   │  DDoS        │            │   │
│  │  │  (L4 load    │   │  (SQL inj,   │   │  Protection  │            │   │                                                                                                          
│  │  │   balancer)  │   │   XSS, bot   │   │  (rate limit │            │   │                                                                                                          
│  │  │              │   │   detection) │   │   per IP)    │            │   │                                                                                                         
│  │  └──────┬───────┘   └──────┬───────┘   └──────┬───────┘            │   │                                                                                                          
│  │         └──────────────────┼──────────────────┘                    │   │                                                                                                        
│  │                            ▼                                       │   │                                                                                                       
│  │                   ┌──────────────────┐                             │   │                                                                                                       
│  │                   │   API GATEWAY    │                             │   │
│  │                   │                  │                             │   │                                                                                                       
│  │                   │  • Auth (JWT)    │                             │   │
│  │                   │  • Rate limiting │                             │   │                                                                                                       
│  │                   │  • Request ID    │                             │   │
│  │                   │  • Routing       │                             │   │                                                                                                       
│  │                   │  • TLS term      │                             │   │                                                                                                       
│  │                   └────────┬─────────┘                             │   │
│  └────────────────────────────┼──────────────────────────────────────┘   │                                                                                                        
│                               │                                          │
│  ┌────────────────────────────┼──────────────────────────────────────┐   │                                                                                                        
│  │  CELL TIER                 ▼                                      │   │
│  │                                                                   │   │                                                                                                       
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │   │
│  │  │   CELL 1     │  │   CELL 2     │  │   CELL N     │           │   │                                                                                                          
│  │  │              │  │              │  │              │           │   │
│  │  │ Serves       │  │ Serves       │  │ Serves       │           │   │                                                                                                          
│  │  │ customers    │  │ customers    │  │ customers    │           │   │                                                                                                          
│  │  │ A-F          │  │ G-M          │  │ ...          │           │   │                                                                                                          
│  │  │              │  │              │  │              │           │   │                                                                                                          
│  │  │ Contains:    │  │ Contains:    │  │ Contains:    │           │   │                                                                                                          
│  │  │ • All domain │  │ • All domain │  │ • All domain │           │   │                                                                                                          
│  │  │   services   │  │   services   │  │   services   │           │   │
│  │  │ • Own data   │  │ • Own data   │  │ • Own data   │           │   │                                                                                                          
│  │  │   stores     │  │   stores     │  │   stores     │           │   │
│  │  │ • Own caches │  │ • Own caches │  │ • Own caches │           │   │                                                                                                          
│  │  │ • Own queues │  │ • Own queues │  │ • Own queues │           │   │                                                                                                          
│  │  └──────────────┘  └──────────────┘  └──────────────┘           │   │                                                                                                          
│  │                                                                 │   │                                                                                                       
│  │  Cell routing: API Gateway hashes customer_id → cell            │   │
│  │  Cell count: 20-50 per region                                   │   │                                                                                                       
│  │  Cell size: ~2-5% of regional traffic each                      │   │                                                                                                        
│  │  Blast radius: cell failure affects ≤5% of customers                │                                                                                                        
│  └───────────────────────────────────────────────────────────────────┘   │                                                                                                       
│                                                                          │                                                                                                        
│  ┌───────────────────────────────────────────────────────────────────┐   │                                                                                                       
│  │  SHARED TIER (region-wide, not per-cell)                          │   │                                                                                                       
│  │                                                                   │   │                                                                                                       
│  │  • Product Catalog (read replicas — same data for all customers)  │   │
│  │  • Search Cluster (Elasticsearch / OpenSearch)                    │   │                                                                                                        
│  │  • Recommendation Engine (ML inference)                           │   │
│  │  • Image / Asset Storage (S3)                                     │   │                                                                                                        
│  │  • Event Bus (Kafka — regional cluster)                           │   │                                                                                                        
│  │  • Observability Stack (metrics, logs, traces)                    │   │                                                                                                        
│  └───────────────────────────────────────────────────────────────────┘   │                                                                                                       
│                                                                          │                                                                                                        
└──────────────────────────────────────────────────────────────────────────┘

Why Cell-Based Architecture

WITHOUT CELLS (monolithic service mesh):                                                                                                                                            
Every service talks to every other service.
A bad deploy to the order service takes down orders for ALL customers.                                                                                                            
A database hotspot affects ALL customers.                                                                                                                                         
Blast radius = 100% of traffic.

WITH CELLS:     
┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐                                                                                                                                            
│Cell 1│  │Cell 2│  │Cell 3│  │Cell 4│                                                                                                                                            
│  ✓   │  │  ✗   │  │  ✓   │  │  ✓   │
│      │  │ BAD  │  │      │  │      │                                                                                                                                            
│      │  │DEPLOY│  │      │  │      │
└──────┘  └──────┘  └──────┘  └──────┘

    Cell 2 has a bad deploy. Only customers routed to cell 2 are affected.                                                                                                            
    Blast radius = 1/N of traffic (e.g., 5% if N=20).
    Other cells are completely isolated — different processes, different                                                                                                              
    database instances, different queues.

Shuffle Sharding (Within Cells)

Basic sharding: Customer → hash → Cell N
Problem: Cell N fails → ALL customers in that shard are down.

Shuffle sharding: Customer → assigned to 2 cells (primary + backup)                                                                                                                 
Cell 2 fails → customer is automatically routed to their backup cell.                                                                                                             
Two customers share the SAME pair of cells only if they're both                                                                                                                   
assigned to the same 2 out of N cells. With 20 cells, choosing 2:                                                                                                                 
C(20,2) = 190 possible pairs. Probability of correlated failure = 1/190.

    ┌──────────────────────────────────────────────┐                                                                                                                                  
    │  Customer Alice:  primary=Cell 3, backup=Cell 7  │                                                                                                                              
    │  Customer Bob:    primary=Cell 3, backup=Cell 12 │                                                                                                                              
    │                                                   │
    │  Cell 3 fails:                                    │                                                                                                                             
    │    Alice → Cell 7  ✓                              │
    │    Bob   → Cell 12 ✓                              │                                                                                                                             
    │                                                   │
    │  Cell 3 AND Cell 7 fail simultaneously:           │                                                                                                                             
    │    Alice → ✗ (both cells down)                    │                                                                                                                             
    │    Bob   → Cell 12 ✓ (unaffected)                 │                                                                                                                             
    │                                                   │                                                                                                                             
    │  Blast radius of 2-cell failure: only customers   │                                                                                                                             
    │  whose specific pair is {3,7}. Tiny fraction.     │                                                                                                                             
    └──────────────────────────────────────────────┘                                                                                                                                  
                                                                                                                                                                                      
---                                                                                                                                                                                 
5. Domain Services — Deep Dive

Service Map (Within a Cell)

┌──────────────────────────────────────────────────────────────────────────┐
│                              CELL N                                       │                                                                                                       
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐ │                                                                                                        
│  │  CUSTOMER-FACING (synchronous, in the request path)                 │ │                                                                                                        
│  │                                                                      │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐│ │                                                                                                           
│  │  │ Product  │ │  Cart    │ │ Checkout │ │  User    │ │  Search  ││ │                                                                                                           
│  │  │ Detail   │ │ Service  │ │ Service  │ │ Profile  │ │ Gateway  ││ │                                                                                                           
│  │  │ Page     │ │          │ │          │ │ Service  │ │          ││ │                                                                                                           
│  │  │ Composer │ │          │ │          │ │          │ │          ││ │                                                                                                           
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘│ │                                                                                                           
│  └──────────────────────────────────────────────────────────────────────┘ │                                                                                                       
│                                                                           │                                                                                                       
│  ┌─────────────────────────────────────────────────────────────────────┐ │                                                                                                        
│  │  BUSINESS LOGIC (mix of sync and async)                             │ │                                                                                                        
│  │                                                                      │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐│ │                                                                                                           
│  │  │ Order    │ │ Payment  │ │Inventory │ │ Pricing  │ │ Promotion││ │                                                                                                           
│  │  │ Service  │ │ Service  │ │ Service  │ │ Service  │ │ Service  ││ │                                                                                                           
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘│ │                                                                                                           
│  │                                                                  │ │                                                                                                       
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐             │ │                                                                                                           
│  │  │ Seller   │ │ Review   │ │Shipping  │ │ Tax      │             │ │
│  │  │ Service  │ │ Service  │ │ Service  │ │ Service  │             │ │                                                                                                           
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘             │ │                                                                                                           
│  └──────────────────────────────────────────────────────────────────────┘ │                                                                                                       
│                                                                           │                                                                                                       
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  DATA LAYER (per-cell, isolated)                                    │ │                                                                                                        
│  │                                                                      │ │                                                                                                       
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐             │ │
│  │  │PostgreSQL│ │DynamoDB  │ │  Redis   │ │  Kafka   │             │ │                                                                                                           
│  │  │(orders,  │ │(cart,    │ │(cache,   │ │(cell-    │             │ │                                                                                                           
│  │  │ users)   │ │ sessions)│ │ locks)   │ │ local    │             │ │                                                                                                           
│  │  │          │ │          │ │          │ │ events)  │             │ │                                                                                                           
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘             │ │                                                                                                           
│  └──────────────────────────────────────────────────────────────────────┘ │                                                                                                       
│                                                                           │                                                                                                       
└──────────────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---
5a. Product Catalog Service

The most read-heavy service in the system. Reads outnumber writes by 10,000:1.

┌───────────────────────────────────────────────────────────────────┐
│  PRODUCT CATALOG                                                   │                                                                                                              
│                                                                    │
│  Write Path (sellers update products):                             │
│                                                                    │                                                                                                              
│   Seller Portal → Catalog Write API → Primary DB (PostgreSQL)      │
│                                         │                          │                                                                                                              
│                                         │ CDC (Change Data Capture)│
│                                         ▼                          │                                                                                                              
│                                    Kafka topic:                    │
│                                    "catalog.changes"               │                                                                                                              
│                                         │                          │
│                     ┌───────────────────┼──────────────┐          │                                                                                                               
│                     ▼                   ▼              ▼          │                                                                                                               
│              Search Index       Cache Invalidator   Read Model    │
│              (OpenSearch)       (Redis/Memcached)   Builder       │                                                                                                               
│                                                     (DynamoDB)    │
│                                                                    │                                                                                                              
│  Read Path (customers browse products):                            │
│                                                                    │                                                                                                              
│   Browser → CDN (edge cache, 30s TTL)                             │
│              │ miss                                                 │                                                                                                             
│              ▼                                                      │                                                                                                             
│         API Gateway → Product Detail Page Composer                 │                                                                                                              
│                        │                                            │                                                                                                             
│                        ├→ Catalog Read API → DynamoDB (read model) │
│                        ├→ Pricing Service → Redis (price cache)    │                                                                                                              
│                        ├→ Inventory Service → "In Stock" / "OOS"  │                                                                                                               
│                        ├→ Review Service → avg rating + count      │                                                                                                              
│                        └→ Recommendation Svc → "related products" │                                                                                                               
│                                                                    │                                                                                                              
│                        Compose into single response.               │
│                        Each sub-call has independent timeout.      │                                                                                                              
│                        Non-critical calls (reviews, recs) degrade. │                                                                                                              
│                                                                    │                                                                                                              
│  Data Model:                                                       │                                                                                                              
│                                                                    │                                                                                                              
│  Write store (PostgreSQL): normalized, relational                  │
│    products, product_variants, product_attributes,                 │                                                                                                              
│    product_images, categories, seller_products                     │                                                                                                              
│                                                                    │                                                                                                              
│  Read store (DynamoDB): denormalized, single-table                 │                                                                                                              
│    PK: PRODUCT#{product_id}                                        │                                                                                                              
│    Contains: all variants, all images, category path,              │                                                                                                              
│              seller info, pre-joined into a single document.       │                                                                                                              
│    Single read to render a product page.                           │                                                                                                              
│                                                                    │                                                                                                              
│  Why two stores:                                                   │                                                                                                              
│    Write store: optimized for sellers editing products             │
│      (normalized, constraints, transactions)                       │                                                                                                              
│    Read store: optimized for customers viewing products            │                                                                                                              
│      (denormalized, single-digit-ms reads, no joins)               │                                                                                                              
│                                                                    │                                                                                                              
│  Scale:                                                            │
│    500M products × ~2KB each = ~1 TB in DynamoDB                   │                                                                                                               
│    DynamoDB handles millions of reads/sec with on-demand capacity  │                                                                                                              
│    Hot products (top 0.1%) cached in Redis: 500K items × 2KB      │                                                                                                               
│    = 1 GB in cache. Cache hit rate: 80%+                          │                                                                                                               
└───────────────────────────────────────────────────────────────────┘

Graceful degradation for product page:

Component          Required?   Fallback if unavailable
────────────────   ─────────   ──────────────────────────────                                                                                                                       
Product details    YES         Return 404 if product not found                                                                                                                      
Price              YES         "Price unavailable — try again"                                                                                                                      
Buy button         YES         Disabled if price unavailable                                                                                                                        
Inventory status   SOFT YES    Default to "In Stock" (risk oversell                                                                                                                 
vs. lose sale — business decides)                                                                                                                    
Reviews            NO          Hide reviews section                                                                                                                                 
Recommendations    NO          Hide "customers also bought"                                                                                                                         
Seller info        NO          Hide seller badge                                                                                                                                    
Q&A                NO          Hide section
                                                                                                                                                                                      
---             
5b. Search Service

┌───────────────────────────────────────────────────────────────────┐
│  SEARCH & DISCOVERY                                                │                                                                                                              
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐    │                                                                                                               
│  │  Indexing Pipeline (offline / near-real-time)               │   │                                                                                                              
│  │                                                             │   │                                                                                                              
│  │  Kafka: "catalog.changes" ──▶ Index Builder ──▶ OpenSearch  │   │                                                                                                               
│  │                                    │                        │   │                                                                                                              
│  │                              ┌─────┴──────┐                 │   │
│  │                              │ Enrichment  │                │   │                                                                                                              
│  │                              │             │                │   │
│  │                              │ • Synonyms  │                │   │                                                                                                              
│  │                              │ • Spell fix │                │   │
│  │                              │ • Category  │                │   │                                                                                                              
│  │                              │   mapping   │                │   │
│  │                              │ • Popularity│                │   │                                                                                                              
│  │                              │   score     │                │   │
│  │                              │ • Seller    │                │   │                                                                                                              
│  │                              │   quality   │                │   │                                                                                                              
│  │                              └─────────────┘                │   │
│  │                                                             │   │                                                                                                              
│  │  Latency: catalog write → searchable = 5-30 seconds        │    │
│  └────────────────────────────────────────────────────────────┘    │                                                                                                               
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐    │                                                                                                               
│  │  Query Pipeline (real-time, in the request path)            │   │                                                                                                              
│  │                                                             │   │                                                                                                              
│  │  User query: "blue running shoes size 10"                   │   │                                                                                                              
│  │       │                                                     │   │                                                                                                              
│  │       ▼                                                     │   │
│  │  ┌──────────┐   ┌──────────┐   ┌──────────┐                │    │                                                                                                                
│  │  │  Query   │──▶│  Query   │──▶│OpenSearch │               │    │                                                                                                               
│  │  │  Parser  │   │  Rewriter│   │  Cluster  │               │    │                                                                                                               
│  │  │          │   │          │   │           │               │    │                                                                                                               
│  │  │ tokenize │   │ spell    │   │ 100+     │               │     │                                                                                                                
│  │  │ intent   │   │ correct  │   │ data     │               │     │                                                                                                                
│  │  │ detect   │   │ expand   │   │ nodes    │               │   │
│  │  │ (filter  │   │ synonyms │   │          │               │   │                                                                                                               
│  │  │  vs.     │   │ personlz │   │ 500M docs│               │   │                                                                                                                
│  │  │  freetext│   │          │   │ ~2 TB    │               │   │                                                                                                                
│  │  │  )       │   │          │   │ sharded  │               │   │                                                                                                                
│  │  └──────────┘   └──────────┘   └─────┬────┘               │   │                                                                                                                
│  │                                       │                    │   │                                                                                                              
│  │                                       ▼                    │   │                                                                                                              
│  │                                ┌──────────┐                │   │                                                                                                               
│  │                                │ Re-Ranker│                │   │
│  │                                │ (ML)     │                │   │                                                                                                               
│  │                                │          │                │   │
│  │                                │ Re-rank  │                │   │                                                                                                               
│  │                                │ by:      │                │   │
│  │                                │ • CTR    │                │   │                                                                                                               
│  │                                │ • conversion              │   │                                                                                                                
│  │                                │ • relevance              │   │
│  │                                │ • personalization        │   │                                                                                                                
│  │                                │ • sponsored rank         │   │                                                                                                                
│  │                                └─────┬────┘                │   │
│  │                                      │                     │   │                                                                                                               
│  │                                      ▼                     │   │
│  │                               Response (top 48 results)    │   │                                                                                                               
│  │                               Latency budget: < 100ms      │   │                                                                                                               
│  └────────────────────────────────────────────────────────────┘   │                                                                                                               
│                                                                    │                                                                                                              
│  Scaling:                                                          │                                                                                                              
│    • 100K queries/sec at steady state                              │                                                                                                              
│    • OpenSearch cluster: 100+ data nodes, 3 master nodes           │
│    • Index sharded by category (electronics, clothing, etc.)       │                                                                                                              
│      → queries filtered by category only hit relevant shards       │                                                                                                              
│    • Hot indices (last 30 days of catalog changes) on SSD          │                                                                                                             
│    • Cold indices (historical) on HDD                              │                                                                                                             
│    • Query cache: top 10K queries cached for 60s → 40% cache hit   │                                                                                                              
└───────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---                                                                                                                                                                                 
5c. Cart Service

┌───────────────────────────────────────────────────────────────────┐
│  CART SERVICE                                                      │
│                                                                    │
│  Requirements:                                                     │
│    • 50K writes/sec at peak                                        │                                                                                                              
│    • Cart must survive node failures (not lost on crash)           │                                                                                                              
│    • Sub-10ms latency (it's in the critical path of every page     │                                                                                                               
│      that shows the cart icon count)                               │                                                                                                              
│    • Merge anonymous → authenticated cart on login                 │
│                                                                    │                                                                                                              
│  Data Store: DynamoDB (or Redis with AOF persistence)              │
│                                                                    │                                                                                                              
│  Schema:                                                           │
│    PK: CART#{customer_id or session_id}                            │                                                                                                              
│    Attributes:                                                     │
│      items: [                                                      │                                                                                                              
│        { sku, quantity, added_at, snapshot_price, price_version }  │                                                                                                              
│      ]                                                             │                                                                                                              
│      cart_version: 47       (optimistic concurrency control)       │                                                                                                              
│      updated_at: timestamp                                         │                                                                                                              
│      ttl: 30 days           (auto-expire abandoned carts)          │
│                                                                    │                                                                                                              
│  Operations:                                                       │
│    add_item(cart_id, sku, qty):                                    │                                                                                                              
│      Conditional write: version = expected_version                 │
│      On conflict: read-merge-write retry (3 attempts)              │                                                                                                              
│                                                                    │                                                                                                              
│    get_cart(cart_id):                                              │                                                                                                             
│      Single read from DynamoDB (< 5ms)                             │                                                                                                              
│      Enrich with live prices (batch call to pricing service)       │
│                                                                    │                                                                                                              
│    merge_carts(anonymous_cart_id, user_cart_id):                   │
│      Read both. Union items. On SKU conflict: sum quantities.      │                                                                                                              
│      Delete anonymous cart. Write merged to user cart.             │                                                                                                              
│      All in a DynamoDB TransactWriteItems (atomic).                │                                                                                                              
│                                                                    │                                                                                                              
│  Why DynamoDB:                                                     │                                                                                                              
│    • Single-digit ms reads and writes at any scale                 │                                                                                                              
│    • Auto-scaling. No connection pool management.                  │                                                                                                              
│    • TTL for automatic cleanup of abandoned carts                  │                                                                                                              
│    • Conditional writes for OCC without external locks             │                                                                                                              
│    • Per-cell DynamoDB tables (cell isolation)                     │                                                                                                              
│                                                                    │                                                                                                              
│  Why NOT Redis:                                                    │                                                                                                              
│    • Redis is faster (sub-ms) but durability requires AOF + repl   │                                                                                                              
│    • A Redis node failure + replication lag = lost carts           │                                                                                                             
│    • DynamoDB is durable by default (3-AZ replication)             │                                                                                                              
│    • Use Redis as a CACHE in front of DynamoDB for hot carts       │                                                                                                              
│                                                                    │                                                                                                              
│  Cache layer:                                                      │                                                                                                              
│    Redis (cell-local) caches recently active carts.                │                                                                                                              
│    Write-through: write to DynamoDB + Redis simultaneously.        │
│    TTL: 1 hour. Cache miss → read from DynamoDB.                   │                                                                                                              
│    Hot carts (users actively shopping): always in cache.           │                                                                                                              
│    Cold carts (abandoned 2 weeks ago): only in DynamoDB.           │                                                                                                              
└───────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---                                                                                                                                                                                 
5d. Checkout & Order Service

The most complex transactional flow. Multiple services must coordinate.

┌───────────────────────────────────────────────────────────────────────┐
│  CHECKOUT FLOW (SAGA ORCHESTRATION)                                    │                                                                                                          
│                                                                        │                                                                                                          
│     Customer clicks "Place Order"                                     │    
│               │                                                                │                                                                                                          
│               ▼                                                               │                                                                                                          
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  CHECKOUT ORCHESTRATOR (saga coordinator)                        │ │                                                                                                          
│  │                                                                  │ │                                                                                                         
│  │  Step 1: VALIDATE                                    Timeout: 2s │ │
│  │  ┌───────────────────────────────────────────────────────────┐   │ │                                                                                                           
│  │  │ In parallel:                                              │   │ │                                                                                                          
│  │  │  • Pricing: resolve final prices (lock)                   │   │ │                                                                                                           
│  │  │  • Inventory: check availability                          │   │ │                                                                                                           
│  │  │  • Address: validate shipping address                     │   │ │
│  │  │  • Tax: calculate tax for (items × destination)           │   │ │                                                                                                           
│  │  │  • Fraud: preliminary fraud score                         │   │ │                                                                                                           
│  │  │                                                           │   │ │                                                                                                          
│  │  │  ALL must succeed. Any failure → abort checkout.          │   │ │                                                                                                           
│  │  └───────────────────────────────────────────────────────────┘   │ │                                                                                                           
│  │       │                                                          │ │                                                                                                          
│  │       ▼ all passed                                               │ │                                                                                                          
│  │                                                                  │ │                                                                                                         
│  │  Step 2: RESERVE                                    Timeout: 3s  │ │
│  │  ┌───────────────────────────────────────────────────────────┐   │ │                                                                                                           
│  │  │ Sequential (order matters for compensation):              │   │ │                                                                                                          
│  │  │  1. Payment: authorize (hold funds, don't capture)        │   │ │                                                                                                           
│  │  │  2. Inventory: reserve units (decrement available count)  │   │ │                                                                                                           
│  │  │                                                           │   │ │                                                                                                          
│  │  │  If payment auth fails → abort, no compensation needed.   │   │ │                                                                                                           
│  │  │  If inventory fails → release payment auth, then abort.   │   │ │                                                                                                           
│  │  └───────────────────────────────────────────────────────────┘   │ │                                                                                                           
│  │       │                                                          │ │                                                                                                          
│  │       ▼ all reserved                                             │ │
│  │                                                                  │ │                                                                                                         
│  │  Step 3: COMMIT                                      Timeout: 2s │ │
│  │  ┌───────────────────────────────────────────────────────────┐   │ │                                                                                                           
│  │  │ 1. Create order record (status: CONFIRMED)                │   │ │
│  │  │ 2. Clear cart                                             │   │ │                                                                                                          
│  │  │ 3. Publish event: "order.confirmed"                       │   │ │                                                                                                           
│  │  │                                                           │   │ │                                                                                                          
│  │  │ The order record is the point of no return.               │   │ │                                                                                                           
│  │  │ After this, we MUST fulfill or explicitly cancel.         │   │ │
│  │  └───────────────────────────────────────────────────────────┘   │ │                                                                                                           
│  │       │                                                          │ │                                                                                                          
│  │       ▼                                                          │ │                                                                                                          
│  │                                                                  │ │                                                                                                         
│  │  Step 4: POST-COMMIT (async, event-driven)                       │ │
│  │  ┌───────────────────────────────────────────────────────────┐   │ │                                                                                                           
│  │  │  Triggered by "order.confirmed" event:                    │   │ │                                                                                                          
│  │  │                                                           │   │ │                                                                                                          
│  │  │  • Payment: capture (move from auth → capture)            │   │ │                                                                                                           
│  │  │  • Fulfillment: create shipment, assign warehouse         │   │ │                                                                                                           
│  │  │  • Notification: send confirmation email                  │   │ │                                                                                                           
│  │  │  • Analytics: record conversion                           │   │ │                                                                                                           
│  │  │  • Loyalty: award points                                  │   │ │                                                                                                           
│  │  │  • Seller: notify seller of sale                          │   │ │                                                                                                           
│  │  │                                                           │   │ │                                                                                                          
│  │  │  These are INDEPENDENT. If email fails, the order         │   │ │                                                                                                           
│  │  │  is still valid. Each has its own retry/DLQ policy.       │   │ │                                                                                                           
│  │  └───────────────────────────────────────────────────────────┘   │ │                                                                                                           
│  └──────────────────────────────────────────────────────────────────┘ │                                                                                                           
│                                                                       │                                                                                                          
│  COMPENSATION (rollback on failure):                                  │
│                                                                       │                                                                                                          
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  If Step 2.2 (inventory) fails after Step 2.1 (payment auth):    │ │                                                                                                            
│  │    → Release payment authorization                               │ │                                                                                                           
│  │    → Return error: "Item went out of stock during checkout"      │ │                                                                                                            
│  │                                                                  │ │                                                                                                          
│  │  If Step 3 (commit) fails after Step 2 (reserves):               │ │                                                                                                            
│  │    → Release inventory reservation                               │ │                                                                                                           
│  │    → Release payment authorization                               │ │
│  │    → Return error: "Order could not be created, please retry"    │ │                                                                                                            
│  │                                                                  │ │                                                                                                          
│  │  If compensation itself fails:                                   │ │
│  │    → Write to "stuck_orders" table                               │ │                                                                                                           
│  │    → Alert on-call                                               │ │                                                                                                          
│  │    → Reconciliation job picks it up within 15 minutes            │ │                                                                                                           
│  └──────────────────────────────────────────────────────────────────┘ │                                                                                                           
│                                                                        │                                                                                                          
│  IDEMPOTENCY:                                                          │                                                                                                          
│                                                                        │                                                                                                          
│  Every checkout attempt has a unique checkout_token (generated         │
│  client-side when the checkout page loads). The orchestrator           │                                                                                                         
│  deduplicates by token. Double-clicks, retries, and network            │                                                                                                          
│  replays are all safe.                                                 │                                                                                                         
│                                                                        │                                                                                                          
│  ┌──────────────────────────────────────────────────────────────────┐  │                                                                                                           
│  │  checkout_attempts table:                                         │ │                                                                                                          
│  │  PK: checkout_token                                               │ │
│  │  status: pending | completed | failed                             │ │                                                                                                          
│  │  order_id: (set on completion)                                    │ │
│  │  created_at: timestamp                                            │ │                                                                                                          
│  │                                                                   │ │
│  │  On duplicate token:                                              │ │                                                                                                          
│  │    if status=completed → return existing order_id                 │ │
│  │    if status=pending → return "checkout in progress"              │ │                                                                                                          
│  │    if status=failed → allow retry (new attempt, same token)      │ │                                                                                                           
│  └──────────────────────────────────────────────────────────────────┘ │                                                                                                           
└───────────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---                                                                                                                                                                                 
5e. Payment Service

┌───────────────────────────────────────────────────────────────────┐
│  PAYMENT SERVICE                                                   │
│                                                                    │
│  Principles:                                                       │
│    1. NEVER store raw card numbers (PCI DSS)                       │                                                                                                              
│    2. Auth then capture (never direct charge)                      │                                                                                                              
│    3. **Every external call has an idempotency key**               │                                                                                                              
│    4. Reconcile daily against payment provider                     │                                                                                                              
│                                                                    │                                                                                                              
│  ┌──────────────────────────────────────────────────────────────┐ │                                                                                                               
│  │  Architecture:                                                │ │                                                                                                              
│  │                                                               │ │                                                                                                             
│  │  Checkout ──▶ Payment Service ──▶ Payment Gateway Adapter     │ │
│  │  Orchestrator       │               │                         │ │                                                                                                              
│  │                     │          ┌────┴────┐                    │ │                                                                                                              
│  │                     │          │         │                    │ │                                                                                                              
│  │                     │       ┌──▼──┐  ┌──▼──┐  ┌──────┐        │ │                                                                                                                 
│  │                     │       │Stripe│  │Braint│  │PayPal│     │ │                                                                                                               
│  │                     │       │      │  │ree   │  │      │     │ │
│  │                     │       └──────┘  └──────┘  └──────┘     │ │                                                                                                               
│  │                     │                                         │ │                                                                                                              
│  │                     ▼                                         │ │                                                                                                              
│  │              Payment Ledger                                   │ │                                                                                                              
│  │              (PostgreSQL)                                     │ │                                                                                                              
│  │                                                               │ │
│  │  Ledger records every state transition:                      │ │                                                                                                              
│  │    payment_id | order_id | state       | amount | provider   │ │                                                                                                               
│  │    ───────────────────────────────────────────────────────── │ │                                                                                                              
│  │    pay_001    | ord_123  | AUTH_PENDING | $50.00 | stripe    │ │                                                                                                               
│  │    pay_001    | ord_123  | AUTHORIZED   | $50.00 | stripe    │ │                                                                                                               
│  │    pay_001    | ord_123  | CAPTURE_PEND | $50.00 | stripe    │ │                                                                                                               
│  │    pay_001    | ord_123  | CAPTURED     | $50.00 | stripe    │ │                                                                                                               
│  │                                                              │ │                                                                                                             
│  │  Append-only. Never update. Full audit trail.                │ │
│  └──────────────────────────────────────────────────────────────┘ │                                                                                                               
│                                                                    │
│  Multi-provider failover:                                          │                                                                                                              
│    Primary: Stripe (90% of transactions)                          │                                                                                                               
│    Fallback: Braintree (if Stripe returns 5xx)                    │
│                                                                    │                                                                                                              
│    if stripe.authorize(amount, idempotency_key) → 5xx:            │
│      retry once with stripe                                        │                                                                                                              
│      if still failing → route to braintree                        │
│      (different idempotency key to avoid cross-provider dedup)    │                                                                                                               
│                                                                    │                                                                                                              
│  Daily reconciliation job:                                         │                                                                                                              
│    Download settlement file from each provider.                   │                                                                                                               
│    Compare with internal ledger.                                   │                                                                                                              
│    Flag discrepancies:                                             │
│      • Auth without capture (should have captured or voided)      │                                                                                                               
│      • Capture without order (orphaned charge)                    │                                                                                                               
│      • Amount mismatch (rounding, currency conversion)            │
│    Route flagged items to finance team.                           │                                                                                                              
└───────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---                                                                                                                                                                                 
5f. Inventory Service

┌───────────────────────────────────────────────────────────────────┐
│  INVENTORY SERVICE                                                 │                                                                                                              
│                                                                    │
│  The inventory service must handle the flash-sale thundering       │                                                                                                              
│  herd: 100K concurrent requests for the same SKU.                  │                                                                                                               
│                                                                    │                                                                                                              
│  ┌──────────────────────────────────────────────────────────────┐ │                                                                                                               
│  │  Architecture: SHARDED COUNTERS                               │ │                                                                                                              
│  │                                                                │ │                                                                                                             
│  │  Problem: single row UPDATE SET qty = qty - 1 WHERE qty > 0  │ │
│  │           → max ~1,000 TPS due to row lock contention         │ │                                                                                                              
│  │                                                                │ │                                                                                                             
│  │  Solution: **split inventory into N shards**                      │ │                                                                                                              
│  │                                                                │ │                                                                                                             
│  │  SKU-123 total inventory: 10,000 units                        │ │
│  │                                                                │ │                                                                                                             
│  │  ┌────────────┐ ┌────────────┐     ┌────────────┐           │ │
│  │  │ Shard 0    │ │ Shard 1    │ ... │ Shard 99   │           │ │                                                                                                                
│  │  │ qty: 100   │ │ qty: 100   │     │ qty: 100   │           │ │                                                                                                                
│  │  └────────────┘ └────────────┘     └────────────┘           │ │                                                                                                                
│  │                                                             │ │                                                                                                             
│  │  Decrement: hash(request_id) % 100 → shard N                 │ │                                                                                                               
│  │             UPDATE shard_N SET qty = qty - 1 WHERE qty > 0    │ │                                                                                                              
│  │             If affected_rows = 0 → try next shard             │ │                                                                                                              
│  │             If all shards exhausted → OUT OF STOCK            │ │                                                                                                              
│  │                                                                │ │                                                                                                             
│  │  Throughput: 100 shards × 1,000 TPS each = 100K TPS          │ │                                                                                                               
│  │                                                                │ │                                                                                                             
│  │  Rebalancing: when shards become uneven (shard 3 has 0,       │ │
│  │  shard 7 has 50), a **background job rebalance**s.                │ │                                                                                                              
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                    │                                                                                                              
│  Multi-warehouse:                                                  │
│                                                                    │                                                                                                              
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │  inventory_by_location:                                       │ │                                                                                                              
│  │                                                                │ │
│  │  sku    │ warehouse │ available │ reserved │ version          │ │                                                                                                              
│  │  ──────────────────────────────────────────────────          │ │                                                                                                               
│  │  SKU-123│ WH-NJ    │ 300       │ 45       │ 1247            │ │                                                                                                                
│  │  SKU-123│ WH-CA    │ 150       │ 22       │ 893             │ │                                                                                                                
│  │  SKU-123│ WH-TX    │ 0         │ 0        │ 445             │ │                                                                                                                
│  │                                                             │ │                                                                                                             
│  │  "In Stock" shown on product page = SUM(available) > 0       │ │                                                                                                               
│  │  Pre-computed and cached. Updated via event on inventory      │ │                                                                                                              
│  │  change. Never computed at read time.                         │ │                                                                                                              
│  │                                                                │ │                                                                                                             
│  │  Warehouse allocation (which warehouse ships?):               │ │                                                                                                              
│  │  Decided at fulfillment time, NOT at checkout time.           │ │                                                                                                              
│  │  Checkout just reserves N units globally.                     │ │                                                                                                              
│  │  Fulfillment optimizer allocates to warehouse based on:       │ │                                                                                                              
│  │    • Proximity to customer (minimize shipping time/cost)      │ │                                                                                                              
│  │    • Current warehouse load (don't overload one warehouse)    │ │                                                                                                              
│  │    • Inventory balance (ship from warehouse with most stock)  │ │                                                                                                              
│  └──────────────────────────────────────────────────────────────┘ │                                                                                                               
└───────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---             
6. Event Backbone

The nervous system of the entire platform. Services communicate primarily through events.

┌──────────────────────────────────────────────────────────────────────┐
│  EVENT BACKBONE (Kafka)                                               │                                                                                                           
│                                                                       │
│  Regional Kafka cluster: 30-50 brokers per region                    │                                                                                                            
│                                                                       │                                                                                                           
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  CORE EVENT TOPICS                                              │  │                                                                                                           
│  │                                                                  │  │                                                                                                          
│  │  catalog.product.changed      (catalog → search, cache, recs)  │  │
│  │  pricing.price.changed        (pricing → cart, catalog cache)  │  │                                                                                                            
│  │  inventory.stock.changed      (inventory → catalog, alerts)    │  │                                                                                                            
│  │  order.confirmed              (checkout → everything below)    │  │                                                                                                            
│  │  order.cancelled              (orders → payment, inventory)    │  │                                                                                                            
│  │  order.shipped                (fulfillment → notification)     │  │
│  │  payment.captured             (payment → orders, accounting)   │  │                                                                                                            
│  │  payment.refunded             (payment → orders, notification) │  │                                                                                                            
│  │  user.registered              (auth → marketing, analytics)    │  │                                                                                                            
│  │  user.address.changed         (profile → orders, fulfillment) │  │                                                                                                             
│  │  review.submitted             (reviews → catalog, moderation) │  │
│  │  seller.product.listed        (seller → catalog, moderation)  │  │                                                                                                             
│  └────────────────────────────────────────────────────────────────┘  │                                                                                                            
│                                                                       │                                                                                                           
│  Event envelope:                                                      │                                                                                                           
│  {                                                                    │                                                                                                           
│    "event_id":     "evt_a1b2c3",         // globally unique          │
│    "event_type":   "order.confirmed",    // topic + type             │                                                                                                            
│    "aggregate_id": "ord_12345",          // entity this is about     │                                                                                                            
│    "version":      3,                    // monotonic per aggregate   │                                                                                                           
│    "timestamp":    "2026-04-20T...",     // event time               │                                                                                                            
│    "source":       "checkout-service",   // producing service        │                                                                                                            
│    "correlation_id": "req_xyz",          // trace across services    │                                                                                                            
│    "data": { ... }                       // event payload            │                                                                                                            
│  }                                                                    │                                                                                                           
│                                                                       │
│  Guarantees:                                                          │                                                                                                           
│    • Ordering: per partition (partition by aggregate_id)              │                                                                                                           
│    • Delivery: at-least-once (consumers must be idempotent)          │                                                                                                            
│    • Durability: replicated 3x, retained for 7 days                  │                                                                                                            
│                                                                       │                                                                                                           
│  Cross-region replication:                                            │
│    MirrorMaker 2 replicates critical topics across regions.          │                                                                                                            
│    Replication lag: 100-500ms (cross-region network latency).        │                                                                                                            
│    Used for: catalog sync, order visibility in all regions.          │                                                                                                            
│    NOT used for: real-time checkout (must be region-local).          │                                                                                                            
└──────────────────────────────────────────────────────────────────────┘

Event Flow Example: Order Lifecycle

order.confirmed ──┬──▶ Payment Service:     capture funds                                                                                                                           
├──▶ Inventory Service:    convert reservation → allocation                                                                                                       
├──▶ Fulfillment Service:  create shipment record                                                                                                                 
├──▶ Notification Service: send confirmation email                                                                                                                
├──▶ Analytics Service:    record conversion event                                                                                                                
├──▶ Loyalty Service:      award points                                                                                                                           
└──▶ Seller Service:       notify seller dashboard

Each consumer is an independent consumer group.
Each processes at its own pace.                                                                                                                                                     
Each has its own retry policy and DLQ.                                                                                                                                              
If Notification Service is down for 10 minutes:
→ email is delayed 10 minutes                                                                                                                                                     
→ order, payment, inventory are unaffected                                                                                                                                        
→ no cascading failure
                                                                                                                                                                                      
---             
7. Data Strategy

Per-Service Database Selection

┌───────────────────────────────────────────────────────────────────────┐                                                                                                           
│  SERVICE              DATABASE              WHY                       │
│  ──────────────────────────────────────────────────────────────────── │                                                                                                           
│  Product Catalog      PostgreSQL (write)    Relational integrity for  │
│    (write model)      + DynamoDB (read)     complex product data.     │                                                                                                           
│                                             DynamoDB for fast reads.  │                                                                                                           
│                                                                       │                                                                                                           
│  Cart                 DynamoDB              Key-value access pattern. │                                                                                                          
│                                             Single-digit ms latency.  │                                                                                                           
│                                             TTL for auto-cleanup.     │                                                                                                           
│                                                                       │                                                                                                           
│  Orders               PostgreSQL            Complex queries (joins,   │
│                       (sharded by           range scans, aggregates). │                                                                                                           
│                        customer_id)         ACID for order state      │
│                                             transitions.              │                                                                                                           
│                                                                       │                                                                                                           
│  Payments             PostgreSQL            Append-only ledger needs  │                                                                                                           
│                                             strict ACID. Auditable.   │                                                                                                           
│                                                                       │                                                                                                           
│  Inventory            PostgreSQL            UPDATE ... WHERE qty > 0  │
│                       (sharded by           needs transactions.       │                                                                                                           
│                        warehouse + sku)     Sharded counters.         │
│                                                                       │                                                                                                           
│  User Profiles        DynamoDB              Simple key-value.         │
│                                             High read volume.         │                                                                                                           
│                                             Low write volume.         │
│                                                                       │                                                                                                           
│  Sessions             Redis (cluster)       Sub-ms reads. Ephemeral.  │
│                                             TTL-based expiry.         │                                                                                                           
│                                                                       │                                                                                                           
│  Search               OpenSearch            Full-text search. Facets. │                                                                                                           
│                                             Relevance scoring.        │                                                                                                           
│                                                                       │
│  Reviews              DynamoDB (store)      Write-heavy (user         │                                                                                                           
│                       + OpenSearch (search)  reviews). Search for     │                                                                                                           
│                                             keyword queries.          │                                                                                                           
│                                                                       │                                                                                                           
│  Recommendations      Feature Store         Pre-computed per-user     │                                                                                                           
│                       (Redis + S3)          recs. ML model serving    │
│                                             needs fast lookups.       │                                                                                                           
│                                                                       │                                                                                                           
│  Analytics /          ClickHouse or         Columnar for analytical   │                                                                                                           
│  Reporting            Redshift              queries. Event-sourced    │                                                                                                           
│                                             from Kafka.               │
└───────────────────────────────────────────────────────────────────────┘

Caching Strategy (Three Tiers)

┌──────────────────────────────────────────────────────────────────┐                                                                                                                
│  TIER 1: CDN EDGE CACHE                                          │
│                                                                  │                                                                                                               
│  What:    Static assets, product page HTML fragments             │
│  TTL:     Images: 1 year (immutable URLs). HTML: 30-60s.        │                                                                                                                 
│  Hit rate: 95% (static), 60% (dynamic)                          │                                                                                                                 
│  Latency: 1-5ms (edge PoP to user)                              │                                                                                                                 
│  Invalidation: Purge API on catalog change.                     │                                                                                                                 
│                 Stale-while-revalidate for soft updates.         │                                                                                                                
│                                                                  │                                                                                                               
├──────────────────────────────────────────────────────────────────┤
│  TIER 2: APPLICATION CACHE (Redis, per-cell)                     │                                                                                                                
│                                                                  │                                                                                                               
│  What:    Product details, prices, stock status, session data    │
│  TTL:     Prices: 60s. Products: 300s. Sessions: 30min.          │                                                                                                                 
│  Hit rate: 80-90%                                                │                                                                                                                
│  Latency: < 1ms (same AZ)                                        │                                                                                                                 
│  Size:    50-100 GB per cell                                     │                                                                                                                
│  Invalidation: Event-driven. Price change event → delete         │                                                                                                                
│                 cached price for that SKU. Write-through         │                                                                                                               
│                 for session data.                                │                                                                                                               
│                                                                  │                                                                                                               
│  Topology:  Redis Cluster (6+ nodes per cell for HA)             │                                                                                                                 
│             Read replicas for read-heavy keys.                   │                                                                                                                
│                                                                  │                                                                                                               
├──────────────────────────────────────────────────────────────────┤                                                                                                                
│  TIER 3: LOCAL IN-PROCESS CACHE                                  │                                                                                                                
│                                                                  │                                                                                                               
│  What:    Config, feature flags, category tree, tax tables       │
│  TTL:     5-60 seconds                                           │                                                                                                                
│  Hit rate: 99%+ (small, stable datasets)                         │                                                                                                                
│  Latency: < 0.01ms (memory access)                               │                                                                                                                
│  Size:    10-100 MB per process                                  │                                                                                                                
│  Invalidation: Poll or event-driven refresh.                     │                                                                                                                
│                                                                  │
│  Risk:    Inconsistency window between processes.                │                                                                                                                
│           Process A has new config, Process B has old.           │
│           Acceptable for config/flags. NOT for prices/inventory. │                                                                                                                
└──────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---                                                                                                                                                                                 
8. Reliability Patterns

Pattern Map

┌──────────────────────────────────────────────────────────────────────┐
│  RELIABILITY PATTERN            WHERE APPLIED                         │                                                                                                           
│                                                                       │                                                                                                           
│  Circuit Breaker                Every synchronous service-to-service  │
│                                 call. Prevent cascading failure.      │                                                                                                           
│                                                                       │                                                                                                           
│  Bulkhead                       Thread pool isolation per dependency. │
│                                 Checkout's call to Payment Service    │                                                                                                           
│                                 uses a separate thread pool from its  │
│                                 call to Inventory Service.            │                                                                                                           
│                                                                       │
│  Retry + Jitter                 Idempotent operations only. Max 3     │                                                                                                           
│                                 retries. Exponential backoff + jitter │                                                                                                           
│                                 to prevent thundering herd.           │                                                                                                           
│                                                                       │                                                                                                           
│  Timeout + Deadline Propagation Every RPC has a timeout. Deadline     │                                                                                                           
│                                 = original_timeout - time_elapsed.    │                                                                                                           
│                                 Propagated in request headers.        │                                                                                                           
│                                                                       │                                                                                                           
│  Load Shedding                  API Gateway drops requests when queue │                                                                                                           
│                                 depth exceeds threshold. Return 503   │                                                                                                           
│                                 immediately rather than queueing.     │
│                                                                       │                                                                                                           
│  Graceful Degradation           Product page renders without reviews, │
│                                 recs, or Q&A if those services are    │                                                                                                           
│                                 down. Hard dependencies: product      │                                                                                                           
│                                 data + price only.                    │                                                                                                           
│                                                                       │                                                                                                           
│  Idempotency                    Every write operation that can be     │                                                                                                           
│                                 retried (payment, order creation,     │
│                                 inventory reservation).               │                                                                                                           
│                                                                       │
│  Rate Limiting                  Per-customer, per-IP, per-seller,     │                                                                                                           
│                                 per-API-key. Layered at edge (coarse) │                                                                                                           
│                                 and service (fine-grained).           │                                                                                                           
│                                                                       │                                                                                                           
│  Backpressure                   Kafka consumers. If processing slows, │                                                                                                           
│                                 consumer lag grows. Alert at lag      │                                                                                                           
│                                 thresholds. Auto-scale consumers.     │                                                                                                           
│                                                                       │                                                                                                           
│  Chaos Engineering              Regularly inject failures in prod:    │                                                                                                           
│                                 kill instances, partition networks,   │                                                                                                           
│                                 degrade dependencies. Verify the      │
│                                 system degrades gracefully.           │                                                                                                           
└──────────────────────────────────────────────────────────────────────┘

Circuit Breaker Detail

┌──────────────────────────────────────────────────────────────────┐
│  CIRCUIT BREAKER (per dependency, per cell)                       │                                                                                                               
│                                                                   │                                                                                                               
│  States:                                                          │                                                                                                               
│                                                                   │                                                                                                               
│  CLOSED (normal) ──── error rate > 50% over 10s ────▶ OPEN       │
│       ▲                                                │          │                                                                                                               
│       │                                                │          │
│       │                              wait 30 seconds   │          │                                                                                                               
│       │                                                ▼          │                                                                                                               
│       │                                           HALF-OPEN      │
│       │                                                │          │                                                                                                               
│       │         success ◀──── allow 5 probe requests ──┘          │
│       │            │                    │                          │                                                                                                              
│       │            │               all fail                       │                                                                                                               
│       │            │                    │                          │                                                                                                              
│       └────────────┘                    └───────▶ OPEN            │                                                                                                               
│                                                                   │                                                                                                               
│  When OPEN:                                                       │
│    Don't call the dependency at all.                              │                                                                                                               
│    Return fallback immediately:                                   │                                                                                                               
│      • Recommendation service down → show popular items           │                                                                                                               
│      • Review service down → hide reviews section                 │                                                                                                               
│      • Payment service down → "checkout temporarily unavailable"  │                                                                                                               
│                                                                   │                                                                                                               
│  Key: circuit breakers are PER-CELL.                              │                                                                                                               
│  Cell 3's circuit breaker to the Review Service can be OPEN       │                                                                                                               
│  while Cell 7's is CLOSED. They're independent.                   │                                                                                                               
└──────────────────────────────────────────────────────────────────┘

Timeout & Deadline Propagation

Customer request has a 3-second total budget.

API Gateway ──[deadline: 3000ms]──▶ Product Page Composer                                                                                                                           
│                                                                                                                                          
┌───────────────┼──────────────┐                                                                                                                           
│               │              │
┌─────▼─────┐  ┌─────▼─────┐ ┌─────▼─────┐
│  Catalog  │  │  Pricing  │ │  Reviews  │                                                                                                                       
│           │  │           │ │           │                                                                                                                       
│ deadline: │  │ deadline: │ │ deadline: │                                                                                                                       
│ 2800ms    │  │ 2800ms    │ │ 2800ms    │                                                                                                                       
│ (3000 -   │  │           │ │           │                                                                                                                       
│  200ms    │  │           │ │           │                                                                                                                       
│  overhead)│  │           │ │           │                                                                                                                       
└─────┬─────┘  └─────┬─────┘ └─────┬─────┘                                                                                                                       
│               │              │
▼               ▼              │                                                                                                                           
Catalog DB       Price Cache         │                                                                                                                           
deadline:        deadline:           │
2500ms           2500ms              │                                                                                                                           
│
Reviews DB                                                                                                                       
deadline: 2500ms

If Reviews takes 2.9 seconds:                                                                                                                                                       
The overall deadline (3000ms) expires.
Product Page Composer returns a response WITHOUT reviews.                                                                                                                         
The customer sees the page. Reviews are missing but everything                                                                                                                    
else is there. The page rendered in 2.9s — slow but not broken.

WITHOUT deadline propagation:                                                                                                                                                       
Reviews takes 2.9s. Catalog response arrives in 50ms, sits in memory.                                                                                                             
Total time: 2.9s (blocked on slowest).                                                                                                                                            
BUT: if Reviews had its own 3s timeout, and IT calls another service                                                                                                              
with a 3s timeout, the chain could take 6s+ total. The original                                                                                                                   
3s deadline would have killed it at 3s — but without propagation,                                                                                                                 
downstream services don't know time has run out.
                                                                                                                                                                                      
---                                                                                                                                                                                 
9. Multi-Region Active-Active

┌─────────────────────────────────────────────────────────────────────┐
│  DATA CLASSIFICATION FOR MULTI-REGION                                │                                                                                                            
│                                                                      │                                                                                                            
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  GLOBAL (replicated to all regions, eventual consistency)     │   │                                                                                                            
│  │                                                               │   │                                                                                                           
│  │  • Product catalog (same products everywhere)                 │   │                                                                                                            
│  │  • Seller data                                                │   │                                                                                                           
│  │  • User account core (email, password hash)                   │   │                                                                                                            
│  │  • Category taxonomy                                          │   │                                                                                                           
│  │                                                               │   │                                                                                                           
│  │  Replication: Kafka MirrorMaker 2 (100-500ms lag)             │   │                                                                                                            
│  │  Conflict resolution: **last-writer-wins on version vector**  │   │                                                                                                            
│  │  Primary: region where the seller/user was created            │   │                                                                                                            
│  │           (writes route to primary, reads from local replica) │   │                                                                                                           
│  └──────────────────────────────────────────────────────────────┘    │                                                                                                             
│                                                                      │                                                                                                            
│  ┌──────────────────────────────────────────────────────────────┐    │                                                                                                             
│  │  REGIONAL (stays in one region, no replication)               │   │                                                                                                            
│  │                                                               │   │
│  │  • Shopping carts (user shops in one region at a time)        │   │                                                                                                            
│  │  • Active checkout sessions                                   │   │                                                                                                           
│  │  • Payment authorizations (must be region-local for latency)  │   │                                                                                                            
│  │  • Inventory reservations                                     │   │                                                                                                           
│  │                                                               │   │                                                                                                           
│  │  If user moves regions (VPN, travel):                         │   │
│  │    Cart is re-fetched from origin region or                   │   │                                                                                                           
│  │    replicated on-demand when user first accesses.             │   │                                                                                                           
│  └──────────────────────────────────────────────────────────────┘   │                                                                                                             
│                                                                      │                                                                                                            
│  ┌──────────────────────────────────────────────────────────────┐   │                                                                                                             
│  │  REGION-SPECIFIC (different data per region)                  │   │                                                                                                            
│  │                                                               │   │                                                                                                           
│  │  • Pricing (different prices per country)                     │   │
│  │  • Tax rules (US tax ≠ EU VAT ≠ India GST)                    │   │                                                                                                             
│  │  • Shipping options (regional carriers)                       │   │                                                                                                            
│  │  • Legal compliance (GDPR in EU, PDPA in India)               │   │                                                                                                            
│  │  • Payment methods (UPI in India, iDEAL in NL, PIX in BR)     │   │                                                                                                             
│  │                                                               │   │                                                                                                           
│  │  This data is not replicated — it's authored per region.      │   │                                                                                                            
│  └──────────────────────────────────────────────────────────────┘    │                                                                                                             
│                                                                      │                                                                                                            
│  FAILOVER:                                                           │                                                                                                            
│                                                                      │                                                                                                            
│  Region US-EAST goes down.                                           │
│                                                                      │                                                                                                            
│  1. Route 53 health checks detect failure (30s)                      │
│  2. DNS stops routing to US-EAST (TTL: 60s)                          │                                                                                                             
│  3. US customers are routed to US-WEST                               │                                                                                                            
│  4. US-WEST has:                                                     │                                                                                                            
│     • Full catalog replica (always in sync via MirrorMaker)          │                                                                                                             
│     • User accounts (replicated)                                     │                                                                                                            
│     • Regional data (pricing, tax — already configured for US)       │                                                                                                             
│     • NO active carts/checkouts from US-EAST users                   │                                                                                                             
│       → Customers must re-add items to cart (acceptable tradeoff     │                                                                                                            
│         vs. the complexity of cross-region cart replication)         │                                                                                                           
│  5. US-EAST recovery:                                                │                                                                                                            
│     • Drain traffic from US-WEST back to US-EAST gradually           │                                                                                                             
│     • Replay events from Kafka (retained 7 days) to rebuild state    │                                                                                                             
│     • Run reconciliation to catch any divergence                     │                                                                                                            
│                                                                      │                                                                                                            
│  RTO (Recovery Time Objective): < 2 minutes (DNS failover)          │                                                                                                             
│  RPO (Recovery Point Objective): < 1 second (Kafka replication lag) │                                                                                                             
└─────────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---                                                                                                                                                                                 
10. Observability

┌──────────────────────────────────────────────────────────────────────┐
│  THREE PILLARS + BUSINESS METRICS                                     │
│                                                                       │                                                                                                           
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  METRICS (Prometheus / CloudWatch)                              │  │                                                                                                           
│  │                                                                  │  │                                                                                                          
│  │  Infrastructure:                                                 │  │
│  │    CPU, memory, disk, network per instance                      │  │                                                                                                           
│  │    Kafka consumer lag per consumer group                        │  │
│  │    Database connection pool utilization                         │  │                                                                                                           
│  │    Cache hit rate per cache tier                                 │  │
│  │                                                                  │  │                                                                                                          
│  │  Application (RED method per service):                          │  │
│  │    Rate:     requests per second                                │  │                                                                                                           
│  │    Errors:   error rate (4xx, 5xx)                              │  │                                                                                                           
│  │    Duration: latency (P50, P90, P99, P99.9)                    │  │                                                                                                            
│  │                                                                  │  │                                                                                                          
│  │  Business:                                                       │  │
│  │    Orders per minute (THE most important metric)                │  │                                                                                                           
│  │    Cart abandonment rate                                        │  │
│  │    Checkout conversion rate                                     │  │                                                                                                           
│  │    Revenue per minute                                           │  │
│  │    Payment decline rate                                         │  │                                                                                                           
│  │    Search null-result rate                                      │  │                                                                                                           
│  └────────────────────────────────────────────────────────────────┘  │                                                                                                            
│                                                                       │                                                                                                           
│  ┌────────────────────────────────────────────────────────────────┐  │                                                                                                            
│  │  DISTRIBUTED TRACING (Jaeger / X-Ray)                          │  │
│  │                                                                  │  │                                                                                                          
│  │  Every request gets a trace_id at the API Gateway.              │  │
│  │  Propagated via headers through all service calls.              │  │                                                                                                           
│  │  Sampling: 1% of all requests + 100% of errors.                │  │                                                                                                            
│  │                                                                  │  │                                                                                                          
│  │  Trace for a checkout:                                          │  │                                                                                                           
│  │  [API GW 3ms]                                                   │  │                                                                                                           
│  │    └─[Checkout Orchestrator 450ms]                              │  │                                                                                                           
│  │        ├─[Pricing Service 12ms]                                 │  │                                                                                                           
│  │        ├─[Inventory Service 8ms]                                │  │                                                                                                           
│  │        ├─[Tax Service 35ms] ← external API, always slow       │  │                                                                                                             
│  │        ├─[Fraud Service 120ms] ← ML inference                  │  │                                                                                                            
│  │        ├─[Payment Auth 180ms] ← Stripe API call                │  │                                                                                                            
│  │        ├─[Order DB Write 15ms]                                  │  │                                                                                                           
│  │        └─[Event Publish 5ms]                                    │  │                                                                                                           
│  └────────────────────────────────────────────────────────────────┘  │                                                                                                            
│                                                                       │                                                                                                           
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  LOGS (structured JSON → Kafka → OpenSearch)                    │  │                                                                                                           
│  │                                                                  │  │                                                                                                          
│  │  { "ts": "...", "level": "ERROR", "service": "payment",       │  │
│  │    "trace_id": "abc123", "msg": "stripe_timeout",              │  │                                                                                                            
│  │    "duration_ms": 5000, "order_id": "ord_789" }                │  │                                                                                                            
│  │                                                                  │  │                                                                                                          
│  │  Correlation: trace_id links logs across all services.          │  │                                                                                                           
│  │  A single checkout error → query by trace_id → see every       │  │                                                                                                            
│  │  log line from every service that participated.                 │  │                                                                                                           
│  └────────────────────────────────────────────────────────────────┘  │                                                                                                            
│                                                                       │                                                                                                           
│  ALERTING HIERARCHY:                                                  │                                                                                                           
│                                                                       │                                                                                                           
│  P0 (page immediately):                                               │                                                                                                           
│    • Orders/min drops > 20% (revenue impact)                          │                                                                                                            
│    • Payment error rate > 5%                                          │                                                                                                           
│    • Any cell completely unresponsive                                  │                                                                                                          
│    • Kafka consumer lag > 1 hour on critical topics                   │                                                                                                           
│                                                                       │                                                                                                           
│  P1 (page within 15min):                                              │                                                                                                           
│    • P99 latency > 2x baseline                                       │                                                                                                            
│    • Error rate > 1% on any service                                   │                                                                                                           
│    • Database replication lag > 30s                                    │                                                                                                          
│    • Cache hit rate drops > 20%                                       │                                                                                                           
│                                                                       │                                                                                                           
│  P2 (ticket, next business day):                                      │                                                                                                           
│    • Disk utilization > 70%                                           │                                                                                                           
│    • Certificate expiration within 30 days                            │
│    • DLQ accumulating messages                                        │                                                                                                           
│    • Search null-result rate increases > 10%                          │
└──────────────────────────────────────────────────────────────────────┘
                  
---                                                                                                                                                                                 
11. Deployment & Operations

┌──────────────────────────────────────────────────────────────────────┐
│  DEPLOYMENT STRATEGY: CELL-BASED PROGRESSIVE ROLLOUT                  │                                                                                                           
│                                                                       │                                                                                                           
│  Deploy to 1 canary cell (lowest-traffic cell)                       │
│       │                                                               │                                                                                                           
│       ▼ monitor 15 minutes                                           │
│  Automated checks:                                                    │                                                                                                           
│    ✓ Error rate ≤ baseline                                           │
│    ✓ Latency P99 ≤ baseline × 1.1                                   │                                                                                                             
│    ✓ Orders/min ≥ baseline × 0.95                                    │                                                                                                            
│    ✓ No new crash loops                                               │                                                                                                           
│       │                                                               │                                                                                                           
│       ▼ pass                                                          │                                                                                                           
│  Deploy to 10% of cells (2-3 cells)                                  │                                                                                                            
│       │                                                               │
│       ▼ monitor 15 minutes + automated checks                        │                                                                                                            
│       │                                                               │
│       ▼ pass                                                          │                                                                                                           
│  Deploy to 50% of cells                                              │
│       │                                                               │                                                                                                           
│       ▼ monitor 15 minutes                                           │
│       │                                                               │                                                                                                           
│       ▼ pass                                                          │
│  Deploy to 100% of cells in this region                              │                                                                                                            
│       │                                                               │
│       ▼ wait 1 hour                                                   │                                                                                                           
│       │                                                               │                                                                                                           
│  Deploy to next region (repeat same progressive rollout)             │
│                                                                       │                                                                                                           
│  ROLLBACK:                                                            │
│    Automated: if any check fails, automatically roll back             │                                                                                                           
│    the affected cells. No human intervention needed.                  │                                                                                                           
│    Time to rollback: < 2 minutes (restart with previous image).      │                                                                                                            
│                                                                       │                                                                                                           
│  Total time for global rollout: ~4 hours (6 regions × 40 min each)  │                                                                                                             
│  This is intentionally slow. Speed of deployment is less important   │                                                                                                            
│  than blast radius containment.                                       │                                                                                                           
│                                                                       │                                                                                                           
│  FEATURE FLAGS (LaunchDarkly / internal):                             │                                                                                                           
│    New features deploy as dormant code behind flags.                  │                                                                                                           
│    Code ships globally. Flag turns on per-cell, per-region,          │                                                                                                            
│    per-user-segment. If something breaks, kill the flag —             │                                                                                                           
│    no redeployment needed.                                            │                                                                                                           
│                                                                       │                                                                                                           
│  DATABASE MIGRATIONS:                                                 │                                                                                                           
│    Three-phase:                                                       │                                                                                                           
│    1. Deploy code that reads old + new schema                        │
│    2. Run migration (add column, backfill)                           │                                                                                                            
│    3. Deploy code that reads only new schema                         │                                                                                                            
│    **Never: ALTER TABLE ADD NOT NULL on a live, large table.**       │                                                                                                            
│    Always: backward-compatible schema changes only.                  │                                                                                                            
└──────────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---             
12. Security Architecture

┌──────────────────────────────────────────────────────────────────────┐
│  SECURITY LAYERS                                                     │
│                                                                      │                                                                                                           
│  EDGE:                                                               │
│    • WAF (SQL injection, XSS, path traversal filtering)              │                                                                                                            
│    • DDoS mitigation (rate limiting per IP, CAPTCHA for bots)        │                                                                                                            
│    • TLS 1.3 termination at CDN/load balancer                        │                                                                                                            
│    • Bot detection (device fingerprinting, behavioral analysis)      │                                                                                                            
│                                                                      │                                                                                                           
│  API GATEWAY:                                                        │
│    • JWT validation (RS256, short-lived: 15 min)                     │                                                                                                            
│    • OAuth 2.0 scopes per endpoint                                   │                                                                                                            
│    • Rate limiting per customer/API key                              │                                                                                                            
│    • Request size limits (prevent payload bombs)                     │                                                                                                            
│    • Input validation (schema enforcement)                           │                                                                                                            
│                                                                      │
│  SERVICE MESH:                                                       │                                                                                                           
│    • mTLS between all services (zero-trust network)                  │                                                                                                            
│    • Service-to-service authorization (RBAC)                         │
│      Payment Service can call Stripe. Catalog Service cannot.        │                                                                                                            
│    • Certificate rotation: automated, no downtime                    │                                                                                                            
│                                                                      │                                                                                                           
│  DATA:                                                               │                                                                                                           
│    • PII encryption at rest (AES-256)                                │                                                                                                            
│    • PII encryption in transit (TLS 1.3)                             │                                                                                                            
│    • Payment data: tokenized via payment gateway                     │
│      (we never see raw card numbers)                                 │                                                                                                            
│    • Column-level encryption for SSN, DOB (if stored)                │
│    • Audit log: who accessed what PII, when, why                     │                                                                                                            
│    • Data retention policies: auto-delete PII after N years          │                                                                                                            
│                                                                      │                                                                                                           
│  SECRETS:                                                            │                                                                                                           
│    • All secrets in Vault (HashiCorp) or AWS Secrets Manager         │                                                                                                            
│    • Dynamic secrets for database credentials                        │                                                                                                            
│      (rotated every 24 hours, never in config files)                 │
│    • No secrets in environment variables, Docker images, or git      │                                                                                                            
│                                                                      │                                                                                                           
│  FRAUD:                                                              │                                                                                                           
│    • ML model scores every transaction (feature vector:              │                                                                                                            
│      device fingerprint, IP geolocation, purchase history,           │                                                                                                            
│      shipping/billing address mismatch, order velocity)              │
│    • High-risk orders: hold for manual review                        │                                                                                                            
│    • Known-bad actors: block at edge (IP/device block list)          │                                                                                                            
│    • Address verification (AVS) + 3D Secure for cards                │                                                                                                             
└──────────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---                                                                                                                                                                                 
13. Complete Request Flow: Customer Buys a Product

Tying everything together with one end-to-end flow:

1. CUSTOMER OPENS PRODUCT PAGE
   ──────────────────────────────                                                                                                                                                   
   Browser → DNS (Route 53, latency-based) → nearest CDN edge PoP

   CDN: cache HIT for product page HTML fragment? → serve (1ms)                                                                                                                     
   CDN: cache MISS → forward to regional API Gateway

   API Gateway → authenticate (JWT, 2ms) → route to Cell N                                                                                                                          
   (hash customer_id → cell assignment)

   Cell N → Product Page Composer (parallel fan-out):                                                                                                                               
   ├─ Catalog Read API → Redis cache (HIT: 0.5ms) → product data                                                                                                                  
   ├─ Pricing Service → Redis cache (HIT: 0.5ms) → $29.99                                                                                                                         
   ├─ Inventory Service → Redis cache (HIT: 0.5ms) → "In Stock"                                                                                                                   
   ├─ Review Service → Redis cache (HIT: 0.5ms) → 4.5★ (2,341 reviews)                                                                                                            
   └─ Recommendation Svc → pre-computed list → "Customers also bought..."

   Compose response: 15ms total (all calls parallel, worst-case wins)                                                                                                               
   Return to CDN → cache for 30s → return to browser

   Total: ~50ms (user in same region) to ~200ms (cross-region)

2. CUSTOMER ADDS TO CART                                                                                                                                                            
   ──────────────────────
   POST /cart/items { sku: "ABC123", qty: 1 }

   API Gateway → Cell N → Cart Service
   Cart Service:                                                                                                                                                                    
   → Read cart from Redis (cache, 0.5ms)                                                                                                                                          
   → Append item with snapshot_price=$29.99, version=42
   → Write to DynamoDB (conditional, 5ms) + Redis (write-through, 0.5ms)                                                                                                          
   → Return updated cart

   Total: ~20ms

3. CUSTOMER STARTS CHECKOUT
   ────────────────────────
   POST /checkout/start { cart_id: "...", checkout_token: "tok_xyz" }

   Checkout Orchestrator (Cell N):

   Step 1 — Validate (parallel):                                                                                                                                                    
   ├─ Pricing: resolve current prices → lock at $29.99 (12ms)
   ├─ Inventory: available? YES (8ms)                                                                                                                                             
   ├─ Address: valid? YES (15ms via USPS API)
   ├─ Tax: $29.99 × 8.875% NJ tax = $2.66 (35ms via Avalara)                                                                                                                      
   └─ Fraud: score = 0.02 (low risk) (120ms via ML model)

   Step 2 — Reserve (sequential):                                                                                                                                                   
   → Payment: authorize $32.65 via Stripe (180ms)                                                                                                                                 
   → Inventory: reserve 1 unit of ABC123 (8ms)

   Return checkout session with locked prices.                                                                                                                                      
   Total: ~250ms

   Customer reviews order summary page. Prices are LOCKED.

4. CUSTOMER CONFIRMS ORDER                                                                                                                                                          
   ────────────────────────
   POST /checkout/confirm { checkout_token: "tok_xyz" }

   Checkout Orchestrator:                                                                                                                                                           
   → Verify checkout_token (idempotency check, 2ms)
   → Re-verify prices (if decreased, use lower; if increased, interstitial)                                                                                                       
   → Create order record in PostgreSQL (15ms)                                                                                                                                     
   INSERT INTO orders (id, customer_id, items, total, status)                                                                                                                   
   VALUES ('ord_999', 'cust_123', [...], 3265, 'CONFIRMED')                                                                                                                     
   → Publish "order.confirmed" event to Kafka (5ms)                                                                                                                               
   → Clear cart in DynamoDB (5ms)                                                                                                                                                 
   → Return order confirmation

   Total: ~50ms

5. POST-ORDER ASYNC PROCESSING                                                                                                                                                      
   ───────────────────────────
   "order.confirmed" event triggers (all independent, parallel):

   Payment Consumer:   capture $32.65 from Stripe (200ms, async)                                                                                                                  
   Fulfillment:        assign warehouse NJ, create shipment (50ms)                                                                                                                
   Notification:       send confirmation email via SES (100ms)                                                                                                                    
   Analytics:          record conversion in ClickHouse (10ms)                                                                                                                     
   Loyalty:            award 33 points (10ms)                                                                                                                                     
   Seller:             notify seller dashboard (20ms)

   Customer sees: "Order confirmed! You'll receive an email shortly."                                                                                                               
   Email arrives 2-5 seconds after order placement.

  ---             
Summary: Why Each Decision

┌────────────────────────────────────┬────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│              Decision              │                                                   Reason                                                   │                                 
├────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Cell-based architecture            │ **Blast radius containment**. Cell failure affects ≤5% of customers.                                       │
├────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Shuffle sharding                   │ Correlated failure affects even fewer. Two-cell failure ≈ 0.5%.                                            │                                 
├────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────┤                                 
│ Event-driven backbone (Kafka)      │ Decouples services. Notification failure doesn't block checkout.                                           │                                 
├────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────┤                                 
│ CQRS (separate read/write stores)  │ **10,000:1 read/write ratio. Optimize each path independently**.                                           │
├────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────┤                                 
│ DynamoDB for cart/sessions         │ Predictable **single-digit ms latency** at any scale. No connection pool drama.                            │
├────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────┤                                 
│ PostgreSQL for orders/payments     │ ACID transactions for money movement. Auditable. Relational queries for support tools.                     │
├────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────┤                                 
│ Three-tier caching                 │ CDN (95% of static), Redis (80% of dynamic), in-process (99% of config). Origin sees <1% of total traffic. │
├────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────┤                                 
│ Progressive cell-by-cell deploys   │ Bad code affects 1 cell for 15 min, not all customers for 1 second.                                        │
├────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────┤                                 
│ Deadline propagation               │ Prevents cascading timeouts. Slow dependency doesn't hold up the entire page.                              │
├────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────┤                                 
│ Auth + capture (not direct charge) │ Payment failure after order creation is recoverable. Direct charge failure is not.                         │
├────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────┤                                 
│ Active-active multi-region         │ Regional outage = automatic failover. No manual intervention. RTO < 2 min.                                 │
├────────────────────────────────────┼────────────────────────────────────────────────────────────────────────────────────────────────────────────┤                                 
│ Saga with compensation (not 2PC)   │ 2PC blocks on coordinator failure. Saga compensates. No distributed locks.                                 │
└────────────────────────────────────┴────────────────────────────────────────────────────────────────────────────────────────────────────────────┘          