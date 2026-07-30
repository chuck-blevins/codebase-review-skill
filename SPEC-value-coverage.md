---
created: 2026-07-29
revised: 2026-07-29 (rescoped — data flow inverted, see Rescope note)
type: spec
status: draft — awaiting red-line
topic: feature-inventory emission for the codebase-review skill
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

Adds one output to the codebase-review skill: a **feature inventory** derived from the code, at
feature grain, with an implementation state and evidence on every row. It is a handoff artifact for
whoever owns revenue positioning. The skill establishes what exists and what is real; a rev ops
reader adds the value judgment.

## Rescope note

The first draft of this spec ran the data the other way: a sales-authored feature inventory
(94 screenshot-sourced features, value-tagged) would be **consumed** by the review and reconciled
against the code. That was wrong in a way worth recording.

The code is the ground truth for what features exist. A screenshot crawl is a lossy, manual,
drift-prone proxy for it. Deriving the inventory from the code makes it accurate by construction,
re-derivable on every review, and — because the skill then owns the id space — actually diffable
across versions.

Inverting the flow removes: the coarse-vs-fine grain matching problem, the four-state reconciliation
enum, `.docx` parsing, inventory-date provenance, and the severity-escalation rule in `rubric.md`.
Five of the previous draft's open questions no longer exist. One new problem appears and is the main
thing to get right: **feature granularity and id stability** (§6).

The consumption half is not dead, only deferred. See *Phase 2, explicitly out of scope*.

## Governing principle

**The schema holds only what the code can prove.**

Value codes, business-value prose, sales categories, and stakeholder archetypes do not appear in
`findings.json` — not even as empty fields. They exist only as blank columns in the emitted CSV, for
a human to fill. An empty field in the schema is an invitation for the model to fill it in; a blank
column in a spreadsheet handed to rev ops is a task assignment.

## Decisions carried forward from the first draft

1. Separate output file, not folded into the founder or technical report.
2. CSV is the working artifact.
3. All state lives in `findings.json`; the CSV is a mechanical projection.
4. Founder report keeps a mandatory one-sentence headline plus a pointer.
5. Scorecard untouched — `scorecard[].dimension` is a fixed enum that deltas key on.

Dropped from the first draft: `meta.valueRepository`, `valueCoverage`, the `rubric.md` value-weighting
axis, `severityBase`/`severityBasis`, `finding.category: "value-coverage"`, the
`feature-coverage-report.html` template, and the `recommendation` value fields. All of them were
downstream of consuming an external document.

## File manifest

