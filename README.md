# Pipelines

> A collection of reusable GitHub Actions workflows and composite actions for use across projects.

---

## Reusable Workflows

Consumer repos reference these via `uses: simonjensen/pipelines/.github/workflows/<name>@main`.

| Workflow | File | Trigger in consumer | Purpose |
|---|---|---|---|
| CI Workflow | `ci.yaml` | `push` / `pull_request` | Runs Composer install, PHPUnit tests, and a Docker build (no push) to verify the image builds cleanly |
| Release Workflow | `release.yaml` | `workflow_dispatch` (manual) | Computes the next semver tag, creates a GitHub Release with generated release notes, and builds + pushes the Docker image to `ghcr.io` |

---

## Composite Actions

These are the building blocks used internally by the reusable workflows. They can also be called directly from consumer workflows if finer control is needed.

| Action | Directory | Purpose |
|---|---|---|
| Composer Install | `actions/composer-install` | Installs PHP dependencies via Composer inside Docker |
| PHPUnit | `actions/phpunittest` | Runs the PHPUnit test suite inside Docker |
| Create Tag | `actions/create-tag` | Computes the next semver version via GitVersion and pushes a git tag |
| Create Release Notes | `actions/create-release-notes` | Generates a `changelog.md` from commits between the last two tags |
| Create Release | `actions/create-release` | Creates a GitHub Release from the latest tag and `changelog.md` |
| Docker Build And Publish | `actions/docker-build-and-publish` | Builds the Docker image; pushes to `ghcr.io` only when on `main` and a token is provided |

---

## Consumer Wiring

### CI — runs on every push and pull request

```yaml
# .github/workflows/ci.yml
on: [push, pull_request]
jobs:
  ci:
    uses: simonjensen/pipelines/.github/workflows/ci.yaml@main
    with:
      composer-install: true
      phpunittest: true
      docker-build: true
      image-name: my-image
```

### Release — triggered manually to cut a release

```yaml
# .github/workflows/release.yml
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
