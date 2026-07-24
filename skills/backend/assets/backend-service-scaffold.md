# Backend service scaffold (12-factor)

Fillable starter checklist for a new backend service, grounded in the 12-Factor App methodology
(12factor.net, MIT-licensed — Adam Wiggins/Heroku). Fill every box for a real service; a box left
unchecked without a stated reason is a gap, not a style choice. Full factor-by-factor rationale
lives at 12factor.net; this asset is the audit skeleton, not a restatement of the essay.

Service name: ____________________  Language/runtime: ____________________  Surface:
[ ] rest-crud  [ ] serverless-edge  [ ] realtime  [ ] event-driven-microservices  [ ] agentic

## I. Codebase
- [ ] One repo (or one clearly-scoped path in a monorepo) tracks this service; no code fork per
  environment — only config varies between deploys.

## II. Dependencies
- [ ] All dependencies are declared in a manifest (package.json / requirements.txt / go.mod / pom.xml
  — name it: ____________________), none assumed present on the host.
- [ ] A lockfile is committed; builds are reproducible from it.

## III. Config
- [ ] All per-deploy values (credentials, hostnames, feature flags, timeouts) come from environment
  variables or a typed config/secrets layer — none hardcoded in source.
- [ ] Config is validated at startup (fail fast on a missing/malformed value) rather than failing
  deep inside a request handler.
- [ ] No secret is committed to the repo, including in test fixtures or `.env.example` with a real
  value.

## IV. Backing services
- [ ] Every backing service (DB, cache, queue, SMTP, third-party API) is reached via a URL/credential
  in config — swappable (e.g. local Postgres ↔ managed Postgres) without a code change.
- [ ] Local dev backing services are declared (docker-compose or equivalent) so parity with prod is
  achievable, not aspirational.

## V. Build, release, run
- [ ] The build stage (compile/bundle/install deps) is separate and repeatable from the release
  stage (build + config → immutable versioned release) and the run stage (execute a release).
- [ ] A prior release can be rolled back to without rebuilding.

## VI. Processes
- [ ] The process is stateless and share-nothing — no in-memory session/cache assumed to survive a
  restart or to be visible to a sibling instance; persistent state goes to a backing store.
- [ ] Sticky-session or local-file-state assumptions are absent, or explicitly flagged as a known
  deviation with an owner.

## VII. Port binding
- [ ] The service is self-contained and exports its interface (HTTP or other) by binding a port —
  not by being injected into an external webserver container at runtime.

## VIII. Concurrency
- [ ] Scaling is via the process model (more instances of the web/worker/clock process types), not
  by assuming a single giant process will grow threads indefinitely.
- [ ] Distinct process types (web / background-worker / scheduled-job) are identified, even if only
  one is running today.

## IX. Disposability
- [ ] The process starts fast (state target: _____ seconds) and handles SIGTERM by finishing
  in-flight requests, closing connections cleanly, then exiting — no reliance on `SIGKILL` to stop.
- [ ] Crash-and-restart is safe: no partial-write state is left inconsistent if the process dies
  mid-operation (pairs with idempotency in [async-and-messaging](../references/async-and-messaging.md)
  where relevant).

## X. Dev/prod parity
- [ ] Dev, staging, and prod use the same backing-service *products* (not, e.g., SQLite in dev
  against Postgres in prod).
- [ ] The gap between writing code and it running in prod is measured in hours, not weeks.

## XI. Logs
- [ ] The app writes logs as an unbuffered event stream to stdout/stderr — it does not manage log
  files, rotation, or routing itself; the execution environment captures/aggregates them.
- [ ] Log lines are structured (not free-text) so the execution environment's aggregator can index
  them — see `operate` for the aggregation/dashboard side of this.

## XII. Admin processes
- [ ] One-off admin tasks (migrations, REPL/console, data backfills) run as one-off processes against
  the same codebase/config as the long-running process — not as ad hoc scripts run only by one
  engineer's local setup.

## Health and readiness (not a 12-factor item, but the operational baseline every service under this
scaffold ships — `operate` wires these into its deploy/monitoring, backend exposes them)
- [ ] A liveness endpoint exists (`/health` or equivalent): returns 200 iff the process itself is up,
  with no dependency checks — used to decide "restart this instance."
- [ ] A readiness endpoint exists (`/ready` or equivalent): returns 200 iff the process can serve
  traffic (backing services reachable) — used to decide "route traffic to this instance."
- [ ] The two are distinct endpoints/semantics — a slow dependency should fail readiness, not
  liveness (failing liveness would restart a healthy process for someone else's outage).

## Deviations recorded (fill in, don't leave implicit)
| Factor | Deviation | Reason | Owner |
|---|---|---|---|
| | | | |
