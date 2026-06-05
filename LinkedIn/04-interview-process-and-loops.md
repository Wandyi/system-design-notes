# 4 · The LinkedIn Staff SWE Interview Loop

This file describes the loop format, what each round actually tests, scoring rubrics, and how to calibrate. Information here is synthesized from publicly reported candidate experiences (Glassdoor, LeetCode discuss, levels.fyi, blind), LinkedIn's own engineering-careers blog, and recruiter-shared overviews. Don't treat any of it as a hard contract — every loop varies slightly.

## 4.1 Stages

### Stage 0 — Recruiter screen (30 min)

- Career narrative, current scope, why LinkedIn, geographic constraints, comp expectations.
- Recruiters at LinkedIn are *technical*; they will ask about the systems you've worked on.
- Calibrating your level happens here. Be honest about scope.

### Stage 1 — Technical phone screen (45–60 min)

- **Coding** in a shared editor (CoderPad or similar). One or two medium problems.
- Topics: arrays, hashmaps, trees, BFS/DFS, intervals, sliding window, basic graphs. Not weird tricks.
- LinkedIn often draws from its own historically-leaked question pool — see `21-coding-problems.md` for the canonical list.
- You're expected to: discuss approach before coding, hit O(n) or O(n log n) typically, handle edge cases, *test* your code at the end.

### Stage 2 — Onsite loop (5 rounds, ~6 hours)

The typical onsite for L5 / Staff is 5 rounds:

| Round | What it tests | Weight at Staff |
|---|---|---|
| **Coding 1** | Standard DSA, medium/hard | Medium |
| **Coding 2** | Often more applied / OO design / API-shaped problem | Medium |
| **System Design 1 — Open-ended** | A consumer-scale system (feed, search, messaging, etc.) | **Very high** |
| **System Design 2 — Object-oriented / API** | Class design, API contracts, evolvability | High |
| **Host Manager / Behavioral / Leadership** | Career story, scope, leadership, conflict, OE | **Very high** |

Some loops swap one System Design round for a **Tech-deep-dive on your past project** ("tell me about the most complex system you built"). Treat that as a hybrid system-design + behavioral round.

### Stage 3 — Team match + offer

- Team-match conversations after onsite passes; you talk to 1–3 hiring managers.
- Final offer once team selected. Compensation negotiation happens with the recruiter.

## 4.2 What "Staff" calibration looks like

LinkedIn's rubric for L5 emphasises three areas:

1. **Technical depth and breadth** — you can go deep on at least one area (data infra, ML, distributed systems, front-end platform, etc.) AND have working knowledge across the stack.
2. **Scope and impact** — your past work moves needles for a team-of-teams or an organisation. You're not just "owning a feature" — you're owning the technical roadmap of a domain.
3. **Leadership without authority** — you influence direction, mentor seniors, write/review architecture docs, drive cross-team changes.

Anti-patterns interviewers code as "Senior, not Staff":

- Talks about implementation details but can't articulate *why* one design beat another.
- Owns code, not outcomes.
- Hasn't mentored or influenced anyone else's work.
- Single-team scope.
- Avoids ambiguity rather than thriving in it.

## 4.3 Coding rounds at staff level

The bar is *not* hard LeetCode. The bar is:

- **You write correct, clean code** the first time, not after 3 hint nudges.
- **You communicate**: state the approach, identify edge cases *before* coding, explain trade-offs.
- **You test**: dry-run your code with a tricky input, fix bugs found.
- **You don't waste time** on tangents — you read the problem, ask the 1–2 clarifying questions that matter, and start.
- **You handle "now make it concurrent" / "now make it distributed"** follow-ups. At staff, a coding problem usually has a "make it scale" extension.

What you *don't* need:
- Memorized DP recipes for problems you'll never see.
- Tricks that depend on knowing one obscure data structure.

## 4.4 System design rounds — what they actually test

### The five things the interviewer is scoring

1. **Requirements gathering** — did you ask the right questions to pin down scope?
2. **Quantitative reasoning** — did you compute QPS, storage, bandwidth before deciding?
3. **Architectural taste** — did you choose appropriate primitives (queue vs. stream, OLTP vs. OLAP, sync vs. async)?
4. **Operational thinking** — failure modes, capacity, monitoring, deploys, multi-region?
5. **Communication** — did you drive the conversation, or did the interviewer have to fish?

### A scoring rubric (paraphrased from many sources)

- **No-hire (junior level)**: gives a single-machine answer; doesn't ask requirements; can't compute throughput.
- **Senior**: gives a microservices answer; uses correct primitives; handles 2 follow-ups.
- **Staff**: drives the conversation, anticipates trade-offs, names 2–3 alternative designs and chooses, talks operational concerns unprompted, knows when to push back on a requirement.
- **Senior Staff+**: in addition, identifies *organizational* concerns (who owns this, how does it migrate from $existing_system, how does it deprecate when superseded).

