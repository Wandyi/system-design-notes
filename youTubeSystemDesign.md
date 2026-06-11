YouTube Platform — Comprehensive High-Level Design
                                                                                                                                                                                      
---                                                                                                                                                                                 
1. Scale Definition
```

┌───────────────────────────────────────────────────────────────────┐                                                                                                                
│  USERS & ENGAGEMENT                                               │                                                                                                               
│                                                                   │                                                                                                               
│  Monthly active users:         2.5 B                              │                                                                                                               
│  Daily active users:           800 M                              │                                                                                                               
│  Peak concurrent viewers:      100-150 M                          │                                                                                                               
│  Avg session duration:         40 minutes                         │                                                                                                               
│  Hours of video watched/day:   1 B                                │                                                                                                               
│                                                                   │                                                                                                               
│  CONTENT                                                          │                                                                                                               
│                                                                   │                                                                                                               
│  Total videos:                 800 M+                             │
│  Videos uploaded per minute:   500 hours of content               │                                                                                                               
│    → ~3,000 videos/min  =  ~50 videos/second                      │                                                                                                                
│  Average video length:         ~10 minutes                        │                                                                                                               
│  Creators with channels:       50 M+                              │                                                                                                               
│                                                                   │                                                                                                               
│  TRAFFIC                                                          │
│                                                                   │                                                                                                               
│  API requests/sec (peak):      10 M+                              │
│  Search queries/sec:           200 K                              │                                                                                                               
│  Video plays started/sec:      500 K                              │                                                                                                               
│  Comments posted/sec:          50 K                               │                                                                                                               
│  Likes/sec:                    200 K                              │                                                                                                               
│                                                                   │                                                                                                               
│  STORAGE & BANDWIDTH                                              │                                                                                                               
│                                                                   │
│  Total video storage:          ~5 exabytes (all transcoded)       │                                                                                                               
│  Raw uploads/day:              ~1.5 PB                            │                                                                                                               
│  After transcoding/day:        ~10 PB (all variants)              │                                                                                                               
│  Egress bandwidth (peak):      ~1 Pbps (petabit/sec)              │                                                                                                                
│  CDN edge cache hit rate:      95%+                               │                                                                                                               
│  Origin read bandwidth:        ~50 Tbps                           │                                                                                                               
│                                                                   │                                                                                                               
│  INFRASTRUCTURE                                                   │                                                                                                               
│                                                                   │                                                                                                               
│  Data centers:                 30+ (global)                       │
│  CDN edge PoPs:                300+                               │                                                                                                               
│  Transcoding cluster:          50,000+ CPU/GPU cores              │
│  Serving instances:            200,000+                           │                                                                                                               
│  ML model serving:             GPU clusters per region            │
│  Microservices:                1,000+                             │                                                                                                               
└───────────────────────────────────────────────────────────────────┘
```                                                                                                                                                                                      
---             
2. Guiding Principles

1. THE PRODUCT IS THE RECOMMENDATION ENGINE
   70%+ of watch time comes from algorithmic recommendations,                                                                                                                       
   not search or direct links. The recommendation system IS
   YouTube. Everything else is infrastructure that supports it.

2. VIDEO IS A WRITE-ONCE, READ-BILLIONS WORKLOAD                                                                                                                                    
   A video is uploaded once, transcoded once, then read billions                                                                                                                    
   of times. Optimize the entire architecture for read throughput.                                                                                                                  
   It's acceptable for uploads to take minutes. Playback must                                                                                                                       
   start in milliseconds.

3. POPULARITY FOLLOWS POWER LAW                                                                                                                                                     
   0.1% of videos account for 80%+ of views.                                                                                                                                        
   99% of videos are watched rarely (long tail).                                                                                                                                    
   Cache and optimize for the head. Serve the tail from origin.
   Don't over-invest in caching content nobody watches.

4. DECOUPLE UPLOAD FROM PLAYBACK                                                                                                                                                    
   The upload/transcode pipeline can be slow (minutes to hours).                                                                                                                    
   The playback pipeline must be instantaneous. These are                                                                                                                           
   completely independent systems sharing only the storage layer.

5. EVENTUAL CONSISTENCY IS ACCEPTABLE (mostly)                                                                                                                                      
   View counts, like counts, subscriber counts can lag by seconds
   to minutes. A video appearing in search 30 seconds after upload                                                                                                                  
   is fine. But: playback must always work if the video exists.

6. GRACEFUL DEGRADATION, NOT TOTAL FAILURE                                                                                                                                          
   If recommendations are down: show trending/popular.                                                                                                                              
   If comments are down: show the video without comments.                                                                                                                           
   If ads are down: show the video without ads (lose revenue,                                                                                                                       
   not users).

  ---                                                                                                                                                                                 
3. Global Infrastructure
```

                               ┌────────────────┐
                               │   Global DNS   │                                                                                                                                  
                               │  (Anycast +    │
                               │   geo-routing) │                                                                                                                                  
                               └───────┬────────┘                                                                                                                                   
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐                                                                                                              
         │                             │                             │
         ▼                             ▼                             ▼
    ┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐                                                                                                     
    │  CDN EDGE PoP   │          │  CDN EDGE PoP   │          │  CDN EDGE PoP   │
    │  (300+ global)  │          │  (300+ global)  │          │  (300+ global)  │                                                                                                     
    │                 │          │                 │          │                 │                                                                                                     
    │  Video cache    │          │  Video cache    │          │  Video cache    │                                                                                                     
    │  (SSD+HDD,      │          │  (SSD+HDD,      │          │  (SSD+HDD,      │                                                                                                        
    │   100TB-1PB     │          │   100TB-1PB     │          │   100TB-1PB     │                                                                                                     
    │   per PoP)      │          │   per PoP)      │          │   per PoP)      │                                                                                                     
    │                 │          │                 │          │                 │                                                                                                     
    │  Static assets  │          │  Static assets  │          │  Static assets  │                                                                                                     
    │  Thumbnail cache│          │  Thumbnail cache│          │  Thumbnail cache│
    └────────┬────────┘          └────────┬────────┘          └────────┬────────┘                                                                                                     
             │ miss                       │ miss                       │ miss
             ▼                            ▼                            ▼                                                                                                              
    ┌──────────────────────────────────────────────────────────────────────────┐
    │  ORIGIN SHIELD (regional mid-tier cache)                                 │                                                                                                     
    │                                                                          │                                                                                                     
    │  Absorbs cache misses from multiple edge PoPs in the same region.        │
    │  Prevents thundering herd on origin when a popular video is requested    │                                                                                                      
    │  from 50 edge PoPs simultaneously (each misses, but origin shield has it)│                                                                                                      
    │                                                                          │                                                                                                     
    │  US-EAST shield    EU-WEST shield    AP-EAST shield                      │                                                                                                      
    │  (500 TB SSD)      (500 TB SSD)      (500 TB SSD)                        │                                                                                                       
    └──────────────────────────────┬───────────────────────────────────────────┘                                                                                                     
                                   │ miss                                                                                                                                             
                                   ▼                                                                                                                                                  
    ┌──────────────────────────────────────────────────────────────────────────┐
    │  ORIGIN (data center, per region)                                         │                                                                                                     
    │                                                                           │                                                                                                     
    │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐               │                                                                                                        
    │  │  Video Blob   │  │  Metadata     │  │  API Serving  │               │                                                                                                        
    │  │  Storage      │  │  Services     │  │  Layer        │               │
    │  │  (Colossus /  │  │               │  │               │               │                                                                                                        
    │  │   S3-like)    │  │               │  │               │               │                                                                                                        
    │  └───────────────┘  └───────────────┘  └───────────────┘               │                                                                                                        
    └──────────────────────────────────────────────────────────────────────────┘                                                                                                      

CDN Video Caching Strategy

TIERED CACHE ADMISSION (not everything gets cached):

┌─────────────────────────────────────────────────────────────┐
│                                                             │                                                                                                                    
│  HOT TIER (SSD, edge PoP):                                  │                                                                                                                     
│    Top 0.1% of videos by recent views.                      │
│    ~800K videos × 10 variants × 500MB avg = ~4 PB globally  │                                                                                                                      
│    Distributed across 300 PoPs → ~13 TB per PoP             │                                                                                                                     
│    Cache hit rate: 80% of all bytes served                  │                                                                                                                    
│                                                             │                                                                                                                    
│  WARM TIER (HDD, edge PoP + origin shield):                 │                                                                                                                     
│    Top 5% of videos by recent views.                        │                                                                                                                     
│    ~40M videos × selective variants = ~100 PB globally      │
│    Not every PoP has every warm video.                      │                                                                                                                     
│    Cache admission: promoted on 2nd request within 1 hour.  │                                                                                                                     
│    Cache hit rate: additional 15%                            │                                                                                                                    
│                                                              │                                                                                                                    
│  COLD TIER (origin only):                                    │                                                                                                                    
│    95% of videos. Rarely watched.                            │
│    Stored in blob storage. Served on-demand.                 │                                                                                                                    
│    Higher latency (origin read), but acceptable for          │                                                                                                                    
│    videos with < 100 views/day.                              │                                                                                                                    
│    Some cold videos are only stored in 2-3 resolutions      │                                                                                                                     
│    (not all 10) to save storage.                            │                                                                                                                    
│                                                             │                                                                                                                    
│  Total CDN cache hit rate: ~95%                             │                                                                                                                    
│  Origin serves only ~5% of bytes = ~50 Tbps peak            │                                                                                                                     
│                                                             │                                                                                                                    
└─────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
```          
4. Video Upload & Processing Pipeline

