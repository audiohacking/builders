# builders

Reusable GitHub Actions workflows for building internal projects and publishing release artifacts back to their source repositories.

## Build and publish workflow

This repository provides `.github/workflows/build-and-publish.yml` as a reusable workflow (`workflow_call`).

It is intended for repositories in this organization that need centralized build automation with a token that can:

- read private source repositories
- run the source repository build/CI command
- write release assets to the same source repository

### Authentication

Workflows checkout and publish to other `audiohacking/*` repositories using the repository secret **`BUILDERS_REPO_ACCESS_TOKEN`** (Actions secret, not a variable). The built-in `GITHUB_TOKEN` cannot access private sibling repositories.

### Permission and data-leakage model

To prevent private code/artifact leakage through this public builders repository:

- `source_repository` is restricted to `audiohacking/*` in the reusable workflow
- checkout uses `persist-credentials: false` so credentials are not persisted in git config
- workflow/job `GITHUB_TOKEN` permissions are explicitly minimized
- artifacts are uploaded only to the same `source_repository` that was built

### Required workflow inputs

- `source_repository`: `owner/repo` to build
- `build_command`: shell command to run the project build
- `artifact_paths`: newline-delimited file paths or glob patterns to upload
- `release_tag`: release tag in the source repository
Optional inputs:

- `source_ref` (defaults to `main`)
- `release_name`
- `release_notes`

### Example caller workflow

```yaml
name: Build release via builders repo

on:
  workflow_dispatch:

jobs:
  build:
    uses: audiohacking/builders/.github/workflows/build-and-publish.yml@main
    with:
      source_repository: audiohacking/my-private-project
      source_ref: main
      build_command: make release
      artifact_paths: |
        dist/*.tar.gz
        dist/*.sha256
      release_tag: v1.2.3
      release_name: v1.2.3
      release_notes: Automated build from builders repo
    secrets: inherit
```

`secrets: inherit` passes `BUILDERS_REPO_ACCESS_TOKEN` from the caller when the caller is also `audiohacking/builders`.

## Release workflow for `audiohacking/aitroce-vst`

This repository includes `.github/workflows/manual-aitroce-vst-build.yml`, a `workflow_dispatch` entrypoint that ports the `aitroce-vst` release build into the builders repository.

What it does:

- resolves/validates release tag and source ref
- ensures the release exists in `audiohacking/aitroce-vst`
- runs a macOS/Linux/Windows matrix build for VST3 (plus AU/.pkg on macOS)
- uploads generated artifacts back to the release in `audiohacking/aitroce-vst`

Required manual input:

- `release_tag` (must start with `v`)

Optional manual input:

- `source_ref` (branch/tag/SHA to build; defaults to `release_tag`)

It requires the **`BUILDERS_REPO_ACCESS_TOKEN`** secret on **audiohacking/builders** (see [Authentication](#authentication)).
