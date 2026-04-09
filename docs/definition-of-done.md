# mailarchiver — Definition of Done

A task is **done** when every item in the checklist below is true. Marking a task Done without satisfying all items is not permitted.

This checklist applies to every task in `docs/implementation.md`, regardless of iteration or track.

---

## 1. Code Quality

- [ ] All CI checks pass on the branch: `ruff check`, `mypy`, `bandit`, `pytest`, Trivy image scan, `promtool check rules`, `amtool check-config`, `docker compose config`
- [ ] No new lint errors, type errors, or test failures introduced anywhere in the repo

## 2. Tests

- [ ] Tests were written before implementation code (TDD — failing test first)
- [ ] All tests pass; no tests skipped without documented justification
- [ ] If this task touches the deletion worker: HALT conditions and state-transition paths have test coverage

## 3. Acceptance Criterion

- [ ] The "Done when" criterion from `docs/implementation.md` has been met
- [ ] The criterion has been verified manually by running the specified command or performing the specified action — not assumed from a passing test

## 4. Security

- [ ] If any of the following were changed, the `security-reviewer` agent has reviewed the diff before the task is closed:
  - Python worker code (`worker/`)
  - Docker Compose file or Dockerfiles
  - Secrets handling or credential flows
- [ ] No new `Not yet defined — gap` mitigations were introduced in any doc as a result of this task
- [ ] If a meaningful architectural change was made, the `threat-check` skill was consulted to confirm no threat model entries are newly unmitigated

## 5. Documentation

- [ ] If a design decision was made during implementation that was previously "Not yet defined" or "Open Decision": the relevant doc has been updated to `Defined in docs/X.md §N`
- [ ] The corresponding `TODOs.md` item (if any) has been updated to `Resolved` with a "Resolved in" reference
- [ ] If docs were modified, the `doc-consistency-checker` agent was run to confirm cross-references are accurate

## 6. Operations

- [ ] If a new operational procedure or command was introduced: it is documented in `docs/operations.md`
- [ ] If a new Docker secret was added: it is listed in `docs/tech-stack.md §Secrets`
- [ ] If a new alert was added: it is listed in `docs/observability.md §Alert Mapping` with its signal type and condition

## 7. Cleanup

- [ ] No commented-out code or leftover debug logging in committed files
- [ ] No placeholder values, hardcoded paths, or `TODO` comments in production code
- [ ] No secret values, tokens, or credentials in any committed file
- [ ] No `@latest` or mutable tag image references introduced in Compose or Dockerfiles
