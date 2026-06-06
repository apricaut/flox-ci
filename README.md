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
