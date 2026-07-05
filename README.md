# flox-ci

Reusable CI/CD for apricaut services:
- **Flox web/server delivery**: build → publish package → push runtime env → promote
- **Tauri client releases**: reusable desktop, Android, and iOS workflows backed by Doppler
- **Release management + validation**: reusable `release-please` and Rust/Postgres check workflows

App repos call it from `.github/workflows/flox.yml`:

```yaml
name: flox
on:
  pull_request:
  push:
    branches: [main]
jobs:
  ci:
    uses: apricaut/flox-ci/.github/workflows/flox-ci.yml@v1
    secrets: inherit
```

The pipeline derives the service name from the caller repo, builds the `[build.<repo>]`
Flox target (→ `apricaut/<repo>`), pushes the `deploy/` runtime env, and promotes by
bumping `flox.dev/revision` in `apricaut/apricaut`. Change the pipeline here; every product
picks it up — no per-repo config drift.

## Tauri release workflows

App repos can keep thin caller workflows and delegate the real release logic here:

```yaml
jobs:
  desktop:
    uses: apricaut/flox-ci/.github/workflows/tauri-desktop.yml@main
    with:
      app_name: my-app
      server_url: https://my-app.example.com
      dx_version: 0.7.9
      doppler_project: my-app
      doppler_config: prd
    secrets: inherit
```

There are matching reusable workflows for:
- `.github/workflows/tauri-desktop.yml`
- `.github/workflows/tauri-android.yml`
- `.github/workflows/tauri-ios.yml`
- `.github/workflows/release-please.yml`
- `.github/workflows/rust-postgres-check.yml`

These workflows assume GitHub only supplies `DOPPLER_TOKEN`; the Apple,
Android, and Tauri signing secrets are read from Doppler at runtime.

## Shared release-please

App repos can keep a tiny `release-please.yml` trigger and delegate the token
minting + release-please wiring here:

```yaml
jobs:
  release-please:
    uses: apricaut/flox-ci/.github/workflows/release-please.yml@main
    permissions:
      contents: write
      pull-requests: write
    secrets: inherit
```

This workflow reads `APRICAUT_APP_ID` and `APRICAUT_APP_PRIVATE_KEY` from
Doppler `platform/common`, mints an `apricaut-platform` installation token,
and uses that token so release tags can fan out into downstream workflows.

## Shared Rust + Postgres validation

Repos whose tests need a disposable Postgres can keep only their triggers and
repo-specific cargo commands locally:

```yaml
jobs:
  ci:
    uses: apricaut/flox-ci/.github/workflows/rust-postgres-check.yml@main
    with:
      postgres_user: my-app
      postgres_password: my-app
      postgres_db: my-app
      postgres_health_cmd: pg_isready -U my-app
      database_url: postgres://my-app:my-app@localhost:5432/my-app
      test_command: cargo test -p my-app-shared --features server
      check_command: cargo check -p my-app-server
```

## axum services — `actions/axum/cicd`

A leaner, DB-free sibling for plain **axum** services (no postgres service, no dioxus
wasm bundle — just lint/test, build the binary, publish, promote). It's a **composite
action** (not a reusable workflow) so the caller keeps control of its own job/triggers,
which suits both standalone app repos and a single crate inside a monorepo.

```yaml
jobs:
  cicd:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: apricaut/flox-ci/.github/actions/axum/cicd@main
        with:
          app: my-service                 # default: repo name
          working-directory: .            # crate dir holding `.flox` (e.g. a monorepo member)
          cargo-scope: "--workspace"      # or "-p my-service" for one member of a workspace
          deploy-manifest: apps/my-service/base/deployment.yaml
          publish: ${{ github.ref == 'refs/heads/main' && github.event_name == 'push' }}
          doppler-token: ${{ secrets.DOPPLER_TOKEN }}
```

Inputs: `app`, `working-directory`, `cargo-scope`, `publish`, `floxhub-owner`,
`deploy-repo`, `deploy-manifest`, `doppler-token`. When `publish=true` it runs the
deliver phase (FloxHub publish + promote PR); otherwise it just verifies.
