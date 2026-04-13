# AGENTS.md

Guidance for coding agents working in this repository.

## Project Overview

**mailarchiver** is a self-hosted email archiving system. It pulls mail from multiple providers into a central Maildir archive, serves that archive back over IMAP, maintains encrypted backup copies, and only deletes provider-side mail after retention and backup-verification checks succeed.

The current documented direction is a multi-user login model: Dovecot login is delegated to an external OIDC/OAuth2 provider, and each human user authenticates with their own identity over OAuth-capable IMAP. The broader archive authorization and writable-IMAP model is still intentionally deferred.

The current architecture is documented, but implementation has **not** started yet. This repository is still primarily a design and execution-planning repo.

## Current State

- Source of truth is the documentation under `docs/`
- `TODOs.md` tracks unresolved gaps, ambiguities, and follow-up work
- Implementation work should follow `docs/implementation.md`
- `docs/definition-of-ready.md` and `docs/definition-of-done.md` are mandatory process gates, not optional references

## Architecture Summary

The system is designed around these major components:

1. `mbsync` / isync for ingest from Gmail, Outlook, Yahoo, and other providers
2. Maildir on a primary VPS as the permanent archive
3. Dovecot serving the archive over IMAPS to authorized Tailscale peers, with per-user OIDC/OAuth2 login via XOAUTH2/OAUTHBEARER
4. Local rsnapshot backup on the primary VPS, snapshotting the gocryptfs ciphertext directories
5. Offsite encrypted backups to Backblaze B2 and Cloudflare R2
6. A Python deletion worker with a SQLite manifest DB gating provider-side deletion
7. Observability via Prometheus, Loki, Grafana, Alertmanager, Grafana Alloy, Docker socket proxy, Pushgateway, and Node Exporter
8. At-rest encryption via gocryptfs + Clevis/Tang on the primary VPS, with Tang hosted on a secondary VPS

## Current Design Boundaries

- Dovecot login is defined: external OIDC/OAuth2, IMAP via XOAUTH2/OAUTHBEARER, one identity per human user
- Dovecot archive authorization is not yet defined: do not invent per-user namespace visibility, ownership, or sharing rules unless the task explicitly adds them
- Writable IMAP semantics are not yet defined: do not assume deletes, moves, appends, folder creation, or flag changes are already reconciled with the manifest, backups, or deletion worker
- Tang clients are limited to the primary VPS only; keep any new Tang-related guidance consistent with that ACL model

## Documentation Map

Read the relevant docs before making changes:

| File | Purpose |
|---|---|
| `README.md` | High-level project summary |
| `docs/concept.md` | Architecture diagram |
| `docs/architecture.md` | Core system design, encryption, retention, resilience, ACLs |
| `docs/tech-stack.md` | Container topology, networking, image pinning, operational stack decisions |
| `docs/observability.md` | Metrics, alerts, dashboards, access controls |
| `docs/cicd.md` | CI/CD design, validation checks, release/deploy pipeline |
| `docs/threat-model.md` | Threats, mitigations, status tracking, gap summary |
| `docs/recovery.md` | Disaster recovery and restore procedure |
| `docs/operations.md` | Day-2 operations and manual procedures |
| `docs/implementation.md` | Iteration plan and task acceptance criteria |
| `docs/definition-of-ready.md` | Preconditions before starting a task |
| `docs/definition-of-done.md` | Requirements before marking a task done |
| `TODOs.md` | Gap tracker; move items to `In Progress` during implementation, and only mark `Resolved` after user review |

## Working Rules

### 1. Docs-first changes

This project is highly cross-coupled at the documentation level. A change to one design doc often requires synchronized edits elsewhere.

When resolving a gap or changing a design decision, check whether the change also requires updates to:

- `docs/threat-model.md`
- `docs/implementation.md`
- `docs/observability.md`
- `docs/operations.md`
- `docs/recovery.md`
- `TODOs.md`

### 2. Keep status bookkeeping consistent

If you implement a documented gap or design decision:

- update the relevant design doc with the actual decision
- update any threat-model mitigation row from `Not yet defined` / `Not yet defined — gap` to `Defined in docs/X.md §N` when appropriate
- update `TODOs.md` status to `In Progress` while the change is awaiting review
- only after explicit user review/approval: update `TODOs.md` status to `Resolved` and populate `Resolved in`
- update the "Summary: Gaps Requiring New Decisions" table in `docs/threat-model.md` when an item is actually resolved or its remaining scope is narrowed

Do not mark follow-up items resolved unless the new text actually closes them.

### 3. Respect Ready / Done gates

Before implementation work begins, use `docs/definition-of-ready.md`.

Before declaring work complete, use `docs/definition-of-done.md`.

These files are part of the repo contract and should guide both planning and execution.

### 4. Preserve existing decisions

Do not casually override existing architectural choices. If you believe a prior decision should change, update all affected docs and clearly reflect the new state in the threat model and TODO tracker.

### 5. User direction overrides stale docs

If the user changes a requirement, treat that instruction as the new source of truth. Update the docs to match the user’s decision rather than arguing from older repository text.

## Local Agents and Skills

The repository already includes local Claude-oriented agents and skills under `.claude/`. They are still useful as project guidance even when another coding agent is working in the repo.

### Agents

| File | Purpose |
|---|---|
| `.claude/agents/security-reviewer.md` | Review security-sensitive code and design changes against the threat model |
| `.claude/agents/doc-consistency-checker.md` | Check cross-references, naming consistency, status vocab, and doc drift |

### Skills

| File | Purpose |
|---|---|
| `.claude/skills/doc-decision/SKILL.md` | Conventions for recording and updating design decisions |
| `.claude/skills/threat-check/SKILL.md` | Review proposed changes against the threat model |
| `.claude/skills/commit/SKILL.md` | Safe commit workflow with secret-guard intent |

### When to use them

- Use `security-reviewer` for:
  - Python worker changes
  - Docker Compose or Dockerfile changes
  - secret-handling changes
  - network exposure, Tailscale, firewall, Tang, or backup-path changes
- Use `doc-consistency-checker` after:
  - resolving a TODO item
  - changing threat-model statuses
  - changing section names or cross-references
  - updating task wording in `docs/implementation.md`
- Use `threat-check` when:
  - making a meaningful architecture change
  - altering trust boundaries
  - changing deletion safety, backup verification, encryption, or recovery behavior

## Authoring Conventions

- Top-level sections use numbered headers like `## 1. Title` where the existing doc uses that scheme
- Cross-references use `docs/X.md §N`
- Threat-model mitigation status values should stay in the existing vocabulary:
  - `Not yet defined`
  - `Not yet defined — gap`
  - `Defined in docs/X.md §N`
- Prefer concise, explicit policy statements over vague intent
- When documenting network policy, specify:
  - source role
  - destination role
  - protocol / port
  - whether the path is public, Docker-internal, or Tailscale-only

## Practical Guidance

- Prefer updating the smallest consistent set of docs rather than making a one-file fix that leaves contradictions behind
- If a doc says something is "the only" or "must", verify that other docs do not describe a conflicting path
- If a TODO item is resolved indirectly by another change, update the bookkeeping explicitly
- If a referenced local agent/skill is missing, do not invent it silently; note the gap

## Non-Goals For This File

`AGENTS.md` is repo guidance. It is not the place to restate the entire architecture in exhaustive detail. The detailed design remains in `docs/`.
