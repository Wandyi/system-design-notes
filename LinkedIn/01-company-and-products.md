# 1 · LinkedIn — Company, Products, and Business Model

LinkedIn is the world's largest professional network: ~1B+ members across 200+ countries, ~67M+ companies represented, the most-trusted dataset on the global labor market. It is wholly owned by Microsoft (acquired December 2016 for $26.2B). Understanding the business is table stakes for the Staff interview — every system design will be more credible if you can tie a design decision back to a business surface.

## 1.1 Founding and short history

- **2002** — founded by Reid Hoffman, Allen Blue, Konstantin Guericke, Eric Ly, and Jean-Luc Vaillant in Hoffman's living room.
- **2003** — public launch (May 5). Slow start; took months to hit 4,500 users.
- **2008** — InMail, mobile apps, and the "People You May Know" (PYMK) recommender — PYMK alone drove a step-change in graph growth.
- **2011** — IPO on NYSE (ticker `LNKD`). Engineering team starts open-sourcing Kafka.
- **2013** — Influencers and SlideShare acquisition (later divested).
- **2015** — Lynda.com acquisition for $1.5B → became **LinkedIn Learning**.
- **2016** — Microsoft acquisition closes ($26.2B).
- **2019–2021** — Project Inversion phase 2; major data-infrastructure consolidations; Pinot becomes the default OLAP store.
- **2023–2024** — Generative-AI integration sweep: AI-assisted writing, premium AI features, Recruiter AI Copilot.
- **2025–2026** — Continued AI investment, content/creator tooling, increased video-first behaviors on Feed.

## 1.2 Products you must be able to discuss

Memorize this map. In system-design rounds, you will design *one* of these. In behavioral rounds, you will be asked which you'd want to own and why.

### Member-facing products

| Product | What it is | Backend hot-spots |
|---|---|---|
| **Profile** | Member identity record | Espresso (primary store), denormalized into Venice for feed/search |
| **News Feed** | Personalized activity stream | FollowFeed, Concourse, hybrid push/pull fanout, ML ranking |
| **Search** | People, company, content, jobs, learning search | Galene, Bobcat, vector search for AI surfaces |
| **Messaging / InMail** | 1:1 and group conversations, real-time | Real-time platform, Kafka, presence service |
| **Notifications** | Email + push + in-app | ATC (Air Traffic Controller), rate limiting, multi-channel |
| **Jobs** | Postings + applications + matching | Job ranking, "Jobs You May Be Interested In" model |
| **Learning** | Courses, on-demand video | Video pipeline, CDN, progress tracking |
| **Premium** | Subscription tier (Career, Business, Sales, Recruiter Lite) | Entitlements, billing, A/B test gating |
| **LinkedIn Live / Video** | Live broadcast + short-form video | Live ingest, transcoding, CDN, real-time chat |
| **Articles / Newsletters** | Long-form content + creator subs | Content service, recommendations, publisher analytics |
| **Polls, Events, Groups** | Engagement surfaces | Standard write-then-fanout to feed |

### Business-facing (commercial) products

| Product | What it is | Why it matters |
|---|---|---|
| **LinkedIn Recruiter** | The flagship enterprise product. Lets recruiters search and InMail candidates at scale. | Highest-revenue product. The "Hire" segment is ~⅔ of LinkedIn revenue. |
| **Sales Navigator** | Sales-team prospecting tool — lead lists, CRM sync, alerts. | "Sales Solutions" — fastest-growing commercial segment. |
| **LinkedIn Marketing Solutions (LMS)** | Ad platform: Sponsored Content, Message Ads, Dynamic Ads, Conversation Ads. | Real-time bidding, auction, budget pacing, attribution. |
| **LinkedIn Learning for Business** | Enterprise rollout of Learning. | Bulk seat licensing, SSO/SAML, learning paths. |
| **Talent Insights** | Workforce analytics for HR leadership. | OLAP on the Economic Graph (Pinot under the hood). |
| **Glint / Talent Hub** | Employee-engagement surveys & ATS. | Acquired (Glint 2018), partially integrated. |

### Revenue mix (publicly disclosed in Microsoft 10-Ks)

LinkedIn's revenue is split into four reporting segments. Approximate split (rounded, FY2024 directional):

- **Talent Solutions (Recruiter + jobs + Learning for Business)** — ~55–60% of revenue.
- **Marketing Solutions (Ads)** — ~20%.
- **Premium Subscriptions** — ~10–15%.
- **Sales Solutions (Sales Navigator)** — ~10%.

Implication for engineering: a feature that improves Recruiter conversion is worth far more than a same-effort feature on consumer Premium. Staff candidates who reason about *which* surface to invest a quarter of engineering capacity in will stand out.

## 1.3 The Economic Graph

This is the single most important phrase to internalize.

> The Economic Graph is LinkedIn's digital representation of the global economy: every member, company, job, skill, school, and the edges between them.

