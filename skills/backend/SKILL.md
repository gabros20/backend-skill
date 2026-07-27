---
name: backend
description: >-
  Implement the server behind the contract — APIs, domain logic, OLTP persistence, auth, async/jobs,
  caching, security controls, integration, and tests — across
  REST/GraphQL/gRPC/realtime/serverless-edge/agentic backends in Node/Python/Go/Rust/JVM. Use to
  build an endpoint, add auth, write a migration, set up a job queue or outbox, add caching/rate
  limiting, harden (OWASP), integrate a webhook, or expose an agent/MCP endpoint. Not for
  system/contract design (→ architecture), client/SSR (→ frontend), OLAP (→ data), agent core (→
  ai), deploy/monitoring (→ operate), or independent QA (→ quality).
---

# Backend Engineering

## Mission and boundary

Implement the production server that sits behind the contract — the APIs, domain logic, OLTP
persistence, authentication/authorization, async work, caching, security controls, outbound
integrations, and the tests that prove it runs. Own the *running server*: the implementation craft
inside the decided shape, not the shape itself.

Own the code, not the decision behind it. The border — with `architecture` (boundaries, contract
shape, NFR budgets, threat model) and with `frontend`/`data`/`ai`/`operate`/`quality` — is enumerated
in the invariants below; consume their artifacts and implement inside them rather than overriding.
Operate independently when invoked alone; when compatible upstream artifacts exist (a
solution-architecture doc, contracts, an NFR budget, a threat model, a `handoff.yaml`), build against
them rather than re-deriving. Recommend adjacent skills when useful; never invoke them automatically
unless the user explicitly requested a composition workflow.

## Route before acting

1. Pick the **one primary job** the request needs (≥1 row below — most requests are one; name it).
2. Add **at most one base surface** — the runtime/system type reshapes how every job applies.
3. **The agentic overlay is additive.** When the system is *also* an LLM/agent system, stack
   `surface-agentic` **on top of** the chosen base surface — it does not replace it. A serverless
   agent backend is `surface-serverless-edge` **+** `surface-agentic`; a realtime one adds
   `surface-realtime` too.
4. **Retrieval-first:** before generating any framework/ORM/DB-specific code, fetch the current
   official docs/types for the target version — backends ship breaking changes fast (see invariants).
5. Read each selected reference **completely** before producing the affected artifact. Load 2–3 at
   most; never preload the pack.

### Primary job (pick one)

| User intent | Read | Contribution |
|---|---|---|
| Build/scaffold/extend endpoints, handlers, routers, resolvers, or RPC methods behind a contract; pick REST/GraphQL/gRPC/tRPC; set pagination, versioning, error shape | [api-implementation](references/api-implementation.md) | Transport pick, request-lifecycle wiring, single-source schema, uniform error shape, cursor pagination, in-code versioning |
| Add/refactor business logic, a service or use-case layer, or domain models; apply DDD tactical / clean / hexagonal; stop rules leaking into handlers | [domain-logic](references/domain-logic.md) | Layered domain/application/infra structure, aggregates/entities/VOs, anti-anemic litmus, dependency rule |
| Design schema/indexes, choose ORM vs raw SQL, write a migration, get transactions/isolation right, size a pool, kill N+1, or push Postgres further | [persistence](references/persistence.md) | Schema+index+expand/contract migration, isolation decision, pooling matched to runtime, DB-license flag |
| Add login/signup/sessions, protect a route, add OAuth/SSO/"sign in with X", store passwords, or decide who-can-do-what | [auth](references/auth.md) | Session-vs-JWT decision, OAuth2/OIDC+PKCE flow, Argon2id config, RBAC/ABAC/ReBAC model + library pick |
| Background jobs, queues, workers, event-driven services, pub/sub, CDC, outbox, saga, idempotency, delivery semantics, or durable workflows (Temporal/Hatchet) | [async-and-messaging](references/async-and-messaging.md) | Broker/queue pick, idempotency-key path, transactional outbox, saga+compensations, delivery stance, durable-execution decision |
| Add a cache and invalidate it, implement rate limiting, or diagnose a slow endpoint/query | [caching-and-performance](references/caching-and-performance.md) | Cache strategy by staleness/loss tolerance, invalidation mechanism, distributed-correct token bucket, measured bottleneck + fix |
| Secure the API, add security headers, fix CORS, prevent injection, store secrets, add CSRF/mTLS — implement a threat model's controls | [server-security](references/server-security.md) | Uniform OWASP controls table (control→impl→verify) with self-audit count, mapped to Top 10:2021 / ASVS |
| Call a third-party API, receive/deliver webhooks, add a gRPC/HTTP client, tune a circuit breaker, or integrate a payment/AI/email provider | [service-integration](references/service-integration.md) | Resilient client wrapper (retry reused from async + circuit breaker), verified idempotent webhooks in/out |
| Write/run/fix backend tests, or "prove this works" — unit / integration (testcontainers) / producer-contract + a runnable verification loop | [backend-testing](references/backend-testing.md) | Layered test suite, real-dependency integration tests, contract-producer check, ✓/✗ verification report |

