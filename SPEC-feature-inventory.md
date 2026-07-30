---
created: 2026-07-29
revised: 2026-07-29 (rev 3 — open loop, perception framing; see Revision history)
type: spec
status: draft — awaiting red-line
topic: feature-inventory CSV emission for the codebase-review skill
target-version: 2.2.0 (from 2.1.0)
domains:
  - software-engineering
concepts:
  - agentic-ai
  - content-governance
tags:
  - spec
  - code-review
  - agent-skill
  - json-schema
---

# Spec — feature inventory emission

Adds one output to the codebase-review skill: a **feature inventory CSV**, derived from the code, at
feature and flow grain, with an implementation state and evidence on every row. It is a starting
point handed to whoever owns revenue positioning. The skill states what it perceives in the code; a
human decides what that is worth and what counts.

## Revision history

**Rev 1 — consume.** A sales-authored inventory (94 screenshot-sourced features, value-tagged) would
be read by the review and reconciled against the code.

**Rev 2 — emit.** Inverted. The code is ground truth for what exists; a screenshot crawl is a lossy
proxy. Deriving the inventory from code makes it accurate by construction. Removed the grain-matching
problem, the reconciliation enum, `.docx` parsing, inventory-date provenance, and the
severity-escalation rule in `rubric.md`.

**Rev 3 — open the loop.** Rev 2 still assumed the enriched document would eventually be read back,
which justified an append-only id registry, a grain-rule version, and churn-suppression rules. It
comes back never. Rev ops takes the CSV offline, revises what qualifies as a feature on their own
terms, and their documents land in the repo as artifacts the skill does not read. That removes the
id-stability apparatus and reframes grain from a correctness problem to a legibility one (§6).

## What the output is for

The inventory is **the skill's perception of the codebase**, not a canonical feature registry. No
list is supplied to it, so there is no right or wrong answer about what counts as a feature.

Its two jobs:

1. **Validation instrument.** A human holds the AI's list against their own understanding of the
   product and finds the gaps in either direction — features they didn't know were built, features
   they believed were built that the code does not support, and descriptions that reveal the reviewer
   misread something.
2. **Head start.** Rev ops begins from a populated sheet with implementation states and code
   citations already filled, instead of a blank one, and then applies their own judgment about
   granularity, naming, and value.

Both jobs are served by a legible list. Neither requires a stable one.

## Governing principle

**The schema holds only what the code can prove.**

Value codes, business-value prose, sales categories, and stakeholder archetypes do not appear in
`findings.json` — not even as empty fields. They exist only as blank columns in the emitted CSV, for
a human to fill. An empty field in the schema is an invitation for the model to fill it; a blank
column in a spreadsheet handed to rev ops is a task assignment.

## The loop is deliberately open

Rev ops revisions do not return to the skill. Their resulting documents live in the repo as artifacts
and are **never** parsed, reconciled against, or updated by this skill. There is no phase 2.

This is a design choice, not a limitation. Reading enriched documents back would make the skill's
output depend on a human's positioning judgment, which would then need reconciling on every review —
which is exactly the complexity rev 1 collapsed under. One direction only.

## Decisions locked

1. Separate output file, not folded into the founder or technical report.
2. CSV is the working artifact; the value/positioning columns are emitted blank.
3. All state lives in `findings.json`; the CSV is a mechanical projection.
4. Founder report keeps a mandatory one-sentence headline plus a pointer.
5. Scorecard untouched — `scorecard[].dimension` is a fixed enum that deltas key on.
6. `rubric.md` untouched.
7. Non-destructive output: never overwrite, always a new version (§7).
8. Emitted by default on every review, not opt-in. It is now a cheap byproduct of the capability map
   rather than a heavy reconciliation, and an inventory nobody asked for still costs less than a
   missing baseline.
9. Id prefix `FI-` (`FI-001`), not `F-`, to avoid confusion with the existing 94-feature document's
   `F001`–`F094` when a rev ops reader holds both.

## File manifest