It is both a *product framing* (Jeff Weiner's vision: "create economic opportunity for every member of the global workforce") and an *engineering substrate* (a literal graph that backs PYMK, search, jobs, ads, and the ML feature store). Mentioning it correctly in interviews signals that you've done the homework.

Key nodes and edges:

- **Nodes**: Members, Companies, Jobs, Skills, Schools, Industries, Geographies, Titles.
- **Edges**: Connection (member↔member, with degree), Employment (member↔company, with title/dates), Education (member↔school), Skill-endorsement (member↔skill with endorsers), Follow (member→company / member→member), Open-to-work, Hiring, etc.

The whole graph is real-time and consistent enough that PYMK recomputes quickly, search reflects new connections, and Recruiter searches return the right counts.

## 1.4 The Microsoft relationship

LinkedIn operates **independently** of Microsoft — separate brand, separate culture, separate engineering org for the most part — but with deep integrations:

- **Azure migration** — LinkedIn has been migrating from its own data centers to Azure for years. Not complete; still uses a mix.
- **Microsoft Entra / Office integrations** — calendar integration, Outlook sign-in, Teams presence.
- **Copilot integrations** — LinkedIn data and Microsoft 365 Copilot share some signals.
- **Engineering tools** — some adoption of Azure DevOps, but LinkedIn's internal tooling (LIX, Inversion, etc.) is still dominant.

For the interview, treat LinkedIn as its own company. If asked about Microsoft, talk about the cloud migration as the most engineering-relevant artifact.

## 1.5 Engineering org (rough mental model)

The exact org changes constantly; below is a generalized view. Most candidates interview into one of these orgs:

- **Core Engineering / Foundations** — feeds, search, messaging, identity, profile.
- **Talent Solutions Engineering** — Recruiter, Jobs, Career.
- **Marketing Solutions Engineering** — Ads, analytics, attribution.
- **Sales Solutions Engineering** — Sales Navigator.
- **Learning Engineering** — Learning, video.
- **Trust Engineering** — anti-abuse, integrity, content moderation, anti-fraud.
- **AI / ML / Foundation Models** — central ML platform, GenAI features, ranking infra.
- **Data & Infrastructure** — Kafka, Pinot, Espresso, Venice, ML feature store, observability, security.
- **Productivity Engineering** — dev infra, CI/CD, IDE platform, build (Bazel-ish), monorepo tooling.
- **Site Reliability / Production Engineering** — capacity, on-call, incident response.

Knowing which org you're interviewing into matters: a Staff role on Ads engineering will weight different competencies than a Staff role on Kafka platform.

## 1.6 Open-source projects that came out of LinkedIn

Mentioning these correctly demonstrates you've done your homework. Each has its own file later, but here's the cheat sheet:

| Project | What it is | Status |
|---|---|---|
| **Kafka** | Distributed log / messaging | Apache project since 2011; LinkedIn still a heavy user and contributor |
| **Voldemort** | Distributed key-value store (Dynamo-style) | Active internally for older workloads; new workloads on Venice |
| **Pinot** | Real-time OLAP datastore | Apache project; powers most of LinkedIn's interactive analytics |
| **Samza** | Stream-processing framework | Apache project; LinkedIn still uses for many pipelines, though Flink adoption is growing |
| **Venice** | Derived-data serving platform | Open-sourced 2022; LinkedIn's "next-gen Voldemort" for ML feature serving and derived datasets |
| **Brooklin** | Streaming-data bridge (Kafka↔Kafka, Kafka↔DB, etc.) | Open-source |
| **Databus** | Older change-data-capture system (Oracle CDC) | Open-source; mostly superseded by Brooklin |
| **Rest.li** | REST framework with strongly typed schemas | Open-source; LinkedIn's primary RPC framework |
| **Ambry** | Distributed immutable-blob store (photos, media, attachments) | Open-source |
| **Helix** | Generic cluster manager (used by Espresso, Pinot, Venice, etc.) | Apache project |
| **Burrow** | Kafka consumer-lag monitoring | Open-source |
| **Cruise Control** | Kafka cluster auto-balancing | Open-source |
| **Datahub** | Metadata platform (data lineage, discovery) | Open-source; spun out as Acryl Data |
| **PalDB** | Embedded read-only KV store | Open-source |
| **PyMK** | (Not OSS but iconic) People You May Know recommender | Internal |
| **FollowFeed** | Pull-model feed back-end | Internal |

Other notable ones: **LIX** (their experimentation framework), **inFlow** (workflow engine), **dr-elephant** (Hadoop perf tuning).

## 1.7 Strategic priorities (recent themes)

Based on public earnings remarks, blog activity, and job postings:

1. **AI everywhere** — generative AI in writing, recruiter copilot, premium chat, learning, ads creative.
2. **Video & creator** — short-form video, Live, newsletters as creator monetization.
3. **Enterprise revenue** — Recruiter and Sales Navigator are the growth engines; consumer Premium is the experimentation surface.
4. **Trust & integrity** — anti-spam, identity verification, deepfake detection. Heavy ML.
5. **Cost discipline** — Azure migration, Pinot replacing more expensive systems, derived-data consolidation onto Venice.

## 1.8 What to read on linkedin.com/engineering before the loop

Spend 4–6 hours on these, in this order:

1. **"A Brief History of Scaling LinkedIn"** (Josh Clemm, 2015) — still the canonical primer on the monolith→SOA journey.
2. **"Project Inversion"** posts — engineering-org transformation story.
3. **Kafka origin posts** by Jay Kreps — the log papers and follow-up blog posts.
4. **Pinot architecture** posts — most recent ones from the last 18 months.
5. **Venice architecture** post — 2022 open-source announcement and subsequent updates.
6. **Espresso** posts — schema evolution, replication, MySQL backing store.
7. **Galene/Search** posts — the in-house search platform.
8. **GenAI** posts from 2024–2025 — Recruiter AI, GAI infrastructure, eval frameworks.
9. **Trust & Safety** ML posts — anti-abuse pipelines.
10. **LinkedIn Live / video** posts — real-time ingest, low-latency CDN.

Bookmark `https://engineering.linkedin.com/blog`. If a topic in this prep doc looks dated, the blog is authoritative.