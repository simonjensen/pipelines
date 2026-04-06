# AGENTS.md — Coding Agent Reference

This repository is a **reusable GitHub Actions pipeline library**. It contains no application source code. All files are YAML (composite actions and reusable workflows), JSON, and Markdown. Changes here affect all consumer repositories that reference `simonjensen/pipelines/.github/...@main`.

---

## Repository Layout

```
pipelines/
├── .github/
│   ├── actions/                  # Composite actions (one per concern)
│   │   ├── composer-install/action.yaml
│   │   ├── create-release-notes/action.yaml
│   │   ├── create-release/action.yaml
│   │   ├── create-tag/action.yaml
│   │   ├── docker-build-and-publish/action.yaml
│   │   └── phpunittest/action.yaml
│   └── workflows/
│       ├── actionlint.yaml       # CI: lints all Actions YAML on every push
│       ├── ci.yaml               # Reusable workflow: build + test (no release)
│       └── release.yaml          # Reusable workflow: tag + release + Docker push
├── renovate.json                 # Renovate Bot config (auto-updates action versions)
└── README.md
```

---

## Build / Lint / Test Commands

### Linting (the only automated check run on this repo itself)

Linting is handled by `actionlint` and runs automatically via GitHub Actions on every push to any branch. There is no local Makefile or task runner.

To lint locally using Docker (mirrors CI exactly):

```bash
docker run --rm -v "$(pwd):/repo" --workdir /repo \
  rhysd/actionlint:latest
```

To lint a single file:

```bash
docker run --rm -v "$(pwd):/repo" --workdir /repo \
  rhysd/actionlint:latest .github/actions/composer-install/action.yaml
```

Alternatively, install `actionlint` directly and run:

```bash
actionlint                                          # lint everything
actionlint .github/actions/composer-install/action.yaml  # lint one file
```

### PHPUnit (for consumer projects, via the phpunittest composite action)

The `phpunittest` action runs tests inside Docker in a consumer project. To replicate it manually:

```bash
# Run full test suite
docker run --rm \
  -v "$(pwd)/<src-path>:/src" \
  -w /src \
  ghcr.io/simonjensen/docker-php8.3:0.0.3 \
  ./vendor/bin/phpunit tests/

# Run a single test class or method (append --filter)
docker run --rm \
  -v "$(pwd)/<src-path>:/src" \
  -w /src \
  ghcr.io/simonjensen/docker-php8.3:0.0.3 \
  ./vendor/bin/phpunit tests/ --filter MyTestClassName::testMethodName
```

### Composer (for consumer projects, via the composer-install composite action)

```bash
docker run --rm \
  -v "$(pwd)/<composer-path>:/src" \
  -w /src \
  ghcr.io/simonjensen/docker-php8.3:0.0.3 \
  composer install --no-progress --no-ansi --no-interaction \
    --optimize-autoloader --prefer-dist
```

---

## YAML Style Guidelines

### File and directory naming

- All file names: `kebab-case` lowercase (`action.yaml`, `docker-build-and-publish/`)
- Use `action.yaml` (not `action.yml`) for composite actions
- Each composite action lives in its own directory under `.github/actions/<kebab-name>/`

### `name:` fields

- Workflows and actions: Title Case with spaces (`"Composer Install"`, `"Docker Build And Release"`)
- Step `id:` values: `kebab-case` (`checkout`, `create-tag`, `docker-build-and-release`)

### Input naming

- Path and string inputs: `kebab-case` (`src-path`, `composer-path`)
- Token/secret inputs: `snake_case` (`github_token`) — follow GitHub Actions conventions
- Boolean feature-flag inputs: `kebab-case`, with sensible defaults
  - Core/always-on steps default to `true`
  - Optional steps default to `false`
- All inputs must either be `required: true` or have an explicit `default:` — never leave an input optional without a fallback

### Output naming

- `kebab-case` (`tag`)

### Environment variables

- `SCREAMING_SNAKE_CASE` for all env vars exported to `$GITHUB_ENV` or `$GITHUB_OUTPUT` (`REGISTRY`, `IMAGE_NAME`, `FINAL_TAG`)
- Local Bash variables within `run:` scripts: `camelCase` (`nextVersion`, `resolvedTag`)

---

## Inline Bash Script Style

