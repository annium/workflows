# annium/workflows

Reusable GitHub Actions workflows shared by Annium .NET repositories.

Each file here is **one job that does one thing**: bring up the toolchain and invoke one `just`
recipe. What the recipe does lives in the calling repository's `justfile`. What shape the pipeline
has — which recipes run, in what order, with what matrix — lives in the caller's workflow.

That is the change from v1, where a single workflow owned the whole pipeline. It fit six similar
repositories and stopped fitting when they merged: one of them now wants its tests split across
parallel jobs while another still wants one, and a shape baked in here cannot be both.

| Workflow | Job |
|----------|-----|
| `dotnet-run.yml` | run one recipe |
| `dotnet-test.yml` | run one recipe, upload its TRX, optionally inside a GitHub Environment |
| `test-report.yml` | collect the TRX artifacts and publish one check |
| `dotnet-release.yml` | run `ci-release <apiKey> <repository> <githubToken>` |

This repository is public so that private repositories can call it without extra access
configuration.

## Usage

### A pipeline with parallel test groups

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
  check:
    uses: annium/workflows/.github/workflows/dotnet-run.yml@v2
    with:
      recipe: ci-check

  test:
    needs: check
    if: ${{ !contains(github.event.pull_request.title, 'skip ci') }}
    strategy:
      fail-fast: false
      matrix:
        group: [ framework, adapters ]
    uses: annium/workflows/.github/workflows/dotnet-test.yml@v2
    with:
      recipe: ci-test-${{ matrix.group }}
      artifact: test-results-${{ matrix.group }}

  report:
    needs: test
    if: ${{ always() && !contains(github.event.pull_request.title, 'skip ci') }}
    uses: annium/workflows/.github/workflows/test-report.yml@v2
```

`fail-fast: false` so that one failing group still lets the other report. Give matrix jobs distinct
artifact names — two uploads under one name collide — and let `test-report.yml` gather them by
pattern.

### A pipeline with one test job

```yaml
jobs:
  check:
    uses: annium/workflows/.github/workflows/dotnet-run.yml@v2
    with:
      recipe: ci-check
  test:
    needs: check
    if: ${{ !contains(github.event.pull_request.title, 'skip ci') }}
    uses: annium/workflows/.github/workflows/dotnet-test.yml@v2
    with:
      recipe: ci-test
      artifact: test-results-all
  report:
    needs: test
    if: ${{ always() }}
    uses: annium/workflows/.github/workflows/test-report.yml@v2
```

### Release

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
  check:
    uses: annium/workflows/.github/workflows/dotnet-run.yml@v2
    with:
      recipe: ci-check
  test:
    needs: check
    strategy:
      fail-fast: false
      matrix:
        group: [ framework, adapters ]
    uses: annium/workflows/.github/workflows/dotnet-test.yml@v2
    with:
      recipe: ci-test-${{ matrix.group }}
      artifact: test-results-${{ matrix.group }}
  release:
    needs: test
    uses: annium/workflows/.github/workflows/dotnet-release.yml@v2
    secrets: inherit
```

`ci-release` no longer runs the tests itself: the jobs above gate it.

`permissions` and `concurrency` belong to the caller — a called workflow can only narrow the token
the caller grants it, and cancellation applies to the caller's run. So does the `skip ci` marker,
which is now an `if:` on the jobs rather than a job of its own computing it.

## Inputs

All four accept `dotnet-version` (default `10.0.x`) and `runs-on` (default `ubuntu-latest`). Set
`runs-on` to a self-hosted label for private repositories — never for public ones, where fork pull
requests would then execute on your machine.

The three that run a recipe also accept `timeout-minutes` — 20 for `dotnet-run.yml`, 45 for
`dotnet-test.yml`, 60 for `dotnet-release.yml`. GitHub's own default is 360 minutes, which is not a
limit so much as an absence of one: a test hanging on a network call burns six hours of runner time
before anyone hears about it. Raise it where a recipe honestly takes longer, rather than leaving it
unset.

| Workflow | Also accepts |
|----------|--------------|
| `dotnet-run.yml` | `recipe` (required) |
| `dotnet-test.yml` | `recipe` (required), `artifact`, `environment`, `test-results-retention-days` |
| `test-report.yml` | `artifact-pattern` (default `test-results-*`) |
| `dotnet-release.yml` | `test-results-retention-days`; requires the `NUGET_API_KEY` secret |

### `environment`

`dotnet-test.yml` takes an `environment` input, and declares it on the job. It has to live here
rather than in the caller: `on.workflow_call` does not accept the `environment` keyword.

That placement is also what makes it useful. A secret attached to a GitHub Environment is visible
only to a job that declares that environment — a pipeline that does not name it cannot read those
secrets at all. Use it for credentials that reach a live system, and the ordinary pull-request
pipeline stays unable to see them no matter what a branch adds to it.

`NUGET_API_KEY` is an organization secret, so `secrets: inherit` is enough; the calling repository
must be on that secret's visibility list.

## Versioning

Callers pin `@v2`, a moving major tag. Move it after a backwards-compatible change:

```bash
git tag -f v2 && git push -f origin v2
```

Cut `v3` for anything that breaks callers — a new required input, a renamed recipe, a dropped
default.

`v1` is frozen at the pipeline-owning shape and stays until nothing points at it.
