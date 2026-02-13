# README

> This repo aim to serve a collection of pipelines to be used in my daily development

## Pipelines

### Docker

* Location: `.github/workflows/docker.yaml`

* Purpose:
  * All branches:
    * Builds your `Dockerfile`
  * `main` branches:
    * Pushes your container to `ghcr.io`
    * Creates a tag, release and release notes

* Inputs:
  * `image-name`: Typically just set this to `${{ github.repository }}` from the calling repo