| File | Change | Risk |
| --- | --- | --- |
| `schema/findings.schema.json` | Additive: 1 new top-level array, 1 meta object, 1 `capabilityMap` field | Low — all optional |
| `SKILL.md` | 6 edits incl. version bump, new review step, grain guidance, non-destructive rule | Low |
| `templates/feature-inventory.csv` | **New** (header/grain contract) | New surface |
| `templates/technical-report.html` | 1 column on an existing table | Low |
| `templates/founder-report.html` | 1 optional block + 1 CSS rule | Low |
| `CONTRIBUTING.md`, `README.md` | Sync per *Keep things in sync* | Low |
| `rubric.md` | **No change** | — |

---

## 1. `schema/findings.schema.json`

### 1a. `featureInventory` — new top-level array

```jsonc
"featureInventory": {
  "type": "array",
  "description": "The reviewer's perception of the product's user-facing features and flows, derived from code. The fine-grained expansion of capabilityMap. Not a canonical registry — see SKILL.md 'Feature grain'.",
  "items": {
    "type": "object",
    "required": ["featureId", "kind", "name", "plainLanguage", "state", "evidence", "confidence", "basis"],
    "properties": {
      "featureId": { "type": "string", "description": "Sequential within this review, prefix 'FI-'. Carried forward from a prior review when the feature is recognizably the same, as a reader convenience. Not a contract." },
      "kind": { "type": "string", "enum": ["feature", "flow"], "description": "feature = a discrete thing a user can do. flow = a multi-step path across features (e.g. need -> RFP -> award -> SOW). Flows are frequently what gets positioned, so they earn rows of their own." },
      "name": { "type": "string", "description": "Short name a non-engineer would recognize." },
      "plainLanguage": { "type": "string", "description": "What it does, for a non-engineer. One or two sentences. Must stand alone — rev ops reads this in a spreadsheet, away from the reports." },
      "area": { "type": "string", "description": "Product area from the code's own structure (route group, directory). Not a sales category." },
      "capability": { "type": "string", "description": "Name of the capabilityMap[] entry this rolls up to." },
      "state": {
        "type": "string",
        "enum": ["implemented", "partial", "ui-only"],
        "description": "implemented = works end to end. partial = some of it real, some not. ui-only = screen renders from mock or sample data, no working backend."
      },
      "evidence": {
        "type": "array",
        "minItems": 1,
        "items": { "$ref": "#/$defs/evidence" },
        "description": "Cite-or-assume applies per row. 'partial' cites both the working part and the gap. 'ui-only' cites the mock or sample-data source."
      },
      "confidence": { "type": "string", "enum": ["high", "medium", "low"] },
      "basis": { "type": "string", "enum": ["documented", "inferred"] }
    }
  }
}
```

No `removed` state and no retention policy. Because every emitted CSV is preserved (§7), diffing two
versions is the removal signal, and it is a better one than a state flag a reader has to filter.

### 1b. `meta.featureInventory` — new, optional

```jsonc
"featureInventory": {
  "type": "object",
  "required": ["count", "headline"],
  "properties": {
    "count": { "type": "integer", "description": "Rows emitted. The denominator for any percentage." },
    "stateCounts": {
      "type": "object",
      "description": "Computed rollup. implemented + partial + uiOnly MUST equal count.",
      "properties": {
        "implemented": { "type": "integer" },
        "partial": { "type": "integer" },
        "uiOnly": { "type": "integer" }
      }
    },
    "grainNote": { "type": "string", "description": "One sentence on the lens that produced this list, so a reader knows what they are comparing against. See SKILL.md 'Feature grain'." },
    "headline": { "type": "string", "description": "One sentence, founder voice, carrying the counts AND the denominator. Rendered in the founder report. Required when featureInventory exists." }
  }
}
```

### 1c. `capabilityMap[]` — one additive field

```jsonc
"featureIds": {
  "type": "array",
  "items": { "type": "string" },
  "description": "featureInventory ids rolling up to this capability. Every inventory row belongs to exactly one capability; a capability with an empty array is a bug in the inventory."
}
```

`capabilityMap` keeps its coarse grain and both existing reports render it unchanged. The inventory is
its expansion, not its replacement.

---

## 2. `SKILL.md`

**2a. Frontmatter.** `version: 2.1.0` → `2.2.0`. Append to `description`: "Emits a code-derived
feature and flow inventory as a CSV with per-row implementation state — a starting point for revenue
and product positioning, and a way to check a team's understanding of the product against what the
code actually supports."