### Time budget for a 60-min round

- 5 min — requirements gathering ("functional + non-functional")
- 5 min — back-of-envelope estimation (QPS, storage, bandwidth)
- 10 min — high-level design / API
- 25 min — deep dive on 2–3 components the interviewer picks
- 10 min — failure modes, operations, trade-offs
- 5 min — buffer for interviewer to switch threads

If you spent 30 minutes on requirements, you bombed pacing. Be brisk.

## 4.5 The OO / API design round

A surprise for many candidates: LinkedIn loves an OO design round.

Typical prompts:

- "Design the **rate limiter** library that every service uses."
- "Design the **abstraction for a feature flag** (à la LIX)."
- "Design the **API for a notifications service**" — what endpoints, what data, what error semantics?
- "Model **memberships in a group**" — classes, relationships, persistence.
- "Design the **Job Posting class hierarchy**" — paid vs. organic, expiration, application types.

What they want:
- Clear class boundaries with cohesive responsibilities.
- Interfaces that are easy to evolve (e.g., adding a new notification channel without changing every caller).
- Idempotency, error semantics, and pagination thought through at the API level.
- Honest trade-offs: composition over inheritance, separating policy from mechanism, etc.

What they don't want:
- 14-level class hierarchies.
- "Generic enterprise" patterns applied dogmatically.
- Forgetting to define the data model behind the classes.

## 4.6 The host-manager / behavioral round

At staff level, **this is half the loop**. Don't underweight it.

Common themes:

- **"Tell me about a project where you had to influence a team you don't own."**
- **"Tell me about a time you disagreed with your manager / a Director / a Principal."**
- **"Walk me through the largest design decision you owned — and what you'd do differently."**
- **"Describe a time you mentored someone through a difficult problem."**
- **"How do you balance new features vs. tech debt?"**
- **"Tell me about a production incident you led."**
- **"What's a project that failed, and why?"**

The host-manager round will also probe **why LinkedIn?** — have a real answer, not boilerplate. The Economic Graph mission, the engineering blog quality, specific projects you'd want to contribute to.

See `23-behavioral-and-leadership.md` for STAR templates and the full question bank.

## 4.7 Deep-dive-on-past-project round

This is sometimes a standalone, sometimes folded into the host manager round.

Format:
- You pick a project (or the interviewer picks one from your resume).
- 45–55 minutes of drilling: the architecture, the alternatives you considered, the trade-offs, the failures, what you'd change.

What separates staff from senior here:
- Staff: explains the *organizational* context (who was the stakeholder, who pushed back, how was the team aligned).
- Staff: can defend the design without hand-waving but also acknowledges what they didn't anticipate.
- Staff: shows how the project influenced decisions *beyond* it.

Pick 2 projects to prep deeply. Have a diagram in your head you can draw on the whiteboard cold.

## 4.8 Levels and comp signal

For calibration, LinkedIn L5 (Staff) ranges (rough US/Bay Area, base + stock + bonus, 2024–2026 data):

- **Total comp**: $380K–$550K typical. Top-of-band higher.
- Mix: base ~50–55%, stock ~35–40%, bonus ~10%.
- Sign-on: variable; can be substantial.

This is *signal*, not a commitment. Treat ranges as conversation-starters with the recruiter; LinkedIn negotiates more like Microsoft (firm but reasonable) than Meta (aggressive).

## 4.9 Calibration anchors — how to know you're hitting Staff

Mid-round self-checks. If you're doing these naturally, you're hitting Staff signal:

- You said the words: "trade-off", "alternatives considered", "I'd want to validate that assumption", "what's the cost", "who owns this", "how do we deprecate the old system".
- You wrote at least one explicit non-functional requirement on the board (latency, availability, consistency).
- You drew a *data flow* not just boxes and arrows.
- You named *who* is upstream/downstream of your system, not just *what*.
- You changed your mind once when the interviewer pushed back, but defended a different point when the pushback was wrong.

If by the end of system design the interviewer is mostly *listening* rather than *driving*, you're hitting Staff.

## 4.10 What to do the week before

- Re-read the case studies in this directory.
- Pick *two* you'd be happy to design cold (recommendation: Feed + Messaging — they cover most patterns).
- Do 1 system-design dry run with a friend.
- Re-read your STAR stories aloud.
- Walk through `22-quick-reference-cheatsheets.md` twice.
- Sleep. Performance falls off a cliff at >8 hours of sustained cognitive load.