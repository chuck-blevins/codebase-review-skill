---
created: 2026-07-29
type: spec
status: draft — awaiting red-line
topic: value-coverage reuse profile for the codebase-review skill
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

# Spec — value-coverage reuse profile

Adds a fourth output to the codebase-review skill: a reconciliation of a repo's **feature/value
repository** (a sales-side feature inventory) against what the code actually implements. Rendered as
a standalone report plus a CSV, both derived from `findings.json`.

Follows the documented extension path in `CONTRIBUTING.md` → *Add reuse profiles*: "new outputs are
just new renderings of it. To add a profile, write a new template that reads the same
`findings.json`."

## Decisions already locked

1. **Separate output file**, not folded into the founder or technical report.
2. **CSV emitted alongside** the HTML.
3. **`valueCoverage` stays the data key**; the file is named for its reader
   (`feature-coverage-report.html`).
4. **All coverage state lives in `findings.json`.** Renderings are projections. Versioning and
   deltas diff the JSON.
5. **Founder report keeps a mandatory one-sentence headline** with the number and its denominator,
   plus a link. Not a table, never zero.
6. **Scorecard is not touched.** Adding a dimension would break comparability with prior reviews,
   since `scorecard[].dimension` is a fixed enum that deltas key on.

## Before touching anything

`/Users/chuckblevins/SynologyDrive/03 Projects/llm_skills/codebase-review` **is not a git
repository** and has no version control. Copy the directory before any edit lands. Every change
below is additive and gated, but there is no undo.

## File manifest

| File | Change | Risk |
|---|---|---|
| `schema/findings.schema.json` | Additive: 1 new top-level object, 4 field groups | Low — all optional |
| `rubric.md` | New section + 1 bullet in an existing section | Medium — touches severity |
| `SKILL.md` | 7 edits incl. version bump and a renumbered step list | Low |
| `templates/technical-report.html` | 2 columns on an existing table | Low |
| `templates/founder-report.html` | 1 optional block + 1 CSS rule | Low |
| `templates/feature-coverage-report.html` | **New file** | New surface |
| `templates/feature-coverage.csv` | **New file** (header/grain contract) | New surface |
| `CONTRIBUTING.md`, `README.md` | Sync per *Keep things in sync* | Low |

---

## 1. `schema/findings.schema.json`

### 1a. `meta.valueRepository` — new, optional

Presence of this object is the **gate** for every other behavior in this spec.

```jsonc
"valueRepository": {
  "type": "object",
  "description": "Optional. A feature/value inventory found in the repo that this review reconciles against. When absent, no coverage outputs are emitted and the review behaves exactly as 2.1.0.",
  "required": ["path", "featureCount", "taxonomy"],
  "properties": {
    "path": { "type": "string", "description": "Repo-relative path to the source inventory." },
    "asOf": { "type": "string", "description": "Date the inventory reflects. Read from a date stated inside the document, or ask. Never infer from file mtime." },
    "featureCount": { "type": "integer", "description": "Total catalogued features. The denominator." },
    "taxonomy": {
      "type": "array",
      "description": "Read verbatim from the source document's legend. Never hardcode a project's codes into the skill.",
      "items": {
        "type": "object",
        "required": ["code", "meaning"],
        "properties": {
          "code": { "type": "string" },
          "meaning": { "type": "string" },
          "revenueAdjacent": {
            "type": "boolean",
            "description": "True for codes that gate revenue capture or retention. Drives the value-weighting rule in rubric.md. Set from the taxonomy's own wording and record the reasoning in methodology.approach."
          }
        }
      }
    },
    "categories": {
      "type": "array",
      "description": "Optional sales-facing grouping, if the source documents provide one.",
      "items": {
        "type": "object",
        "required": ["name"],
        "properties": {
          "name": { "type": "string" },
          "modules": { "type": "array", "items": { "type": "string" } },
          "featureIds": { "type": "array", "items": { "type": "string" } },
          "speaksTo": { "type": "array", "items": { "type": "string" }, "description": "Stakeholder archetypes, verbatim from the source. Renders only in the coverage report." }
        }
      }
    }
  }
}
```

