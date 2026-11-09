# GitHub

## Introduction

Shared GitHub Actions workflows and Renovate config for the `simonerigoni` Python repos.

## Reusable CI workflow: `python-uv-ci.yml`

One `checks` job on `ubuntu-24.04` (20-minute timeout, read-only token) that:

1. checks out the repo (`persist-credentials: false`)
2. installs uv with caching (`astral-sh/setup-uv`)
3. `uv lock --locked` (lock file is up to date)
4. `uv sync --locked --all-groups`
5. `ruff check` (GitHub annotations) and `ruff format --check`
6. `pyright` (optional)
7. `pytest` with coverage; the markdown coverage report is added to the run summary

Steps 5–7 all run even if an earlier check fails, so one run shows every problem.

**Python version:** nothing is hard-coded. uv uses the calling repo's `.python-version`
(and `requires-python`), so each repo can be on its own Python version.

### Usage

`.github/workflows/ci.yml` in a project repo:

```yaml
name: Continuous Integration

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  ci:
    uses: simonerigoni/github/.github/workflows/python-uv-ci.yml@v1
```

Inputs (all optional):

| Input | Type | Default | Meaning |
|---|---|---|---|
| `pyright` | boolean | `true` | Run `pyright .` |
| `pytest` | boolean | `true` | Run pytest + coverage |
| `pytest-args` | string | `-v --durations=0` | Extra pytest arguments |

Secrets (optional): `QUANDL_API_KEY` is exposed to the pytest step. Pass it with
`secrets: { QUANDL_API_KEY: ${{ secrets.QUANDL_API_KEY }} }` (or `secrets: inherit`).
The job also sets `ENV=github`.

Example with options:

```yaml
jobs:
  ci:
    uses: simonerigoni/github/.github/workflows/python-uv-ci.yml@v1
    with:
      pyright: false
      pytest-args: "-v tests"
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

Callers use the `v1` tag. After changing the workflow, move the tag (or cut `v2` for breaking
changes).

## List of activities

In the [TODO.md](TODO.md) file you can find the list of tasks and on going activities.

## Licensing and acknowledgements

Have a look at [LICENSE.md](LICENSE.md).

## Outro

I hope this repository was interesting and thank you for taking the time to check it out. Let me know if you have any question and if you like the content that I create feel free to [buy me a coffee](https://www.buymeacoffee.com/simonerigoni).
