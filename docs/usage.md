# Usage reference

## Activation

`backend` should trigger on requests to implement **the running server behind the contract** —
endpoints, domain logic, persistence, auth, async work, caching, security controls, integrations,
and tests. Realistic triggers: "build/scaffold this endpoint," "add auth / OAuth / sessions," "write
a migration," "set up a job queue or outbox," "add caching / rate limiting," "harden this API
(OWASP)," "integrate this webhook," "expose an agent/MCP endpoint," "write backend tests / prove
this works."

It should **not** trigger on: "design the system / decide the boundaries / set NFR budgets"
(`architecture`), "build the client / React component / SSR page" (`frontend`), "build the
OLAP/analytics pipeline" (`data`), "design the agent core / prompt / eval harness" (`ai`), "deploy
this / set up monitoring" (`operate`), or "run an independent load test / security scan" (`quality`)
— unless the request also requires server implementation.

Use `/backend` as documentation shorthand only. Codex explicit invocation is `$backend`; other
clients use an `@` mention, a skill tool, or natural-language activation.

## Mission and boundary

`backend` owns the server-side half of the Build stage — the *running server*: API/handler code,
domain/service logic, OLTP persistence, auth, async/messaging, caching, security controls, outbound
integrations, and the tests that prove it works. It sits between `architecture` (which decides the
boundaries, contract shape, NFR budgets, and threat model) and `operate`/`quality` (which deploy,
monitor, and independently verify). It owns the implementation craft inside the decided shape, not
the decision behind it. It consumes upstream artifacts when they exist (a solution-architecture doc,
contracts, an NFR budget, a threat model, a `handoff.yaml`) rather than re-deriving them, works
standalone when they don't, recommends its sibling skills for work it does not own, and never
silently invokes them.

## Workflow

1. Pick **one primary job** and **at most one base surface**; stack `surface-agentic` additively on
   top of the base surface when the system is also an LLM/agent backend.
2. **Retrieval-first:** before generating any framework/ORM/DB-specific code, fetch the current
   official docs/types for the target version — never rely on memorized APIs for a fast-moving
   target.
3. Read only the selected references (2–3 at most — never preload the pack).
4. Produce the artifact grounded in the framework the reference names: a concrete pick + what NOT to
   build + a trade-off, capped at 3–5 decisions per pass, weighted by reversibility.
5. Run the prove-it-works verification loop (lint → migrate → test → hit the endpoint →
   diff-review), reporting ✓/✗ per phase — never asserted from memory.
6. Emit a compact `handoff.yaml` when downstream build/verify work is expected.

## Route by job

| User intent | Reference | Expected contribution |
|---|---|---|
| Build/scaffold/extend endpoints, handlers, routers, resolvers, or RPC methods; pick REST/GraphQL/gRPC/tRPC; set pagination, versioning, error shape | `references/api-implementation.md` | Transport pick, request-lifecycle wiring, single-source schema, uniform error shape, cursor pagination, in-code versioning |
| Add/refactor business logic, a service or use-case layer, or domain models; apply DDD tactical / clean / hexagonal | `references/domain-logic.md` | Layered domain/application/infra structure, aggregates/entities/VOs, anti-anemic litmus, dependency rule |
| Design schema/indexes, choose ORM vs raw SQL, write a migration, get transactions/isolation right, size a pool, kill N+1 | `references/persistence.md` | Schema+index+expand/contract migration, isolation decision, pooling matched to runtime, DB-license flag |
| Add login/signup/sessions, protect a route, add OAuth/SSO, store passwords, decide who-can-do-what | `references/auth.md` | Session-vs-JWT decision, OAuth2/OIDC+PKCE flow, Argon2id config, RBAC/ABAC/ReBAC model + library pick |
| Background jobs, queues, workers, event-driven services, pub/sub, CDC, outbox, saga, idempotency, durable workflows | `references/async-and-messaging.md` | Broker/queue pick, idempotency-key path, transactional outbox, saga+compensations, delivery stance, durable-execution decision |
| Add a cache and invalidate it, implement rate limiting, or diagnose a slow endpoint/query | `references/caching-and-performance.md` | Cache strategy by staleness/loss tolerance, invalidation mechanism, distributed-correct token bucket, measured bottleneck + fix |
| Secure the API, add security headers, fix CORS, prevent injection, store secrets, implement a threat model's controls | `references/server-security.md` | Uniform OWASP controls table (control→impl→verify) with self-audit count, mapped to Top 10:2021 / ASVS |
| Call a third-party API, receive/deliver webhooks, add a gRPC/HTTP client, tune a circuit breaker | `references/service-integration.md` | Resilient client wrapper (retry + circuit breaker), verified idempotent webhooks in/out |
| Write/run/fix backend tests, or "prove this works" | `references/backend-testing.md` | Layered test suite, real-dependency integration tests, contract-producer check, ✓/✗ verification report |

Surface overlays (`surface-rest-crud`, `surface-serverless-edge`, `surface-realtime`,
`surface-event-driven-microservices`, `surface-agentic`) reshape the primary job for that
runtime/system type. Always stack `surface-agentic` additively on top of the base surface for an
LLM/agent backend — it changes the control plane, the contracts, and the token model.

## Outputs and completion

| Output | Complete when |
|---|---|
| Endpoint/handler implementation | Transport, error shape, pagination, and versioning decided and wired, distinguishing facts/decisions/assumptions |
| Domain/service layer | Aggregates/entities/VOs modeled, anti-anemic litmus passed, dependency rule respected |
| Schema, migration, or pooling change | Isolation level and pooling matched to runtime; DB-license implication flagged when relevant |
| Auth implementation | Session-vs-JWT or OAuth2/OIDC+PKCE flow wired; password/RBAC-ABAC-ReBAC config recorded |
| Async/messaging setup | Idempotency-key path and delivery-semantics stance stated; outbox/saga wired where at-least-once demands it |
| OWASP/security controls artifact | Every applicable Top 10/ASVS control audited as a table row, self-audit count shown |
| Verification loop | ✓/✗ reported per phase (lint, migrate, test, hit endpoint, diff-review) — never asserted from memory |

When downstream work is expected, provide the `handoff.yaml` companion with artifact paths,
decisions, constraints, risks, unresolved questions, and the recommended next skill — and, when a
threat model was consumed, state the border between backend-implemented controls and
quality-verified controls.