### 1b. `valueCoverage` — new top-level, optional

```jsonc
"valueCoverage": {
  "type": "object",
  "description": "Reconciliation of the value repository against the code. Emitted only when meta.valueRepository is present.",
  "required": ["denominator", "headline", "features", "byCode"],
  "properties": {
    "denominator": {
      "type": "string",
      "description": "Plain-language statement of what every count is out of, e.g. '94 catalogued UI-observed features as of 2026-07-29'. Rendered verbatim anywhere a percentage appears."
    },
    "headline": {
      "type": "string",
      "description": "One sentence, founder voice, carrying the number AND its denominator. Rendered in the founder report. Required when this object exists."
    },

    "features": {
      "type": "array",
      "description": "THE GRAIN. One row per catalogued feature. byCode and byCategory are computed from this, and the CSV is a direct projection of it.",
      "items": {
        "type": "object",
        "required": ["featureId", "name", "valueCode", "state"],
        "properties": {
          "featureId": { "type": "string", "description": "Stable id from the source inventory, e.g. 'F033'." },
          "name": { "type": "string" },
          "module": { "type": "string", "description": "Source inventory's module grouping." },
          "category": { "type": "string", "description": "Matches a meta.valueRepository.categories[].name." },
          "valueCode": { "type": "string", "description": "Must be one of meta.valueRepository.taxonomy[].code." },
          "state": {
            "type": "string",
            "enum": ["implemented", "partial", "ui-only", "absent"],
            "description": "implemented = working code end to end. partial = some of it real, some not. ui-only = screen renders, no working backend. absent = catalogued but not found in code at all."
          },
          "capability": { "type": "string", "description": "Name of the capabilityMap[] entry this rolls up to." },
          "evidence": {
            "type": "array",
            "minItems": 1,
            "items": { "$ref": "#/$defs/evidence" },
            "description": "Cite-or-assume applies. For 'partial', cite both the working part and the gap. For 'ui-only', cite the mock/sample data source."
          },
          "confidence": { "type": "string", "enum": ["high", "medium", "low"] }
        }
      }
    },

    "byCode": {
      "type": "array",
      "description": "Computed from features[]. Integrity rule: implemented + partial + uiOnly + absent MUST equal catalogued on every row.",
      "items": {
        "type": "object",
        "required": ["code", "catalogued", "implemented", "partial", "uiOnly", "absent", "businessMeaning"],
        "properties": {
          "code": { "type": "string" },
          "catalogued": { "type": "integer" },
          "implemented": { "type": "integer" },
          "partial": { "type": "integer" },
          "uiOnly": { "type": "integer" },
          "absent": { "type": "integer" },
          "businessMeaning": { "type": "string", "description": "One plain-language sentence. Founder voice." }
        }
      }
    },

    "byCategory": {
      "type": "array",
      "description": "Same counts at the sales-facing grain. Omit when meta.valueRepository.categories is absent.",
      "items": {
        "type": "object",
        "required": ["name", "catalogued", "implemented", "partial", "uiOnly", "absent"],
        "properties": {
          "name": { "type": "string" },
          "catalogued": { "type": "integer" },
          "implemented": { "type": "integer" },
          "partial": { "type": "integer" },
          "uiOnly": { "type": "integer" },
          "absent": { "type": "integer" },
          "speaksTo": { "type": "array", "items": { "type": "string" } },
          "businessMeaning": { "type": "string" }
        }
      }
    },

    "uncataloguedCapabilities": {
      "type": "array",
      "description": "The reverse gap: working code the inventory does not catalogue. The only findings that flow back INTO the sales document.",
      "items": {
        "type": "object",
        "required": ["capability", "evidence"],
        "properties": {
          "capability": { "type": "string" },
          "plainLanguage": { "type": "string" },
          "evidence": { "type": "array", "minItems": 1, "items": { "$ref": "#/$defs/evidence" } },
          "suggestedValueCode": { "type": "string", "description": "Inferred. Mark basis accordingly." }
        }
      }
    },

    "developerQuestions": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Generated from features[] where state is ui-only or partial. Feeds the coverage report's closing section."
    }
  }
}
```

