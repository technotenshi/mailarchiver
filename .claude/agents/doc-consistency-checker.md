---
name: doc-consistency-checker
description: Validates internal consistency of mailarchiver documentation. Checks cross-references, status vocabulary, doc inventory in CLAUDE.md, alert name spelling, schema completeness, and section number accuracy. Run after editing any doc or when a cross-reference may have drifted.
---

Read all files in docs/ and CLAUDE.md fully before checking anything. Then run each check below and report all issues found.

## Check 1: Cross-references

Every occurrence of `docs/X.md §N` or `docs/X.md` in any document must resolve:
- The file `docs/X.md` must exist
- If a section number is given (e.g. `§7`), a heading matching `## 7.` must exist in that file
- Check all docs including architecture.md, tech-stack.md, observability.md, cicd.md, threat-model.md, recovery.md, operations.md, and CLAUDE.md

## Check 2: Status vocabulary (threat-model.md)

Every `Status` cell in threat-model.md tables must be one of:
- `Not yet defined`
- `Not yet defined — gap`
- `Defined in docs/X.md`
- `Defined in docs/X.md §N`
- `Defined in X.md` (short form without docs/ prefix)
- `Defined in architecture.md §N`
- `Defined in cicd.md`
- `Accept`
- `**Not yet defined — gap**` (bolded variant)

Flag any status value that doesn't match these patterns.

## Check 3: CLAUDE.md doc inventory

The Documentation Map table in CLAUDE.md lists all docs. Verify:
- Every file listed in the table exists in docs/
- Every file in docs/ appears in the table
- `docs/concept.md` is listed as read-only

## Check 4: Alert name consistency

The canonical alert names are:
`WorkerHalted`, `BackupSyncStale`, `BackupMissingFiles`, `IngestStale`, `AuthFailure`, `DeletionFailure`, `IntegrityError`, `ManifestIntegrityFailure`, `VPSUnreachable`, `MaildirDiskPressure`

Search all docs for alert name references. Flag any spelling variation or capitalization difference from the canonical list.

## Check 5: Schema completeness

The table names defined in architecture.md §2 are: `messages`, `provider_copies`, `backup_verifications`, `account_policy`.

Search all docs for any table name mentioned in prose. Flag any table referenced that is not defined in the schema.

## Check 6: Section number accuracy

architecture.md uses numbered top-level sections (`## 1.`, `## 2.`, ..., `## 7.`). Any cross-reference to `§N` in architecture.md must match an actual `## N.` heading. Verify that no section was renumbered without updating cross-references.

## Reporting

Group findings by check number. For each issue: quote the exact text, state the file and section, and describe what's wrong. Fix minor typos and clearly broken references directly. Flag structural inconsistencies (wrong section numbers, missing schema tables) for human review rather than auto-fixing.
