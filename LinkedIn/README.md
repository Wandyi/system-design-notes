# LinkedIn — Staff Software Engineer Interview Prep

This directory is your end-to-end prep kit for the LinkedIn Staff Software Engineer loop. It is intentionally exhaustive — read what you need, skim what you don't. Every file is self-contained so you can study them out of order.

> **Note on the directory name.** The folder is `Linux/` because that was the agreed name when this prep was kicked off. The content here is for **LinkedIn** (the Microsoft-owned professional network), not the Linux kernel. The two are not related.

## How to use this directory

- **6+ weeks out**: read `01` → `04` to internalize the company, products, engineering culture, and reference architecture. Then start one case study a day.
- **3–4 weeks out**: do the system-design files (`05`–`14`). Re-draw each design on a whiteboard from scratch by the end.
- **1–2 weeks out**: coding patterns (`19`), behavioral (`21`), and the cheat sheets (`22`).
- **Last 48 hours**: re-read `22-quick-reference-cheatsheets.md`, the case-study you're most worried about, and your STAR stories.

## File map

| # | File | What it covers |
|---|------|----------------|
| 01 | `01-company-and-products.md` | LinkedIn the business, product surfaces, monetization, recent strategy |
| 02 | `02-engineering-culture-and-stack.md` | Tech stack, eng culture, open-source projects, monorepo vs. multi-repo, internal platforms |
| 03 | `03-linkedin-macro-architecture.md` | High-level architecture: edge, services, data, ML, batch — how it all hangs together |
| 04 | `04-interview-process-and-loops.md` | The actual loop format, what each round tests, calibration, level expectations |
| 05 | `05-system-design-feed.md` | News Feed: fan-out, FollowFeed, ranking, deduplication |
| 06 | `06-system-design-search.md` | Galene, Bobcat, federated search, typeahead, vector search for AI |
| 07 | `07-system-design-messaging.md` | InMail, real-time chat, presence, read receipts, fanout |
| 08 | `08-system-design-notifications.md` | Air Traffic Controller (ATC), rate limiting, multi-channel delivery |
| 09 | `09-system-design-connections-graph.md` | Economic Graph, LIquid graph store, 2nd/3rd-degree queries |
| 10 | `10-system-design-jobs-and-recs.md` | Job postings, recommender systems, "Jobs You May Be Interested In" |
| 11 | `11-system-design-ads-platform.md` | LinkedIn Marketing Solutions, auction, budget pacing, attribution |
| 12 | `12-system-design-analytics-pipeline.md` | Tracking, Kafka ingestion, Pinot/Druid OLAP, Spark batch, member analytics |
| 13 | `13-system-design-realtime-and-live.md` | Real-time platform, LinkedIn Live, presence, typing indicators |
| 14 | `14-system-design-questions-bank.md` | 30+ real questions reported in interviews, ranked by frequency |
| 15 | `15-kafka-deep-dive.md` | Kafka — born at LinkedIn; internals, exactly-once, partitioning, MirrorMaker |
| 16 | `16-voldemort-espresso-ambry.md` | Voldemort (KV), Espresso (document), Ambry (blob) — their three homegrown stores |
| 17 | `17-pinot-and-samza.md` | Pinot (OLAP), Samza (stream processing), comparison vs. Druid/Flink |
| 18 | `18-venice-databus-brooklin.md` | Derived data serving (Venice), CDC pipes (Databus, Brooklin), feature stores |
| 19 | `19-restli-and-service-infra.md` | Rest.li, D2 service discovery, Linkerd's ancestor, GraphQL adoption |
| 20 | `20-distributed-systems-fundamentals.md` | CAP, consistency, consensus, replication, sharding, idempotency — staff depth |
| 21 | `21-coding-problems.md` | DSA patterns, LinkedIn-reported problems, Go-specific tips |
| 22 | `22-staff-engineer-topics.md` | Scope, tech leadership, what staff means at LinkedIn vs. other FAANGs |
| 23 | `23-behavioral-and-leadership.md` | STAR stories, LinkedIn's leadership competencies, common questions |
| 24 | `24-quick-reference-cheatsheets.md` | Last-48-hours flashcards: numbers, latencies, formulas, names |

## What "Staff" means at LinkedIn

LinkedIn uses the standard tech-industry ladder. Their levels (engineering IC track):

- L2 — Software Engineer (new grad)
- L3 — Software Engineer
- L4 — Senior Software Engineer
- L5 — **Staff Software Engineer** ← target
- L6 — Senior Staff Software Engineer
- L7 — Principal Staff Software Engineer
- L8 — Distinguished Engineer
- L9 — Fellow

**Staff (L5) at LinkedIn** is typically scoped at *team-of-teams* or *org* level. You're expected to:

- Own the technical direction of a system or a few coordinated systems.
- Mentor senior engineers, set up architectural review processes, write design docs that get cited.
- Influence outside your team: cross-org alignment, deprecation campaigns, RFCs.
- Take a vague business problem and reduce it to an engineering plan.
- Be a calibration anchor for hiring at the senior level.

It is **not** an "individual brilliance" role — output is measured through your team and adjacent teams. The interview loop reflects this: behavioral rounds carry as much weight as system design.

## A note on currency of information

LinkedIn's engineering blog is the canonical source. Wherever this prep references an internal system (Espresso, Pinot, Venice, etc.), the blog has the deepest write-up. Where the prep is dated or wrong, trust the blog, then the original Kafka/Pinot/Samza papers (LinkedIn open-sourced most of them and the academic papers are excellent).

Recent strategic shifts to be aware of (as of 2025–2026):

- Heavy investment in **generative-AI features** across the product (writing assistance, recruiter copilot, premium AI features). Expect questions on serving LLMs at LinkedIn scale, RAG over the Economic Graph, and ML-ops at member scale.
- **Move off the JVM in selected places** — some new services in Go and Rust, though Java + Scala remain dominant.
- **Continued consolidation** onto Azure (the Microsoft mandate), though LinkedIn still operates significant own-DC capacity.
- Several long-running **data-infrastructure migrations**: Espresso → next-gen document store conversations, Pinot continues to absorb workloads from Druid/Hive, Venice for derived-data serving has displaced a lot of Voldemort.

Don't bluff currency. If asked about something recent, say "I read the blog through about $DATE — happy to reason from first principles about anything after that." Staff candidates lose more points pretending than admitting.