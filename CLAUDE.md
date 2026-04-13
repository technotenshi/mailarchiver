# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**mailarchiver** is a self-hosted email archiving system. It pulls email from multiple providers (Gmail, Outlook, Yahoo, and others) via IMAP/POP, stores a permanent copy in a central Maildir archive on a VPS/server, and maintains encrypted off-site backups. A deletion worker removes provider-side copies only after verifying retention and backup conditions are met.

## Architecture (from `docs/concept.md`)

The system has five layers:

1. **Email Ingest** — **mbsync (isync)** pulls from providers via IMAP (OAuth2 for Gmail/Outlook, app passwords for others)
2. **Primary Archive** — Central Maildir store on a cloud/VPS (Copy #1)
3. **IMAP Service** — Dovecot serves the archive to mail clients
4. **Local Backup** — rsnapshot of the gocryptfs ciphertext directories on the primary VPS (Copy #2)
5. **Offsite Backup** — Backblaze B2 and Cloudflare R2, both encrypted (Copies #3 and #4)

A **deletion worker** gates removal of email from provider accounts on successful retention checks and confirmed backup presence across all copies.

Key technical decisions: gocryptfs + Clevis/Tang for at-rest encryption; SQLite manifest DB (four-table schema: messages, provider_copies, backup_verifications, account_policy); Prometheus + Loki + Grafana + Alloy observability stack; Docker Compose v2 on a single VPS; Dovecot with OIDC/OAuth2 (XOAUTH2/OAUTHBEARER) for archive IMAP access.

## Status

Design documentation phase complete. No implementation exists yet. All architectural decisions are finalized in `docs/`. Next phase: implementation.

## Documentation Map

| Doc | Covers |
|---|---|
| `docs/concept.md` | Mermaid architecture diagram — **read-only**; reflects decisions recorded in other docs; update only when those docs change |
| `docs/architecture.md` | Encryption, deletion worker schema + algorithm, retention policy, error handling, auth, resilience, gocryptfs startup |
| `docs/tech-stack.md` | Container stack decisions, Docker Compose layout, network segmentation, image pinning |
| `docs/observability.md` | Metrics, alerts, Grafana/Loki/Prometheus/Alertmanager, external heartbeat |
| `docs/cicd.md` | GitHub Actions pipelines, image build/push, deploy procedure |
| `docs/threat-model.md` | 18 threats with mitigations and status; security gaps summary |
| `docs/recovery.md` | Disaster recovery procedure, RPO/RTO |
| `docs/operations.md` | Routine operations: add account, rotate credentials, health check, Tang key rotation |
| `docs/implementation.md` | 4-iteration implementation plan with per-task acceptance criteria |
| `docs/definition-of-ready.md` | Checklist: preconditions any task must meet before development starts |
| `docs/definition-of-done.md` | Checklist: criteria any task must meet to be considered done |
| `TODOs.md` | Living tracker of undefined/ambiguous/gap items across all docs; update when gaps are resolved |

## Claude Code Automations

| Type | File | Purpose |
|---|---|---|
| Agent | `.claude/agents/security-reviewer.md` | Reviews code against T1–T18 threat model |
| Agent | `.claude/agents/doc-consistency-checker.md` | Validates cross-references, status vocab, alert names |
| Skill | `.claude/skills/doc-decision/` | Claude-only: conventions for recording decisions |
| Skill | `.claude/skills/threat-check/` | Evaluates a proposed change against the threat model |
| Skill | `.claude/skills/commit/` | `/commit` — conventional commit with no-secrets guard |
| Hook | `.claude/settings.json` | Blocks edits to secrets files and sha256 digest lines; runs ruff on .py files |
| MCP | `.mcp.json` | context7 (live library docs), GitHub (issues/PRs/Actions) |

## Doc Authoring Conventions

- **Cross-doc changes**: resolving a gap or changing a design decision typically requires synchronized edits in the threat model, TODOs.md, architecture.md, implementation.md, and/or operations.md — check all of them before closing an item
- **Agents**: run `doc-consistency-checker` after any doc edit that changes section names or cross-references; run `security-reviewer` before closing any security-relevant gap (worker code, Docker config, secrets, network/firewall changes)
- Section headers: `## N. Title` (numbered) for top-level, `### Title` for subsections
- Cross-references: `docs/X.md §N` (e.g. `docs/architecture.md §7`)
- Decision tables: columns `| Mitigation | Severity | Status |`; status values: `Not yet defined`, `Not yet defined — gap`, `Defined in docs/X.md §N`
- Mermaid or ASCII diagrams for architecture visuals; Markdown tables for decisions
