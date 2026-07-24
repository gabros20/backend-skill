# Verification Loop Template

A fillable ✓/✗ report + a minimal CI skeleton for the `backend-testing` prove-it-works loop. Fill in
the report after actually running each phase — do not check a box from memory.

## Verification report skeleton

```
# Verification Report — <change/PR description>
Date: <date>          Branch/commit: <ref>

| Phase | Result | Notes |
|---|---|---|
| 1. Environment (deps installed, config/env vars present) | ✓/✗ | |
| 2. Lint / type-check | ✓/✗ | |
| 3. Migrations applied cleanly, no drift | ✓/✗ | |
| 4. Tests + coverage (unit / integration / contract) | ✓/✗ | coverage: __% (by layer if tracked) |
| 5. Endpoints hit for real (not just unit-mocked) | ✓/✗ | endpoint(s) tested: |
| 6. Diff review (no `FIXME`/task-marker/print/debug-flag/pdb left) | ✓/✗ | |

RECOMMENDATION: <ship / fix N before ship / blocked on X>
NEXT STEPS: <bulleted list, or "none">
```

## Minimal CI YAML skeleton

Adapt job names/commands to the actual stack — this mirrors the local loop 1:1 so CI and local runs
never diverge.

```yaml
name: backend-verify

on:
  pull_request:
  push:
    branches: [main]

jobs:
  verify:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10

    steps:
      - uses: actions/checkout@v4

      # 1. Environment
      - name: Install dependencies
        run: <install command — e.g. npm ci / pip install -r requirements.txt / go mod download>

      # 2. Lint / type-check
      - name: Lint and type-check
        run: <e.g. npm run lint && npm run typecheck / ruff check . && mypy .>

      # 3. Migrations
      - name: Apply migrations
        run: <e.g. npx drizzle-kit migrate / alembic upgrade head / python manage.py migrate>
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/postgres

      # 4. Tests + coverage
      - name: Run tests with coverage
        run: <e.g. npm test -- --coverage / pytest --cov / go test ./... -cover>

      # 5. Security scan (dependency audit — deeper scanning is quality's independent gate)
      - name: Dependency audit
        run: <e.g. npm audit --audit-level=high / pip-audit / safety check>

      # 6. Smoke
      - name: Boot server and smoke-test endpoints
        run: <start server in background, curl/httpie the changed endpoint(s), assert 2xx>
```
