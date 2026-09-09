# Shared CI

`dporkka/Tools@shared-ci-v1` is the bootstrap host for reusable GitHub Actions used across David Porkka's repositories.

## Design

- Caller workflows own triggers, concurrency, permissions, secrets, and repository-specific policy.
- Reusable workflows own generic language setup and deterministic validation.
- `reusable-detect.yml` classifies changed files before expensive language jobs are scheduled.
- `runner_json` makes the same workflow usable on GitHub-hosted or self-hosted runners.
- Existing repository-specific CI should be migrated incrementally rather than deleted all at once.

## Runner selectors

GitHub-hosted:

```yaml
with:
  runner_json: '"ubuntu-latest"'
```

Nulang Cloud self-hosted pool:

```yaml
with:
  runner_json: '["self-hosted","Linux","X64"]'
```

## Standard caller

```yaml
name: Shared CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: shared-ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  changes:
    uses: dporkka/Tools/.github/workflows/reusable-detect.yml@shared-ci-v1

  node:
    needs: changes
    if: needs.changes.outputs.node == 'true' || needs.changes.outputs.ci == 'true' || github.event_name == 'workflow_dispatch'
    uses: dporkka/Tools/.github/workflows/reusable-node.yml@shared-ci-v1
    with:
      node_version: '22'
      package_manager: pnpm
      lint_command: pnpm lint
      test_command: pnpm test
      build_command: pnpm build
```

## Rust

```yaml
rust:
  needs: changes
  if: needs.changes.outputs.rust == 'true' || needs.changes.outputs.ci == 'true'
  uses: dporkka/Tools/.github/workflows/reusable-rust.yml@shared-ci-v1
  with:
    rust_version: stable
    check_args: --workspace --all-targets
    clippy_args: --workspace --all-targets
    clippy_deny: warnings
    test_args: --workspace
```

## Go

```yaml
go:
  needs: changes
  if: needs.changes.outputs.go == 'true' || needs.changes.outputs.ci == 'true'
  uses: dporkka/Tools/.github/workflows/reusable-go.yml@shared-ci-v1
  with:
    go_version: '1.24.x'
```

## Python

```yaml
python:
  needs: changes
  if: needs.changes.outputs.python == 'true' || needs.changes.outputs.ci == 'true'
  uses: dporkka/Tools/.github/workflows/reusable-python.yml@shared-ci-v1
  with:
    python_version: '3.13'
    install_command: pip install -e '.[dev]'
    lint_command: ruff check .
    test_command: pytest -q
```

## Migration policy

1. Add the shared workflow alongside existing CI.
2. Verify cross-repository access, runner assignment, and command parity.
3. Move generic jobs such as language setup, lint, typecheck, unit tests, and builds into the shared layer.
4. Keep service-heavy integration tests, deployments, secrets-dependent jobs, database migrations, and repository-specific policy local until a purpose-built reusable workflow exists.
5. Add path/affected-area gating to legacy workflows so a docs or workflow-only edit does not start unrelated database, browser, container, or deployment jobs.
6. After the bootstrap stabilizes, move these workflows to a dedicated `dporkka/platform-ci` repository and pin callers to a version tag or immutable commit SHA.

## Current pilots

- `dporkka/nulang-cloud` — shared Rust validation on the existing self-hosted Linux/X64 pool.
- `dporkka/adacavo` — shared Node/pnpm typecheck, workspace lint, and root Jest tests.
- `dporkka/apex` — shared Go, Node, and Rust validation; protobuf and deployment policy remain local.

`nulang-org/nulang` is ready for the same migration, but the current GitHub App installation cannot create refs in that organization; grant the app repository write access before adding its caller branch.
