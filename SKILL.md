---
name: codebase-review
description: Review a local git repository and produce a versioned, stakeholder-ready codebase assessment. Emits a structured findings.json (source of truth) plus two rendered reports — a plain-language founder/board summary and an evidence-cited technical report — covering purpose, capability→code map, architecture, data model, roles, security, quality, operations, AI-provenance, and prioritized recommendations. Also emits a code-derived feature and flow inventory as a CSV with per-row implementation state: a starting point for revenue and product positioning, and a way to check a team's understanding of the product against what the code actually supports. Supports version-over-version deltas and re-use as a basis for audit and how-to documentation.
version: 2.2.0
license: MIT
---

# Codebase Review Skill

Inspect a codebase and produce a **versioned** assessment that non-engineers can act on and
engineers can verify. The audience is twofold — founders/board and the development team — so the
skill emits one machine-readable findings file and **two** audience-tuned reports rendered from it.

## When to use this skill
When asked to review a repo, audit a codebase, document what a system does, assess AI-assisted code,
or produce a technical/business analysis for stakeholders — especially when the reader is
non-technical or when the report needs to be tracked as the code evolves.

## Core principle: findings first, reports second
Do **not** hand-write HTML narratives. Produce data, then render.

1. **`findings.json`** — the single source of truth. Conforms to `schema/findings.schema.json`.
   Versioning, deltas, and every report all derive from this file.
2. **`founder-report.html`** — plain-language summary for non-engineers. Renders from
   `templates/founder-report.html`. Leads with a traffic-light scorecard and version deltas;
   frames every finding as "what this means for you"; hides technical detail.
3. **`technical-report.html`** — evidence-cited report for the dev team / auditors. Renders from
   `templates/technical-report.html`. Every claim carries `file:line` evidence and a
   documented-vs-inferred basis.
4. **`feature-inventory.csv`** — the reviewer's list of user-facing features and flows, each with what
   it does, where it lives, and whether it is backed by working code. Projected mechanically from
   `featureInventory[]` using the header in `templates/feature-inventory.csv`. The columns a revenue
   owner needs (value category, business value, sales grouping, buyer archetype) are emitted
   **blank**. This output states what the code appears to do. It does not assign business value, and
   nothing a human does with it downstream returns to the review.

This split is what makes the output **versionable** (diff the JSON, not the HTML) and **reusable**
(re-render the same findings into audit or how-to docs without re-scanning).

## Output location & versioning convention
Write to the repository's own `documents/` directory, one folder per review, named by date + short SHA:

```text
<repo>/documents/code-review/<YYYY-MM-DD>_<shortSHA>/
    findings.json
    founder-report.html
    technical-report.html
    feature-inventory.csv
```

- Read the date and SHA from git (`git rev-parse --short HEAD`, `git show -s --format=%cd`); never
  fabricate them. If not a git repo, ask the user for a version label.
- Before writing, look for the most recent prior `findings.json` under `documents/code-review/`.
  If one exists, populate the `deltas` object by comparing finding ids and scores against it, and
  reuse stable finding ids (`SEC-01`, …) so items can be tracked across versions. Apply the
  **score-band rules in `rubric.md`**: carry findings forward on unchanged code (do not re-score),
  and only report a delta that crosses a rating boundary or reflects a real code change — not
  reviewer-judgment wobble. Pin finding `severity` with the stage anchors in `rubric.md`.
- `documents/` may be git-ignored (it often is) — that's fine; versioning is by dated folder, not by commit.

### Non-destructive output

Write **only** inside the dated review folder, and never modify anything a human put in `documents/`.

- If a target file already exists (re-running a review at the same date and SHA), emit a new version
  rather than overwriting: `findings.v2.json`, `founder-report.v2.html`, `feature-inventory.v2.csv`,
  then `.v3`, and so on. Never overwrite.
- Documents a human placed in `documents/` — enriched inventories, value repositories, category
  guides, positioning decks — are **read-only** to this skill. Do not parse them, reconcile against
  them, or update them. If asked to revise one, emit a new version alongside and leave the original
  untouched.

## How to review
1. Start at the repo root. Capture provenance (repo name, SHA, ref, date, languages, LOC, file count).
2. Scan README, docs, package manifests, lockfiles, CI/CD config, Docker/K8s/infra files, and source.
3. Identify purpose, target personas, and the problems the system solves.
4. Build the **capability → code map**: every user-facing feature mapped to the files that implement
   it. This is the direct answer to "we don't know what the code does."