| File | Change | Risk |
| --- | --- | --- |
| `schema/findings.schema.json` | Additive: 1 new top-level array, 1 meta object, 1 `capabilityMap` field | Low — all optional |
| `SKILL.md` | 6 edits incl. version bump, new review step, grain rule | Medium — the grain rule is load-bearing |
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
  "description": "Code-derived inventory of user-facing features, at the grain defined by SKILL.md's feature grain rule. The fine-grained expansion of capabilityMap. Optional: emitted when the reviewer is asked for an inventory or when the repo already has one under documents/.",
  "items": {
    "type": "object",
    "required": ["featureId", "name", "plainLanguage", "state", "evidence", "confidence", "basis"],
    "properties": {
      "featureId": {
        "type": "string",
        "description": "Minted by the skill. Append-only, never renumbered or reused. See SKILL.md id stability rule."
      },
      "name": { "type": "string", "description": "Short feature name a non-engineer would recognize." },
      "plainLanguage": { "type": "string", "description": "What it does, for a non-engineer. One or two sentences." },
      "area": { "type": "string", "description": "Product area or module the feature sits in, from the code's own structure (route group, directory). Not a sales category." },
      "capability": { "type": "string", "description": "Name of the capabilityMap[] entry this rolls up to." },
      "state": {
        "type": "string",
        "enum": ["implemented", "partial", "ui-only", "removed"],
        "description": "implemented = works end to end. partial = some of it real, some not. ui-only = screen renders from mock or sample data, no working backend. removed = present in a prior review's inventory, no longer in the code."
      },
      "evidence": {
        "type": "array",
        "minItems": 1,
        "items": { "$ref": "#/$defs/evidence" },
        "description": "Cite-or-assume applies. 'partial' cites both the working part and the gap. 'ui-only' cites the mock or sample-data source."
      },
      "confidence": { "type": "string", "enum": ["high", "medium", "low"] },
      "basis": { "type": "string", "enum": ["documented", "inferred"] },
      "firstSeen": { "type": "string", "description": "Review date the id was minted. Carried forward unchanged." }
    }
  }
}
```

### 1b. `meta.featureInventory` — new, optional

Present whenever `featureInventory` is. Exists to keep the id space honest across versions.

```jsonc
"featureInventory": {
  "type": "object",
  "required": ["count", "idHighWater"],
  "properties": {
    "count": { "type": "integer", "description": "Rows in featureInventory, excluding state 'removed'. The denominator for any percentage." },
    "idHighWater": { "type": "string", "description": "Highest id minted so far, e.g. 'F094'. Next review mints from here. Never decreases." },
    "grainRuleVersion": { "type": "integer", "description": "Bump when SKILL.md's grain rule changes. Two reviews with different values are not comparable — say so in deltas rather than reporting churn." },
    "stateCounts": {
      "type": "object",
      "description": "Computed rollup. implemented + partial + uiOnly MUST equal count.",
      "properties": {
        "implemented": { "type": "integer" },
        "partial": { "type": "integer" },
        "uiOnly": { "type": "integer" },
        "removed": { "type": "integer" }
      }
    },
    "headline": {
      "type": "string",
      "description": "One sentence, founder voice, carrying the counts AND the denominator. Rendered in the founder report. Required when featureInventory exists."
    }
  }
}
```

### 1c. `capabilityMap[]` — one additive field

```jsonc
"featureIds": {
  "type": "array",
  "items": { "type": "string" },
  "description": "featureInventory ids that roll up to this capability. Every inventory row belongs to exactly one capability; every capability with an empty array is a bug in the inventory."
}
```

`capabilityMap` keeps its existing coarse grain and both existing reports render it unchanged. The
inventory is its expansion, not its replacement.

---

## 2. `SKILL.md`

**2a. Frontmatter.** `version: 2.1.0` → `2.2.0`. Append to `description`: "Can emit a code-derived
feature inventory with per-feature implementation state, as a handoff artifact for revenue and
product positioning."

**2b. *Core principle: findings first, reports second*** — add a fourth item, numbered `4.`:

> **`feature-inventory.csv`** — a code-derived list of user-facing features, each with what it
> does, where it lives, and whether it is backed by working code. Projected from
> `featureInventory[]`. The columns a revenue owner needs to add (value category, business value,
> sales grouping) are emitted **blank**. The review establishes what exists and what is real; it
> does not assign business value.

**2c. *Output location & versioning convention*** — extend the tree:

```text
<repo>/documents/code-review/<YYYY-MM-DD>_<shortSHA>/
    findings.json
    founder-report.html
    technical-report.html
    feature-inventory.csv           # when an inventory was requested or already exists
