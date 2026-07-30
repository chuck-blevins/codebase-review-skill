---
created: 2026-07-09
type: note
topic: codebase-review synthetic example output
domains:
  - software-engineering
  - ai-ml
concepts:
  - agentic-ai
tags:
  - notes
  - code-review
  - agent-skill
  - example-output
  - findings-json
---

# Example output

A synthetic review of **Larkspur**, a fictional team-wiki SaaS. Nothing here is a real product,
company, or codebase — it exists only to show the shape of what the skill produces.

- **`findings.json`** — the structured source of truth (trimmed: a real run has more findings and
  diagrams).
- **`founder-report.html`** — the plain-language founder/board summary rendered from it.
- **`technical-report.html`** — the evidence-cited technical report rendered from it.

Open the two HTML files in a browser to see the reports. The technical report loads Mermaid from a
CDN to draw diagrams, so that view needs internet access.

A real review writes these three files to `<repo>/documents/code-review/<date>_<sha>/`.