This is the most complex offline pipeline. A single video upload triggers dozens of processing steps.
```
┌────────────────────────────────────────────────────────────────────────────┐
│  VIDEO UPLOAD & PROCESSING PIPELINE                                         │                                                                                                     
│                                                                             │                                                                                                     
│           ┌──────────┐                                                      │
│           │  Creator │                                                      │                                                                                                     
│           │  uploads │                                                      │                                                                                                     
│           │  video   │                                                      │
│           └─────┬────┘                                                      │                                                                                                      
│                 │                                                           │                                                                                                     
│                 ▼                                                           │
│  ┌──────────────────────────────────────────────────────────────────────┐   │                                                                                                      
│  │  UPLOAD SERVICE                                                       │  │                                                                                                     
│  │                                                                       │  │
│  │  • Resumable upload (tus protocol or proprietary)                     │  │                                                                                                     
│  │    → large files (4K video = 10+ GB) need resume on network failure   │  │                                                                                                     
│  │  • Client chunks file into 8MB segments                               │  │                                                                                                     
│  │  • Each segment uploaded independently, can retry individually        │  │                                                                                                     
│  │  • Server reassembles on completion                                   │  │                                                                                                     
│  │  • Upload to nearest regional blob storage                            │  │                                                                                                     
│  │                                                                        │  │                                                                                                    
│  │  Rate: 50 uploads/sec × avg 1 GB = 50 GB/s ingest                    │  │
│  │                                                                        │  │                                                                                                    
│  │  On upload complete:                                                   │  │
│  │    → Write raw file to blob storage (Colossus / S3)                   │  │                                                                                                     
│  │    → Create video metadata record (status: PROCESSING)                │  │                                                                                                     
│  │    → Publish event: "video.uploaded"                                   │  │                                                                                                    
│  │    → Return upload_id to creator                                       │  │                                                                                                    
│  └──────────────────────────────────────────────────────────────────────┘  │                                                                                                      
│        │                                                                    │                                                                                                     
│        │ event: video.uploaded                                              │                                                                                                     
│        ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │                                                                                                      
│  │  PROCESSING ORCHESTRATOR (DAG-based workflow engine)                   │  │                                                                                                    
│  │                                                                        │  │                                                                                                    
│  │  Manages a directed acyclic graph of tasks per video:                 │  │                                                                                                     
│  │                                                                        │  │                                                                                                    
│  │                    ┌──────────────┐                                    │  │
│  │                    │  Raw Video   │                                    │  │                                                                                                    
│  │                    │  Uploaded    │                                    │  │                                                                                                    
│  │                    └──────┬───────┘                                    │  │
│  │                           │                                            │  │                                                                                                    
│  │              ┌────────────┼────────────┐                              │  │                                                                                                     
│  │              ▼            ▼            ▼                              │  │
│  │        ┌──────────┐ ┌──────────┐ ┌──────────┐                       │  │                                                                                                       
│  │        │ Validate │ │ Extract  │ │ Generate │                       │  │
│  │        │ format,  │ │ metadata │ │ **video  │                       │  │                                                                                                       
│  │        │ codecs,  │ │ (duration│ │ finger-  │                       │  │
│  │        │ check    │ │  resolut-│ │ print**  │                       │  │                                                                                                       
│  │        │ corrupt  │ │  ion,    │ │ (Content │                       │  │                                                                                                       
│  │        │          │ │  fps)    │ │  ID)     │                       │  │                                                                                                       
│  │        └────┬─────┘ └────┬─────┘ └────┬─────┘                       │  │                                                                                                       
│  │             │            │            │                             │  │
│  │             ▼            ▼            ▼                             │  │                                                                                                      
│  │        ┌─────────────────────────────────────┐                      │  │
│  │        │          VALIDATION GATE            │                      │  │                                                                                                      
│  │        │  All 3 must pass before transcoding │                      │  │
│  │        └──────────────┬──────────────────────┘                      │  │                                                                                                       
│  │                       │                                             │  │                                                                                                      
│  │         ┌─────────────┼─────────────┐                               │  │                                                                                                       
│  │         ▼             ▼             ▼                               │  │                                                                                                       
│  │    ┌─────────┐  ┌──────────┐  ┌──────────┐                        │  │
│  │    │Transcode│  │Transcode │  │Transcode │  ... (parallel)        │  │                                                                                                         
│  │    │ 360p    │  │ 720p     │  │ 1080p    │                        │  │                                                                                                         
│  │    │ H.264   │  │ H.264    │  │ H.264    │                        │  │                                                                                                         
│  │    └────┬────┘  └────┬─────┘  └────┬─────┘                        │  │                                                                                                         
│  │         │            │             │                              │  │                                                                                                        
│  │    ┌────┴────┐  ┌────┴─────┐  ┌────┴─────┐                        │  │
│  │    │Transcode│  │Transcode │  │Transcode │  ... (parallel)        │  │                                                                                                         
│  │    │ 360p    │  │ 720p     │  │ 1080p    │                        │  │                                                                                                         
│  │    │ VP9     │  │ VP9      │  │ VP9      │                        │  │                                                                                                         
│  │    └────┬────┘  └────┬─────┘  └────┬─────┘                        │  │                                                                                                         
│  │         │            │             │                              │  │                                                                                                        
│  │    ┌────┴────┐  ┌────┴─────┐  ┌────┴─────┐                        │  │
│  │    │Transcode│  │Transcode │  │Transcode │  (lower priority,      │  │                                                                                                         
│  │    │ 360p    │  │ 720p     │  │ 1080p    │   AV1 is CPU-heavy)    │  │                                                                                                         
│  │    │ AV1     │  │ AV1      │  │ AV1      │                        │  │                                                                                                         
│  │    └────┬────┘  └────┬─────┘  └────┬─────┘                        │  │                                                                                                         
│  │         │            │             │                               │  │                                                                                                        
│  │         └────────────┼─────────────┘                               │  │                                                                                                        
│  │                      ▼                                             │  │                                                                                                       
│  │              ┌───────────────┐                                     │  │
│  │              │  PACKAGING    │                                     │  │                                                                                                       
│  │              │               │                                     │  │
│  │              │ • Generate    │                                     │  │                                                                                                       
│  │              │   DASH/HLS   │                                      │  │                                                                                                        
│  │              │   manifests  │                                      │  │
│  │              │ • Segment    │                                      │  │                                                                                                        
│  │              │   videos     │                                      │  │                                                                                                        
│  │              │   (2-6s      │                                      │  │
│  │              │   chunks)    │                                      │  │                                                                                                        
│  │              │ • Generate   │                                      │  │
│  │              │   thumbnails │                                      │  │                                                                                                        
│  │              │   (multiple) │                                      │  │
│  │              │ • Extract    │                                      │  │                                                                                                        
│  │              │   audio-only │                                      │  │
│  │              │   track      │                                      │  │                                                                                                        
│  │              │ • Generate   │                                      │  │                                                                                                        
│  │              │   subtitles  │                                      │  │
│  │              │   (auto-     │                                      │  │                                                                                                        
│  │              │   caption ML)│                                      │  │                                                                                                        
│  │              └───────┬──────┘                                      │  │
│  │                      │                                             │  │                                                                                                       
│  │         ┌────────────┼────────────┐                                │  │                                                                                                        
│  │         ▼            ▼            ▼                                │  │
│  │   ┌──────────┐ ┌──────────┐ ┌──────────┐                         │  │                                                                                                          
│  │   │ Content  │ │ Copyright│ │ Safety   │                         │  │                                                                                                          
│  │   │ Modera-  │ │ Check    │ │ Classi-  │                         │  │                                                                                                          
│  │   │ tion (ML)│ │(ContentID│ │ fier(age │                         │  │                                                                                                          
│  │   │          │ │ finger-  │ │ restrict,│                         │  │                                                                                                          
│  │   │ violence,│ │ print    │ │ sensitive│                         │  │
│  │   │ nudity,  │ │ matching)│ │ content) │                         │  │                                                                                                          
│  │   │ spam,    │ │          │ │          │                         │  │                                                                                                          
│  │   │ misinfo  │ │          │ │          │                         │  │                                                                                                          
│  │   └────┬─────┘ └────┬─────┘ └────┬─────┘                         │  │                                                                                                          
│  │        │            │            │                               │  │                                                                                                         
│  │        └────────────┼────────────┘                               │  │                                                                                                         
│  │                     ▼                                              │  │
│  │             ┌───────────────┐                                      │  │                                                                                                        
│  │             │  PUBLISH GATE │                                      │  │                                                                                                        
│  │             │               │                                      │  │
│  │             │ All checks OK │──▶ status = PUBLISHED                │  │                                                                                                         
│  │             │               │    Publish "video.published" event   │  │                                                                                                        
│  │             │               │    Video appears in search/recs      │  │                                                                                                         
│  │             │               │                                      │  │                                                                                                        
│  │             │ Copyright hit │──▶ status = COPYRIGHT_CLAIM          │  │                                                                                                        
│  │             │               │    Notify creator. Options:         │  │                                                                                                         
│  │             │               │    dispute, mute audio, block.      │  │                                                                                                         
│  │             │               │                                     │  │                                                                                                        
│  │             │ Safety fail   │──▶ status = UNDER_REVIEW            │  │                                                                                                         
│  │             │               │    Queue for human review.          │  │
│  │             │               │    Auto-block if high confidence.   │  │                                                                                                         
│  │             └───────────────┘                                     │  │
│  └──────────────────────────────────────────────────────────────────┘   │                                                                                                          
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘

Transcoding Strategy

┌──────────────────────────────────────────────────────────────────┐
│  TRANSCODING MATRIX                                              │
│                                                                  │                                                                                                               
│  Resolution   H.264 (most compatible)   VP9 (better)   AV1 (best)│
│  ─────────── ─────────────────────── ──────────── ────────────   │                                                                                                                  
│  144p         ✓ immediate               ✓ immediate  ✗ not worth │                                                                                                                
│  240p         ✓ immediate               ✓ immediate  ✗ not worth │                                                                                                                
│  360p         ✓ immediate               ✓ immediate  ✓ delayed   │                                                                                                                
│  480p         ✓ immediate               ✓ immediate  ✓ delayed   │                                                                                                                
│  720p         ✓ immediate               ✓ immediate  ✓ delayed   │                                                                                                                
│  1080p        ✓ immediate               ✓ immediate  ✓ delayed   │                                                                                                                
│  1440p        ✓ if source ≥ 1440p       ✓ delayed    ✓ deferred  │
│  2160p (4K)   ✓ if source ≥ 4K          ✓ delayed    ✓ deferred  │                                                                                                                
│                                                                   │                                                                                                               
│  "immediate": starts as soon as validation passes                 │                                                                                                               
│  "delayed":   starts after H.264 variants are done                │                                                                                                               
│  "deferred":  only transcoded if the video reaches a view         │
│               threshold (e.g., >10K views) — saves compute        │                                                                                                               
│               for the 95% of videos that will never be popular    │                                                                                                               
│                                                                   │                                                                                                               
│  WHY MULTIPLE CODECS:                                             │                                                                                                               
│    H.264: universal playback (every browser, every device)        │                                                                                                               
│    VP9:   30-50% better compression, supported on Chrome/Android  │                                                                                                               
│    AV1:   50-70% better compression, newer devices only           │                                                                                                               
│           AV1 saves massive bandwidth on popular videos.          │                                                                                                               
│           At 1M views, 50% bandwidth saving = terabytes saved.    │                                                                                                               
│                                                                   │                                                                                                               
│  CONTENT-AWARE ENCODING:                                          │                                                                                                               
│    • Animation (low motion): lower bitrate, same quality          │                                                                                                               
│    • Sports (high motion): higher bitrate needed                  │                                                                                                               
│    • Talking head (static background): very low bitrate           │
│    • Per-scene bitrate adaptation within the same video           │                                                                                                               
│      (CRF / 2-pass encoding adjusts quality per GOP)              │                                                                                                               
│                                                                   │                                                                                                               
│  COMPUTE:                                                         │                                                                                                               
│    50 videos/sec × ~20 transcode jobs each = 1,000 jobs/sec       │                                                                                                                
│    Average transcode time: 2-10 minutes per variant               │                                                                                                               
│    Concurrency: ~50,000 transcode workers                         │                                                                                                               
│    Hardware: CPU for H.264/VP9, GPU for AV1 (hardware encoders)   │                                                                                                               
│    Cost optimization: spot/preemptible instances for deferred jobs│                                                                                                              
│                                                                   │                                                                                                               
│  PRIORITY QUEUE:                                                  │                                                                                                              
│    P0: Premieres, live-to-VOD, verified creators > 1M subs        │                                                                                                                
│    P1: Verified creators, scheduled publishes                     │                                                                                                               
│    P2: Regular uploads                                            │                                                                                                              
│    P3: Deferred AV1 transcodes, re-encodes                        │                                                                                                               
│                                                                   │                                                                                                               
│    P0 videos are transcoded within 1-2 minutes.                  │
│    P2 videos may wait 10-30 minutes during peak upload hours.    │                                                                                                                
└──────────────────────────────────────────────────────────────────┘
```
Progressive Availability

The creator and viewers don't wait for all variants. The video becomes playable as soon as the first variants are done.

T+0:00   Upload complete. Raw file in blob storage.                                                                                                                                 
T+0:05   Validation passed. Metadata extracted.                                                                                                                                     
T+0:30   360p H.264 done → video is PLAYABLE (low quality only)                                                                                                                     
T+0:45   720p H.264 done → 720p available to viewers                                                                                                                                
T+1:00   1080p H.264 done → HD available                                                                                                                                            
T+1:30   720p VP9 done → Chrome/Android viewers get better codec                                                                                                                    
T+2:00   1080p VP9 done                                                                                                                                                             
T+3:00   All H.264 + VP9 variants done
T+5:00   Thumbnails, auto-captions, content moderation complete                                                                                                                     
→ video.published event fired                                                                                                                                              
→ video appears in search and recommendations

T+24:00  If video reaches >10K views: AV1 transcode queued                                                                                                                          
T+48:00  AV1 variants available for compatible devices
                                                                                                                                                                                      