- Keep `run:` scripts minimal and focused
- Do not add `set -e` or `set -o pipefail` — GitHub Actions fails steps on non-zero exit by default
- Do not add explicit error traps or `|| exit 1` unless truly necessary
- Prefer writing computed values to `$GITHUB_OUTPUT` or `$GITHUB_ENV` using the `>> "$GITHUB_OUTPUT"` heredoc syntax (not deprecated `::set-output`)
- Use `${{ inputs.<name> }}` for action inputs inside `run:` scripts, not shell variable assignment

---

## Action Reference Conventions (`uses:`)

- **External actions**: pin to a major version tag, no digest pinning
  ```yaml
  uses: actions/checkout@v6
  uses: docker/build-push-action@v7
  ```
- **Internal (self-referencing) actions**: always pin to `main`
  ```yaml
  uses: simonjensen/pipelines/.github/actions/create-tag@main
  ```
- Renovate Bot (`renovate.json`) manages version updates automatically — do not manually update external action versions; let Renovate open PRs
- `"pinDigests": false` is intentional — do not add digest pins

---

## Architectural Patterns

### Composite actions vs. reusable workflows

- **Composite actions** (`.github/actions/*/action.yaml`): one per discrete concern (tag, release notes, release, composer, phpunit, docker build). Keep each action single-purpose.
- **Reusable workflows** (`.github/workflows/ci.yaml` and `.github/workflows/release.yaml`): orchestrate composite actions with feature-flag boolean inputs so consumers can enable/disable stages without forking.

### CI vs. CD split

There are two separate reusable workflows that consumer repos wire up independently:

| Workflow | Consumer trigger | Purpose |
|---|---|---|
| `ci.yaml` | Every push / pull request | Composer install, PHPUnit, Docker build (no push) |
| `release.yaml` | `workflow_dispatch` (manual) | Tag, release notes, GitHub Release, Docker build + push |

Consumer repos call them like this:

```yaml
# .github/workflows/ci.yml — runs on every push and PR
on: [push, pull_request]
jobs:
  ci:
    uses: simonjensen/pipelines/.github/workflows/ci.yaml@main
    with:
      composer-install: true
      phpunittest: true
      docker-build: true
      image-name: my-image
    # permissions: contents: read is the default — no block needed

# .github/workflows/release.yml — triggered manually to cut a release
on:
  workflow_dispatch:
jobs:
  release:
    permissions:
      contents: write
      packages: write
      id-token: write
    uses: simonjensen/pipelines/.github/workflows/release.yaml@main
    secrets: inherit
    with:
      image-name: my-image
```

The `if: github.ref == 'refs/heads/main'` guards inside the composite actions remain in place as a second line of defence — release/push steps will not run if the workflow is triggered from a non-main branch.

### Feature flags

All optional pipeline stages are controlled by boolean inputs on the orchestrating workflows. New stages should follow the same pattern:

```yaml
inputs:
  my-new-stage:
    type: boolean
    default: false
```

And guarded in steps:

```yaml
- if: ${{ inputs.my-new-stage }}
  uses: simonjensen/pipelines/.github/actions/my-new-stage@main
```

### Tooling via Docker

All heavy tooling (PHP, Composer, PHPUnit, GitVersion) runs inside Docker containers via `docker run --rm`. This keeps the runner dependency-free and makes tool versions explicit. New tooling should follow the same pattern — do not install tools directly onto the runner with `apt-get` or similar.

### Container registry

All images (tooling and output) use `ghcr.io` (GitHub Container Registry). Do not introduce other registries.

---

## Commit Message Style

Follow Conventional Commits:

```
feat: add support for multi-platform Docker builds
fix: resolve tag duplication on re-runs
chore: update actions/checkout to v6
chore(deps): bump docker/build-push-action from v6 to v7
```

Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `ci`

---

## Adding a New Composite Action

1. Create `.github/actions/<kebab-name>/action.yaml`
2. Set `name:`, `description:`, `inputs:`, `outputs:` (if any), and `runs.using: composite`
3. Add a boolean input to the appropriate reusable workflow (`ci.yaml` for build/test steps, `release.yaml` for release steps) to expose it as a feature flag
4. Call it from the workflow with `if: ${{ inputs.<flag> }}` and `uses: simonjensen/pipelines/.github/actions/<name>@main`
5. Document the new action in `README.md`
