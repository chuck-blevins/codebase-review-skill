---
created: 2026-07-09
type: note
topic: customizing the codebase-review skill
domains:
  - software-engineering
  - ai-ml
concepts:
  - agentic-ai
  - content-governance
tags:
  - notes
  - code-review
  - agent-skill
  - rubric
  - json-schema
  - mermaid
---

# Contributing & customizing

This skill is meant to be adapted. The most common changes:

## Adjust the scoring rubric
`rubric.md` defines each scorecard dimension 0–5 and the score→traffic-light mapping. Keep the
definitions **fixed** within a project — that's what makes reviews comparable across versions. If
you retune a dimension, re-review prior versions or note the rubric change so old and new scores
aren't compared naively.

## Add or change scorecard dimensions
Dimensions live in `schema/findings.schema.json` (`scorecard[].dimension`) and are echoed in both
report templates. If you add one (e.g. "Accessibility", "Data privacy"), add a rubric definition
for it too, or the scores won't be reproducible.

## Add reuse profiles
`findings.json` is the source of truth; new outputs are just new renderings of it. Existing hooks:
- **Audit package** — populate `auditMappings` (OWASP / SOC 2 / GDPR) and lead the technical report
  with the control table.
- **How-tos** — `workflows[].steps` are written to double as step-by-step user guides.
- **Feature inventory** — `featureInventory[]` projects to `feature-inventory.csv` at feature and flow
  grain, with the value/positioning columns emitted blank for a revenue owner to fill offline.
To add a profile, write a new template that reads the same `findings.json`.

**The enrichment loop is deliberately one-way.** Whatever a human produces from the inventory CSV is an
artifact the skill does not read back. Do not add a profile that consumes it. Reading enriched documents
in would make the review's output depend on positioning judgment and require reconciling two
independently-authored artifacts on every run — which is the design this feature replaced. If you want
that, fork it; do not bolt it on here.

## Tune the voice
The founder report uses a deliberate register (concise product/SaaS leader; no "it's not X, it's Y"
antithesis, no metaphors, sparing em-dashes). The rules and before/after examples are in the
`## Voice` section of `SKILL.md`. Change them there, not per-report.

## Mermaid diagram gotchas
Diagrams are Mermaid source stored in `findings.json` and rendered in the technical report.
**Validate every diagram** before shipping a review. Two failures pass casual reading:
- A `;` inside a `sequenceDiagram` message — it's a statement separator. Use a comma.
- A flowchart node label that starts with `[/` — it parses as a parallelogram shape. Reword it.
For fully offline reports, replace the CDN-rendered Mermaid with the documented ASCII fallback.

## Reconcile stale docs
If the target repo already has architecture docs, check them against the current code and correct
anything that has drifted (e.g. a role that has since grown a full portal) rather than copying it
forward. The review should reflect the code as it is now.

## Keep things in sync
`STEP_ORDER`-style constants, the scorecard dimensions, the schema, and the templates reference each
other. When you change one, check the others. Validate `schema/findings.schema.json` and any
`findings.json` with a JSON tool before shipping.
