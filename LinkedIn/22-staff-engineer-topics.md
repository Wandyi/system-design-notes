# 22 · What Staff Means at LinkedIn

The label "Staff Software Engineer" varies wildly across companies. At LinkedIn, **L5 / Staff** is the inflection point where you transition from "owns features" to "owns systems and shapes direction." This file unpacks what they actually look for, with concrete signals.

## 22.1 The published rubric (paraphrased)

LinkedIn's internal Staff rubric has these axes (paraphrased from public talks and exit interviews):

| Axis | Senior (L4) | Staff (L5) | Senior Staff (L6) |
|---|---|---|---|
| **Scope** | A team's domain | A team-of-teams or domain across teams | A whole org or function |
| **Technical depth** | Goes deep in one area | Deep in one area + working depth in adjacent | Deep across multiple areas; recognized expert |
| **Influence** | Influences team | Influences cross-team; recognized in org | Influences cross-org; recognized company-wide |
| **Mentorship** | Mentors peers; helps junior | Mentors L4s, calibrates L4 candidates | Mentors L5s; calibrates L5 candidates |
| **Project leadership** | Leads multi-week projects | Leads multi-quarter projects with multi-team involvement | Leads multi-year strategic initiatives |
| **Ambiguity** | Solves well-defined problems | Defines the problem; reduces ambiguity for the team | Defines the *strategy*; reduces ambiguity for the org |
| **Trade-off articulation** | Knows the trade-offs of their design | Selects between several alternatives, defends choice | Frames the trade-off space others operate within |

You will not be asked these dimensions directly. They show up implicitly throughout the loop.

## 22.2 What Staff is NOT at LinkedIn

A common misconception list:

- **Not** "the smartest IC who can implement anything." Smart matters, but reach matters more.
- **Not** a manager. Staff is the IC track. You don't have direct reports.
- **Not** a code-throughput contest. A Staff engineer who writes 10x the code of a Senior is probably doing the wrong things.
- **Not** about specific technologies. A Staff distributed-systems engineer and a Staff frontend engineer are both Staff.
- **Not** automatic from years of experience. Some Senior engineers stay Senior for a decade because they don't grow into the scope dimension.

## 22.3 Skills that distinguish Staff

### 1. Reduce ambiguity for others

A Senior engineer asks "what should we build?" A Staff engineer says "based on these constraints, here are three options; here's the one I recommend and why."

**In the interview**: when given a vague design prompt, *frame the problem* before solving. State what you'll cover, what you're treating as out-of-scope, and check with the interviewer.

### 2. Make trade-offs explicit

A Senior engineer picks the best technical option. A Staff engineer surfaces *which* trade-offs the team should weigh:
- "We can have lower latency or stronger consistency, not both. Here's why I'd pick consistency for this use case."
- "We can build this in 6 weeks with tech debt, or 14 weeks clean. Here's how I'd discuss it with product."

**In the interview**: don't say "we'll use Kafka." Say "I'd use Kafka because we need durable replay; the alternative would be a synchronous gRPC call, which avoids the storage tier but loses replay-ability."

### 3. Operate at the system level, not the service level

A Senior engineer thinks about their service. A Staff engineer thinks about how their service fits into the system, the surrounding teams' services, and the deprecation/migration path of what was there before.

**In the interview**: when you design feed, talk about how feed integrates with notifications, search, trust, ads. Don't draw your service in isolation.

### 4. Mentor and calibrate

A Senior engineer helps when asked. A Staff engineer proactively raises the team's bar — design reviews, code reviews, paired debugging, RFC writing.

**In the interview**: behavioral round will probe. Have ≥2 mentorship stories. The best stories show **specific change** in the mentee's behavior or outcomes.

### 5. Drive cross-team initiatives

A Senior engineer ships their team's work. A Staff engineer drives initiatives where the team boundary is fuzzy: a migration, a new shared library, an org-wide consistency push.

**In the interview**: have a story about leading a multi-team effort *without authority* — by writing, presenting, negotiating, and persisting.

### 6. Choose what NOT to do

A Senior engineer takes on every task they can. A Staff engineer says "this isn't worth it; here's what I'd do instead."

**In the interview**: when scope creep happens (the interviewer adds requirements), name it: "I want to make sure we cover what you actually need — do you want me to dive deeper into the auction or into multi-region?"

### 7. Communicate to executives

A Senior engineer writes excellent technical docs. A Staff engineer also writes execs-ready summaries: 1-pager TL;DRs that surface the right trade-offs to decision-makers.

**In the interview**: when you give your final design summary, do it in 90 seconds with the 3 key choices and the rationale. Not in 10 minutes.

## 22.4 Concrete behavioral signals to demonstrate

The behavioral round is partly a vocabulary test. Use these phrases authentically:

