# flox-ci

Reusable Flox CI/CD for apricaut services: **build → publish package → push runtime env → promote**.

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
Flox target (→ `tyler-harpool/<repo>`), pushes the `deploy/` runtime env, and promotes by
bumping `flox.dev/revision` in `apricaut/apricaut`. Change the pipeline here; every product
picks it up — no per-repo config drift.

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