---             
5. Video Storage Architecture
```
┌──────────────────────────────────────────────────────────────────────┐
│  VIDEO BLOB STORAGE                                                   │                                                                                                           
│                                                                       │
│  Not a traditional filesystem. A purpose-built distributed blob store │                                                                                                           
│  optimized for large sequential reads (video streaming).              │                                                                                                           
│                                                                       │                                                                                                           
│  ┌──────────────────────────────────────────────────────────────┐    │                                                                                                            
│  │  STORAGE TIERING                                              │    │                                                                                                           
│  │                                                                │    │
│  │  ┌─────────────────────────────────────────────┐              │    │                                                                                                           
│  │  │  HOT (SSD, replicated 3x across AZs)        │              │    │                                                                                                           
│  │  │                                              │              │    │                                                                                                          
│  │  │  Videos uploaded in last 7 days              │              │    │                                                                                                          
│  │  │  + Videos with >1K views/day                 │              │    │                                                                                                          
│  │  │  ~5% of catalog = ~40M videos                │              │    │                                                                                                          
│  │  │  ~200 PB                                     │              │    │                                                                                                          
│  │  │  Avg read latency: 5-10ms                    │              │    │                                                                                                          
│  │  └─────────────────────────────────────────────┘              │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  ┌─────────────────────────────────────────────┐              │    │                                                                                                           
│  │  │  WARM (HDD, replicated 3x)                   │              │    │                                                                                                          
│  │  │                                              │              │    │                                                                                                          
│  │  │  Videos with 10-1K views/day                 │              │    │
│  │  │  ~15% of catalog = ~120M videos              │              │    │                                                                                                          
│  │  │  ~1 EB                                       │              │    │
│  │  │  Avg read latency: 20-50ms                   │              │    │                                                                                                          
│  │  └─────────────────────────────────────────────┘              │    │
│  │                                                                │    │                                                                                                          
│  │  ┌─────────────────────────────────────────────┐              │    │
│  │  │  COLD (HDD, erasure-coded 10+4)              │              │    │                                                                                                          
│  │  │                                              │              │    │                                                                                                          
│  │  │  Videos with <10 views/day                   │              │    │
│  │  │  80% of catalog = ~640M videos               │              │    │                                                                                                          
│  │  │  ~3.5 EB                                     │              │    │
│  │  │  Avg read latency: 100-500ms                 │              │    │                                                                                                          
│  │  │                                              │              │    │
│  │  │  Erasure coding: 10 data + 4 parity shards  │              │    │                                                                                                           
│  │  │    = 1.4x storage overhead (vs 3x for repl) │              │    │
│  │  │    Can tolerate 4 simultaneous shard losses  │              │    │                                                                                                          
│  │  │    Saves ~50% storage vs replication         │              │    │                                                                                                          
│  │  │    at ~5 EB scale = ~2.5 EB saved            │              │    │                                                                                                          
│  │  └─────────────────────────────────────────────┘              │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  Automatic migration between tiers based on access patterns:   │    │                                                                                                          
│  │    Hot → Warm: no access for 7 days                            │    │
│  │    Warm → Cold: no access for 30 days                          │    │                                                                                                          
│  │    Cold → Warm: accessed 3+ times in a day (viral resurgence) │    │                                                                                                           
│  └──────────────────────────────────────────────────────────────┘    │                                                                                                            
│                                                                       │                                                                                                           
│  VIDEO SEGMENT LAYOUT:                                                │                                                                                                           
│                                                                       │                                                                                                           
│  Videos are NOT stored as single large files.                         │                                                                                                           
│  Each video is segmented during packaging:                            │
│                                                                       │                                                                                                           
│  video_id: v_abc123                                                   │
│  ├── manifest.mpd (DASH) / playlist.m3u8 (HLS)                      │                                                                                                             
│  ├── 1080p_h264/                                                      │                                                                                                           
│  │   ├── init.mp4          (initialization segment)                   │                                                                                                           
│  │   ├── seg_000.m4s       (0:00 - 0:04, 4-second chunk)            │                                                                                                             
│  │   ├── seg_001.m4s       (0:04 - 0:08)                            │                                                                                                             
│  │   ├── seg_002.m4s       (0:08 - 0:12)                            │                                                                                                             
│  │   └── ...                                                          │                                                                                                           
│  ├── 720p_h264/                                                       │                                                                                                           
│  │   └── ...                                                          │
│  ├── 720p_vp9/                                                        │                                                                                                           
│  │   └── ...                                                          │                                                                                                           
│  ├── thumbnails/                                                      │
│  │   ├── default.jpg                                                  │                                                                                                           
│  │   ├── sprite_sheet.jpg  (hover preview thumbnails)                │
│  │   └── storyboard.vtt    (thumbnail timecodes)                     │                                                                                                            
│  └── captions/                                                        │                                                                                                           
│      ├── en_auto.vtt       (auto-generated)                          │                                                                                                            
│      └── es_manual.vtt     (creator-uploaded)                        │                                                                                                            
│                                                                       │                                                                                                           
│  WHY SEGMENTS:                                                        │                                                                                                           
│    • Seeking: client requests seg_075.m4s directly, no server-side   │                                                                                                            
│      byte-range calculation                                           │                                                                                                           
│    • Caching: individual segments are cache-friendly (small, fixed    │                                                                                                           
│      size, immutable after creation)                                  │                                                                                                           
│    • Adaptive: client can switch quality per-segment                  │                                                                                                           
│    • CDN: segments are individual objects, standard HTTP caching      │                                                                                                           
└──────────────────────────────────────────────────────────────────────┘

Storage for Non-Video Data

┌──────────────────────────────────────────────────────────────────┐                                                                                                                
│  METADATA STORAGE (separate from video blobs)                     │
│                                                                   │                                                                                                               
│  Service                Store              Size                   │
│  ────────────────────── ────────────────── ────────────────────  │                                                                                                                 
│  Video metadata          Vitess (sharded   800M rows, ~2 TB      │
│  (title, description,    MySQL)            Sharded by video_id   │                                                                                                                
│  channel, upload date)                                            │                                                                                                               
│                                                                   │                                                                                                               
│  View counts             Redis + Cassandra Counters, 800M keys   │                                                                                                                
│  (real-time approx +                       Redis for real-time   │                                                                                                                
│   batch-accurate)                          Cassandra for durable │                                                                                                                
│                                                                   │                                                                                                               
│  User profiles           Bigtable /        2.5B rows             │                                                                                                                
│  (watch history,         DynamoDB                                 │                                                                                                               
│   subscriptions)                                                  │                                                                                                               
│                                                                   │                                                                                                               
│  Comments                Vitess (sharded   200B+ comments        │                                                                                                                
│                          MySQL)            Sharded by video_id   │
│                                                                   │                                                                                                               
│  Likes / reactions       Cassandra         Counter columns       │
│                                            per video_id          │                                                                                                                
│                                                                   │
│  Subscriptions           Bigtable          50M creators ×        │                                                                                                                
│  (creator → subscribers)                   avg 10K subs          │                                                                                                                
│                                                                   │                                                                                                               
│  Search index            Custom inverted   500M documents        │                                                                                                                
│                          index / Elastic   ~5 TB index           │                                                                                                                
│                                                                   │                                                                                                               
│  Recommendations         Feature store     Pre-computed per-user │                                                                                                                
│  (pre-computed)          (Redis + Bigtable) candidate lists      │                                                                                                                
│                                                                   │                                                                                                               
│  Ad targeting            Bigtable +        User interest graphs  │                                                                                                                
│  (user interest graph)   ML feature store                        │                                                                                                                
│                                                                   │                                                                                                               
│  Analytics               ClickHouse /      Event-sourced from    │                                                                                                                
│  (creator dashboard)     BigQuery          Kafka. Columnar.      │                                                                                                                
└──────────────────────────────────────────────────────────────────┘
  ```                                                                                                                                                                                    