5. Expand the capability map into a **feature and flow inventory** (`featureInventory[]`). Enumerate
   what a user can do, at the grain described in *Feature grain* below, and give each row a `state`
   (`implemented` / `partial` / `ui-only`) with evidence. Include multi-step flows as their own rows
   (`kind: "flow"`). Leave every value and positioning column blank — that is a revenue owner's
   judgment, not the reviewer's.
6. Extract workflows (reusable later as how-to docs) and the internal role/permission matrix.
7. Map architecture to the depth of a standalone architecture report: a layered tech stack
   (`architecture.techStack`), several complementary Mermaid diagrams (`architecture.diagrams` —
   at least a system-context graph, a layered/dependency flowchart, a persona/route map, and a
   cross-actor lifecycle sequence), cross-cutting concerns (`architecture.crossCutting` — payments,
   AI, verification, state management, design system, etc.), a feature×persona matrix
   (`architecture.featureMatrix`), runtime flow, and external dependencies.
8. Extract the data model: entities, relationships, storage, data flow.
9. Assess security, code quality, and operations. Score each scorecard dimension against
   `rubric.md` — use the fixed rubric so scores are comparable across versions.
10. Scan for **AI-provenance signals** (unused deps, duplicated blocks, dead code, inconsistent
    patterns, untested areas, hallucinated APIs) — relevant when code was AI-assisted.
11. Assemble `findings.json`, then render both reports and the inventory CSV from the templates.

## Non-negotiable rules
- **Cite or mark as assumption.** Every finding needs ≥1 evidence ref (`file:line`) OR
  `basis: "inferred"` with justification. Never state an unverifiable claim as fact.
- **Business impact is required** on every finding — plain language: cost, risk, customer impact,
  or timeline. The founder report shows only this; if you can't write it, the finding isn't ready.
- **Confidence is visible.** Set `confidence` honestly; when evidence is thin, score lower.
- **Declare scope.** `scopeExcluded` and `methodology.limitations` must be filled — say what you
  did NOT review. Silent gaps read as false completeness.
- **Determinism for diffs.** Stable finding ids and the fixed rubric make version-over-version
  comparison meaningful.
- **Never author a business-value judgment.** Value category, business-value prose, sales grouping,
  and buyer archetype are out of scope for this skill. Blank is the correct output. A plausible guess
  is worse than a blank, because a blank gets filled and a guess gets shipped.
- **Cite-or-assume applies per inventory row.** Never mark a row `implemented` without `file:line`
  evidence. `partial` requires evidence of both the working part and the gap; `ui-only` requires
  evidence of the mock or sample-data source.
- **Every count states its denominator.** Render `meta.featureInventory.count` alongside any
  percentage. A bare percentage reads as the whole product.
- **Never overwrite, never write outside the review folder.** See *Non-destructive output*.

## Feature grain

The inventory is **the skill's perception of the codebase**, not a canonical feature registry. No list
is supplied to match against, so there is no right or wrong answer about what counts as a feature. Its
job is to be legible enough that a human can hold it against their own understanding of the product
and see where the two disagree — features they didn't know were built, features they believed were
built that the code does not support, and descriptions that reveal the reviewer misread something.

That makes grain a legibility problem, not a correctness one.

**Grain guidance.** Aim for the grain a demo or a pricing page would use. Derive it by enumerating
routes and screens from the code's own structure, then the distinct user actions within each, then the
multi-step paths that cross them (`kind: "flow"`).

- A navigation affordance is not a row unless it is the only entry point to a capability.
- A CRUD set on one entity is one row, not four, unless the states are separately meaningful to a user
  (an approval workflow is not the same thing as record creation).
- **When unsure, split rather than merge.** A too-fine row is easy for a reader to combine; a
  too-coarse row hides a feature they would otherwise have noticed was missing. The asymmetry favors
  splitting.

State the lens you used in `meta.featureInventory.grainNote`, so a reader knows what they are
comparing against.

**Ids.** Sequential within a review, prefix `FI-` (`FI-001`). On re-review, carry a prior id forward
when a row is recognizably the same feature — match on name, then capability, then code location. This
is a convenience for anyone diffing two CSVs, not a contract: no high-water mark, no append-only
invariant, no renumbering rules.

