---
name: threat-check
description: Given a proposed design change or new feature, identify which of the 14 threats in docs/threat-model.md are affected, whether existing mitigations still hold, and whether new gaps or accepted risks need to be recorded
---

When asked to threat-check a proposed change:

1. Read `docs/threat-model.md` in full to understand all 14 threats (T1–T14) and current mitigation status

2. For each threat, evaluate whether the proposed change:
   - **Expands the attack surface** for that threat (e.g., a new network port, a new credential type, a new container)
   - **Weakens an existing mitigation** (e.g., a change that bypasses network segmentation or logs a credential)
   - **Adds a new vector** not currently modeled (e.g., a new third-party dependency, a new external service)
   - **Has no impact** — skip it

3. For each affected threat, report:
   - The T-number and threat name
   - What specifically changes about the risk
   - Whether existing mitigations in the table still cover it or need updating
   - Whether a new row should be added to the mitigation table
   - Whether the change should be reflected in the "Summary: Gaps" table

4. Identify any completely new threat not covered by T1–T14 that the change introduces. If one exists, draft the new threat section following the existing format:
   - `### TN — Title`
   - Impact statement
   - Vectors list
   - Mitigation table with `| Mitigation | Severity | Status |` columns

5. Summarize: list affected threats, recommended doc updates, and any new threats to add.

Use the doc-decision skill conventions when suggesting edits to threat-model.md.
