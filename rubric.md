---
created: 2026-07-09
type: note
topic: codebase-review scoring rubric and severity anchors
domains:
  - software-engineering
  - ai-ml
concepts:
  - governance-in-practice
  - adversarial-thinking
  - assurance-levels
tags:
  - notes
  - code-review
  - scoring-rubric
  - severity
  - security
  - bus-factor
---

# Scoring Rubric

A fixed rubric is what makes reports **comparable across versions**. Score each scorecard
dimension 0–5 using these definitions, then map the score to a traffic-light rating.
Score conservatively: when evidence is thin, score lower and set `confidence: low`.

## Score → rating mapping

| Score | Rating | Meaning |
|-------|--------|---------|
| 4.0–5.0 | 🟢 green | Healthy. No action needed beyond normal upkeep. |
| 2.5–3.9 | 🟠 amber | Workable but with real gaps. Plan to address. |
| 0.0–2.4 | 🔴 red | Material risk. Needs attention soon. |

## Severity anchors (per-finding)

Finding `severity` drifts between reviewers when it is left to judgment. Pin it to two things:
**what could happen** and **the deployment stage the code is actually at**. State the stage in
`meta` and apply it consistently.

| Severity | What it means |
|----------|---------------|
| **critical** | Exploitable/observable now, causing data loss, account takeover, fund loss, or full outage. Fix before anything else. |
| **high** | Serious weakness that is real at the code's current stage, or a certainty the moment it ships. Fix this cycle. |
| **medium** | Real gap, but bounded impact or gated behind a stage the code has not reached. Plan to address. |
| **low** | Minor or cosmetic; hygiene, or theoretical until other conditions change. |
| **info** | Not a defect: a positive confirmation, a design note, or context. |

**Stage calibration (the anti-drift rule).** The same code fact takes a *different* severity
depending on deployment stage. Decide the stage once, then rate against it — do not re-litigate
per finding:

- **Prototype / no real backend, no real users or data** → weaknesses in auth, authorization,
  rate limiting, and data-at-rest are typically **medium** (real once shipped, not exploitable
  now). Only score **high/critical** here for things wrong *today* regardless of stage (committed
  real secrets, a live exploitable endpoint, data loss).
- **Handles real users / real data / real money** → those same weaknesses jump to **high or
  critical**. Access-control and authN/authZ gaps are **critical**; missing tests around money or
  permissions is **high**.

Write the stage-conditional in the finding when it matters, e.g. "Medium as a prototype; Critical
once this carries real accounts." That single sentence is what keeps two reviewers (or two runs)
from splitting High vs Medium on the same fact.

### Overall health

Holistic read across the other dimensions plus general coherence of the codebase.

- **5** — Well-structured, documented, tested, low risk. A new engineer is productive in days.
- **3** — Functional and shippable, but onboarding is slow and some areas are fragile.
- **1** — Fragile, opaque, or sprawling. Changes are risky and slow.

### Security

Auth, access control, secret handling, sensitive-data exposure, dependency risk, common vulns.

- **5** — Secrets managed, authz enforced, no obvious vulns, dependencies current.
- **3** — Basics in place but with notable gaps (e.g. weak input validation, stale deps).
- **1** — Exposed secrets, missing/naive auth, or known-vulnerable dependencies.

### Maintainability

Organization, consistency, complexity, coupling, readability, dependency hygiene.

- **5** — Clear structure, consistent patterns, low coupling, easy to change safely.
- **3** — Mixed patterns and some tangled areas; changeable with care.
- **1** — Inconsistent, highly coupled, or copy-pasted; small changes ripple widely.

### Test coverage

Presence, breadth, and meaningfulness of automated tests; edge-case coverage.

- **5** — Meaningful coverage of core logic and edge cases; runs in CI.
- **3** — Some tests, uneven coverage, gaps around critical paths.
- **1** — Little or no automated testing; regressions are likely and invisible.

### Documentation

README, architecture notes, inline docs, onboarding material, decision records.

- **5** — A newcomer can understand purpose, architecture, and how to run it without help.
- **3** — Partial docs; key knowledge lives in people's heads.
- **1** — Little to no documentation; the code is the only source of truth.

### Key-person risk

How concentrated is understanding of the system? (a.k.a. bus factor)

- **5** — Well-documented and conventional; many people could safely own it.
- **3** — A few undocumented or idiosyncratic areas depend on specific individuals.
- **1** — Critical, opaque areas understood by one person; losing them stalls the product.

### Operational readiness

Deployment pipeline, environment setup, config management, observability, rollback.

- **5** — Reproducible deploys, monitored, easy to set up and roll back.
- **3** — Deployable but manual or under-observed; incidents are hard to diagnose.
- **1** — Fragile or undocumented deploys; little visibility into production health.

## Comparing across versions (score bands)

Scores carry reviewer judgment, so treat them as **±0.5 bands, not exact points**. An independent
re-scan of *unchanged* code will typically land within 0.5 of the prior score on the softer
dimensions (documentation, key-person risk, operational readiness) and on borderline severities —
that variance is noise, not a code change.

Rules that keep version-over-version deltas honest:

- **Unchanged code → carry findings forward; do not re-score.** Same commit + clean tree means the
  assessment is unchanged by definition. Re-scoring would manufacture fake deltas. (Re-scan only to
  audit reproducibility, and label it as such — never overwrite the versioned review with it.)
- **Only report a scorecard delta when it crosses a rating boundary** (green↔amber↔red) *or* moves
  by **more than 0.5**. A 3.5→3.2 wobble inside amber is not a "decline"; a 2.6→2.3 (amber→red) is.
- **Only report a severity delta when the code changed**, not when a reviewer would have graded the
  same fact differently. If two runs split High vs Medium on identical code, that is a stage-anchor
  gap (see *Severity anchors*), not a finding that changed — fix the anchor, don't log a delta.
- When a delta is real, name the code change that caused it (a fixed finding, a new dependency, a
  scope change), not just the number.
