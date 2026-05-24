# builders

Reusable GitHub Actions workflows for building internal projects and publishing release artifacts back to their source repositories.

## Build and publish workflow

This repository provides `.github/workflows/build-and-publish.yml` as a reusable workflow (`workflow_call`).

It is intended for repositories in this organization that need centralized build automation with a token that can:

- read private source repositories
- run the source repository build/CI command
- write release assets to the same source repository

### Authentication (why not `github.token`?)

Workflows in this repository target **other** `audiohacking/*` repositories (often private). The built-in `GITHUB_TOKEN` (`${{ github.token }}`) is scoped only to **audiohacking/builders** — it cannot checkout private source trees or upload release assets elsewhere. That is why `gh` fails with “set the GH_TOKEN environment variable” when no cross-repo credential is configured.

Use one of these (configured on **audiohacking/builders**):

1. **GitHub App (recommended)** — short-lived installation tokens via [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token), wired through [`.github/actions/cross-repo-token`](.github/actions/cross-repo-token):
   - Repository variable: `BUILDERS_GITHUB_APP_ID`
   - Repository (or org) secret: `BUILDERS_GITHUB_APP_PRIVATE_KEY`
   - Create an org GitHub App installed on `audiohacking` with **Contents: Read and write** on the private repos you build, and **Metadata: Read**.
2. **PAT fallback** — repository secret `BUILDERS_REPO_ACCESS_TOKEN` with the same effective access (classic or fine-grained PAT).

Every `gh` step and `actions/checkout` for a source repo uses the token from that composite action, not `${{ github.token }}`.

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
- `repo_access_token` (secret, optional): PAT override when not using the org GitHub App on builders

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
    # secrets.repo_access_token is optional when BUILDERS_GITHUB_APP_* is set on builders
```

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

It requires cross-repo authentication on **audiohacking/builders** (see [Authentication](#authentication-why-not-githubtoken)) — either the org GitHub App (`BUILDERS_GITHUB_APP_ID` + `BUILDERS_GITHUB_APP_PRIVATE_KEY`) or `BUILDERS_REPO_ACCESS_TOKEN`.