**Why `features[]` is the grain, and why it matters.** The 2026-07-13 review has 17 coarse
capabilities against 94 fine-grained features. One capability spans many features, and a capability
is frequently `partial` precisely because some of its features are real and others are not. If state
lived only on `capabilityMap[]`, the per-code counts would have to attribute one state to every
feature under that capability — which would be wrong, and the counts are the whole point. State
belongs at feature grain; the capability keeps a coarse rollup badge for the technical report.

Cost: roughly 94 small rows, ~15–20KB on an 86KB `findings.json`. Acceptable.

### 1c. `capabilityMap[]` — three additive fields

```jsonc
"featureIds": { "type": "array", "items": { "type": "string" }, "description": "Catalogued features this capability implements. Empty when the capability is uncatalogued." },
"implementationState": { "type": "string", "enum": ["implemented", "partial", "ui-only", "absent"], "description": "Coarse rollup of the featureIds' states, for the technical report badge. Computed, not authored: any ui-only or absent among them makes the rollup 'partial' at best." },
"uncatalogued": { "type": "boolean", "description": "Working code absent from the inventory. Kept as a separate flag rather than an implementationState value, because it answers a different question than 'how much of this is real'." }
```

### 1d. `finding` — three additive fields, one enum extension

```jsonc
"severityBase": { "type": "string", "enum": ["critical","high","medium","low","info"], "description": "Stage-anchored severity BEFORE value weighting. Version-over-version deltas compare this field, never `severity`." },
"severityBasis": { "type": "string", "enum": ["stage", "stage+value"], "description": "Which axes produced `severity`." },
"featureIds": { "type": "array", "items": { "type": "string" }, "description": "Catalogued features this finding touches. Drives value weighting." }
```

Extend `finding.category` enum with `"value-coverage"`.

### 1e. `recommendation` — two additive fields

```jsonc
"unblocksFeatureIds": { "type": "array", "items": { "type": "string" } },
"valueCodes": { "type": "array", "items": { "type": "string" }, "description": "Derived from unblocksFeatureIds. Drives ordering within each recommendation bucket." }
```

---

## 2. `rubric.md`

### 2a. New section, inserted after *Stage calibration (the anti-drift rule)*

> ## Value weighting (third axis, optional)
>
> Applies **only** when `meta.valueRepository` is present. The first two axes (what could happen ×
> deployment stage) always run first and produce `severityBase`.
>
> - A finding whose `featureIds` include any feature tagged with a **revenue-adjacent** code
>   (`taxonomy[].revenueAdjacent: true`) escalates **one** band from `severityBase`.
> - Escalate **at most one band, at most once**, however many such features are touched.
> - Never escalate above `critical`. Never escalate `info`.
> - Codes that are purely internal-efficiency never escalate.
>
> Record both: `severityBase` (stage-anchored), `severity` (post-escalation), and `severityBasis`.
>
> This axis is mechanical on purpose. It reads a tag off a document; it is not a judgment call. If
> applying it requires a judgment, the taxonomy's `revenueAdjacent` flags are wrong — fix those.

### 2b. New bullet in *Comparing across versions (score bands)*

> - **Compare `severityBase`, never `severity`.** If `severity` moved but `severityBase` did not,
>   the sales taxonomy changed, not the code. Do not log a delta. A document edit must never
>   manufacture a code-review delta.

---

## 3. `SKILL.md`

**3a. Frontmatter.** `version: 2.1.0` → `2.2.0`. Append to `description`: "Optionally reconciles a
repo's feature/value inventory against the code to report which catalogued features are backed by
working code."

**3b. *Core principle: findings first, reports second*** — add a fourth item:

> 4. **`feature-coverage-report.html` + `feature-coverage.csv`** — emitted only when a feature/value
>    inventory exists in the repo. Answers "what can we safely claim in a deal": which catalogued
>    features are backed by working code, which are demo-only, and which working capabilities the
>    inventory misses. Renders from `templates/feature-coverage-report.html`.