**2b. *Core principle: findings first, reports second*** — add a fourth item, numbered `4.`:

> **`feature-inventory.csv`** — the reviewer's list of user-facing features and flows, each with what
> it does, where it lives, and whether it is backed by working code. Projected from
> `featureInventory[]`. The columns a revenue owner needs (value category, business value, sales
> grouping) are emitted **blank**. This output states what the code appears to do; it does not assign
> business value, and nothing a human does with it downstream returns to the review.

**2c. *Output location & versioning convention*** — extend the tree:

```text
<repo>/documents/code-review/<YYYY-MM-DD>_<shortSHA>/
    findings.json
    founder-report.html
    technical-report.html
    feature-inventory.csv
```

Add a pointer to the non-destructive rule (§7 of this spec → new `SKILL.md` subsection).

**2d. *How to review*** — insert as new step numbered `5.`, renumbering old 5–10 to 6–11:

> **Expand the capability map into a feature and flow inventory.** Enumerate what a user can do, at
> the grain described in *Feature grain*, and assign each row a `state` with evidence. Include
> multi-step flows as their own rows. Leave every value and positioning column blank — that is a
> revenue owner's judgment, not the reviewer's.

**2e. *Non-negotiable rules*** — four new bullets:

> - **Never author a business-value judgment.** Value category, business-value prose, sales grouping,
>   and buyer archetype are out of scope. Blank is the correct output. A plausible guess is worse than
>   a blank, because a blank gets filled and a guess gets shipped.
> - **Cite-or-assume applies per row.** Never mark a row `implemented` without `file:line` evidence.
>   `partial` requires evidence of both the working part and the gap; `ui-only` requires evidence of
>   the mock or sample-data source.
> - **Every count states its denominator.** Render `meta.featureInventory.count` alongside any
>   percentage. A bare percentage reads as the whole product.
> - **Never overwrite, never write outside the review folder.** See *Non-destructive output*.

**2f. *Reuse profiles*** — add:

> - **Feature inventory** — populate `featureInventory[]`; projects to a CSV at feature and flow grain
>   for a revenue owner to enrich offline. One direction only: enriched documents are artifacts, and
>   this skill does not read them back.

---

## 3. `templates/feature-inventory.csv` — new

Header contract. One row per feature or flow. The first block is filled by the review; the second is
emitted **empty, with headers present**.

```csv
feature_id,kind,feature_name,what_it_does,area,capability,state,code_locations,evidence,confidence,basis,value_code,business_value,sales_category,speaks_to
```

| Columns | Owner |
| --- | --- |
| `feature_id` … `basis` | The review. Mechanical projection of `featureInventory[]`. |
| `value_code`, `business_value`, `sales_category`, `speaks_to` | Rev ops. Always emitted blank. |

- UTF-8, RFC 4180 quoting, `;`-joined multi-values inside a quoted cell.
- Row order is `feature_id` order, so two preserved versions diff cleanly.
- No preamble comment line — it breaks spreadsheet import. Provenance lives in the containing dated
  folder and in `findings.json`.

## 4. `templates/technical-report.html`

One column on the existing capability-map table, line 94 `<thead>` and the row body at 97–102:

```html
<!-- INVENTORY COL (delete th and td when featureInventory is absent) -->
<th>Features</th>
...
<td class="fid"><!-- REPEAT featureIds[] -->{{featureId}} <!-- /REPEAT --></td>
```

No new section, no TOC entry. Per-row detail lives in the CSV; a 90-row table in an HTML report is
worse than a spreadsheet at every job a reader would use it for.

## 5. `templates/founder-report.html`

One optional block after line 91 (`<p class="lead">{{executiveSummary}}</p>`):

```html
  <!-- ============ INVENTORY CALLOUT (delete whole block if featureInventory is absent) ============ -->
  <p class="inventory-callout">
    {{meta.featureInventory.headline}}
    <span class="meta-row">Full inventory: <code class="path">feature-inventory.csv</code></span>
  </p>
```

`{{meta.featureInventory.headline}}` is required when the block renders — that enforces locked
decision 4. Add one CSS rule (left rule, tinted background, matching `.lead` type scale).

The headline speaks only in implementation terms: "Of 90 user-facing features and flows, 22 are
demo-only." It cannot say which *value* is exposed, because the review does not assign value.

