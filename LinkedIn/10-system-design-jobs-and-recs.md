# 10 · System Design — Jobs and Job Recommendations

The Jobs surface is LinkedIn's most directly monetizable consumer surface — Recruiter pays for Job Slots, members apply via Easy Apply, the recommender ("Jobs You May Be Interested In", **JYMBII**) drives ~30%+ of all applications. A staff candidate will frequently be asked some variant of "design the Jobs system" or specifically "design JYMBII".

## 10.1 Requirements

### Functional

- Companies post jobs (with title, description, requirements, location, salary, perks, application options).
- Postings can be **paid** (sponsored, distributed widely) or **free** (organic, smaller reach).
- Members browse, search, filter (location, remote, salary, skill, experience level).
- Recommended jobs (JYMBII) on the home page, feed, email digests.
- Members apply: Easy Apply (in-product), external apply (employer's ATS), Direct-Message.
- Recruiter side: track applicants, contact, schedule, hire.
- Saved jobs, job alerts, set-and-forget search subscriptions.
- Skills assessments + recommendations.

### Non-functional

- **Scale**: tens of millions of active job postings. ~5M members applying daily. ~100M JYMBII recommendations served/day.
- **Latency**: JYMBII page-load < 300ms server. Job search p95 < 500ms. Apply flow < 1s.
- **Freshness**: new posting indexed within minutes. Job application status updated near-real-time.
- **Availability**: 99.95%; degradation acceptable on JYMBII (cached recommendations).

## 10.2 Architecture

```
       Member side                                     Recruiter side
            │                                                │
            ▼                                                ▼
      ┌──────────┐                                     ┌──────────┐
      │ Jobs UI  │                                     │Recruiter │
      └────┬─────┘                                     └────┬─────┘
           │                                                │
           ▼                                                ▼
      ┌────────────────────────────────────────────────────────┐
      │                  BFF / API gateway                    │
      └────────────────────────────────────────────────────────┘
           │
           ├─► Job Service (Espresso)
           ├─► Search (Galene job-index)
           ├─► JYMBII Recommender Service
           ├─► Applications Service
           ├─► Job Alert subscription Service
           └─► Tracking/Analytics
```

Key services:

- **Job Service** — CRUD on job postings; stores the canonical posting in Espresso. Schema versioned (jobs change templates over years).
- **Job Indexer** — consumes Job Service CDC → builds Galene job index.
- **JYMBII Recommender** — generates personalized job recommendations.
- **Applications Service** — manages member→job applications (states: applied, viewed by Recruiter, shortlisted, rejected, hired).
- **Job Alerts** — subscriptions and email digests.

## 10.3 Data Model

```
jobs {
  job_id: BIGINT (PK, Snowflake)
  company_id: BIGINT
  recruiter_id: BIGINT (poster)
  title: STRING
  normalized_title_id: BIGINT  // from a canonical title taxonomy
  description: TEXT
  required_skills: ARRAY<skill_id>
  preferred_skills: ARRAY<skill_id>
  location: GEO (city + lat/long)
  work_type: ENUM (REMOTE, HYBRID, ONSITE)
  seniority: ENUM
  salary_min, salary_max: INT
  posted_at, expires_at: TIMESTAMP
  status: ENUM (DRAFT, ACTIVE, PAUSED, EXPIRED, FILLED)
  is_promoted: BOOL
  budget_remaining: DECIMAL
}

applications {
  application_id: BIGINT (PK)
  job_id: BIGINT
  member_id: BIGINT
  applied_at: TIMESTAMP
  state: ENUM (APPLIED, VIEWED, IN_REVIEW, INTERVIEWED, REJECTED, HIRED, WITHDRAWN)
  resume_ref: STRING (Ambry pointer)
  cover_letter: TEXT?
  source: ENUM (EASY_APPLY, EXTERNAL, REFERRAL)
  ats_external_id: STRING?  // for tracking when synced to an ATS
}

job_alerts {
  subscription_id: BIGINT
  member_id: BIGINT
  criteria: JSON  // search facets
  frequency: ENUM (DAILY, WEEKLY)
  last_sent: TIMESTAMP
}
```

Sharding: jobs by `job_id`; applications by `job_id` (so all applicants for a job co-located, fast for Recruiter UI) with a secondary lookup by member.

## 10.4 Search (jobs sub-index)

A specialized Galene index:
- Documents have rich structured fields: title, normalized-title-id, description text, required-skills array, location (geo + radius search), salary, etc.
- Heavy use of **structured filters** (Boolean facets) and **geographic radius** queries.
- Ranking blends relevance (BM25 / title match) with member-personalization (skills overlap, location preference, prior application behavior).

Subtle: "Senior Software Engineer" titles vary wildly. Normalization to canonical title IDs is essential. LinkedIn maintains a curated **titles taxonomy** (thousands of canonical titles, with skill mappings).

## 10.5 JYMBII (Jobs You May Be Interested In)

JYMBII is the recommendation system. Treated separately from search.

### Two-stage pipeline

**Stage 1 — Candidate generation (offline + nearline)**

For each member, generate ~1000 candidate jobs based on multiple signals:

- **Profile match** — member's skills × job's required-skills overlap.
- **Title progression** — likely next title given current title and trajectory.
- **Company affinity** — similar companies the member has shown interest in.
- **Location preference** — member's location + commute radius / remote preference.
- **Network signal** — jobs at companies where the member has connections.
- **Past behavior** — jobs the member viewed/applied/dismissed.
- **Collaborative filtering** — "members like you also applied to ..."
- **Embedding similarity** — member embedding × job embedding (cosine ANN).

Run as a Spark batch nightly, with Samza-driven incremental updates (e.g., when a member updates their profile, regenerate candidates).

Candidate lists stored in Venice keyed by member_id.

**Stage 2 — Online ranking**

At surface time:
- Load member's candidate list from Venice.
- Filter expired / already-applied / dismissed.
- Rerank with online features (very recent activity, freshness of posting).
- Apply business rules (promote sponsored jobs subject to relevance threshold).
- Surface top 25.

### Ranking model

- Multi-objective: optimize for click-through *and* application-quality (engagement that leads to a real conversion, not a low-effort apply).
- Features: skill overlap score, title-distance score, company size, salary alignment, member's recent search history, prior application rate.
- Calibration: predicted CTR + predicted Apply rate.
- Online learning for some features (e.g., a job's recent CTR).

### Sponsorship / paid distribution

- Paid jobs get a bid; auctioned for placement.
- "Promoted" badge shown.
- Relevance threshold prevents irrelevant promoted jobs from appearing.
- Budget pacing across the day.

## 10.6 Application flow

Member taps Apply on a job:
1. **Easy Apply** (in-product): pre-filled application form (resume from profile, optional questions). One submit → `applications` row.
2. Application event written to Espresso + emitted to Kafka.
3. **Recruiter dashboard** updates near-real-time.
4. **External apply**: redirect to employer's ATS. Tracking ping when redirect happens; no in-product application created.
5. **Application state** updates over time as Recruiter views/processes.

Subtle:
- Resume parsing: ML pipeline extracts skills + experience from uploaded resumes → enrich application metadata.
- **Quality screening**: ML flags spam applications / mass-applies; deprioritized in Recruiter view.
- **Disclosures**: ATS data syncing has compliance constraints.

## 10.7 Job alerts (subscription digests)

- Subscription stored in `job_alerts`.
- A daily batch job runs the criteria as a search; collects matching new postings; emits a notification candidate to ATC.
- ATC decides channel + timing; sends an email digest.
- Click-throughs from digests heavily monetized; tracked.

## 10.8 Recruiter side

Recruiters interact with:
- Job posting editor (templates, bulk-post).
- Applicant tracking surface — see all applicants per job; filter by skills/experience; rate; bulk-message.
- Search & sourcing — search candidates by Boolean facets (this is where Bobcat shines).
- Pipeline analytics — track conversion rates.

Recruiter is the highest-ARR product, so the engineering bar is rigorous: SLAs are tighter, audit logs are stricter (compliance with hiring laws), and the data model supports complex hiring workflows (interview loops, scorecards, requisitions).

## 10.9 Multi-region

- Job postings: globally replicated to all regions (every member must see all relevant jobs).
- Applications: regionally homed; sync to global ATS connectors via Brooklin.
- JYMBII candidates: per-region precompute (cheaper) with member's home region as primary.

## 10.10 Failure modes

- **JYMBII candidate gen failure** — serve last successful snapshot; alert.
- **Search index outage on jobs** — fall back to a smaller index (e.g., only paid jobs); show "limited results" banner.
- **Application service outage** — Easy Apply degrades to "we couldn't submit, please try again"; pre-filled form preserved client-side; idempotency key prevents duplicates.
- **Recruiter dashboard slow** — eventual-consistency banner; degraded mode.
- **Promoted job over-budget** — auto-pause distribution.

## 10.11 Operational concerns

- **Fairness**: avoid biased recommendations (gender, race, age). LinkedIn has invested heavily in fairness-aware ranking; explicit guardrails.
- **Transparency**: "Why am I seeing this job?" UI surfaces match signals.
- **Compliance**: certain locales (Germany, China) have specific job-posting rules.
- **Spam jobs**: anti-fraud detects duplicate / scam postings; pre-publish ML check.
- **A/B testing**: ramping any ranking change carefully on application metrics + member feedback (dismiss, hide).

## 10.12 Common follow-ups

> **"How would you bootstrap JYMBII for a new member with no history?"**
Use profile signals heavily (skills, title, education, current employer). Fall back to popular jobs in their region/title cohort. Collaborative filtering kicks in after a few interactions.

> **"How do you A/B test a new ranking model when applications take weeks to convert to hires?"**
Short-term proxy metrics (click, apply, save) + longer-term retention metrics (interview-rate from hires reported back to LinkedIn). Need long-running experiments for definitive impact.

> **"How would you support 'apply directly via email' for some employers?"**
A separate Application channel; same `applications` row; track delivery; deprioritize in ranking if external-apply has lower conversion.

> **"How do you handle ghost jobs (posted but no real intent to hire)?"**
ML detects: low applicant-view rate, never-marked-as-filled, posting age > N days with no Recruiter action. Penalize in ranking; flag for manual review.

> **"How would you redesign for a fully-remote-jobs-only feature?"**
Filter on `work_type = REMOTE`. Re-rank weighted by member's remote-preference signal. Easy on the back-end; subtle UX work to make it discoverable.