---                                                                                                                                                                                 
6. Video Playback / Streaming
```
┌────────────────────────────────────────────────────────────────────────┐
│  VIDEO PLAYBACK FLOW                                                   │                                                                                                         
│                                                                        │
│  User clicks a video:                                                  │                                                                                                         
│                                                                        │
│  1. PLAYER INITIALIZATION (client-side)                                │                                                                                                          
│     ┌──────────────────────────────────────────────────────────────┐   │                                                                                                          
│     │  Browser / App                                               │   │
│     │                                                              │   │                                                                                                        
│     │  GET /api/video/v_abc123/player_config                       │   │
│     │  → Returns:                                                  │   │                                                                                                        
│     │    {                                                         │   │                                                                                                        
│     │      manifest_url: "https://cdn.yt/v_abc123/manifest.mpd",   │   │                                                                                                          
│     │      available_qualities: [360, 720, 1080, 1440],             │   │                                                                                                         
│     │      available_codecs: ["h264", "vp9", "av1"],                │   │                                                                                                         
│     │      captions: [{lang:"en", url:"..."}],                      │   │                                                                                                         
│     │      ad_breaks: [{offset: 0, type: "pre-roll"},               │   │                                                                                                         
│     │                  {offset: 180, type: "mid-roll"}],            │   │                                                                                                         
│     │      cdn_host: "edge-lax-042.cdn.yt",                         │   │                                                                                                          
│     │      initial_quality: 720   (server-recommended based         │   │                                                                                                         
│     │                              on user's historical bandwidth)  │   │                                                                                                         
│     │    }                                                          │   │                                                                                                        
│     └──────────────────────────────────────────────────────────────┘    │                                                                                                          
│                                                                         │                                                                                                         
│  2. MANIFEST FETCH                                                      │
│     ┌──────────────────────────────────────────────────────────────┐    │                                                                                                          
│     │  GET https://cdn.yt/v_abc123/manifest.mpd                     │   │                                                                                                         
│     │                                                                │  │
│     │  DASH manifest lists all available AdaptationSets:            │   │                                                                                                         
│     │    <AdaptationSet mimeType="video/mp4" codecs="avc1.64001f"> │   │
│     │      <Representation bandwidth="800000"  width="640"  .../>  │   │                                                                                                          
│     │      <Representation bandwidth="2400000" width="1280" .../>  │   │                                                                                                          
│     │      <Representation bandwidth="5000000" width="1920" .../>  │   │                                                                                                          
│     │    </AdaptationSet>                                           │   │                                                                                                         
│     │    <AdaptationSet mimeType="video/webm" codecs="vp9">       │   │
│     │      ...                                                      │   │                                                                                                         
│     │    </AdaptationSet>                                           │   │
│     │    <AdaptationSet mimeType="audio/mp4" codecs="mp4a.40.2">  │   │                                                                                                           
│     │      ...                                                      │   │                                                                                                         
│     │    </AdaptationSet>                                           │   │
│     └──────────────────────────────────────────────────────────────┘   │                                                                                                          
│                                                                         │                                                                                                         
│  3. ADAPTIVE BITRATE STREAMING (ABR)                                   │
│     ┌──────────────────────────────────────────────────────────────┐   │                                                                                                          
│     │                                                                │   │
│     │  Player's ABR algorithm continuously decides quality:          │   │                                                                                                        
│     │                                                                │   │                                                                                                        
│     │  Bandwidth                                                     │   │
│     │  estimate     Segment quality chosen                           │   │                                                                                                        
│     │    │                                                           │   │                                                                                                        
│     │    │    8 Mbps ──── 1080p ────────────────                    │   │                                                                                                         
│     │    │                        \                                  │   │                                                                                                        
│     │    │                         \___ bandwidth drops              │   │                                                                                                        
│     │    │    3 Mbps ───────────────────── 720p ───                 │   │                                                                                                         
│     │    │                                        \                  │   │                                                                                                        
│     │    │    1 Mbps ──────────────────────────────── 360p          │   │                                                                                                         
│     │    │                                                           │   │                                                                                                        
│     │    └──────────────────────────────────────────▶ Time          │   │                                                                                                         
│     │                                                                │   │                                                                                                        
│     │  For each segment:                                             │   │                                                                                                        
│     │    1. Estimate current bandwidth (moving avg of last 5 segs)  │   │                                                                                                         
│     │    2. Check buffer level (how many seconds buffered ahead)    │   │                                                                                                         
│     │    3. If buffer is healthy (>10s): try higher quality         │   │                                                                                                         
│     │    4. If buffer is draining (<5s): drop to lower quality      │   │                                                                                                         
│     │    5. Request next segment at chosen quality from CDN         │   │                                                                                                         
│     │                                                               │   │                                                                                                        
│     │  GET https://edge-lax-042.cdn.yt/v_abc123/1080p_vp9/seg_014  │   │                                                                                                          
│     │  → CDN edge cache HIT → 1ms response                         │   │                                                                                                          
│     │  → CDN edge cache MISS → origin shield → origin → 50-200ms   │   │                                                                                                          
│     │                                                              │   │                                                                                                        
│     └──────────────────────────────────────────────────────────────┘   │                                                                                                          
│                                                                         │                                                                                                         
│  4. SEEKING                                                            │
│     ┌──────────────────────────────────────────────────────────────┐   │                                                                                                          
│     │  User scrubs to 5:32 in the video.                            │   │                                                                                                         
│     │                                                                │   │                                                                                                        
│     │  5:32 / 4s per segment = segment #83                          │   │                                                                                                         
│     │                                                                │   │                                                                                                        
│     │  Player:                                                       │   │
│     │    1. **Flush current buffer**                                     │   │                                                                                                        
│     │    2. Request seg_083.m4s from CDN                             │   │                                                                                                        
│     │    3. If seg_083 is not in CDN cache (cold segment of a       │   │                                                                                                         
│     │       rarely-watched part of the video), it's fetched from    │   │                                                                                                         
│     │       origin. Latency: 50-200ms.                              │   │                                                                                                         
│     │    4. Begin playback from seg_083                              │   │                                                                                                        
│     │                                                                │   │                                                                                                        
│     │  Hover preview (thumbnail scrub bar):                          │   │                                                                                                        
│     │    Sprite sheet loaded at player init.                         │   │                                                                                                        
│     │    100 thumbnails in a single image file (~200KB).             │   │                                                                                                        
│     │    CSS background-position used to show the right frame.      │   │                                                                                                         
│     └──────────────────────────────────────────────────────────────┘   │                                                                                                          
│                                                                         │                                                                                                         
└────────────────────────────────────────────────────────────────────────┘

CDN Segment Request Routing

Client request for seg_014.m4s:

    Client ──▶ DNS (Anycast) ──▶ Nearest CDN PoP                                                                                                                                      
                                      │
                                Cache lookup: seg_014.m4s                                                                                                                             
                                      │                                                                                                                                               
                      ┌───────────────┼───────────────┐
                      │               │               │                                                                                                                               
                    HIT            MISS             MISS
                    (95%)          (popular)        (cold)                                                                                                                            
                      │               │               │                                                                                                                               
                      ▼               ▼               ▼
                Serve from        Origin Shield    Origin                                                                                                                             
                edge cache        (regional)       (data center)                                                                                                                      
                (1ms)             (10-30ms)        (50-200ms)                                                                                                                         
                                      │                                                                                                                                               
                                Cache on shield                                                                                                                                       
                                + cache on edge                                                                                                                                       
                                for subsequent
                                requests                                                                                                                                              
                  
    For popular videos (top 0.1%):                                                                                                                                                    
      Edge cache hit rate ≈ 99%
      Almost zero origin reads.                                                                                                                                                       
                                                                                                                                                                                      
    For a video with 10 views/day:                                                                                                                                                    
      Edge cache hit rate ≈ 0%                                                                                                                                                        
      Every view goes to origin or shield.
      This is fine — it's 10 requests, not 10 million.                                                                                                                                
                                                                                                                                                                                      
---                                                                                                                                                                                 
7. Recommendation Engine

This is the core of the product. More complex than any other component.

┌────────────────────────────────────────────────────────────────────────┐
│  RECOMMENDATION PIPELINE                                                │                                                                                                         
│                                                                         │
│  Two phases: CANDIDATE GENERATION (fast, broad)                        │                                                                                                          
│              RANKING (slow, precise)                                    │                                                                                                         
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │                                                                                                          
│  │  PHASE 1: CANDIDATE GENERATION                                    │  │
│  │                                                                    │  │                                                                                                        
│  │  Goal: reduce 800M videos → ~10,000 candidates in < 50ms         │ ] │                                                                                                          
│  │                                                                    │  │                                                                                                        
│  │  Multiple sources, run in parallel:                                │  │                                                                                                        
│  │                                                                    │  │                                                                                                        
│  │  ┌──────────────────┐  Output: ~3,000 candidates                  │  │                                                                                                         
│  │  │ Collaborative     │  "Users who watched X also watched Y"      │  │                                                                                                         
│  │  │ Filtering         │  Model: two-tower neural network           │  │                                                                                                         
│  │  │ (user embedding   │  User embedding (512d) → ANN search        │  │                                                                                                         
│  │  │  ×                │  against video embedding index             │  │                                                                                                         
│  │  │  video embedding) │  (FAISS / ScaNN, billions of vectors)      │  │                                                                                                         
│  │  └──────────────────┘                                              │  │                                                                                                        
│  │                                                                    │  │                                                                                                        
│  │  ┌──────────────────┐  Output: ~2,000 candidates                  │  │                                                                                                         
│  │  │ Content-Based     │  "Videos similar to what you just watched" │  │                                                                                                         
│  │  │ (video features:  │  Features: title embedding, visual         │  │
│  │  │  topic, visual,   │  features (from video frames),             │  │                                                                                                         
│  │  │  audio, text)     │  audio features, topic classification     │  │                                                                                                          
│  │  └──────────────────┘                                              │  │                                                                                                        
│  │                                                                    │  │                                                                                                        
│  │  ┌──────────────────┐  Output: ~2,000 candidates                  │  │
│  │  │ Subscriptions     │  Recent videos from channels user          │  │                                                                                                         
│  │  │                   │  subscribes to. Weighted by engagement     │  │                                                                                                         
│  │  │                   │  with that channel.                        │  │                                                                                                         
│  │  └──────────────────┘                                              │  │                                                                                                        
│  │                                                                    │  │                                                                                                        
│  │  ┌──────────────────┐  Output: ~1,000 candidates                  │  │                                                                                                         
│  │  │ Trending /        │  Country-specific trending.                │  │                                                                                                         
│  │  │ Popular           │  Category-specific trending.               │  │
│  │  │                   │  Breaking news boost.                      │  │                                                                                                         
│  │  └──────────────────┘                                              │  │                                                                                                        
│  │                                                                    │  │                                                                                                        
│  │  ┌──────────────────┐  Output: ~1,000 candidates                  │  │                                                                                                         
│  │  │ Exploration       │  Random sampling to break filter bubbles.  │  │                                                                                                         
│  │  │ (serendipity)     │  New creators. Cross-category discovery.   │  │
│  │  │                   │  5-10% of recommendations are exploratory. │  │                                                                                                         
│  │  └──────────────────┘                                              │  │
│  │                                                                    │  │                                                                                                        
│  │  Union + dedup → ~10,000 unique candidates                        │  │
│  └──────────────────────────────────────────────────────────────────┘  │                                                                                                          
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │                                                                                                          
│  │  PHASE 2: RANKING                                                 │  │                                                                                                         
│  │                                                                    │  │
│  │  Goal: rank 10,000 candidates → top 50 to display                │  │                                                                                                          
│  │  Latency budget: 100-200ms                                        │  │                                                                                                         
│  │                                                                    │  │                                                                                                        
│  │  Deep neural network (transformer-based):                         │  │                                                                                                         
│  │                                                                    │  │                                                                                                        
│  │  Input features per (user, video) pair:                           │  │
│  │  ┌────────────────────────────────────────────────────────────┐  │  │                                                                                                          
│  │  │  User features:                                             │  │  │                                                                                                         
│  │  │    • Watch history (last 200 videos, with watch %)         │  │  │                                                                                                          
│  │  │    • Search history (last 50 queries)                      │  │  │                                                                                                          
│  │  │    • Demographics (age bucket, country, language)           │  │  │
│  │  │    • Time of day, day of week                              │  │  │                                                                                                          
│  │  │    • Device type (mobile, desktop, TV)                     │  │  │                                                                                                          
│  │  │    • Session context (what they just watched)              │  │  │                                                                                                          
│  │  │                                                             │  │  │                                                                                                         
│  │  │  Video features:                                            │  │  │
│  │  │    • Title/description embedding                           │  │  │                                                                                                          
│  │  │    • Channel reputation (subscriber count, avg engagement) │  │  │
│  │  │    • Video age (freshness)                                 │  │  │                                                                                                          
│  │  │    • Historical CTR, avg watch time, avg % watched         │  │  │                                                                                                          
│  │  │    • Like/dislike ratio                                    │  │  │                                                                                                          
│  │  │    • Topic classification (sports, music, tech, etc.)      │  │  │                                                                                                          
│  │  │    • Language, region                                      │  │  │                                                                                                          
│  │  │                                                             │  │  │                                                                                                         
│  │  │  Cross features:                                            │  │  │                                                                                                         
│  │  │    • User's history with this channel                      │  │  │                                                                                                          
│  │  │    • User's affinity for this topic                        │  │  │                                                                                                          
│  │  │    • Time since user last watched this category            │  │  │                                                                                                          
│  │  └────────────────────────────────────────────────────────────┘  │  │                                                                                                          
│  │                                                                    │  │                                                                                                        
│  │  Output: multi-objective prediction                               │  │
│  │    P(click)              — will the user click?                   │  │                                                                                                         
│  │    P(watch > 50%)        — will they watch more than half?       │  │                                                                                                          
│  │    P(like)               — will they like it?                    │  │                                                                                                          
│  │    P(share)              — will they share it?                   │  │                                                                                                          
│  │    E(watch_time)         — expected watch time (seconds)         │  │                                                                                                          
│  │    P(satisfied)          — composite satisfaction metric         │  │                                                                                                          
│  │                                                                    │  │                                                                                                        
│  │  Final score = weighted combination:                              │  │                                                                                                         
│  │    score = w1 × P(click) × E(watch_time)                        │  │                                                                                                           
│  │          + w2 × P(satisfied)                                      │  │                                                                                                         
│  │          + w3 × P(share)                                          │  │                                                                                                         
│  │          - w4 × P(regret)   ← "did they wish they hadn't"       │  │                                                                                                           
│  │                                                                    │  │                                                                                                        
│  │  This is NOT just "maximize clicks." Clickbait gets high         │  │
│  │  P(click) but low E(watch_time) and high P(regret).              │  │                                                                                                          
│  │  The combined score penalizes clickbait.                          │  │
│  └──────────────────────────────────────────────────────────────────┘  │                                                                                                          
│                                                                         │                                                                                                         
│  ┌──────────────────────────────────────────────────────────────────┐  │                                                                                                          
│  │  PHASE 3: RE-RANKING / POLICY LAYER                               │  │                                                                                                         
│  │                                                                   │  │                                                                                                        
│  │  After ML ranking, apply business rules:                          │  │
│  │                                                                   │  │                                                                                                        
│  │  • Diversity: don't show 10 videos from the same channel.        │  │
│  │    Max 3 from any single channel in top 20.                      │  │                                                                                                          
│  │  • Freshness: boost videos published in last 24h.                │  │                                                                                                          
│  │  • Demotion: reduce rank for borderline content (not violating   │  │                                                                                                          
│  │    but low quality: spam, misleading titles).                    │  │                                                                                                          
│  │  • Age restriction: filter age-restricted content for            │  │                                                                                                          
│  │    non-verified users.                                           │  │                                                                                                         
│  │  • Regional: comply with country-specific content laws.          │  │                                                                                                          
│  │  • Ad-friendly: if ads are enabled, prefer ad-friendly content   │  │                                                                                                          
│  │    in top positions.                                             │  │                                                                                                         
│  │                                                                  │  │                                                                                                        
│  │  Output: final ordered list of ~50 video IDs                     │  │                                                                                                          
│  └──────────────────────────────────────────────────────────────────┘  │                                                                                                          
│                                                                         │                                                                                                         
│  INFRASTRUCTURE:                                                        │                                                                                                         
│                                                                         │                                                                                                         
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                                                                  │  │                                                                                                        
│  │  Candidate generation: CPU-based, ANN index (FAISS/ScaNN)        │  │
│  │    Billions of embeddings, updated hourly.                       │  │                                                                                                         
│  │    Latency: 10-30ms per source.                                  │  │                                                                                                         
│  │                                                                  │  │                                                                                                        
│  │  Ranking model: GPU inference (TensorRT / TFServing)              │  │                                                                                                         
│  │    Batch scoring: 10K candidates in one GPU pass.                 │  │                                                                                                         
│  │    Latency: 50-100ms.                                             │  │                                                                                                         
│  │    Cluster: 1,000+ GPUs dedicated to ranking inference.           │  │                                                                                                         
│  │                                                                   │  │                                                                                                        
│  │  Feature store (Redis + Bigtable):                                │  │                                                                                                         
│  │    User features: updated in real-time (watch events → Kafka      │  │                                                                                                         
│  │      → feature pipeline → Redis). Read latency: < 2ms.           │  │                                                                                                          
│  │    Video features: batch-computed daily. Read from Bigtable.     │  │                                                                                                          
│  │                                                                  │  │                                                                                                        
│  │  Model retraining:                                               │  │                                                                                                         
│  │    Candidate models: retrained daily on last 30 days of data.    │  │                                                                                                          
│  │    Ranking model: retrained every 6 hours on recent data.        │  │                                                                                                          
│  │    A/B testing: 5-10 ranking experiments running simultaneously. │  │                                                                                                         
│  │    Evaluation metric: overall watch time per user session.       │  │                                                                                                         
│  │                                                                  │  │                                                                                                        
│  └──────────────────────────────────────────────────────────────────┘  │                                                                                                          
│                                                                        │                                                                                                         
│  GRACEFUL DEGRADATION:                                                 │                                                                                                         
│                                                                        │                                                                                                         
│  If ranking model is down:                                             │
│    → Fall back to pre-computed recommendations (batch, last 6 hours)  │                                                                                                           
│  If candidate generation is down:                                       │                                                                                                         
│    → Fall back to subscription-based feed + trending                   │                                                                                                          
│  If feature store is down:                                              │                                                                                                         
│    → Use default features (demographic averages). Quality degrades    │
│      but recommendations still work.                                   │                                                                                                          
│  If everything is down:                                                 │
│    → Show globally trending videos (static list, updated hourly)      │                                                                                                           
│                                                                         │                                                                                                         
└────────────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---             
8. View Count System

View counting sounds simple. At YouTube scale, it's one of the hardest problems.

┌──────────────────────────────────────────────────────────────────────┐
│  VIEW COUNT ARCHITECTURE                                              │                                                                                                           
│                                                                       │                                                                                                           
│  Requirements:                                                        │
│    • 500K video plays started per second at peak                     │                                                                                                            
│    • Counts must be approximately real-time (seconds, not minutes)   │                                                                                                            
│    • Must be resistant to fraud (bot views, view farms)              │                                                                                                            
│    • Must be eventually accurate (creators are paid based on views)  │                                                                                                            
│    • Can't use a single counter row (write hotspot at 500K/s)        │                                                                                                             
│                                                                      │                                                                                                           
│  ┌──────────────────────────────────────────────────────────────┐    │                                                                                                            
│  │  TWO-PATH ARCHITECTURE                                        │   │                                                                                                           
│  │                                                               │   │
│  │  PATH 1: REAL-TIME (approximate, for display)                 │    │                                                                                                           
│  │                                                               │    │                                                                                                          
│  │  Client → View Event API → Kafka topic: "view.events"         │    │
│  │                                │                               │    │                                                                                                          
│  │                                ▼                               │    │
│  │                      ┌──────────────────┐                     │    │                                                                                                           
│  │                      │ Stream Processor  │                     │    │                                                                                                          
│  │                      │ (Flink / custom)  │                     │    │
│  │                      │                   │                     │    │                                                                                                          
│  │                      │ • Window: 5 sec   │                     │    │
│  │                      │ • Aggregate count │                     │    │                                                                                                          
│  │                      │   per video_id    │                     │    │
│  │                      │ • Basic dedup     │                     │    │                                                                                                          
│  │                      │   (user_id +      │                     │    │
│  │                      │    video_id +     │                     │    │                                                                                                          
│  │                      │    5min window)   │                     │    │
│  │                      └────────┬─────────┘                     │    │                                                                                                           
│  │                               │                               │    │
│  │                               ▼                               │    │                                                                                                           
│  │                      ┌──────────────────┐                     │    │                                                                                                           
│  │                      │  Redis counters  │                     │    │
│  │                      │                  │                     │    │                                                                                                           
│  │                      │  INCRBY video_id │                     │    │                                                                                                           
│  │                      │  count_delta     │                     │    │                                                                                                           
│  │                      │                  │                     │    │                                                                                                           
│  │                      │  Sharded across  │                     │    │                                                                                                           
│  │                      │  100+ Redis nodes│                     │    │                                                                                                           
│  │                      └──────────────────┘                     │    │
│  │                                                                │    │                                                                                                          
│  │  Displayed to users: Redis counter value.                     │    │
│  │  Latency: 5-15 seconds behind real time. Approximate.         │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  PATH 2: BATCH (accurate, for monetization)                   │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  Kafka: "view.events" → Data Lake (hourly batch)              │    │                                                                                                           
│  │                              │                                 │    │                                                                                                          
│  │                              ▼                                 │    │                                                                                                          
│  │                      ┌──────────────────┐                     │    │                                                                                                           
│  │                      │  Batch Processor  │                     │    │                                                                                                          
│  │                      │  (MapReduce /     │                     │    │
│  │                      │   Spark)          │                     │    │                                                                                                          
│  │                      │                   │                     │    │
│  │                      │ • Full dedup      │                     │    │                                                                                                          
│  │                      │ • Fraud detection │                     │    │                                                                                                          
│  │                      │   (bot patterns,  │                     │    │
│  │                      │    IP clustering, │                     │    │                                                                                                          
│  │                      │    watch duration │                     │    │
│  │                      │    analysis)      │                     │    │                                                                                                          
│  │                      │ • Precise count   │                     │    │
│  │                      └────────┬─────────┘                     │    │                                                                                                           
│  │                               │                               │    │
│  │                               ▼                               │    │                                                                                                           
│  │                      ┌──────────────────┐                     │    │                                                                                                           
│  │                      │  Cassandra        │                     │    │
│  │                      │  (durable,        │                     │    │                                                                                                          
│  │                      │   authoritative)  │                     │    │                                                                                                          
│  │                      │                   │                     │    │
│  │                      │  Overwrites Redis │                     │    │                                                                                                          
│  │                      │  every hour with  │                     │    │                                                                                                          
│  │                      │  verified count.  │                     │    │
│  │                      └──────────────────┘                     │    │                                                                                                           
│  │                                                                │   │
│  │  Used for: creator analytics, ad revenue calculation,         │    │                                                                                                           
│  │  copyright payment distribution.                              │    │                                                                                                           
│  └──────────────────────────────────────────────────────────────┘     │                                                                                                            
│                                                                       │                                                                                                           
│  FRAUD DETECTION:                                                     │                                                                                                           
│                                                                       │                                                                                                           
│  A "view" counts only if:                                             │
│    • Video played for > 30 seconds (or to completion if shorter)     │                                                                                                            
│    • Not from the same user + video within 30 minutes                │                                                                                                            
│    • Client fingerprint passes bot detection                         │
│    • IP is not on known VPN/proxy/datacenter blocklist               │                                                                                                            
│    • Watch pattern is organic (not 1000 views from same /24 subnet)  │                                                                                                             
│                                                                      │                                                                                                           
│  View count may DECREASE after batch processing strips fraudulent    │                                                                                                            
│  views. This is visible to creators in analytics ("adjusted views"). │                                                                                                            
└──────────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---                                                                                                                                                                                 
9. Live Streaming

┌────────────────────────────────────────────────────────────────────────┐
│  LIVE STREAMING ARCHITECTURE                                            │
│                                                                         │                                                                                                         
│  Fundamentally different from VOD:                                      │
│    VOD:  pre-transcoded, cached, served from storage.                  │                                                                                                          
│    Live: real-time ingest, real-time transcode, real-time delivery.    │                                                                                                          
│          Latency target: 3-10 seconds (standard), <1s (ultra-low).    │
│                                                                         │                                                                                                         
│  ┌──────────────────────────────────────────────────────────────────┐  │                                                                                                          
│  │  INGEST                                                           │  │                                                                                                         
│  │                                                                    │  │                                                                                                        
│  │  Creator's encoder (OBS, hardware encoder)                        │  │                                                                                                         
│  │       │                                                           │  │
│  │       │  RTMP / SRT (reliable streaming protocol)                │  │                                                                                                          
│  │       │  Bitrate: 6-50 Mbps (depending on quality)               │  │                                                                                                          
│  │       ▼                                                           │  │                                                                                                         
│  │  ┌──────────────────┐                                             │  │                                                                                                         
│  │  │  Ingest Edge     │  Nearest regional ingest PoP              │  │                                                                                                           
│  │  │  Server          │  Terminates the RTMP/SRT stream            │  │                                                                                                          
│  │  │                  │  Validates stream key (authentication)     │  │                                                                                                          
│  │  │  Redundancy:     │  Sends to 2 ingest servers simultaneously │  │                                                                                                           
│  │  │  creator sends   │  (primary + hot backup)                    │  │                                                                                                          
│  │  │  to 2 endpoints  │                                             │  │                                                                                                         
│  │  └────────┬─────────┘                                             │  │                                                                                                         
│  │           │                                                       │  │                                                                                                         
│  │           ▼                                                       │  │                                                                                                         
│  │  ┌──────────────────┐                                             │  │
│  │  │  Live Transcoder │  Real-time transcoding:                     │  │                                                                                                         
│  │  │  Cluster         │  Input: single high-quality stream          │  │                                                                                                         
│  │  │                  │  Output: all ABR variants simultaneously    │  │                                                                                                         
│  │  │  GPU-accelerated │    • 1080p60, 720p60, 480p30, 360p30      │  │                                                                                                           
│  │  │  (NVENC/QSV)     │    • H.264 only (VP9/AV1 too slow          │  │                                                                                                          
│  │  │                  │      for real-time at all resolutions)      │  │                                                                                                         
│  │  │  Latency: <1s    │                                             │  │                                                                                                         
│  │  │  per variant     │  Each variant segmented into 2-second      │  │                                                                                                          
│  │  │                  │  chunks (vs 4-6s for VOD — smaller for     │  │                                                                                                          
│  │  │                  │  lower latency)                              │  │                                                                                                        
│  │  └────────┬─────────┘                                             │  │                                                                                                         
│  │           │                                                       │  │                                                                                                         
│  │           ▼                                                       │  │
│  │  ┌──────────────────┐                                             │  │                                                                                                         
│  │  │  Live Origin     │  Holds the last ~60 seconds of segments   │  │
│  │  │                  │  in memory. Writes to blob storage for     │  │                                                                                                          
│  │  │  Generates live  │  DVR (rewind) and VOD (post-stream).      │  │                                                                                                           
│  │  │  DASH/HLS        │                                             │  │                                                                                                         
│  │  │  manifest that   │  Manifest updated every 2 seconds          │  │                                                                                                          
│  │  │  points to the   │  (new segment appended, old removed).      │  │                                                                                                          
│  │  │  latest segments │                                             │  │                                                                                                         
│  │  └────────┬─────────┘                                             │  │                                                                                                         
│  │           │                                                       │  │                                                                                                         
│  │           ▼                                                       │  │                                                                                                         
│  │  CDN edge PoPs fetch segments every 2 seconds.                   │  │
│  │  Cache TTL: 1 second (very short — live content changes fast).   │  │                                                                                                          
│  │  Viewers' players poll manifest every 2 seconds for new segments.│  │                                                                                                          
│  └──────────────────────────────────────────────────────────────────┘  │                                                                                                          
│                                                                         │                                                                                                         
│  LIVE CHAT (concurrent with stream):                                    │                                                                                                         
│                                                                         │                                                                                                         
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Popular stream: 500K concurrent viewers, 10K messages/sec       │  │                                                                                                          
│  │                                                                    │  │                                                                                                        
│  │  Architecture:                                                     │  │                                                                                                        
│  │    Messages → Chat Ingest API                                     │  │                                                                                                         
│  │      → Moderation (ML: spam, harassment, banned words) (5ms)     │  │                                                                                                          
│  │      → Kafka topic: "live_chat.{stream_id}" (partitioned)        │  │                                                                                                          
│  │      → Fan-out service                                            │  │                                                                                                         
│  │        → WebSocket connections to viewers                         │  │                                                                                                         
│  │                                                                    │  │                                                                                                        
│  │  Scaling fan-out for 500K viewers:                                │  │
│  │    NOT 500K individual WebSocket pushes per message.              │  │                                                                                                         
│  │    Instead:                                                        │  │                                                                                                        
│  │    • Server batches messages: 50 messages every 2 seconds        │  │                                                                                                          
│  │    • Each WebSocket server handles ~10K connections               │  │                                                                                                         
│  │    • 50 WS servers per stream (500K / 10K)                       │  │                                                                                                          
│  │    • Each server reads from Kafka and pushes to its connections  │  │                                                                                                          
│  │    • At 10K msg/sec, client receives batches of ~20 messages     │  │                                                                                                          
│  │      every 2 seconds (client-side rendering handles smooth       │  │                                                                                                          
│  │      scrolling)                                                    │  │                                                                                                        
│  │                                                                    │  │                                                                                                        
│  │  Slow mode: if chat is too fast (>1000 msg/sec), auto-enable    │  │                                                                                                           
│  │    rate limit: 1 message per user per 5 seconds.                  │  │                                                                                                         
│  └──────────────────────────────────────────────────────────────────┘  │                                                                                                          
│                                                                         │                                                                                                         
│  LIVE-TO-VOD TRANSITION:                                                │                                                                                                         
│                                                                         │                                                                                                         
│  When stream ends:                                                      │
│    1. All live segments already in blob storage (written during stream) │                                                                                                         
│    2. Concatenate into single video file                                │                                                                                                         
│    3. Queue for full VOD transcoding (VP9, AV1, more resolutions)      │                                                                                                          
│    4. Generate thumbnails, chapters, auto-captions                      │                                                                                                         
│    5. Switch URL from live manifest to VOD manifest                     │                                                                                                         
│    6. Stream becomes a regular video (searchable, recommendable)       │                                                                                                          
│                                                                         │                                                                                                         
└────────────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---                                                                                                                                                                                 
10. Ad Serving

┌──────────────────────────────────────────────────────────────────────┐
│  AD SERVING ARCHITECTURE                                              │                                                                                                           
│                                                                       │
│  YouTube's primary revenue source. Must be:                          │                                                                                                            
│    • Low latency (< 200ms to select and return an ad)                │                                                                                                            
│    • High throughput (500K ad decisions/sec at peak)                  │                                                                                                           
│    • Precisely targeted (user interests, video context, advertiser   │                                                                                                            
│      targeting criteria — all matched in real-time)                   │                                                                                                           
│    • Pacing-aware (don't spend an advertiser's daily budget in       │                                                                                                            
│      the first hour)                                                  │                                                                                                           
│                                                                       │                                                                                                           
│  ┌──────────────────────────────────────────────────────────────┐    │                                                                                                            
│  │  AD DECISION FLOW                                             │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  Video player reaches an ad break point                       │    │
│  │       │                                                       │    │                                                                                                           
│  │       ▼                                                       │    │
│  │  ┌──────────────┐                                             │    │                                                                                                           
│  │  │  Ad Request   │  From player: user_id, video_id,           │    │                                                                                                            
│  │  │  API          │  device_type, geo, video_category,         │    │                                                                                                            
│  │  │               │  ad_position (pre/mid/post-roll)           │    │                                                                                                           
│  │  └──────┬────────┘                                            │    │                                                                                                           
│  │         │                                                     │    │                                                                                                           
│  │         ▼                                                     │    │                                                                                                           
│  │  ┌──────────────┐   ┌───────────────────────┐                  │    │                                                                                                            
│  │  │ Targeting     │  │ User interest profile │                  │    │
│  │  │ Matcher       │──│ (from feature store)  │                  │    │                                                                                                         
│  │  │               │  │                       │                  │    │                                                                                                         
│  │  │ Match user    │  │ Interests: technology,│                  │    │                                                                                                         
│  │  │ profile       │  │ gaming, cooking       │                  │    │                                                                                                         
│  │  │ against       │  │ Demographics: M, 25-34│                  │    │                                                                                                         
│  │  │ advertiser    │  │ Recent searches: ...  │                  │    │                                                                                                         
│  │  │ targeting     │  └──────────────────────┘                  │    │                                                                                                           
│  │  │ criteria      │                                            │    │                                                                                                           
│  │  │               │  → ~1,000 eligible ad campaigns            │    │                                                                                                           
│  │  └──────┬────────┘                                            │    │                                                                                                           
│  │         │                                                     │    │                                                                                                           
│  │         ▼                                                     │    │                                                                                                           
│  │  ┌──────────────┐                                             │    │
│  │  │  Ad Auction   │  Real-time auction among eligible ads:     │    │                                                                                                            
│  │  │               │                                            │    │                                                                                                           
│  │  │  Each ad bids:│  bid = advertiser_max_bid                  │    │                                                                                                           
│  │  │               │      × P(click | user, video, ad)         │    │                                                                                                            
│  │  │  eCPM = bid   │      × quality_score                      │    │                                                                                                            
│  │  │    × P(click) │      × pacing_factor                      │    │                                                                                                            
│  │  │    × quality  │                                           │    │                                                                                                           
│  │  │               │  Pacing: if advertiser spent 80% of daily │    │                                                                                                            
│  │  │  Winner:      │  budget and it's only noon, reduce their  │    │                                                                                                            
│  │  │  highest eCPM │  pacing_factor to spread budget over the  │    │                                                                                                            
│  │  │               │  remaining hours.                          │    │                                                                                                           
│  │  │  Price: second│                                            │    │                                                                                                           
│  │  │  -price (pay  │  P(click) predicted by ML model (similar   │    │                                                                                                            
│  │  │  bid of 2nd   │  architecture to rec ranking model).       │    │                                                                                                           
│  │  │  place + $0.01│                                            │    │                                                                                                           
│  │  └──────┬────────┘                                            │    │                                                                                                           
│  │         │                                                     │    │                                                                                                           
│  │         ▼                                                     │    │
│  │  ┌──────────────┐                                             │    │                                                                                                           
│  │  │  Brand Safety │  Check: is this ad appropriate for this    │    │                                                                                                            
│  │  │  Filter       │  video's content?                          │    │                                                                                                           
│  │  │               │  Advertiser blocklists × video content     │    │                                                                                                           
│  │  │               │  classification. Filter unsafe pairings.   │    │                                                                                                           
│  │  └──────┬────────┘                                            │    │                                                                                                           
│  │         │                                                     │    │                                                                                                           
│  │         ▼                                                     │    │                                                                                                           
│  │  Return winning ad creative URL to player.                    │    │
│  │  Player fetches ad video from ad CDN and plays it.            │    │                                                                                                           
│  │  Total latency: < 200ms.                                      │    │                                                                                                           
│  └──────────────────────────────────────────────────────────────┘     │                                                                                                            
│                                                                       │                                                                                                           
│  AD EVENT TRACKING (for billing):                                     │                                                                                                           
│                                                                       │                                                                                                           
│  Player reports: impression (ad shown), view (ad watched >30s or     │
│  to completion), click (user clicked), skip (user hit "skip ad").    │                                                                                                            
│  → Kafka → batch verification (fraud filtering) → billing pipeline.  │                                                                                                            
│  Advertisers are charged based on verified events only.               │                                                                                                           
└──────────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---                                                                                                                                                                                 
11. Content Moderation & Copyright

┌──────────────────────────────────────────────────────────────────────┐
│  CONTENT MODERATION                                                   │
│                                                                       │                                                                                                           
│  AUTOMATED (at upload time, before publishing):                      │
│                                                                       │                                                                                                           
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  ML classifiers (run in parallel during processing):          │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  • Visual: nudity, violence, gore, self-harm                  │    │                                                                                                           
│  │    (frame-level CNN, sampled every 2 seconds)                 │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  • Audio: hate speech, extremist content                      │    │                                                                                                           
│  │    (speech-to-text → NLP classifier)                          │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  • Text: title, description, tags                             │    │
│  │    (spam, misleading, dangerous health claims)                │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  • Copyright: Content ID fingerprint matching                 │    │                                                                                                           
│  │    (audio fingerprint + visual fingerprint matched             │    │                                                                                                          
│  │     against database of 100M+ reference files from            │    │                                                                                                           
│  │     copyright holders)                                        │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  Decision matrix:                                              │    │                                                                                                          
│  │    Confidence > 95% violation → auto-block                    │    │                                                                                                           
│  │    Confidence 70-95%          → queue for human review        │    │                                                                                                           
│  │    Confidence < 70%           → publish, monitor              │    │                                                                                                           
│  └──────────────────────────────────────────────────────────────┘    │                                                                                                            
│                                                                       │                                                                                                           
│  HUMAN REVIEW:                                                        │                                                                                                           
│                                                                       │
│  ~50K videos/day flagged for human review.                           │                                                                                                            
│  Review queue prioritized by: reach (subscriber count),              │
│  severity (violence > spam), and confidence (lower confidence =      │                                                                                                            
│  higher priority for human judgment).                                 │                                                                                                           
│                                                                       │                                                                                                           
│  POST-PUBLISH MONITORING:                                             │                                                                                                           
│                                                                       │                                                                                                           
│  Videos that passed initial moderation are re-evaluated if:          │
│    • User reports spike (>N reports in M minutes)                    │                                                                                                            
│    • View velocity is abnormally high (potential viral harmful)      │                                                                                                            
│    • Comment sentiment analysis detects alarm patterns               │                                                                                                            
│                                                                       │                                                                                                           
│  CONTENT ID (Copyright):                                              │                                                                                                           
│                                                                       │                                                                                                           
│  ┌──────────────────────────────────────────────────────────────┐    │                                                                                                            
│  │  Copyright holder uploads reference file                      │    │
│  │    → System generates audio + visual fingerprint              │    │                                                                                                           
│  │    → Stored in fingerprint database                           │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  Every uploaded video's fingerprint is compared against DB.   │    │                                                                                                           
│  │  Match found → copyright holder chooses policy:               │    │                                                                                                           
│  │    • Block: video is removed                                  │    │                                                                                                           
│  │    • Monetize: video stays, ad revenue goes to holder         │    │                                                                                                           
│  │    • Track: video stays, holder gets view analytics           │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  Scale: 500 hours uploaded/min × fingerprint check            │    │                                                                                                           
│  │         against 100M+ references = massive compute.           │    │                                                                                                           
│  │         Uses locality-sensitive hashing (LSH) for fast        │    │                                                                                                           
│  │         approximate nearest-neighbor search on fingerprints.  │    │                                                                                                           
│  └──────────────────────────────────────────────────────────────┘    │                                                                                                            
└──────────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---                                                                                                                                                                                 
12. Search

┌──────────────────────────────────────────────────────────────────────┐
│  SEARCH ARCHITECTURE                                                  │
│                                                                       │
│  200K queries/sec. 800M videos indexed.                              │
│                                                                       │                                                                                                           
│  Query: "how to change bike tire"                                    │
│       │                                                               │                                                                                                           
│       ▼                                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────────────────────┐   │                                                                                                             
│  │ Query    │  │ Query    │  │ Retrieval                         │   │
│  │ Under-   │→ │ Expansion│→ │                                   │   │                                                                                                            
│  │ standing │  │          │  │ Phase 1: keyword match (inverted  │   │                                                                                                            
│  │          │  │ "bike"   │  │   index, BM25). 10K candidates.  │   │                                                                                                             
│  │ intent:  │  │ → "cycle"│  │                                   │   │                                                                                                            
│  │ tutorial │  │ → "bicycl│  │ Phase 2: semantic match (embed-   │   │                                                                                                            
│  │          │  │   e"     │  │   ding similarity, ANN search).  │   │                                                                                                             
│  │ entity:  │  │          │  │   5K more candidates.             │   │                                                                                                            
│  │ bicycle  │  │ spell    │  │                                   │   │                                                                                                            
│  │ tire     │  │ correct  │  │ Union + dedup: ~12K candidates.  │   │                                                                                                             
│  └──────────┘  └──────────┘  └──────────┬───────────────────────┘   │                                                                                                             
│                                          │                           │
│                                          ▼                           │                                                                                                            
│                              ┌──────────────────────┐               │                                                                                                             
│                              │ Ranking               │               │
│                              │                       │               │                                                                                                            
│                              │ Features:             │               │                                                                                                            
│                              │ • Text relevance      │               │
│                              │ • Video quality (CTR, │               │                                                                                                            
│                              │   watch time, likes)  │               │
│                              │ • Channel authority    │               │                                                                                                           
│                              │ • Freshness            │               │
│                              │ • Personalization      │               │                                                                                                           
│                              │   (user's language,    │               │                                                                                                           
│                              │    watch history)      │               │
│                              │ • Engagement velocity  │               │                                                                                                           
│                              │   (trending factor)    │               │                                                                                                           
│                              └──────────┬───────────┘               │
│                                          │                           │                                                                                                            
│                                          ▼                           │
│                              Top 20 results returned.                │                                                                                                            
│                              Latency: < 100ms.                       │                                                                                                            
│                                                                       │
│  INDEX UPDATE:                                                        │                                                                                                           
│    New video published → event → index builder adds to inverted     │                                                                                                             
│    index + embedding index. Searchable within 30 seconds.            │                                                                                                            
│                                                                       │                                                                                                           
│  AUTOCOMPLETE:                                                        │                                                                                                           
│    Separate service. Trie-based prefix matching + popularity.        │                                                                                                            
│    Updated hourly from query logs. Latency: < 20ms.                  │                                                                                                            
│    Personalized: user's own search history boosted.                  │                                                                                                            
└──────────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---                                                                                                                                                                                 
13. Engagement Services (Likes, Comments, Subscriptions)

┌──────────────────────────────────────────────────────────────────────┐
│  ENGAGEMENT SERVICES                                                  │                                                                                                           
│                                                                       │                                                                                                           
│  LIKES (200K/sec):                                                    │
│  ┌──────────────────────────────────────────────────────────────┐    │                                                                                                            
│  │  Like API → Kafka → Counter Service (Redis + Cassandra)      │    │                                                                                                           
│  │                                                              │    │                                                                                                          
│  │  Redis: real-time approximate count (INCRBY, sharded).       │    │                                                                                                           
│  │  Cassandra: durable per-user like record                      │    │                                                                                                           
│  │    (user_id, video_id, timestamp)                             │    │                                                                                                           
│  │    Used for: "did I already like this?" check (fast lookup).  │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  Idempotency: user likes same video twice → only one record. │    │                                                                                                            
│  │  Unlike: delete record + DECRBY counter.                      │    │                                                                                                           
│  └──────────────────────────────────────────────────────────────┘    │                                                                                                            
│                                                                       │                                                                                                           
│  COMMENTS (50K/sec):                                                  │                                                                                                           
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │  Write:                                                        │    │                                                                                                          
│  │    Comment API → Spam/toxicity ML check (10ms)                │    │                                                                                                           
│  │      → Vitess (sharded MySQL) by video_id                    │    │                                                                                                            
│  │      → Publish "comment.created" event                        │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  Read (loading comments for a video):                         │    │                                                                                                           
│  │    Top-level comments: sorted by "top" (engagement score)     │    │                                                                                                           
│  │      or "newest." Paginated (20 per page).                    │    │                                                                                                           
│  │    Replies: loaded on-demand when "View N replies" clicked.   │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  Schema:                                                       │    │                                                                                                          
│  │    PK: (video_id, comment_id)                                 │    │                                                                                                           
│  │    parent_comment_id: NULL (top-level) or FK (reply)          │    │                                                                                                           
│  │    user_id, text, created_at, like_count, reply_count         │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  Sharded by video_id: all comments for a video on one shard. │    │                                                                                                            
│  │  Hot videos (millions of comments) → dedicated shard.         │    │                                                                                                           
│  │                                                                │    │                                                                                                          
│  │  Moderation: auto-hide comments flagged by ML. Creator can    │    │                                                                                                           
│  │    set keyword filters. "Held for review" queue.              │    │                                                                                                           
│  └──────────────────────────────────────────────────────────────┘    │                                                                                                            
│                                                                       │                                                                                                           
│  SUBSCRIPTIONS:                                                       │                                                                                                           
│  ┌──────────────────────────────────────────────────────────────┐     │                                                                                                            
│  │  Subscribe: write (user_id, channel_id, subscribed_at)        │    │
│  │    to Bigtable. Increment channel's subscriber count          │    │                                                                                                           
│  │    (Redis counter, reconciled with Cassandra).                │    │                                                                                                           
│  │                                                               │    │                                                                                                          
│  │  Subscription feed ("Subscriptions" tab):                     │    │                                                                                                           
│  │    Fan-out on write vs. fan-out on read:                      │    │                                                                                                           
│  │                                                               │    │                                                                                                          
│  │    Small creator (< 100K subs):                               │    │                                                                                                           
│  │      Fan-out on WRITE. When creator uploads,                  │    │                                                                                                           
│  │      push to each subscriber's feed (Bigtable).               │    │                                                                                                            
│  │      100K writes, but only once per upload.                   │    │                                                                                                           
│  │                                                               │    │                                                                                                          
│  │    Large creator (> 1M subs):                                 │    │                                                                                                           
│  │      Fan-out on READ. Subscriber's feed is assembled          │    │                                                                                                           
│  │      at read time by querying "latest videos from             │    │                                                                                                           
│  │      channels I subscribe to."                                │    │                                                                                                           
│  │      Avoids 100M writes when a mega-creator uploads.          │    │                                                                                                           
│  │                                                               │    │                                                                                                          
│  │    Hybrid threshold: ~500K subscribers.                       │    │                                                                                                           
│  │                                                               │    │                                                                                                          
│  │  Notification (bell icon):                                    │    │                                                                                                           
│  │    "All notifications" enabled → push notification on upload. │    │                                                                                                           
│  │    For 100M-subscriber channels: notification is batched,     │    │                                                                                                           
│  │    staggered over minutes (not 100M pushes simultaneously).   │    │                                                                                                           
│  └──────────────────────────────────────────────────────────────┘    │                                                                                                            
└──────────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---             
14. Reliability & Fault Tolerance

What Fails and How We Handle It

┌──────────────────────────────────────────────────────────────────────┐
│  FAILURE MODE          IMPACT               MITIGATION               │                                                                                                            
│  ──────────────────── ──────────────────── ───────────────────────── │                                                                                                              
│                                                                      │                                                                                                           
│  CDN edge PoP down    Users in that region   Anycast DNS routes to   │                                                                                                            
│                       see higher latency     next-nearest PoP. Auto  │                                                                                                            
│                       (hitting origin        within 30s. No manual   │                                                                                                            
│                       shield instead)        intervention.           │                                                                                                           
│                                                                      │                                                                                                           
│  Origin shield down   Cache misses go to     Multiple shields per    │                                                                                                            
│                       origin directly.       region. Traffic shifts  │                                                                                                            
│                       Higher origin load.    to surviving shields.   │                                                                                                            
│                                                                      │                                                                                                           
│  Blob storage         Videos unplayable if   3x replication (hot) or │                                                                                                             
│  node failure         all replicas lost.     14-shard erasure coding │                                                                                                            
│                       Unlikely with 3x       (cold). Tolerates       │                                                                                                             
│                       replication.           multiple shard losses.  │                                                                                                            
│                                                                      │                                                                                                           
│  Transcoding cluster  New uploads delayed.   Queue absorbs burst.    │                                                                                                             
│  partial failure      Not an outage —        Priority queue ensures  │                                                                                                            
│                       existing videos        high-value videos are   │                                                                                                            
│                       unaffected.            transcoded first.       │                                                                                                            
│                                                                      │                                                                                                           
│  Recommendation       Home page shows stale  Pre-computed recs in    │                                                                                                            
│  engine down          or generic recs.       cache (Redis). Updated  │                                                                                                            
│                       Reduced engagement,    every 6 hours. Fallback │                                                                                                            
│                       not a hard failure.    to trending/popular.    │                                                                                                            
│                                                                      │                                                                                                           
│  Search cluster       Search returns errors  Multiple search cluster │                                                                                                            
│  down                 or degraded results.   replicas per region.    │                                                                                                            
│                       Autocomplete may fail. Circuit breaker returns │                                                                                                            
│                                              cached results for top  │                                                                                                            
│                                              1000 queries.           │                                                                                                            
│                                                                      │
│  Metadata DB (Vitess) Video pages can't      Vitess has multi-AZ     │                                                                                                             
│  primary down         load titles/           replicas. Automatic     │                                                                                                            
│                       descriptions. Play     primary failover in    │                                                                                                             
│                       still works (video     <30s. Reads continue   │                                                                                                             
│                       blobs separate).       from replicas during   │`                                                                                                             
│                                              failover.               │                                                                                                            
│                                                                      │                                                                                                           
│  Kafka cluster down   Events stop flowing.   Multi-AZ replicas.     │                                                                                                             
│                       View counts stale.     Producers buffer        │                                                                                                            
│                       Recs not updating.     locally (disk) for up  │                                                                                                             
│                       Uploads queued.        to 1 hour.             │                                                                                                             
│                                                                       │                                                                                                           
│  Ad serving down      No ads shown. Videos   Fallback: show video   │                                                                                                             
│                       play without ads.      without ads. Revenue    │                                                                                                            
│                       Revenue loss, not      loss, not user-visible  │                                                                                                            
│                       user-facing outage.    degradation.            │                                                                                                            
│                                                                       │
│  Redis cluster down   Cache miss → all       Redis Cluster with     │                                                                                                             
│                       reads hit DB.          replicas. If totally    │                                                                                                            
│                       Massive DB load.       down: `circuit breaker`   │                                                                                                            
│                       Potential cascade.     → serve degraded        │                                                                                                            
│                                              (no like counts, approx │                                                                                                            
│                                              view counts from batch).│                                                                                                            
│                                                                       │                                                                                                           
│  Entire region down   All traffic rerouted   Active-active multi-   │                                                                                                             
│                       to other regions.      region. DNS failover   │                                                                                                             
│                       Higher latency for     in <60s. All data      │                                                                                                             
│                       affected users.        replicated. Capacity   │                                                                                                             
│                                              reserved in each region │                                                                                                            
│                                              to absorb a neighbor's  │                                                                                                            
│                                              traffic (N+1 capacity). │                                                                                                            
└──────────────────────────────────────────────────────────────────────┘