```

Add: before minting ids, read the most recent prior `findings.json` for
`meta.featureInventory.idHighWater` and the prior `featureInventory[]`, and follow the id stability
rule in *Feature grain and id stability*.

**2d. *How to review*** — insert as new step numbered `5.`, renumbering old 5–10 to 6–11:

> **Expand the capability map into a feature inventory, if asked for one** (or if the repo already
> has one under `documents/`). Enumerate user-facing features at the grain defined in *Feature
> grain and id stability*, assign each a `state` with evidence, and attach each to its parent
> capability. Leave every value and positioning column blank — that is a revenue owner's judgment,
> not the reviewer's.

**2e. *Non-negotiable rules*** — three new bullets:

> - **Never author a business-value judgment.** Value category, business-value prose, sales grouping,
>   and buyer archetype are out of scope for this skill. Blank is the correct output. A plausible
>   guess is worse than a blank, because a blank gets filled and a guess gets shipped.
> - **Cite-or-assume applies per feature.** Never mark a feature `implemented` without `file:line`
>   evidence. `partial` requires evidence of both the working part and the gap; `ui-only` requires
>   evidence of the mock or sample-data source.
> - **Every count states its denominator.** Render `meta.featureInventory.count` alongside any
>   percentage. A bare percentage reads as the whole product.

**2f. *Reuse profiles*** — add:

> - **Feature inventory** — populate `featureInventory[]`; projects to a CSV at feature grain for a
>   revenue owner to enrich. Once enriched, that document becomes an input a future version can read
>   back (not yet supported).

---

## 3. `templates/feature-inventory.csv` — new

Header contract. Grain is one row per feature. The first block is filled by the review; the second
block is emitted **empty, with headers present**.

```csv
feature_id,feature_name,what_it_does,area,capability,state,code_locations,evidence,confidence,basis,value_code,business_value,sales_category,speaks_to
```

| Columns | Owner |
| --- | --- |
| `feature_id` … `basis` | The review. Mechanical projection of `featureInventory[]`. |
| `value_code`, `business_value`, `sales_category`, `speaks_to` | Rev ops. Always emitted blank. |

- UTF-8, RFC 4180 quoting, `;`-joined multi-values inside a quoted cell.
- Row order is `feature_id` order, so two versions diff cleanly.
- Rows with `state: removed` are included, so a reader learns a feature they were positioning is gone.
- A one-line preamble comment is **not** used — it breaks spreadsheet import. Provenance lives in the
  containing dated folder and in `findings.json`.

## 4. `templates/technical-report.html`

One column on the existing capability-map table, line 94 `<thead>` and the row body at 97–102:

```html
<!-- INVENTORY COL (delete th and td when featureInventory is absent) -->
<th>Features</th>
...
<td class="fid"><!-- REPEAT featureIds[] -->{{featureId}} <!-- /REPEAT --></td>
```

No new section, no TOC entry. The per-feature detail lives in the CSV; a 90-row table in an HTML
report is worse than a spreadsheet at every job a reader would use it for.

## 5. `templates/founder-report.html`

One optional block after line 91 (`<p class="lead">{{executiveSummary}}</p>`):

```html
  <!-- ============ INVENTORY CALLOUT (delete whole block if featureInventory is absent) ============ -->
  <p class="inventory-callout">
    {{meta.featureInventory.headline}}
    <span class="meta-row">Full inventory: <code class="path">feature-inventory.csv</code></span>
  </p>