**3c. *Output location & versioning convention*** — extend the tree:

```
<repo>/documents/code-review/<YYYY-MM-DD>_<shortSHA>/
    findings.json
    founder-report.html
    technical-report.html
    feature-coverage-report.html    # only when a value repository was found
    feature-coverage.csv            # only when a value repository was found
```

**3d. *How to review*** — insert as new step 5, renumber old 5–10 to 6–11:

> 5. **Reconcile against a feature/value inventory, if one exists.** Look in `documents/` for a
>    feature inventory: a table with stable feature ids, a value-category legend, and a value tag per
>    feature. If found, record it in `meta.valueRepository` and resolve **every** catalogued feature
>    id to exactly one `state` (`implemented` / `partial` / `ui-only` / `absent`) with evidence.
>    Then compute `byCode` and `byCategory`, and record the reverse gap in
>    `uncataloguedCapabilities`. Prefer the machine-readable form of the inventory (`.md`/`.csv`)
>    over a `.docx`/`.xlsx` mirror of the same data.

**3e. *Non-negotiable rules*** — two new bullets:

> - **Cite-or-assume applies to coverage.** Never mark a catalogued feature `implemented` without
>   `file:line` evidence. `partial` requires evidence of both the working part and the gap;
>   `ui-only` requires evidence of the mock or sample-data source.
> - **Every coverage percentage states its denominator.** Render
>   `valueCoverage.denominator` verbatim alongside any percentage. A bare percentage reads as full
>   product coverage when it only ever describes the catalogued subset.

**3f. *Reuse profiles*** — add:

> - **Feature coverage** — populate `meta.valueRepository` and `valueCoverage`; renders a
>   sales/GTM-facing report plus a CSV at feature grain for filtering and pivoting.

**3g. *Output format*** — add:

> - `feature-coverage.csv` is a mechanical projection of `valueCoverage.features[]`. Regenerate it;
>   never hand-edit it, and never let it drift from `findings.json`.

---

## 4. `templates/technical-report.html`

Two columns on the existing capability-map table. No new section, no TOC entry.

Line 94 — `<thead>`:
```html
<thead><tr><th>Capability</th><th>Description</th><th>Code locations</th><th>Basis</th>
  <!-- COVERAGE COLS (delete both th and their td when valueCoverage is null) -->
  <th>Features</th><th>State</th></tr></thead>
```

Lines 97–102 — row body, appended before `</tr>`:
```html
        <!-- COVERAGE COLS -->
        <td class="fid"><!-- REPEAT featureIds[] -->{{featureId}} <!-- /REPEAT --></td>
        <td class="state-{{implementationState}}">{{implementationState}}</td>
```

Add CSS for `.state-implemented`, `.state-partial`, `.state-ui-only`, `.state-absent` matching the
existing severity-chip treatment.

## 5. `templates/founder-report.html`

One optional block after line 91 (`<p class="lead">{{executiveSummary}}</p>`):

```html
  <!-- ============ COVERAGE CALLOUT (delete whole block if valueCoverage is null) ============ -->
  <p class="coverage-callout">
    {{valueCoverage.headline}}
    <a href="feature-coverage-report.html">Full feature coverage breakdown &rarr;</a>
  </p>
```

`{{valueCoverage.headline}}` is required when the block renders — that is the mechanism enforcing
locked decision 5. Add one CSS rule (left rule, tinted background, matching `.lead` type scale).

Nothing else in this template changes. Scorecard untouched.

## 6. `templates/feature-coverage-report.html` — new

Same self-contained inline-CSS pattern and the founder report's **voice** (this is a business
document, not a technical one). Print-friendly.

