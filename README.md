---
created: 2026-07-09
type: note
topic: codebase-review agent skill overview
domains:
  - software-engineering
  - ai-ml
concepts:
  - agentic-ai
  - governance-in-practice
  - adversarial-thinking
tags:
  - notes
  - code-review
  - agent-skill
  - static-analysis
  - findings-json
---

# codebase-review

An agent **skill** that reviews a local git repository and produces a versioned,
stakeholder-ready assessment: what the code does, how it's built, and where the risks are —
written so a non-engineering founder can act on it and an engineer can verify every claim.

It is designed for the situation where a team has an existing (often AI-assisted) codebase and
isn't fully sure what it does, how healthy it is, or what to fix first.

## What it produces

One review run writes four artifacts to `<repo>/documents/code-review/<date>_<sha>/`:

| File | Audience | Contents |
| --- | --- | --- |
| `findings.json` | machine / source of truth | Every finding, score, capability, diagram, and recommendation as structured data. Everything else renders from this. |
| `founder-report.html` | founders / board | Plain-language traffic-light scorecard, "what this means for you" per finding, version deltas, glossary, questions to ask your team. Print-to-PDF friendly. |
| `technical-report.html` | dev team / auditors | Evidence-cited findings (`file:line`), tech stack, architecture diagrams, capability→code map, data model, cross-cutting concerns, AI-provenance signals, OWASP mapping. |
| `feature-inventory.csv` | revenue / product | Every user-facing feature and flow the reviewer found in the code, with what it does, where it lives, and whether it is backed by working code or only renders from mock data. Value and positioning columns are emitted blank, for a human to fill. |

Because the findings are structured, the same review re-renders into other formats (audit package,
how-to docs) and **diffs across versions** so you can track a codebase over time.

## What the feature inventory is for

The CSV is the reviewer's **perception of the codebase**, not a canonical feature list. Nothing is
supplied to it to match against, so there is no right answer about what counts as a feature. It does
two jobs. It lets you hold the AI's list against your own understanding of the product and find where
they disagree — features you didn't know were built, features you believed were built that the code
does not support. And it gives whoever owns positioning a populated sheet to start from, with
implementation states and code citations already filled in, instead of a blank one.

**The skill never assigns business value.** Value category, business-value prose, sales grouping, and
buyer archetype are emitted as empty columns. That judgment belongs to a human, and a plausible guess
is worse than a blank: a blank gets filled, a guess gets shipped.

**The handoff is one-way.** Whatever you build from the CSV is yours. The skill does not read it back,
reconcile against it, or update it.

## Why several outputs from one source

`findings.json` is the single source of truth. The two HTML reports and the CSV are just *renderings*
of it. That split is what makes the output **versionable** (diff the JSON, not the HTML) and
**reusable** (re-target the same findings without re-scanning the code).

## Requirements

- An agent host that supports skills (e.g. Claude Code / the Claude Agent SDK). This is a
  prompt-and-template skill; there is no code to install and no dependencies to build.
- A local **git** repository to review (the skill reads the commit SHA/date for versioning).
- Optional: **Node** or any JSON tool to validate `findings.json`, and a browser to view the
  reports. The technical report renders Mermaid diagrams from a CDN, so viewing diagrams needs
  internet — or switch diagrams to the documented ASCII fallback for fully offline/air-gapped use.

## Install

Copy this folder into wherever your agent host discovers skills, then invoke it by name.

```text
your-skills-dir/
  codebase-review/        <- this folder (the directory name is up to you)
    SKILL.md
    rubric.md
    schema/findings.schema.json
    templates/founder-report.html
    templates/technical-report.html
    templates/feature-inventory.csv
    examples/             <- synthetic sample so you can see the output shape
```

> The skill's declared `name:` is `codebase-review`; the folder may be named anything.

## Use

Point your agent at a repository and ask it to run the codebase review, e.g.:

> "Run the codebase-review skill against this repo."

The skill will scan the code, score each dimension against `rubric.md`, assemble `findings.json`,
and render both reports plus the feature inventory. On a repo it has reviewed before, it fills in the
version-over-version `deltas` automatically.

The default output location is `<repo>/documents/code-review/<date>_<sha>/`. Change the path in
`SKILL.md` if your project stores docs elsewhere.

**Nothing is overwritten.** Re-running a review at the same date and commit emits a new version
alongside the old one — `findings.v2.json`, `feature-inventory.v2.csv`, and so on — so an earlier run
is never lost. Diffing two versions of the inventory is how you see what changed.

Documents *you* put in `documents/` are read-only to the skill. It will not parse them, reconcile
against them, or edit them. That includes anything built from an exported inventory: an enriched sheet,
a value repository, a positioning deck. Ask it to revise one and it writes a new version alongside and
leaves your original alone.

## Files

- **`SKILL.md`** — the instructions the agent follows (process, rules, voice, versioning).
- **`rubric.md`** — the fixed 0–5 scoring definitions that make reviews comparable across versions.
- **`schema/findings.schema.json`** — the contract for `findings.json`.
- **`templates/`** — the two HTML report scaffolds and the feature-inventory CSV header contract.
- **`examples/`** — a synthetic sample review of a fictional app.

## Important limitations

- **This is an AI-generated assessment. Verify before you rely on it.** The skill requires every
  finding to cite `file:line` evidence or be marked as an inference precisely so a human can check
  it. Treat scores and claims as a well-organized starting point for a human reviewer, not a
  certified audit.
- A default run is **static** — it reads source, it does not execute the app. Runtime, visual/UX,
  accessibility, and performance issues are out of scope unless you extend it.
- Severity is calibrated to the codebase's stage; a "medium" in a prototype may be a "critical" in
  production. The report states its own scope and limitations — read them.
- The feature inventory is one reading of the code, not a definitive product catalogue. Where it draws
  the line between "one feature" and "three" is a judgment call, and it will differ from yours. That is
  the point — the disagreements are where you learn something. Treat the row count as a denominator for
  that conversation, not as a fact about the product.

## License

MIT — see [LICENSE](LICENSE).
