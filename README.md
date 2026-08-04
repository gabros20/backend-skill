# backend — Backend Engineering

**Visual guide:** [backendengskill.vercel.app](https://backendengskill.vercel.app)

**Family hub:** [digital-product-skill.vercel.app](https://digital-product-skill.vercel.app) — the 15-skill Digital Product family and its composer.

Implement **the running server behind the contract** — API endpoints, domain logic, OLTP
persistence, auth, async jobs and messaging, caching, security controls, outbound integrations, and
the tests that prove it works — across REST/GraphQL/gRPC/realtime/serverless-edge/agentic backends
in Node, Python, Go, Rust, and the JVM. `backend` is the server-side half of the
**Build** stage of the [Digital Product Skills](https://github.com/gabros20) family, peer of
`frontend`: an agent skill for implementing production endpoints, business/domain logic, schema and
migrations, authentication and authorization, background jobs, caching and rate limiting, security
hardening, third-party integrations, and agent/MCP-serving backends.

It sits between `architecture` (which decides the boundaries, the contract shape, the NFR budgets,
and the threat model) and `operate`/`quality` (which deploy, monitor, and independently verify what
gets built). It owns the **running server and the implementation craft inside the decided shape** —
not the decision behind it. It is an **implementation router grounded in the current stack, not a
system-design tool, and not a single-vendor or single-framework skill.**

## What it does

A **faceted router**: pick one primary job × at most one base surface (the agentic overlay stacks
additively on top).

**Primary jobs** — API implementation · domain logic · persistence · auth · async & messaging ·
caching & performance · server security · service integration · backend testing.

**Surface overlays** — REST/CRUD service (default) · serverless/edge · realtime · event-driven
microservices · **agentic** (additive). The base surface reshapes how every job applies for that
runtime/system type; `surface-agentic` stacks on top of whichever base surface applies rather than
replacing it.

**The differentiator:** first-class **agentic-backend** support as the flagship overlay — durable
execution as the default control plane, tool/MCP endpoints, agent-legible typed contracts and
errors, and RAG-retrieval plumbing that calls `ai`'s core without owning it. Plus three invariants no
single-vendor or single-framework skill carries: **retrieval-first / anti-staleness** (fetch current
framework/ORM/DB docs before generating anything version-specific — Prisma 7, Drizzle 1.0, Go 1.22
routing, Kafka 4.0's KRaft-only cutover all ship breaking changes fast), **default-stack-then-deviate**
(name the current modal pick per language, but every recommendation is a concrete choice + what NOT
to build + a trade-off, weighted by reversibility — never trend-chasing), and **DB-license-teaching**
(SSPL, BSL, AGPL, and source-available self-host/resale implications, with the permissive escape
hatches named) — a distinctive value-add most backend material skips entirely.

Every method is grounded in its named source: OWASP Top 10 and ASVS, the 12-Factor App, OAuth
2.0/OIDC with PKCE (RFC 9700), Domain-Driven Design tactical patterns, the transactional-outbox and
saga patterns, Zanzibar-style ReBAC, Google/AWS/Azure well-architected guidance on pooling and
durable execution (Temporal, Hatchet), and the current official docs for each covered
framework/ORM/datastore. Full attribution in [SOURCES.md](SOURCES.md).

## Install

```bash
npx skills add gabros20/backend-skill -g -y
```

Or use it in Codex with `$backend`.

## Examples

```text
$backend build the REST endpoints behind this contract — pagination, versioning, error shape

$backend design the domain/service layer for checkout so rules stop leaking into handlers

$backend add OAuth2/OIDC login with PKCE and decide sessions vs JWT

$backend set up the outbox and saga for this order-fulfillment workflow

$backend expose an MCP/tool endpoint for our agent backend — durable execution, agent-legible errors
```

## Route by job

| User intent | Reads | Contribution |
|---|---|---|
| Build/scaffold/extend endpoints, handlers, routers, resolvers, or RPC methods; pick REST/GraphQL/gRPC/tRPC; set pagination, versioning, error shape | [api-implementation](skills/backend/references/api-implementation.md) | Transport pick, request-lifecycle wiring, single-source schema, uniform error shape, cursor pagination, in-code versioning |
| Add/refactor business logic, a service or use-case layer, or domain models; apply DDD tactical / clean / hexagonal; stop rules leaking into handlers | [domain-logic](skills/backend/references/domain-logic.md) | Layered domain/application/infra structure, aggregates/entities/VOs, anti-anemic litmus, dependency rule |
| Design schema/indexes, choose ORM vs raw SQL, write a migration, get transactions/isolation right, size a pool, kill N+1 | [persistence](skills/backend/references/persistence.md) | Schema+index+expand/contract migration, isolation decision, pooling matched to runtime, DB-license flag |
| Add login/signup/sessions, protect a route, add OAuth/SSO, store passwords, decide who-can-do-what | [auth](skills/backend/references/auth.md) | Session-vs-JWT decision, OAuth2/OIDC+PKCE flow, Argon2id config, RBAC/ABAC/ReBAC model + library pick |
| Background jobs, queues, workers, event-driven services, pub/sub, CDC, outbox, saga, idempotency, durable workflows | [async-and-messaging](skills/backend/references/async-and-messaging.md) | Broker/queue pick, idempotency-key path, transactional outbox, saga+compensations, delivery stance, durable-execution decision |
| Add a cache and invalidate it, implement rate limiting, or diagnose a slow endpoint/query | [caching-and-performance](skills/backend/references/caching-and-performance.md) | Cache strategy by staleness/loss tolerance, invalidation mechanism, distributed-correct token bucket, measured bottleneck + fix |
| Secure the API, add security headers, fix CORS, prevent injection, store secrets, implement a threat model's controls | [server-security](skills/backend/references/server-security.md) | Uniform OWASP controls table (control→impl→verify) with self-audit count, mapped to Top 10:2021 / ASVS |
| Call a third-party API, receive/deliver webhooks, add a gRPC/HTTP client, tune a circuit breaker | [service-integration](skills/backend/references/service-integration.md) | Resilient client wrapper (retry + circuit breaker), verified idempotent webhooks in/out |
| Write/run/fix backend tests, or "prove this works" | [backend-testing](skills/backend/references/backend-testing.md) | Layered test suite, real-dependency integration tests, contract-producer check, ✓/✗ verification report |

**Surface overlay (add at most one base; the agentic overlay stacks additively):**

| System type | Reads | Reshapes |
|---|---|---|
| Standard long-running request/response service — modular monolith or a few services (default) | [surface-rest-crud](skills/backend/references/surface-rest-crud.md) | Concern-scoping model pick, adapter-isolates-runtime seam, per-language idioms, 12-factor scaffold |
| Serverless / edge — Workers, Lambda, Bun-edge, Vercel/Netlify functions | [surface-serverless-edge](skills/backend/references/surface-serverless-edge.md) | Statelessness, the connection-pooling failure mode (txn-mode pooler or HTTP-native driver), D1/KV/R2/Durable-Objects mapping |
| Realtime — WebSocket/SSE/pub-sub, stateful connections, presence, rooms | [surface-realtime](skills/backend/references/surface-realtime.md) | SSE-vs-WS transport, actor-per-connection/room, presence abstraction, per-message reliability, fan-out/scaling |
| Event-driven microservices — independently deployed polyglot services over gRPC + an event backbone | [surface-event-driven-microservices](skills/backend/references/surface-event-driven-microservices.md) | Modular-monolith-first gate, database-per-service consequences (saga/CQRS), inter-service contract evolution, sync-vs-async classification |
| **Agentic** — backend serving an LLM/agent system, stacked additively on the base surface | [surface-agentic](skills/backend/references/surface-agentic.md) | **Durable execution as default control plane, tool/MCP endpoints, agent-legible typed contracts + errors, RAG plumbing, agent-scoped short-lived tokens** |

Plus [handoff](skills/backend/references/handoff.md) — the `handoff.yaml` companion to `operate`
(deploy/monitor) and `quality` (independent verification) when downstream work is expected.

## Outputs

A running implementation plus the decision record behind it: endpoint/handler code, a domain/service
layer, schema and migrations, auth wiring, job/queue setup, cache and rate-limit configuration, an
OWASP controls table, resilient outbound client wrappers, and the test suite that proves it all
runs — plus fillable checklists and a `handoff.yaml` companion that feeds `operate`/`quality`
downstream, grounded in the framework each artifact names, never a generic template dump.

## Reference and asset library

15 references under `skills/backend/references/` (9 primary jobs, 5 surface overlays, 1 handoff
contract) and 8 fillable assets under `skills/backend/assets/`:

- **Assets:** `backend-service-scaffold.md` · `migration-template.md` ·
  `auth-implementation-checklist.md` · `owasp-backend-controls-checklist.md` ·
  `idempotency-and-outbox-recipes.md` · `api-contract-review-checklist.md` ·
  `verification-loop-template.md` · `handoff.yaml`

## Digital Product lifecycle

`backend` is the server-side half of the **Build** stage, peer of `frontend`: it consumes
`architecture`'s contracts, NFR budgets, threat model, and `handoff.yaml` upstream, and hands its
own running services and telemetry hooks to `operate` and its test suite to `quality` downstream,
with `data` and `ai` consumed rather than owned for OLAP pipelines and the agent core respectively.
It works standalone with no upstream artifact required, and never silently invokes a sibling skill.

## Repository layout

```text
skills/backend/    runtime skill (SKILL.md + 15 references + 8 assets + agents/openai.yaml)
evals/              activation · traversal · output · compression-ablation fixtures
docs/                installation and usage
.codex-plugin/      plugin manifest
```

## Documentation & releases

- [docs/installation.md](docs/installation.md) · [docs/usage.md](docs/usage.md) ·
  [docs/recipes.md](docs/recipes.md)
- [CHANGELOG.md](CHANGELOG.md) — release history (current: v0.1.0)
- [SOURCES.md](SOURCES.md) — framework attribution and license posture
- [CONTRIBUTING.md](CONTRIBUTING.md) — local validation (`scripts/lint-skill`, `scripts/check-sync`)

## License

MIT — see [LICENSE](LICENSE).