### Surface overlay (add at most one base surface)

Pick the one base surface that matches the runtime/system type; `surface-agentic` stacks additively on
top of it (step 3 above). Surfaces **reshape** how the jobs apply — they point to the job references,
they don't redefine them.

| System type | Read | Reshapes |
|---|---|---|
| Standard long-running request/response service — modular monolith or a few services, HTTP + relational store (**default / fallback**) | [surface-rest-crud](references/surface-rest-crud.md) | Concern-scoping model pick, adapter-isolates-runtime seam, per-language idioms, 12-factor scaffold |
| Serverless / edge — Workers, Lambda, Bun-edge, Vercel/Netlify functions; cold start, scale-to-zero, per-invocation billing | [surface-serverless-edge](references/surface-serverless-edge.md) | Statelessness, the connection-pooling failure mode (txn-mode pooler or HTTP-native driver), D1/KV/R2/Durable-Objects mapping |
| Realtime — WebSocket/SSE/pub-sub, stateful connections, presence, rooms; chat, collaboration, live dashboards, multiplayer | [surface-realtime](references/surface-realtime.md) | SSE-vs-WS transport, actor-per-connection/room, presence abstraction, per-message reliability, fan-out/scaling |
| Event-driven microservices — independently deployed polyglot services over gRPC + an event backbone (Kafka/NATS), service mesh | [surface-event-driven-microservices](references/surface-event-driven-microservices.md) | Modular-monolith-first gate, database-per-service consequences (saga/CQRS), inter-service contract evolution, sync-vs-async classification |
| **Agentic ⭐** — backend serving an LLM/agent system: tool/MCP endpoints, RAG plumbing, agent-legible contracts. **Additive — stacks on top of the base surface, does not replace it.** | **[surface-agentic](references/surface-agentic.md)** | **Durable execution as default control plane, tool/MCP endpoints, agent-legible typed contracts + errors, RAG plumbing (not core), agent-scoped short-lived tokens** |

### Handoff

When downstream build/verify work is expected (`operate` will deploy/monitor, `quality` will
independently verify, `frontend` is coding to the same contract), read
[handoff](references/handoff.md) and emit the `handoff.yaml` companion. Skip it for a standalone
request nothing downstream will read.

## Universal invariants

- **Implement inside the decision.** Own the running server, not what to build and why, its
  acceptance criteria, or its priority (→ `product`), the boundary/contract-shape/NFR-budget/
  threat-model *decision* (→ `architecture`), OLAP/pipelines (→ `data`), the LLM/agent *core* (→
  `ai`), deploy/CI/monitoring (→ `operate`), the independent load-test/scan/gate (→ `quality`), or a
  **business process spanning vendors you cannot deploy to** (→ `automation`). Consume upstream
  artifacts; when they don't exist (standalone run), say so explicitly rather than inventing a
  contract or threat model to fill the gap.
- **Your substrate is one you control.** `async-and-messaging`'s saga, idempotency and outbox assume
  the other participant is your own service, or a webhook you can dedupe against. When the
  participant is **opaque, un-scriptable, and partly human** — no rollback because the vendor has no
  un-send, credentials expiring across a vendor fleet, a human approval step, or drift that only a
  reconciliation loop against the vendor's own state will catch — that is `automation`. The
  primitives here still apply; the operating discipline around them is theirs.
- **Retrieval-first / anti-staleness.** Backend frameworks/ORMs/DBs ship breaking changes fast
  (Prisma 7, Drizzle 0.x→1.0, Go 1.22 routing, Kafka 4.0 KRaft-only, Redis licensing, PG19). Before
  generating any framework/ORM/DB-specific code, fetch the current official docs/types and version-gate
  idioms to the target release — never rely on memorized APIs for a fast-moving target.
