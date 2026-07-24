# Backend Testing

Purpose: Prove the backend actually works — unit tests, integration tests against real dependencies
(testcontainers), contract tests for API consumers, and a runnable verification loop (lint → migrate →
test → smoke) — as the skill's own "did this work" check, distinct from an independent release gate.

Read when:
- Request asks to write, run, or fix backend tests, or to "verify"/"prove" a backend change works.
- A change touched routing, domain logic, or persistence and needs functional verification before
  being called done.
- Request mentions testcontainers, contract testing, or a pre-PR/pre-deploy verification checklist.

Skip when:
- The ask is an independent load test, security scan, or a release/quality gate separate from the
  team's own dev-loop verification — that's `quality`'s job; this reference covers the skill's
  self-verification only.
- The ask is designing the test *strategy* for a whole system's CI pipeline / deploy gate — `operate`
  owns CI/deploy; this reference owns what runs *inside* that pipeline's test step.

Inputs: test framework already in use (state explicitly when absent — do not assume one), whether real
dependencies (DB, queue, cache) are available via testcontainers/Docker or must be mocked, the layer
being verified (domain logic / API / persistence).

Produces: a passing test suite exercising the change at the appropriate level (unit/integration/
contract), and a filled-in verification report (✓/✗ per phase, see asset) confirming the change
actually runs.

## Contents
- Test pyramid for a backend service
- Integration tests against real dependencies (testcontainers)
- Contract tests (producer side)
- The prove-it-works verification loop
- CI shape

## Procedure

### 1. Test pyramid for a backend service
- **Unit tests** — domain/entity/aggregate methods and pure use-case logic, no I/O. If the domain
  layer follows `domain-logic`'s dependency rule, these run with no DB/HTTP attached — that's the
  actual test of whether the layering held (a domain test that needs a real server/DB signals a
  dependency-rule violation, not a testing gap).
- **Integration tests** — the layer that touches real infrastructure (repository implementations,
  handler-to-DB round trips). Prefer real dependencies over mocks here — mocking the DB driver hides
  the exact bugs (SQL errors, constraint violations, transaction behavior) integration tests exist to
  catch.
- **Contract tests (producer side)** — verify the API you *serve* matches the contract consumers
  depend on (OpenAPI schema conformance, or a Pact/consumer-driven-contract verification if consumers
  publish contracts). This reference owns the producer side only; a consumer verifying someone else's
  contract is out of scope here.
- **End-to-end/smoke** — a small number of real-request checks (hit the actual running endpoint) as
  the last step of the verification loop below, not a substitute for the layers above.

### 2. Integration tests against real dependencies (testcontainers)
Default to spinning up the real dependency in a container for integration tests rather than mocking it
— a Postgres testcontainer catches constraint violations, migration drift, and driver-specific
behavior that an in-memory fake cannot. Concrete shape: start the container once per test session/class
(not per test — container startup cost dominates otherwise), run migrations against it, then run tests
in a transaction that rolls back after each test (or truncate between tests) for isolation. For HTTP-
level API tests without a real socket, several frameworks give an in-process test client that still
exercises the full framework stack — e.g. Hono's `app.request()` / `testClient()`, FastAPI's
`TestClient` (sync interface over the async app), NestJS's `Test.createTestingModule()` (builds the
real DI graph, letting you override just the providers you need to fake, so the test still exercises
the actual wiring). Prefer these over hand-wired fakes: they test the real routing/DI graph, not a
reimplementation of it.

For external services you don't control (a payment gateway, a third-party API), use WireMock or an
equivalent HTTP stub server bound to a dynamic port (avoids parallel-test port collisions) rather than
mocking the HTTP client library itself — this exercises the real serialization/deserialization path.

### 3. Contract tests (producer side)
If the API has a published contract (OpenAPI spec, proto file, GraphQL SDL) or consumer-driven
contracts (Pact), verify the running implementation against it as part of the test suite — generate/
validate the schema and fail the build on drift, rather than trusting the contract file and the code to
stay in sync by convention. This is the mechanical proof that `api-implementation`'s schema-as-single-
source-of-truth claim actually holds at runtime.

### 4. The prove-it-works verification loop
Run this loop before calling a backend change done — not narrative, a runnable sequence with a ✓/✗
outcome per phase (see `assets/verification-loop-template.md` for the fillable report and CI YAML):
1. **Environment** — dependencies installed, env vars/config present for the target environment.
2. **Lint/type-check** — static checks pass (framework-appropriate: mypy/ruff, tsc/eslint, etc.).
3. **Migrations** — pending migrations detected and applied cleanly against a real (or testcontainer)
   database; no drift between migration files and the live schema.
4. **Tests + coverage** — unit + integration + contract tests pass; track coverage by layer rather than
   one blended number (domain/service logic held to a higher bar than glue code) — set the exact
   per-layer targets per repo, don't invent a house number here.
5. **Endpoints** — hit the actual running server for the changed endpoints (curl/httpie or the
   framework's in-process test client) and confirm real request/response, not just unit-level mocks.
6. **Diff review** — grep the diff for debug artifacts left behind (`FIXME`/task-comment markers,
   stray `print`/`console.log`, a flipped debug flag, `import pdb`) before calling it done.

### 5. CI shape
Mirror the local verification loop in CI so "works on my machine" and "passes in CI" are the same
sequence: env setup → lint → migrations → test+coverage → security-scan (dependency audit at minimum;
deeper scanning is `quality`'s independent gate, not duplicated here) → smoke. See the CI YAML skeleton
in `assets/verification-loop-template.md`.

## Validation
- [ ] Domain/use-case logic has unit tests requiring no real DB/HTTP (confirms the dependency rule
      held — a failure here is a `domain-logic` architecture problem, not a test-writing problem).
- [ ] Integration tests run against a real (testcontainer or equivalent) dependency, not a hand-mocked
      driver, for anything touching the DB/queue/cache.
- [ ] If a contract artifact exists (OpenAPI/proto/SDL/Pact), the implementation is verified against it
      in the suite, not just trusted to match by convention.
- [ ] The verification loop (lint → migrate → test → endpoint smoke → diff review) was actually run,
      not asserted from memory — report ✓/✗ per phase.
- [ ] No debug artifacts (stray prints, `FIXME`/task-comment markers, flipped debug flags) left in the diff.

## Failure modes and handoff
- Test failure traces back to business logic living in the wrong layer (a domain test needs a live
  server to pass) → route to `domain-logic` to fix the layering, don't just add more mocks to paper
  over it.
- Need an independent load test, security/vulnerability scan, or a release-readiness gate distinct from
  this dev-loop verification → `quality`.
- CI pipeline wiring itself (runners, secrets, deploy triggers) → `operate`; this reference only owns
  what commands run inside the pipeline's test step.