Graceful Degradation Hierarchy

The watch page is composed of ~15 independent components.                                                                                                                           
Each has its own failure boundary.

┌──────────────────────────────────────────────────────────────────┐                                                                                                                
│  WATCH PAGE COMPONENT       CRITICALITY    IF UNAVAILABLE        │
│  ────────────────────────── ────────────── ───────────────────── │                                                                                                                
│                                                                   │
│  Video player + segments     CRITICAL       Page is broken.       │                                                                                                               
│                                             Return 500.           │                                                                                                               
│                                                                   │                                                                                                               
│  Video manifest (DASH/HLS)   CRITICAL       Can't play. 500.     │                                                                                                                
│                                                                   │                                                                                                               
│  Video title / description   HIGH           Show placeholder.     │
│                                             "Loading..."          │                                                                                                               
│                                                                   │
│  Channel name / avatar       HIGH           Show generic avatar.  │                                                                                                               
│                                                                   │                                                                                                               
│  View count                  MEDIUM         Hide or show "—"     │                                                                                                                
│                                                                   │                                                                                                               
│  Like / dislike buttons      MEDIUM         Disable buttons.      │
│                                             "Temporarily          │                                                                                                               
│                                             unavailable."         │
│                                                                   │                                                                                                               
│  Comments                    LOW            "Comments are         │
│                                             unavailable."         │                                                                                                               
│                                                                   │
│  Recommended videos          LOW            Show trending or      │                                                                                                               
│  (sidebar)                                  popular instead.      │
│                                                                   │                                                                                                               
│  Ad pre-roll                 LOW            Skip ad. Play video   │
│                              (for UX)       immediately. Revenue  │                                                                                                               
│                                             loss, not UX loss.    │                                                                                                               
│                                                                   │                                                                                                               
│  Subscribe button            LOW            Disable button.       │                                                                                                               
│                                                                   │                                                                                                               
│  Captions                    LOW            No captions. Video    │
│                                             plays without them.   │                                                                                                               
│                                                                   │                                                                                                               
│  Description links           LOW            Show plain text, no   │
│                                             rich previews.        │                                                                                                               
│                                                                   │                                                                                                               
│  End screen / cards          LOW            Don't show. Video     │
│                                             plays to end.         │                                                                                                               
│                                                                   │
│  Only the video blob + manifest are truly critical.               │                                                                                                               
│  Everything else degrades independently.                          │                                                                                                               
└──────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---             
15. Multi-Region Strategy