- **"I aligned the team around..."**
- **"I escalated when..."**
- **"I made the trade-off between X and Y..."**
- **"I deprecated $thing because..."**
- **"I influenced $person's decision by..."**
- **"I changed my mind when..."**
- **"I let someone else lead..."**

Avoid:
- "I single-handedly..." (Staff is rarely a solo achievement.)
- "I had to push back hard against..." (Staff finds alignment; doesn't pick fights.)
- "Management didn't get it, but..." (Bad signal; Staff bridges, doesn't bypass.)

## 22.5 Operational excellence — the meta-competency

LinkedIn explicitly calls out OE as a Staff competency. It encompasses:

- **On-call hygiene** — owning the rotation, writing runbooks, drilling response.
- **Capacity planning** — quarterly capacity reviews, growth projections.
- **Post-mortems** — leading retrospectives, driving action items to closure.
- **Tech debt management** — surfacing it, prioritizing it, paying it down deliberately.
- **Security posture** — threat modeling, secret management, dependency scanning.
- **Cost awareness** — $/QPS, $/storage-TB, $/engineer-hour spent.
- **Dependency management** — knowing what your service depends on; managing version churn.

A Staff candidate is expected to volunteer at least 2–3 of these unprompted in the behavioral round.

## 22.6 Writing a design doc — the Staff skill

Staff engineers write more design docs than they write code. The shape LinkedIn (and most FAANGs) expect:

1. **TL;DR** (3–5 sentences).
2. **Context / motivation** (why this matters; what's broken; what the metric is).
3. **Goals + non-goals** (explicit boundaries).
4. **Proposal** (the actual design).
5. **Alternatives considered** (at least 2; for each, why it was rejected).
6. **Trade-offs** (consistency vs. availability; cost vs. performance; etc.).
7. **Migration / rollout plan** (canary, ramp, decommission).
8. **Monitoring + SLOs** (how you'll know it's working).
9. **Risk + open questions**.
10. **Appendix** (calculations, diagrams).

Practice writing one of these for a system you've built — bring it to the interview if asked about your past projects.

## 22.7 Architecture review

At LinkedIn, big designs go through an architecture review board (or org-level equivalent). Staff candidates are often invited to participate.

What reviewers look for:
- Is the problem clearly stated?
- Are the alternatives well-considered?
- Is the cost / capacity sized?
- Is the migration plan realistic?
- Is on-call burden accounted for?
- Are downstream teams represented / consulted?

When asked "have you ever led an arch review?" in behavioral, the best answer is **specific**: which review, what was the proposal, what feedback you got, how you responded.

## 22.8 The "scope" question — what to say

The interviewer will ask, in some form, "describe your scope." Don't fumble this.

A clean answer template:

> "My team is responsible for **X**. Within that, I personally own the technical direction of **Y**. I lead **N** engineers (matrix; not direct reports). My output is measured by **M** (a business metric, an engineering metric, or both). The biggest project I've led recently is **P**, which **business-impact-statement**."

Adapt for your actual situation. The structure: team responsibility → personal ownership → scope-quantification → recent project.

## 22.9 Tell-tale junior signals to avoid

Things Staff candidates *don't* say:

- "I just did what my manager asked." → no agency.
- "I escalated to management." → can't operate at peer level.
- "I optimized this function." (when asked about scope) → wrong altitude.
- "The right way is X." → no acknowledgment of context.
- "I don't know that area." (with no follow-up) → no curiosity / no first-principles reasoning.

## 22.10 The Staff "shadow" — what staff engineers actually do day-to-day

If asked "what does a typical week look like for a staff engineer?", here's a representative answer:

- Monday: 1:1s with team members; review 2 design docs; respond to pages.
- Tuesday: paired with a senior on a hard bug; afternoon code review.
- Wednesday: lead architecture review for a cross-team migration; lunch with a partner team's tech lead.
- Thursday: write half a day, attend a design review of an adjacent team's work; an oncall handover.
- Friday: half-day deep work on the technical strategy doc; a peer-review of a launch readiness review.

If your week is "feature implementation 90%", you're operating as Senior.

## 22.11 What you should be able to articulate by the end of the loop

When asked any of these, you should have a tight answer:

- **"What's the most impactful design decision you've made?"**
- **"What's a project you wish you'd handled differently?"**
- **"What's your operating principle when you disagree with a peer?"**
- **"How do you decide what to work on?"**
- **"How do you grow more junior engineers?"**
- **"What technology / approach do you think is overrated?"**
- **"What technology / approach do you think is underrated?"**

The last two are tells — Staff engineers have opinions. Senior candidates often deflect ("they're all fine for their use case"). Staff candidates have specific takes (e.g., "ML pipelines are over-engineered for problems where a heuristic with a clean monitoring loop is faster to iterate on").