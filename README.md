# annium/workflows

Reusable GitHub Actions workflows shared by every Annium .NET sub-project.

The workflows here own only the *plumbing* — checkout, .NET SDK, `just`, artifact
upload, test reporting. Everything a pipeline actually does lives in the calling
repository's `justfile`, under these recipes:

| Recipe | Called by |
|--------|-----------|
| `ci-merge-request-short` | `dotnet-merge-request.yml`, when the PR title contains `skip ci` |
| `ci-merge-request-full` | `dotnet-merge-request.yml`, otherwise |
| `ci-release <apiKey> <repository> <githubToken>` | `dotnet-release.yml` |

That split is deliberate: adding a step to a pipeline is a change in the
sub-project that owns it, while changing *how* pipelines are wired is a change
here — applied to every sub-project at once.

This repository is public so that private sub-projects can call it without any
extra access configuration.

## Usage

Both callers are thin stubs. Copy them into `.github/workflows/` of the
sub-project.

`.github/workflows/merge-request.yml`:

```yaml
name: Merge Request

on:
  pull_request:

permissions:
  contents: read
  actions: read
  checks: write
  pull-requests: write

concurrency:
  group: merge-request-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  ci:
    uses: annium/workflows/.github/workflows/dotnet-merge-request.yml@v1
```

`.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    branches:
      - main

permissions:
  contents: write
  actions: read

concurrency:
  group: release-${{ github.ref }}
  cancel-in-progress: false

jobs:
  release:
    uses: annium/workflows/.github/workflows/dotnet-release.yml@v1
    secrets: inherit
```

`permissions` and `concurrency` belong to the caller: a called workflow can only
narrow the token the caller grants it, and cancellation applies to the caller's
run.

## Inputs

Both workflows accept:

| Input | Default | Purpose |
|-------|---------|---------|
| `dotnet-version` | `10.0.x` | Passed to `actions/setup-dotnet` |
| `runs-on` | `ubuntu-latest` | Runner label. Set to a self-hosted label for private repos — never for public ones, where fork PRs would execute on your machine |
| `test-results-retention-days` | `30` | TRX artifact retention |

`dotnet-merge-request.yml` also takes `short-pipeline-marker` (default `skip ci`)
— the PR-title substring that downgrades the run to the short pipeline.

`dotnet-release.yml` also takes `skip-ci-marker` (default `skip ci`) — the
head-commit substring that skips the release, and requires the `NUGET_API_KEY`
secret. It is an organization secret, so `secrets: inherit` is enough; the
calling repository must be on that secret's visibility list.

## Versioning

Callers pin `@v1`, a moving major tag. Move it after a backwards-compatible
change:

```bash
git tag -f v1 && git push -f origin v1
```

Cut `v2` for anything that breaks callers — a new required input, a renamed
recipe, a dropped default.
