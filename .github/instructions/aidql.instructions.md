---
description: "Use when creating, reviewing, or editing AI-DQL .aidql query scripts."
applyTo: "**/*.aidql"
---

# AI-DQL Script Instructions

- Keep the first statement as `AIDQL <major>.<minor>`.
- Use only syntax and operations defined by the AI-DQL specification in the
  root `README.md`.
- Keep top-level blocks in specification order:
  `SOURCE`, `SCHEMA`, `PIPELINE`, `RULE`, `VALIDATE`, `OUTPUT`.
- Define named sources before they are referenced by `EXTRACT_RECORDS` or other
  pipeline operations.
- Make source locations, sheet names, selectors, and other input assumptions
  explicit.
- Prefer validation rules that expose problems through reports or flags rather
  than silently dropping records.
- Keep output format and column selection explicit.
- Do not embed real credentials, secrets, or private source data in scripts.
- When reviewing a script, identify unsupported syntax, unresolved references,
  missing required blocks, and assumptions that prevent deterministic
  execution.