┌────────────────────────────────────────────────────────────────────────┐
│  DATA CLASSIFICATION                                                    │                                                                                                         
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │                                                                                                          
│  │  GLOBAL (replicated everywhere)                                   │  │                                                                                                         
│  │                                                                    │  │
│  │  • Video blobs (replicated to 2+ regions per video).             │  │                                                                                                          
│  │    Popular videos: all regions. Cold videos: 2 regions.          │  │                                                                                                          
│  │  • Video metadata (title, description, channel).                 │  │                                                                                                          
│  │    Replicated via async Kafka cross-region replication.          │  │                                                                                                          
│  │  • User accounts (core auth data).                               │  │                                                                                                          
│  │  • Channel data, subscription graph.                             │  │                                                                                                          
│  │  • Search index (rebuilt per-region from replicated metadata).   │  │                                                                                                          
│  │                                                                    │  │                                                                                                        
│  │  Consistency: eventual (100-500ms cross-region lag).              │  │                                                                                                         
│  │  A video uploaded in US may not be searchable in EU for ~1s.     │  │                                                                                                          
│  │  Acceptable tradeoff for global availability.                     │  │                                                                                                         
│  └──────────────────────────────────────────────────────────────────┘  │                                                                                                          
│                                                                         │                                                                                                         
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  REGIONAL (stays local)                                           │  │                                                                                                         
│  │                                                                    │  │                                                                                                        
│  │  • Watch history (written to local region, async-replicated      │  │
│  │    for cross-region recommendation quality).                      │  │                                                                                                         
│  │  • Live stream ingest (processed in region closest to creator).  │  │                                                                                                          
│  │  • Ad auction state (ad budgets, pacing — per-region).           │  │                                                                                                          
│  │  • Recommendation feature store (per-region, real-time signals). │  │                                                                                                          
│  │  • CDN cache state (per-PoP, not replicated).                    │  │                                                                                                          
│  └──────────────────────────────────────────────────────────────────┘  │                                                                                                          
│                                                                         │                                                                                                         
│  ┌──────────────────────────────────────────────────────────────────┐  │                                                                                                          
│  │  REGION-SPECIFIC (authored differently per region)                │  │                                                                                                         
│  │                                                                    │  │
│  │  • Content restrictions (blocked in country X per law).          │  │                                                                                                          
│  │  • Age restriction policies (country-specific).                  │  │                                                                                                          
│  │  • Language-specific trending lists.                              │  │                                                                                                         
│  │  • Local payment methods for Premium subscriptions.              │  │                                                                                                          
│  └──────────────────────────────────────────────────────────────────┘  │                                                                                                          
│                                                                         │                                                                                                         
│  VIDEO BLOB REPLICATION STRATEGY:                                       │                                                                                                         
│                                                                         │                                                                                                         
│  New upload:                                                            │
│    1. Transcoded in the region closest to the creator.                 │                                                                                                          
│    2. Immediately available in that region.                             │
│    3. Async replicated to 1-2 other regions within 1 hour.            │                                                                                                           
│    4. If the video goes viral (>100K views in first hour):             │                                                                                                          
│       → emergency replication to ALL regions within 15 minutes.       │                                                                                                           
│    5. Cold videos (< 10 views/day after 30 days):                      │                                                                                                          
│       → reduce to 2-region replication + erasure coding.              │                                                                                                           
│       → save storage by not keeping copies everywhere.                 │                                                                                                          
│                                                                         │                                                                                                         
│  Playback from a region that doesn't have the video:                   │                                                                                                          
│    CDN miss → origin in that region doesn't have it                    │                                                                                                          
│    → cross-region fetch from the region that does                      │                                                                                                          
│    → 100-200ms extra latency on first request                          │                                                                                                          
│    → cached locally after first fetch for subsequent viewers           │                                                                                                          
│                                                                         │                                                                                                         
│  FAILOVER:                                                              │                                                                                                         
│    Region goes down:                                                    │                                                                                                         
│    • DNS redirects to nearest healthy region (60s)                     │
│    • Video blobs: served from other region's copy                      │                                                                                                          
│    • Metadata: available from replicated store                         │                                                                                                          
│    • Watch history: recent events in the failed region are lost        │                                                                                                          
│      (Kafka hasn't replicated them yet). Acceptable — recs degrade    │                                                                                                           
│      slightly, not a data loss the user notices.                       │                                                                                                          
│    • Uploads: creators retry to a different region's ingest endpoint. │                                                                                                           
│    • Live streams: creator's encoder must reconnect to new ingest     │                                                                                                           
│      endpoint. ~10s disruption for live viewers.                       │                                                                                                          
│                                                                         │                                                                                                         
│  RTO: < 90 seconds (DNS + reconnection)                                │                                                                                                          
│  RPO: < 1 second for metadata, < 5 seconds for view events            │                                                                                                           
└────────────────────────────────────────────────────────────────────────┘
                                                                                                                                                                                      
---                                                                                                                                                                                 
16. Complete Request Flow: User Opens YouTube Home Page

1. DNS RESOLUTION
   youtube.com → Anycast DNS → nearest CDN PoP IP.

2. TLS + HTTP/2 CONNECTION                                                                                                                                                          
   Client establishes TLS 1.3 to CDN edge.                                                                                                                                          
   Connection reused for all subsequent requests (multiplexed).

3. PAGE SHELL (static)                                                                                                                                                              
   GET / → CDN serves cached HTML shell + JS bundle (1ms).
   JS executes, makes API calls for dynamic content.

4. RECOMMENDATIONS API (the main payload)                                                                                                                                           
   GET /api/feed?user_id=U123&device=mobile&page=1

   CDN: not cacheable (personalized). Forward to origin.                                                                                                                            
   API Gateway: validate JWT (2ms). Route to user's region.

   Recommendation Service (200ms budget):                                                                                                                                           
   a. Read user features from Redis feature store (2ms)                                                                                                                           
   → watch history, interests, device, time-of-day                                                                                                                             
   b. Candidate generation — parallel:                                                                                                                                            
   → Collaborative filtering: 3,000 candidates (20ms)                                                                                                                          
   → Content-based: 2,000 candidates (15ms)                                                                                                                                    
   → Subscriptions: 2,000 candidates (10ms)                                                                                                                                    
   → Trending: 1,000 candidates (5ms)
   → Exploration: 1,000 candidates (5ms)                                                                                                                                       
   c. Dedup: 10,000 unique candidates (2ms)                                                                                                                                       
   d. Feature enrichment: fetch video features for 10K videos                                                                                                                     
   from Bigtable (20ms, batch)                                                                                                                                                 
   e. Ranking model: GPU inference, score all 10K (80ms)                                                                                                                          
   f. Re-ranking + policy: diversity, freshness, safety (5ms)                                                                                                                     
   g. Top 50 video IDs selected.

   Video metadata batch fetch:                                                                                                                                                      
   50 video IDs → Vitess (MySQL) → titles, channel names,
   durations, thumbnail URLs, view counts. (10ms, cache hit)

   Response: 50 video cards with all display data.                                                                                                                                  
   Total API latency: ~250ms.

5. THUMBNAIL LOADING
   50 thumbnails requested from CDN.                                                                                                                                                
   All cached (thumbnails are among the most-cached objects).                                                                                                                       
   Loaded in parallel. Visible within 300ms of page load.

6. USER SCROLLS → INFINITE SCROLL                                                                                                                                                   
   Client requests page 2 (next 50 recs) when user nears bottom.                                                                                                                    
   Same pipeline, but candidate generation uses "already shown"                                                                                                                     
   as a negative filter.

7. USER CLICKS A VIDEO                                                                                                                                                              
   → Watch page loads (see playback flow in Section 6)
   → "video.click" event → Kafka → updates user features in real-time                                                                                                               
   → Next recommendation request (when user returns to home page)                                                                                                                   
   incorporates the click signal with < 1 second delay.

Total home page load: ~400ms (shell + API + thumbnails).                                                                                                                            
Time to first meaningful paint: ~200ms (shell + skeleton UI).                                                                                                                       
Time to fully interactive: ~600ms.
                                                                                                                                                                                      
---                                                                                                                                                                                 
17. Summary: Architecture Diagram

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │                                                                                                    
│                          ┌──────────┐                                       │
│                          │  Client  │                                       │                                                                                                     
│                          │(browser /│                                       │                                                                                                     
│                          │   app)   │                                       │
│                          └─────┬────┘                                       │                                                                                                     
│                                │                                             │
│                          ┌─────▼─────┐                                      │                                                                                                     
│                          │  CDN Edge │ ←── video segments, thumbnails,      │
│                          │  (300+    │     static assets (95% served here)  │                                                                                                     
│                          │   PoPs)   │                                      │                                                                                                     
│                          └─────┬─────┘                                      │                                                                                                     
│                                │ API calls, cache misses                    │                                                                                                     
│                          ┌─────▼─────┐                                      │                                                                                                     
│                          │  API      │                                      │                                                                                                     
│                          │  Gateway  │ ←── auth, rate limit, routing       │                                                                                                      
│                          └─────┬─────┘                                      │                                                                                                     
│                                │                                             │
│          ┌─────────────────────┼─────────────────────┐                      │                                                                                                     
│          ▼                     ▼                     ▼                      │
│  ┌───────────────┐   ┌────────────────┐   ┌──────────────────┐            │                                                                                                       
│  │ WATCH PATH    │   │ DISCOVERY PATH │   │ UPLOAD PATH      │            │
│  │               │   │                │   │                  │            │                                                                                                       
│  │ • Video       │   │ • Rec Engine   │   │ • Upload Service │            │                                                                                                       
│  │   Serving     │   │ • Search       │   │ • Transcoding    │            │                                                                                                       
│  │ • Player      │   │ • Trending     │   │   Pipeline       │            │                                                                                                       
│  │   Config      │   │ • Subscription │   │ • Content ID     │            │                                                                                                       
│  │ • Ad Serving  │   │   Feed         │   │ • Moderation     │            │                                                                                                       
│  │ • View Count  │   │ • Ad Matching  │   │ • Publishing     │            │                                                                                                       
│  └───────┬───────┘   └────────┬───────┘   └────────┬─────────┘            │                                                                                                       
│          │                    │                     │                       │                                                                                                     
│          └────────────────────┼─────────────────────┘                       │                                                                                                     
│                               │                                             │                                                                                                     
│                    ┌──────────▼───────────┐                                 │
│                    │   EVENT BACKBONE     │                                 │                                                                                                     
│                    │   (Kafka)            │                                 │                                                                                                     
│                    │                      │                                 │
│                    │ view.events          │                                 │                                                                                                     
│                    │ video.published      │                                 │
│                    │ comment.created      │                                 │                                                                                                     
│                    │ user.subscribed      │                                 │                                                                                                     
│                    │ ad.impression        │                                 │
│                    └──────────┬───────────┘                                 │                                                                                                     
│                               │                                             │                                                                                                     
│          ┌────────────────────┼──────────────────────┐                      │
│          ▼                    ▼                      ▼                      │                                                                                                     
│  ┌───────────────┐  ┌────────────────┐  ┌───────────────────┐             │                                                                                                       
│  │ ENGAGEMENT    │  │ ANALYTICS      │  │ ML PLATFORM       │             │                                                                                                       
│  │               │  │                │  │                   │             │                                                                                                       
│  │ • Likes       │  │ • Creator      │  │ • Model Training  │             │                                                                                                       
│  │ • Comments    │  │   Studio       │  │   (batch, daily)  │             │                                                                                                       
│  │ • Subscriptions│ │   Analytics   │  │ • Feature Store   │             │                                                                                                        
│  │ • Notifications│ │ • Revenue     │  │   (real-time)     │             │                                                                                                        
│  │ • Live Chat   │  │   Reporting   │  │ • Embedding Index │             │                                                                                                        
│  │               │  │ • A/B Testing │  │   (FAISS/ScaNN)   │             │                                                                                                        
│  └───────────────┘  └───────────────┘  └───────────────────┘             │                                                                                                       
│                                                                             │                                                                                                    
│  ┌──────────────────────────────────────────────────────────────────────┐   │                                                                                                     
│  │  DATA LAYER                                                          │   │                                                                                                     
│  │                                                                      │   │                                                                                                    
│  │  Blob Storage     Vitess       Bigtable     Redis      Cassandra    │   │
│  │  (video files)    (metadata,   (user        (cache,    (view counts,│   │                                                                                                      
│  │  5+ exabytes      comments)    profiles,    features)  likes,       │   │                                                                                                      
│  │                                subs)                   durable      │   │                                                                                                      
│  │                                                        counters)    │   │                                                                                                      
│  └─────────────────────────────────────────────────────────────────────┘   │                                                                                                     
│                                                                             │                                                                                                    
└─────────────────────────────────────────────────────────────────────────────┘
  
---                                                                                                                                                                                 
Key Architectural Decisions Summarized

┌─────────────────────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                          Decision                           │                                                     Reason                                                      │   
├─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ 3-tier CDN (edge → shield → origin)                         │ 95% cache hit rate. Origin sees <5% of traffic. Without this, origin would need to serve 1 Pbps.                │
├─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Segment-based video storage (not monolithic files)          │ Seeking = fetch one segment. Caching = per-segment. ABR = switch quality per-segment.                           │   
├─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   
│ Erasure coding for cold storage                             │ 80% of videos are cold. Erasure coding (1.4x) vs. replication (3x) saves exabytes.                              │   
├─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   
│ Progressive transcoding (H.264 first, AV1 deferred)         │ Video playable in 30 seconds. AV1 saves bandwidth but is CPU-heavy — only worth it for popular videos.          │
├─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   
│ Two-path view counting (real-time approx + batch accurate)  │ 500K events/sec can't go through a durable counter. Redis approximation for display, batch for billing.         │
├─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤   
│ Hybrid fan-out for subscriptions (write for small, read for │ Fan-out-on-write for a 100M-sub channel = 100M writes per upload. Impractical. Read-time assembly for           │
│  large)                                                     │ mega-creators.                                                                                                  │   
├─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Multi-objective recommendation ranking                      │ Optimizing for clicks alone → clickbait. Combined metric (watch time × satisfaction - regret) aligns with       │   
│                                                             │ long-term engagement.                                                                                           │   
├─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Active-active multi-region with popularity-based            │ Popular videos everywhere. Cold videos in 2 regions. Saves exabytes while maintaining global availability.      │   
│ replication                                                 │                                                                                                                 │   
├─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Ads degrade to "no ad" (not "no video")                     │ Ad system failure = revenue loss, not user experience loss. Video plays immediately. Never block a video for an │   
│                                                             │  ad failure.                                                                                                    │   
├─────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ DAG-based processing pipeline for uploads                   │ Parallel transcoding across resolutions/codecs. Independent retry per task. One slow transcode doesn't block    │   
│                                                             │ others.                                                                        