- **Default-stack, then deviate.** Name the current modal default per stack, but every recommendation
  is a **concrete pick + what NOT to build + the trade-off**, weighted by reversibility. Anchor the
  fundamentals (SQL-before-ORM, modular-monolith-first, at-least-once + idempotent consumers,
  parameterized queries). Adoption numbers are engagement, not gospel — don't trend-chase. Cap each
  pass at 3–5 decisions.
- **Exhaustive checklists use a uniform table + self-audit count, never a narrative.** For any "cover
  N controls/items" task (OWASP controls, API-contract review, 12-factor audit, ASVS mapping), fill one
  row per item and count covered rows against the total before calling it done — a single narrative
  pass reliably drops rows.
- **Prove it works before done.** Ship the verification loop (lint → migrate → test → hit the
  endpoint → diff-review) as the completion gate, reporting ✓/✗ per phase — distinct from `quality`'s
  independent gate; never assert it from memory.
- **Distinguish facts, decisions, assumptions, and proposals.** Preserve upstream decisions and user
  constraints or flag the conflict explicitly; prefer repository and artifact evidence over generic
  defaults; state unknowns as TBD rather than inventing a value.

## Core workflow

1. Inspect the request, the codebase, and any upstream artifacts (contracts, NFR budgets, threat
   model, ADRs, a `handoff.yaml`); record material assumptions and unresolved inputs with owners.
2. Route (see above): one primary job, ≤1 base surface, `surface-agentic` stacked additively if the
   caller is an agent. Retrieval-first before any framework/ORM/DB code; read selected refs fully.
3. Produce the artifact grounded in the framework the reference names — concrete pick + what-NOT +
   trade-off, capped at 3–5 decisions per pass.
4. Run the prove-it-works verification loop (✓/✗ per phase); emit `handoff.yaml` when downstream
   build/verify work is expected.

## Artifact contract

Each reference defines its own artifact — a running implementation plus the decision record behind it.
Every backend artifact must record: the decision made and the options considered; the trade-off
accepted; facts/decisions/assumptions distinguished; and, for any framework/ORM/DB-specific code, that
current docs were checked for the target version. Exhaustive-coverage artifacts (OWASP controls, API
review, 12-factor) are a uniform table with a self-audit count, never a narrative. A full backend pass
produces up to five kinds of artifact — the running service, the contract as implemented, its own
tests, telemetry hooks, and the fillable checklists in [assets/](assets/) — plus the machine-readable
`handoff.yaml` companion whenever downstream work is expected; see [handoff](references/handoff.md).

## Completion and handoff

Before completion:

- Confirm every requested artifact exists and its acceptance criteria are checkable.
- Confirm the prove-it-works loop was actually run (✓/✗ per phase), not asserted.
- Confirm every exhaustive-checklist artifact is a table with a self-audit count, not a narrative.
- Record decisions, assumptions, risks, and unresolved questions with named owners.
- When downstream build/verify work is expected, emit the `handoff.yaml` companion with artifact
  paths, decisions, constraints, risks, and the recommended next skill — and, if a threat model was
  consumed, state the border between backend-implemented controls and quality-verified controls.
  Never silently invoke a build skill; name it in `recommended_next`.

## Resources

Load only what the selected route requires; never preload. The routing tables above name each
reference's contribution — this is the index.

- **Primary jobs** ([references/](references/)): api-implementation · domain-logic · persistence ·
  auth · async-and-messaging · caching-and-performance · server-security · service-integration ·
  backend-testing.
- **Surface overlays** ([references/](references/)): surface-rest-crud (default) ·
  surface-serverless-edge · surface-realtime · surface-event-driven-microservices ·
  surface-agentic ⭐ (additive).
- **Pipeline:** [handoff](references/handoff.md) — standalone vs pipeline behavior and the
  `handoff.yaml` companion consumed by `operate`, `quality`, `frontend`, `data`, and `ai`.
- **Assets** ([assets/](assets/)): backend-service scaffold (12-factor) · zero-downtime migration
  checklist · auth-implementation checklist · OWASP backend-controls checklist ·
  idempotency-and-outbox recipes · API-contract review checklist · verification-loop template ·
  `handoff.yaml` envelope.