---

## 6. Feature grain — guidance, not law

New `SKILL.md` section. Rev 2 treated grain as a correctness problem requiring a stable id registry.
It is not. Nothing is supplied to match against and nothing returns, so grain is a **legibility**
problem: the list has to be clear enough that a human can compare it to their own mental model and
see where the two disagree.

**Grain guidance.** Aim for the grain a demo or a pricing page would use. Derive it by enumerating
routes and screens from the code's own structure, then the distinct user actions within each, then
the multi-step paths that cross them (`kind: flow`).

- A navigation affordance is not a row unless it is the only entry point to a capability.
- A CRUD set on one entity is one row, not four, unless the states are separately meaningful to a
  user (approval workflow ≠ record creation).
- **When unsure, split rather than merge.** A too-fine row is easy for a reader to combine; a
  too-coarse row hides a feature they would otherwise have noticed was missing. The asymmetry favors
  splitting.

State the lens in `meta.featureInventory.grainNote` so a reader knows what produced the list.

**Ids.** Sequential within a review, prefix `FI-`. On re-review, carry forward a prior id when a row
is recognizably the same feature (name, then capability, then code location) — a convenience for
anyone diffing two CSVs, not a contract. No high-water mark, no append-only invariant.

**Deltas.** The inventory does not participate in `deltas`. Granularity movement between reviews is
reviewer perception, not a code change, and reporting it as churn would be exactly the false-delta
problem `rubric.md` already guards against for scores. `deltas` stays at the capability and finding
level, as in 2.1.0.

## 7. Non-destructive output

New `SKILL.md` subsection under *Output location & versioning convention*.

The skill writes **only** inside its own dated review folder, and never modifies anything a human put
in `documents/`.

- If `feature-inventory.csv` already exists in the target folder (re-running a review at the same
  date and SHA), write `feature-inventory.v2.csv`, then `.v3.csv`. Never overwrite.
- The same applies to `findings.json` and both HTML reports: on collision, emit a new version and
  preserve the original.
- Rev ops artifacts — enriched CSVs, category guides, value repositories — live wherever the team puts
  them and are **read-only** to this skill. It does not parse them, reconcile against them, or update
  them.
- Any pre-existing document elsewhere in `documents/` is likewise never edited in place. Emit a new
  version, preserve the original.

## 8. Doc sync

- `CONTRIBUTING.md` → *Add reuse profiles*: add feature inventory to "Existing hooks", and state that
  the enrichment loop is deliberately one-way.
- `CONTRIBUTING.md` → *Keep things in sync*: name the new coupling (`featureInventory[]` ↔ CSV header
  ↔ `capabilityMap[].featureIds` ↔ the grain guidance).
- `README.md`: add the CSV to the outputs list, and state plainly that the skill does not assign
  business value.

## 9. Gating and acceptance

`featureInventory` absent → no CSV, inventory column deleted from the technical report, founder
callout deleted, `meta.featureInventory` omitted. **Output is byte-comparable to 2.1.0.**

Since locked decision 8 makes emission the default, the gate exists for repos with no meaningful
user-facing surface (a library, a CLI with no product features) where an inventory would be noise.

---

## Open question

**Do flows share the CSV with features, or stay in `workflows[]`?** This spec puts both in one file
behind a `kind` column, because "features/flows" is the unit rev ops positions — "need to signed SOW
in one sitting" sells harder than any single feature in that path — and a separate flow file would
just get joined back together. The cost is that `workflows[]` and `kind: flow` rows now describe
overlapping things at different depth: `workflows[].steps` is a step-by-step sequence written to
double as user documentation, while a flow row is one line with a state. Defensible as different
depths for different readers, but it is duplication, and if you'd rather flows stay out of the CSV
entirely that is a one-line change to the schema enum.

## Not in scope

- Reading enriched rev ops documents back in, in any form. The loop is one-way by design.
- Comparing the inventory against the existing 94-feature document. That comparison is the point of
  the output, but a human does it by eye; it is not a skill feature.
- Backfilling the two prior reviews with an inventory.
- Any change to `rubric.md`, the seven scorecard dimensions, or the score→traffic-light mapping.
- Assigning business value, in any form.