```

`{{meta.featureInventory.headline}}` is required when the block renders — that enforces carried
decision 4. Add one CSS rule (left rule, tinted background, matching `.lead` type scale).

Note the headline can only speak in implementation terms in v1: "Of 90 user-facing features, 22 are
demo-only." It cannot say which *value* is exposed, because the review does not assign value. That
sentence gets sharper in phase 2.

---

## 6. Feature grain and id stability — the load-bearing rule

This is the one problem the inversion creates rather than removes, and the whole output's
diffability rests on it. New `SKILL.md` section.

**Grain.** One row per distinct thing a user can do that could plausibly appear on a pricing page or
in a demo. Derive it by enumerating routes and screens from the code's own structure, then the
distinct user actions within each. Two calibration rules:

- A navigation affordance is not a feature unless it is the only entry point to a capability.
- A CRUD set on one entity is one feature, not four, unless the states are separately sold
  (e.g. approval workflow ≠ record creation).

State the grain rule version in `meta.featureInventory.grainRuleVersion` and bump it whenever this
section changes.

**Id stability.**

- Ids are minted once and are append-only. Never renumber, never reuse a retired id.
- On re-review, match each candidate feature against the prior `featureInventory[]` by name, parent
  capability, and code location — **in that order of precedence** — before minting a new id. A
  renamed feature keeps its id. A feature that moved files keeps its id.
- A feature no longer present in the code stays in the inventory with `state: "removed"`. Do not
  delete rows.
- `idHighWater` never decreases.

**Delta honesty.** Granularity drift between two reviews produces fake churn — a feature "split into
three" is a reviewer artifact, not a product change. If `grainRuleVersion` differs from the prior
review, say so in `deltas.summary` and do not report per-feature churn for that cycle. This is the
same discipline `rubric.md` already applies to score bands.

## 7. Doc sync

- `CONTRIBUTING.md` → *Add reuse profiles*: add feature inventory to "Existing hooks".
- `CONTRIBUTING.md` → *Keep things in sync*: name the new coupling (`featureInventory[]` ↔ CSV header
  ↔ `capabilityMap[].featureIds` ↔ the grain rule).
- `README.md`: add the CSV wherever outputs are listed, and state plainly that the skill does not
  assign business value.

## 8. Gating

`featureInventory` absent → no CSV, inventory column deleted from the technical report, founder
callout deleted, `meta.featureInventory` omitted. **Output is byte-comparable to 2.1.0.** That is the
acceptance test.

---

## Open questions for red-line

**Q1 — Is the inventory opt-in or always-on?** Always-on makes it a dependable artifact and every
review comparable, at the cost of real work on every run for repos where nobody wants it. Opt-in
risks an id space that only advances on the runs someone remembered to ask. I lean **always-on when
a prior review already has an inventory, opt-in otherwise** — once the id space exists, abandoning it
is what breaks.

**Q2 — Id prefix.** `F001` collides with the existing Techifuze document's ids, which is either
convenient or actively confusing depending on Q3. Alternative: `FI-001`. Cheap to decide now,
annoying later, because the ids are append-only forever.

**Q3 — What happens to the existing 94-feature document?** Three options, and this is the one I'd
most like your read on:

- **Discard.** The code-derived inventory supersedes it. Cleanest, throws away the value tagging and
  the eight-category structure, which are real work and genuinely outside the skill's competence.
- **Seed.** Hand-map the 94 existing rows onto the new inventory once, so the value tags and
  categories carry forward and rev ops starts from a filled sheet rather than a blank one. Best
  outcome, one-time manual cost.
- **Compare.** Run the new inventory against the 94 as a one-off analysis to find what the screenshot
  crawl missed and what it recorded that has no code behind it. Not a skill feature — a one-time
  document, and a good validation of whether the derived inventory is actually better.

Seed and compare are the same exercise done to different depth. I'd do **compare, then seed**.

**Q4 — Does `plainLanguage` on the inventory duplicate `capabilityMap[].plainLanguage`?** At 90 rows
against 17, the inventory prose is finer but will restate its parent a lot. Acceptable duplication,
or should inventory rows carry only a name and let the parent capability supply the description? I
lean keep it: the CSV has to stand alone when rev ops opens it away from the reports.

## Phase 2, explicitly out of scope

Reading an enriched inventory back in — value codes, sales categories, archetypes — and rendering the
coverage report the first draft described. Much cheaper once this ships, because the ids and the grain
are already the skill's own and there is nothing to match. Revisit after one enriched cycle exists,
not before; the shape of the enrichment rev ops actually produces should drive that schema, rather
than being guessed at now.

## Not in scope

- Backfilling the two prior reviews with an inventory. The first one is a `new` baseline, no deltas.
- Any change to `rubric.md`, the seven scorecard dimensions, or the score→traffic-light mapping.
- Assigning business value, in any form. That is the point of the rescope.
