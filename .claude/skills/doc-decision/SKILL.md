---
name: doc-decision
description: Record a new architectural decision, security gap, or resolved decision in the appropriate doc following project conventions (status vocabulary, cross-reference format, table structure)
user-invocable: false
---

Use these conventions when recording any decision or gap in the mailarchiver docs.

## Cross-reference format

Always use: `docs/X.md §N` (e.g. `docs/architecture.md §7`, `docs/tech-stack.md`)

Never use relative paths, anchor links, or short forms without the `docs/` prefix in prose.

## Status values (threat-model.md tables)

| Value | When to use |
|---|---|
| `Not yet defined` | Identified requirement, not yet addressed |
| `Not yet defined — gap` | Identified security gap requiring action before deploy |
| `Defined in docs/X.md §N` | Decision captured; link to the section |
| `Accept` | Accepted risk with explicit rationale in the Accepted Risks section |

## Severity levels

`Critical` → `High` → `Medium` → `Low`

## Adding a new threat to threat-model.md

1. Assign the next T-number in sequence (read the file to find the last one)
2. Add a `### TN — Title` section with Impact, Vectors, and a mitigation table with columns `| Mitigation | Severity | Status |`
3. If any mitigations are `Not yet defined — gap`, add a row to the "Summary: Gaps Requiring New Decisions" table at the bottom
4. If it's an accepted risk, add a row to the "Accepted Risks" table

## Adding a resolved decision to architecture.md

1. Add a bullet to the "Resolved Decisions" section at the bottom of the file
2. If it was in "Open Decisions", remove it from there
3. Format: `- **Decision name** — one-line summary. See \`docs/X.md\`.`

## Adding an operations procedure to operations.md

Follow the existing section structure:
- Numbered `## N. Title` heading
- One-sentence context line
- Numbered steps with fenced code blocks for commands
- Note any prerequisites at the top of the section

## Before editing any doc

Always read the full target file first to:
- Find the correct insertion point
- Confirm current section numbering (architecture.md §N must match actual headings)
- Avoid duplicating an existing decision