**The inventory does not participate in `deltas`.** Granularity movement between reviews is reviewer
perception, not a code change, and logging it as churn is the same false-delta problem `rubric.md`
guards against for scores. `deltas` stays at the capability and finding level. Because every emitted
CSV is preserved, diffing two versions is the change signal.

## Output format
- Reports are valid, self-contained HTML rendered from the templates (inline CSS; founder report
  is print/PDF-friendly for board decks).
- Diagrams: Mermaid source in `findings.json` (rendered in the technical report), or ASCII fallback.
  Validate every Mermaid diagram before shipping. Two gotchas that pass casual reading but fail to
  render: a `;` inside a `sequenceDiagram` message (it is a statement separator — use a comma), and a
  flowchart node label that starts with `[/` (parses as a parallelogram shape). If a repo already has
  architecture docs/diagrams, reconcile against the current code and correct anything stale rather than
  copying it forward (e.g. a role that has since grown a full portal).
- Fill every `{{TOKEN}}`, repeat each marked block per array item, and delete optional blocks
  (e.g. "Since last review", the founder report's inventory callout) when their data is absent.
- `feature-inventory.csv` is a mechanical projection of `featureInventory[]` onto the header in
  `templates/feature-inventory.csv`. Never hand-author it and never let it drift from `findings.json`.
  UTF-8, RFC 4180 quoting, `;`-joined multi-values inside a quoted cell, rows in `feature_id` order so
  two preserved versions diff cleanly. No preamble comment line — it breaks spreadsheet import;
  provenance lives in the containing dated folder and in `findings.json`.
- Emit the inventory on every review. Skip it only for repos with no meaningful user-facing surface (a
  library, a CLI with no product features), where it would be noise — and say so in
  `methodology.limitations` when you skip it.

## Voice (applies to all rendered founder-facing prose)
Write the founder report as a concise, experienced product/SaaS leader would — the way a memo to
a board reads, not the way an AI summarizes. This governs every prose field the founder report
renders (executiveSummary, scorecard `businessMeaning`, each finding's `businessImpact`,
recommendations, and any narrative). Apply the same register to the technical report's summary prose.

Do:
- Lead with the claim. Short, declarative sentences. State what is true, then why it matters.
- Use plain operator language a non-engineer trusts. Prefer periods over dashes.
- Let positives stand as positives and risks stand as risks, plainly.

Do NOT:
- Use antithesis / correction constructions: "it's not X, it's Y", "not just X but Y",
  "This isn't a bug, it's…", "None of this… all of it…". These are the top AI tell — ban them.
- Reach for extended metaphors or analogies (no "think of it as a showroom house…"). Describe the
  system directly.
- Pile up em-dashes. At most one per paragraph, and only where a period won't do.
- Hedge with filler ("it's worth noting that", "at the end of the day", "genuinely", "truly").

Before → after:
- "It's not a finished product — it's a convincing demo." → "The UI is complete. The backend is not built yet."
- "Zero tests, so nothing catches a mistake before it reaches users — the biggest gap." →
  "No automated tests. Nothing catches a regression before users do. The biggest gap in the codebase."
- "Think of the app as a showroom house with no plumbing." → "The screens are built; the data is
  sample content in the code, not a live database."

## Reuse profiles (optional, same findings.json)
- **Audit package** — populate `auditMappings` (OWASP Top 10 / SOC 2 / GDPR) and lead the technical
  report with the control table; findings already carry evidence for traceability.
- **How-tos / user docs** — the `workflows[].steps` are written to double as step-by-step guides.
- **Board one-pager** — the founder report's scorecard + deltas + top findings, printed to PDF.
- **Feature inventory** — `featureInventory[]` projects to a CSV at feature and flow grain for a
  revenue owner to enrich offline. **One direction only.** Whatever they produce — an enriched sheet, a
  value repository, a category guide — is an artifact this skill does not read back. Reading it back
  would make the review's output depend on a human's positioning judgment and require reconciling it
  on every run. Emit, hand off, stop.

## Large repositories
If the codebase is too large for one pass, chunk by service/top-level directory: review each,
capture partial findings, then synthesize into one `findings.json`. Record what was sampled vs
exhaustively reviewed in `methodology`, and never let sampling masquerade as full coverage.

## Notes
- Multiple services/microservices: describe each and the integration points between them.
- Always separate quick wins from higher-risk changes in `recommendations`.
- Include a "questions to ask your developers" set in the founder report — grounded in the findings.
