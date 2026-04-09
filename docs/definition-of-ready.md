# mailarchiver — Definition of Ready

A task is **ready to start** when every item in the checklist below is true. If any item cannot be checked, resolve it before picking up the task.

This checklist applies to every task in `docs/implementation.md`, regardless of iteration or track.

---

## 1. Design and Documentation

- [ ] The design doc for this task area has been read in full (see `docs/` mapping in `CLAUDE.md`)
- [ ] All "Not yet defined" or "Open Decision" items in the relevant docs that directly affect this task are resolved and recorded
- [ ] No `TODOs.md` item marked Critical or High is left Open that would block correct implementation of this task
- [ ] The task has a clear, verifiable "Done when" criterion (as written in `docs/implementation.md`)

## 2. Dependencies

- [ ] Every task listed as a prerequisite in `docs/implementation.md` is in `Done` state
- [ ] Required infrastructure is provisioned (VPS instances, Tailscale tailnet, cloud buckets as applicable)
- [ ] All secrets and credentials needed for this task are available in the Docker secret store, or a concrete plan to obtain them exists before the first run

## 3. Security

- [ ] Relevant threat model entries (T1–T18 in `docs/threat-model.md`) for this task area have been read
- [ ] If this task involves any of the following, the `security-reviewer` agent has been consulted on the design before coding begins:
  - Python worker code
  - Docker Compose changes
  - Secrets handling or credential flows
  - Network configuration (port mappings, container networks)

## 4. Testing

- [ ] At least one failing test can be written before any implementation code is written (TDD first principle)
- [ ] The acceptance criterion is observable and verifiable by a specific command or action — not subjective

## 5. CI

- [ ] CI is green on `main` before the branch is created
- [ ] The working branch is up to date with `main`