| # | Section | Content |
|---|---|---|
| 1 | Masthead | `{{repoName}} — Feature Coverage`, sub: "What the code backs · sales & GTM" |
| 2 | Stamp | Commit, review date, `{{meta.valueRepository.path}}`, `asOf`, reviewer |
| 3 | Headline | `{{valueCoverage.headline}}` + `{{valueCoverage.denominator}}` |
| 4 | By category | Category · catalogued · working · partial · demo-only · absent · **speaks to**. Omit if `byCategory` absent. |
| 5 | By value code | Same counts per taxonomy code, with `meaning` and `businessMeaning` |
| 6 | Demo-only detail | `features[]` filtered to `ui-only`/`absent`, grouped by category: id, name, value code, evidence. The section a founder acts on. |
| 7 | Uncatalogued capabilities | `uncataloguedCapabilities[]` — working code nobody is selling |
| 8 | Questions for your developers | `developerQuestions[]`, generated not authored |
| 9 | Methodology footer | Denominator restated, `methodology.limitations`, inventory path and date |

Sort rule: sections 4 and 5 order by demo-only count descending, so the worst exposure reads first.

## 7. `templates/feature-coverage.csv` — new

Header contract. Grain is **one row per catalogued feature** — the grain sales ops can pivot, which
is why no summary CSV is needed.

```csv
feature_id,feature_name,module,category,value_code,state,capability,code_locations,evidence,confidence,speaks_to
```

- Direct projection of `valueCoverage.features[]`, joined to `categories[].speaksTo` on `category`.
- UTF-8, RFC 4180 quoting, `;`-joined multi-values inside a quoted cell.
- Row order matches `features[]` (source-inventory id order) so two versions diff cleanly.

## 8. Doc sync

- `CONTRIBUTING.md` → *Add reuse profiles*: add feature coverage to the "Existing hooks" list.
- `CONTRIBUTING.md` → *Keep things in sync*: name the new coupling (taxonomy in `findings.json` ↔
  three templates ↔ CSV header).
- `README.md`: add the two new outputs wherever it lists what the skill emits.

## 9. Gating — behavior when no inventory exists

With `meta.valueRepository` absent: no coverage files written, coverage columns deleted from the
technical report, founder callout block deleted, no severity escalation (`severityBasis: "stage"`),
`valueCoverage` omitted. **Output is byte-comparable to 2.1.0.** This is the acceptance test for the
change.

---

## Open questions for red-line

**Q1 — Is `TRUST` revenue-adjacent?** This sets how much escalates. `REV-ACCEL` (3) + `REV-GROWTH`
(1) alone is 4 of 94 features, so value weighting would almost never fire. Adding `TRUST` (5) makes
it 9 of 94. I lean yes: `TRUST` is the retention and close mechanism, and F064 (escrow explainer) and
F062 (verified profile) are exactly the features a skeptical buyer tests. But it is your call, and
it should be recorded in `methodology.approach` either way.

**Q2 — `asOf` for the Techifuze inventory.** All three files have an mtime of 2026-07-29 because
they were just added, and the `.md` states no date. The spec forbids inferring from mtime. Options:
ask at review time, or add a date line to the inventory document itself. The second is better and
costs one line in a document you control.

**Q3 — Does the `.docx` get parsed?** `categories[]` and `speaksTo` only exist in the category guide
`.docx`. Parsing it is real work for one table and one column. Alternative: keep the eight categories
and archetypes in a small sidecar `.md`/`.yml` in `documents/`, hand-maintained, and have the skill
read that. Cheaper and more portable, at the cost of a second thing to keep current.

**Q4 — `uncatalogued` as a separate boolean.** I split it from `implementationState` because it
answers a different question. The alternative is a fifth enum value, which is simpler to render but
conflates two axes. Low stakes, but it is a schema shape you will live with.

**Q5 — Does `features[]` belong under `valueCoverage`, or as its own top-level array?** Under
`valueCoverage` keeps one coherent object. Top-level (`featureCoverage[]`) reads better if it grows
independently. I chose nesting; reversing it later is a breaking schema change, so it is worth
deciding now.

## Not in scope

- Backfilling the two prior reviews (`2026-07-09_2787804`, `2026-07-13_1ee4fbd`) with coverage data.
  They stay as-is; the first coverage report is a `new` baseline with no deltas.
- Any change to the seven scorecard dimensions or the score→traffic-light mapping.
- Reconciling the inventory's own accuracy. The skill takes the inventory as given and reports
  against it. If a catalogued feature is described wrongly, that is a finding, not a correction.
