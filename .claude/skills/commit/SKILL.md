---
name: commit
description: Stage and commit changes with a conventional commit message (feat/fix/docs/chore/security). Summarizes what changed and why. Never stages secrets or credential files.
disable-model-invocation: true
---

1. Run `git diff --stat` and `git status` to see what changed
2. Draft a conventional commit message:
   - `docs:` for documentation-only changes
   - `feat:` for new functionality
   - `fix:` for bug fixes
   - `security:` for threat model or security configuration changes
   - `chore:` for tooling, hooks, agents, CI config
   - Subject line: imperative mood, under 72 chars, no period
   - Body: one sentence on *why*, not *what* (the diff shows what)
3. Stage only relevant files — never stage: `.env`, `*secret*`, `*token*`, `*credential*`, `rclone.conf`, any file matching the secrets hook pattern
4. Commit with:
   ```
   git commit -m "$(cat <<'EOF'
   <type>: <subject>

   <body>

   Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
   EOF
   )"
   ```
5. Show the resulting `git log --oneline -1` to confirm
