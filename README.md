# workflows

Reusable GitHub Actions workflows for the repositories under `chri5bot`,
`cybergarden-team` and `TechDevStudio`. The repository is public because a
reusable workflow can only be called across accounts when it is public; it
holds no secrets.

## ci.yml

One pipeline for the bun repositories: a static job (lint, format, typecheck,
build), a unit-test job and an optional end-to-end job with the Playwright
browser cache and report upload. PostgreSQL can be started as a service for the
test and e2e jobs.

Caller, minimal:

```yaml
name: CI

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
  push:
    branches: [main]

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

permissions:
  contents: read

jobs:
  ci:
    uses: chri5bot/workflows/.github/workflows/ci.yml@main
    with:
      static: bun run lint && bun run typecheck && bun run build
```

Caller with end-to-end tests and PostgreSQL:

```yaml
jobs:
  ci:
    uses: chri5bot/workflows/.github/workflows/ci.yml@main
    with:
      static: bun run lint && bun run format:check && bun run typecheck && bun run build
      e2e: bun run test:e2e
      postgres: true
      postgres-user: orbit
      postgres-password: orbit
      postgres-db: orbit_crm_test
```

### Inputs

| Input | Default | Meaning |
| --- | --- | --- |
| `static` | `bun run lint && bun run typecheck && bun run build` | Command of the static job |
| `test` | `bun run test` | Command of the unit-test job; empty skips it |
| `e2e` | empty | Command of the e2e job; empty skips it. Never runs on draft pull requests |
| `e2e-report` | `playwright-report/` | Path uploaded as the `playwright-report` artifact |
| `node-version` | `24` | Node.js installed next to bun |
| `bun-version` | empty | bun version; empty reads `packageManager` from `package.json` |
| `postgres` | `false` | Start PostgreSQL for the test and e2e jobs |
| `postgres-image` | `postgres:17-alpine` | Service image |
| `postgres-user`, `postgres-password`, `postgres-db` | `app`, `app`, `app_test` | Service credentials |
| `database-url-env` | `TEST_DATABASE_URL` | Environment variable that receives the connection URL |
| `static-timeout`, `test-timeout`, `e2e-timeout` | `10`, `10`, `25` | Job timeouts in minutes |

### Conventions the callers keep

- Triggers and `concurrency` stay in the caller: the repository decides when
  CI runs and which runs cancel each other.
- `ready_for_review` in the pull-request trigger lets the e2e job run when a
  draft becomes ready.
- `bun install --frozen-lockfile` fails when `package.json` changed without
  the matching `bun.lock`.
- `bun run test`, never `bun test`: bun's native runner is not Vitest.

### Validate a change

```sh
actionlint .github/workflows/ci.yml
```
