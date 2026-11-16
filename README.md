# GitHub

[![Workflow lint](https://github.com/simonerigoni/github/actions/workflows/workflow-lint.yml/badge.svg?branch=main)](https://github.com/simonerigoni/github/actions/workflows/workflow-lint.yml?query=branch%3Amain) [![Release](https://img.shields.io/github/v/release/simonerigoni/github)](https://github.com/simonerigoni/github/releases) [![License](https://img.shields.io/github/license/simonerigoni/github)](LICENSE.md) [![Renovate](https://img.shields.io/badge/renovate-enabled-brightgreen?logo=renovatebot)](https://renovatebot.com)

## Introduction

Shared GitHub Actions workflows and Renovate config for the `simonerigoni` Python repos.

## Reusable CI workflow: `python-uv-ci.yml`

Two jobs on `ubuntu-26.04` with a read-only token:

**checks** (20-minute timeout):

1. checks out the repo (`persist-credentials: false`)
2. installs uv with caching (`astral-sh/setup-uv`)
3. `uv lock --locked` (lock file is up to date)
4. `uv sync --locked --all-groups`
5. `ruff check` (GitHub annotations) and `ruff format --check`
6. `pyright` (optional)
7. `pytest` with coverage; the markdown coverage report is added to the run summary

Steps 5–7 all run even if an earlier check fails, so one run shows every problem.
The job sets `ENV=github`.

**pip-audit** (15-minute timeout, on by default): exports the locked dependencies
(including dev groups, without hashes) and audits them with `pip-audit`. The
markdown report is added to the run summary. Packages missing from PyPI (for
example a CUDA build of torch) are skipped rather than failing the job; known
vulnerabilities still fail it.

**Python version:** nothing is hard-coded. uv uses the calling repo's `.python-version`
(and `requires-python`), so each repo can be on its own Python version.

### Usage

`.github/workflows/ci.yml` in a project repo:

```yaml
name: Continuous Integration

on:
  push:
    branches: [main, master]
  pull_request:
  workflow_dispatch:
  schedule:
    - cron: "0 6 * * 1"

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

permissions:
  contents: read

jobs:
  ci:
    uses: simonerigoni/github/.github/workflows/python-uv-ci.yml@b40071710b2343ed80815f6ddefce8d1e32c9718 # v1.3
```

The weekly `schedule` runs every Monday at 06:00 UTC so a repo nobody has
pushed to still gets a pip-audit. Cron minutes are spread per repo so the
runs do not all start at once. `cancel-in-progress` is limited to pull
requests, so a new push to `main` or `master` does not cancel an in-flight
run.

Pin the reusable workflow to a commit SHA and keep the `# v1.3` comment.
Renovate (`helpers:pinGitHubActionDigests`) keeps the SHA and the version
comment up to date. The floating `@v1` tag also works and currently points
at the same commit as `v1.3`.

Inputs (all optional):

| Input | Type | Default | Meaning |
|---|---|---|---|
| `pyright` | boolean | `true` | Run `pyright .` |
| `pytest` | boolean | `true` | Run pytest + coverage |
| `pytest-args` | string | `-v --durations=0` | Extra pytest arguments |
| `pip-audit` | boolean | `true` | Audit locked dependencies with pip-audit |
| `pip-audit-ignore` | string | `""` | Space- or comma-separated vulnerability IDs to ignore (for example `GHSA-xxxx` or `CVE-2024-xxxx`) |

Secrets (optional): `QUANDL_API_KEY` is exposed to the pytest step. Pass it with
`secrets: { QUANDL_API_KEY: ${{ secrets.QUANDL_API_KEY }} }` (or `secrets: inherit`).

Example with options:

```yaml
jobs:
  ci:
    uses: simonerigoni/github/.github/workflows/python-uv-ci.yml@b40071710b2343ed80815f6ddefce8d1e32c9718 # v1.3
    with:
      pyright: false
      pytest-args: "-v tests"
      pip-audit-ignore: "GHSA-xxxx"
    secrets:
      QUANDL_API_KEY: ${{ secrets.QUANDL_API_KEY }}
```

Each project needs `ruff`, `pyright` (if enabled) and `pytest-cov` in a dependency group,
plus a pinned `[tool.ruff.lint] select` so ruff upgrades can't silently add rules.

## Shared Renovate preset: `renovate-config.json`

A project's `renovate.json` becomes:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>simonerigoni/github:renovate-config"]
}
```

The preset provides:

- `config:recommended`, Europe/Amsterdam timezone, schedule `before 9am on monday`
- lock file maintenance (same schedule, automerge) and pre-commit updates enabled
- `rebaseWhen: behind-base-branch` so lock-file PRs are rebased after other merges, `prHourlyLimit: 0`
- one grouped PR for all minor/patch updates (automerge), one for GitHub Actions, one for pre-commit hooks
- `helpers:pinGitHubActionDigests`: actions are pinned to commit SHAs and kept up to date

It does **not** pin Python. Repos that must stay on a Python version keep that rule in their
own `renovate.json` `packageRules`.

## Versioning

Callers pin a commit SHA with a version comment (currently
`@b40071710b2343ed80815f6ddefce8d1e32c9718 # v1.3`). Renovate keeps that pin
and comment current. `@v1` also works and now points at `v1.3`. After a
compatible change, cut a new `v1.x` tag and move `v1` there (or cut `v2`
for breaking changes).

## Workflow lint

This repository's own `workflow-lint.yml` runs actionlint 1.7.12 and zizmor on
`ubuntu-26.04`. A repo-local `.github/actionlint.yaml` lists the `ubuntu-26.04`
and `ubuntu-26.04-arm` labels, because actionlint 1.7.12 does not know them yet
([rhysd/actionlint#682](https://github.com/rhysd/actionlint/issues/682)).

## List of activities

In the [TODO.md](TODO.md) file you can find the list of tasks and on going activities.

## Licensing and acknowledgements

Have a look at [LICENSE.md](LICENSE.md).

## Outro

I hope this repository was interesting and thank you for taking the time to check it out. Let me know if you have any question and if you like the content that I create feel free to [buy me a coffee](https://www.buymeacoffee.com/simonerigoni).
