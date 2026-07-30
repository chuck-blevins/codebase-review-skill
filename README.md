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

One review run writes three artifacts to `<repo>/documents/code-review/<date>_<sha>/`:

| File | Audience | Contents |
|------|----------|----------|
| `findings.json` | machine / source of truth | Every finding, score, capability, diagram, and recommendation as structured data. Everything else renders from this. |
| `founder-report.html` | founders / board | Plain-language traffic-light scorecard, "what this means for you" per finding, version deltas, glossary, questions to ask your team. Print-to-PDF friendly. |
| `technical-report.html` | dev team / auditors | Evidence-cited findings (`file:line`), tech stack, architecture diagrams, capability→code map, data model, cross-cutting concerns, AI-provenance signals, OWASP mapping. |

Because the findings are structured, the same review re-renders into other formats (audit package,
how-to docs) and **diffs across versions** so you can track a codebase over time.

## Why two reports from one source

`findings.json` is the single source of truth. The two HTML reports are just *renderings* of it.
That split is what makes the output **versionable** (diff the JSON, not the HTML) and **reusable**
(re-target the same findings without re-scanning the code).

## Requirements

- An agent host that supports skills (e.g. Claude Code / the Claude Agent SDK). This is a
  prompt-and-template skill; there is no code to install and no dependencies to build.
- A local **git** repository to review (the skill reads the commit SHA/date for versioning).
- Optional: **Node** or any JSON tool to validate `findings.json`, and a browser to view the
  reports. The technical report renders Mermaid diagrams from a CDN, so viewing diagrams needs
  internet — or switch diagrams to the documented ASCII fallback for fully offline/air-gapped use.

## Install

Copy this folder into wherever your agent host discovers skills, then invoke it by name.

```
your-skills-dir/
  codebase-review/        <- this folder (the directory name is up to you)
    SKILL.md
    rubric.md
    schema/findings.schema.json
    templates/founder-report.html
    templates/technical-report.html
    examples/             <- synthetic sample so you can see the output shape
```

> The skill's declared `name:` is `codebase-review`; the folder may be named anything.

## Use

Point your agent at a repository and ask it to run the codebase review, e.g.:

> "Run the codebase-review skill against this repo."

The skill will scan the code, score each dimension against `rubric.md`, assemble `findings.json`,
and render both reports. On a repo it has reviewed before, it fills in the version-over-version
`deltas` automatically.

The default output location is `<repo>/documents/code-review/<date>_<sha>/`. Change the path in
`SKILL.md` if your project stores docs elsewhere.

## Files

- **`SKILL.md`** — the instructions the agent follows (process, rules, voice, versioning).
- **`rubric.md`** — the fixed 0–5 scoring definitions that make reviews comparable across versions.
- **`schema/findings.schema.json`** — the contract for `findings.json`.
- **`templates/`** — the two HTML report scaffolds.
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

## License

MIT — see [LICENSE](LICENSE).
