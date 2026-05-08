# AGENTS.md — Coding Agent Reference

This repo is a **reusable GitHub Actions pipeline library** — YAML only, no application code. Changes to `main` immediately affect all consumer repositories that reference `simonjensen/pipelines/.github/...@main`.

---

## Actual Repository Layout

```
.github/
  actions/
    composer-install/action.yaml       # Docker-based Composer install
    phpunittest/action.yaml            # Docker-based PHPUnit runner
    docker-build-and-publish/action.yaml  # Docker build + optional push
  workflows/
    actionlint.yaml   # CI: lints all Actions YAML on every push
    ci.yaml           # Reusable workflow: build + test (no release)
    release.yaml      # Reusable workflow: semver release + Docker push
renovate.json         # Auto-updates external action pins via Renovate
```

> The old AGENTS.md listed `create-release-notes`, `create-release`, and `create-tag` actions. These no longer exist. `release.yaml` now uses `huggingface/semver-release-action@v1.1.1` directly to handle tagging and release creation.

---

## Linting (only automated check in this repo)

CI runs `raven-actions/actionlint@v2` on every push to any branch.

Lint locally (mirrors CI exactly):
```bash
# All files
docker run --rm -v "$(pwd):/repo" --workdir /repo rhysd/actionlint:latest

# Single file
docker run --rm -v "$(pwd):/repo" --workdir /repo rhysd/actionlint:latest .github/actions/composer-install/action.yaml
```

Or with `actionlint` installed directly:
```bash
actionlint
actionlint .github/actions/composer-install/action.yaml
```

---

## Key Implementation Details

### docker-build-and-publish — push gate

Push to `ghcr.io` is gated on `github_token` input being non-empty:
```yaml
push: ${{ inputs.github_token != '' }}
```
In `ci.yaml` the action is called without a token (build-only). In `release.yaml` `secrets.GITHUB_TOKEN` is passed, enabling push.

### docker-build-and-publish — tag resolution

If `tag` input is empty, the action falls back to `github.ref_name` with slashes replaced by dashes (e.g. `feature/foo` → `feature-foo`). Set `REGISTRY` and `IMAGE_NAME` env vars at the workflow level before calling this action — the action reads them from `${{ env.REGISTRY }}` and `${{ env.IMAGE_NAME }}`.

### release.yaml — semver and release creation

`release.yaml` does **not** use internal composite actions for tagging or release notes. It calls `huggingface/semver-release-action@v1.1.1` directly, which computes the next semver tag, creates a GitHub Release, and exposes a `tag` output consumed by the Docker step:
```yaml
tag: ${{ steps.create-release.outputs.tag }}
```

### ci.yaml — input names for path overrides

Default paths for both workflows are `src`. Override with:
- `composer-install-path` (not `composer-path`)
- `phpunittest-src-path` (not `src-path`)

These differ from the input names on the underlying composite actions.

### release.yaml — docker flag name

The flag is `docker-build-and-release` (not `docker-build` as in `ci.yaml`), and it defaults to `true`.

---

## YAML Conventions

- File extension: `action.yaml` not `action.yml`
- File/dir names: `kebab-case`
- Action/workflow `name:`: Title Case
- Step `id:`: `kebab-case`
- String/path inputs: `kebab-case`; token/secret inputs: `snake_case` (`github_token`)
- All inputs: `required: true` or explicit `default:` — never leave optional without fallback
- Env vars written to `$GITHUB_ENV` / `$GITHUB_OUTPUT`: `SCREAMING_SNAKE_CASE`
- Local Bash vars in `run:` blocks: `camelCase`
- No `set -e`, no `|| exit 1` — GitHub Actions fails on non-zero exit by default
- Use `>> "$GITHUB_OUTPUT"` heredoc syntax, not deprecated `::set-output`

---

## MD Conventions

- When adding or updating tables, make sure the columns are spaced to align visually even when viewing the raw markdown

---

## Action Reference Conventions

- External actions: pin to major version tag, no digest pins
  ```yaml
  uses: actions/checkout@v6
  uses: docker/build-push-action@v7
  ```
- Internal self-references: always `@main`
  ```yaml
  uses: simonjensen/pipelines/.github/actions/composer-install@main
  ```
- Do not manually bump external action versions — Renovate opens PRs. `"pinDigests": false` is intentional.

---

## Adding a New Composite Action

1. Create `.github/actions/<kebab-name>/action.yaml` with `runs.using: composite`
2. Add a boolean feature-flag input (default `false`) to `ci.yaml` or `release.yaml`
3. Guard the step: `if: ${{ inputs.<flag> }}`
4. Reference with `uses: simonjensen/pipelines/.github/actions/<name>@main`
5. Update `README.md`

All tooling runs inside Docker (`docker run --rm`). Do not install tools on the runner with `apt-get`.

---

## Commit Style

Conventional Commits: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `ci:`
```
feat: add multi-platform Docker build support
chore(deps): bump docker/build-push-action from v6 to v7
```
