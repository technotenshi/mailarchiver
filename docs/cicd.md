# mailarchiver — CI/CD Pipelines

## Platform & Decisions

| Concern | Choice |
|---|---|
| CI/CD platform | **GitHub Actions** (private repo) |
| Deploy trigger | **Git tag `v*`** — CI runs on every push/PR; deploy only on semver tag |
| Container registry | **ghcr.io** (GitHub Container Registry, built-in `GITHUB_TOKEN`) |
| Deployment target | Single VPS via SSH → `docker compose pull && up -d` |

## Custom Images

Two containers require custom-built images. All others use off-the-shelf images.

| Image | Base | What's custom |
|---|---|---|
| `mailarchiver-worker` | `python:3.12-alpine` | Python deletion worker daemon + dependencies |
| `mailarchiver-mbsync` | `alpine` | mbsync binary + OAuth2 external password helper script |

---

## Pipeline 1 — CI

**Triggers:** push to `main`, pull_request targeting `main`

Three jobs run in parallel. All must pass. No images are pushed from CI.

### Job: `lint-and-test` (Python worker)

| Step | Tool | Purpose |
|---|---|---|
| Lint | `ruff check` | Style and correctness |
| Type check | `mypy` | Static type analysis |
| Security scan | `bandit` | Common Python security issues |
| Unit tests | `pytest` | Worker logic |

### Job: `build-and-scan` (Docker images)

| Step | Tool | Purpose |
|---|---|---|
| Build worker image | `docker build` | Verify Dockerfile is valid |
| Build mbsync image | `docker build` | Verify Dockerfile is valid |
| Scan worker | `trivy image` | CVE scan: fail on HIGH or CRITICAL |
| Scan mbsync | `trivy image` | CVE scan: fail on HIGH or CRITICAL |
| Scan off-the-shelf images | `trivy image` | Scan pinned digests of dovecot, rclone, prometheus, loki, grafana, alertmanager |

Images are built but not pushed. The scan gate covers both custom and off-the-shelf images. Off-the-shelf images are pulled by their pinned digest for scanning.

### Job: `validate-configs`

| Step | Tool | Validates |
|---|---|---|
| Compose file | `docker compose config` | YAML validity + schema |
| Prometheus | `promtool check config` | Scrape config + alerting rules |
| Alertmanager | `amtool check-config` | Routing tree + receiver config |
| Remaining YAML | `python -c "import yaml; yaml.safe_load(...)"` | Ofelia, Loki, Promtail configs |

---

## Pipeline 2 — Release + Deploy

**Trigger:** push of a tag matching `v*` (e.g. `v1.2.3`)

Jobs run sequentially. A failure at any stage stops the pipeline and leaves the previous deployment running.

### Job: `ci`

Reruns all three CI jobs from Pipeline 1. Must pass before proceeding.

### Job: `build-and-push` (depends on `ci`)

1. Authenticate to ghcr.io using built-in `GITHUB_TOKEN` — no extra secret needed
2. Build and push both custom images, each tagged three ways:

```
ghcr.io/<org>/mailarchiver-worker:v1.2.3     ← semver tag
ghcr.io/<org>/mailarchiver-worker:<git-sha>  ← immutable, enables rollback
ghcr.io/<org>/mailarchiver-worker:latest     ← floating, used by Compose
```

### Job: `deploy` (depends on `build-and-push`)

1. SSH to VPS using `VPS_SSH_KEY`
2. `docker compose pull` — pulls new worker and mbsync images
3. `docker compose up -d` — rolling restart
4. Wait 30 seconds for containers to initialize
5. **Health checks:**
   - `docker compose ps` — all containers show `running`
   - `curl worker:8000/metrics` — worker HTTP endpoint responds
6. Ping `DEPLOY_WEBHOOK_URL` (Slack/Discord) with success or failure notification

On failure: webhook fires, job exits non-zero — previous containers remain running.

---

## GitHub Secrets

| Secret | Purpose | Used by |
|---|---|---|
| `VPS_SSH_KEY` | Private SSH key for deploy user | `deploy` job |
| `VPS_HOST` | VPS hostname or IP | `deploy` job |
| `VPS_USER` | SSH user on VPS | `deploy` job |
| `DEPLOY_WEBHOOK_URL` | Slack/Discord webhook for deploy notifications | `deploy` job |

`GITHUB_TOKEN` is built-in — no additional registry credential needed.

**Not in CI secrets:** OAuth2 refresh tokens, rclone crypt passphrase, app passwords, Alertmanager SMTP/webhook credentials. These live on the VPS as Docker secrets and are never exposed to GitHub Actions.

---

## Rollback

Each deploy produces an immutable SHA-tagged image. To roll back:

1. Push a new tag pointing to the previous commit: `git tag v1.2.2-rollback <previous-sha>`
2. Push the tag — Pipeline 2 runs, builds the old code, deploys it
3. Or: SSH to VPS, edit `docker-compose.yml` to reference the previous SHA tag, `docker compose up -d`

---

## Pipeline Security Requirements

### GitHub Actions — pin by commit SHA, not tag

All Actions references must use the full commit SHA, not a mutable version tag. A tag like `@v4` can be silently updated to point at malicious code; a SHA is immutable.

```yaml
# Wrong — mutable tag
uses: actions/checkout@v4

# Correct — pinned SHA
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
```

Renovate manages SHA updates via PRs.

### Branch protection on `main`

Required settings on the `main` branch:
- Require pull request before merging (no direct push)
- Require CI to pass before merge (`lint-and-test`, `build-and-scan`, `validate-configs`)
- Require at least one approval (or self-review for solo project)
- Restrict who can push tags matching `v*` to authorized maintainers only

### Workflow permissions

GitHub Actions workflows must set minimal permissions at the top level:

```yaml
permissions:
  contents: read
```

Grant write permissions only where explicitly needed (e.g., `packages: write` for the `build-and-push` job only).

---

## Dependency Freshness

Renovate (or Dependabot) should be configured to open PRs automatically when:
- Alpine base image **digests** are updated (not just tags — all images pinned by `sha256:`)
- Python dependencies (`pyproject.toml` / `requirements.txt`) have new releases
- Off-the-shelf image digests change (Prometheus, Loki, Grafana, Dovecot, etc.)
- GitHub Actions commit SHAs have newer upstream releases

All such PRs run through the full CI pipeline before merge.

---

## Pipeline Diagram

```
On push / PR to main:

  ┌──────────────┐  ┌───────────────┐  ┌──────────────────┐
  │lint-and-test │  │build-and-scan │  │ validate-configs  │
  │ruff, mypy,   │  │docker build   │  │ compose, promtool │
  │bandit, pytest│  │trivy image    │  │ amtool, yaml lint │
  └──────────────┘  └───────────────┘  └──────────────────┘
  (no deploy — main branch is CI-only)

On git tag v*:

  ┌────────────────────────────────────┐
  │  ci  (all 3 jobs above, required)  │
  └──────────────────┬─────────────────┘
                     │
  ┌──────────────────▼─────────────────┐
  │  build-and-push                    │
  │  ghcr.io: worker + mbsync images   │
  │  tags: v*, sha, latest             │
  └──────────────────┬─────────────────┘
                     │
  ┌──────────────────▼─────────────────┐
  │  deploy                            │
  │  SSH → VPS                         │
  │  docker compose pull + up -d       │
  │  health check → notify webhook     │
  └────────────────────────────────────┘
```
