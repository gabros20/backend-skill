# Backend Skill — Synthesis & Build Gate

**Date:** 2026-07-24 · **Stage:** Build (server-side) · **Peer of:** the shipped `frontend` skill.
**Inputs:** 4 source-linked corpora — [skills](skills-corpus.md), [github](github-corpus.md),
[x](x-corpus.md), [web-canon](web-canon-corpus.md) — ~20-agent swarm, all four channels complete.

This is the gate: sign off the wedge, the skill shape, and the policy rulings, and I start
composition. Nothing is written until then.

---

## 1. The wedge — is the niche open?

**Yes, the same way architecture's was.** Everything in the ecosystem is one of:

- **Single-vendor** deep-but-narrow skills — supabase (304K installs), prisma (6-skill family),
  better-auth (78K), neon, redis, convex, clerk. Excellent within one product; not a backend method.
- **Single-framework / single-language** — mcollina/fastify, yusukebe/hono, elysiajs, samber's Go
  family (one concern per skill, but Go-only), Spring/Rails/Django one-offs.
- **Thin breadth aggregators** — mindrally (widest coverage, template-generated near-duplicate
  boilerplate), affaan-m, jeffallan (persona family), giuseppe, ccheney.
- **The closest structural precedent — `wshobson/agents` `backend-development` plugin** (MIT):
  nodejs-backend-patterns, api-design-principles, postgresql-table-design, fastapi-templates,
  database-migration, microservices-patterns, python-background-jobs, dotnet-backend-patterns. But
  it's a *loose collection of per-topic skills*, not a requirements-gated, language/DB-agnostic
  router that decides the approach and holds the borders.

**No unified backend-engineering skill exists that** (a) routes by the *engineering job*, (b) speaks
the current hottest stack fluently as a *default* while teaching when to deviate, (c) is a superset
across languages/runtimes/datastores, (d) holds the border with architecture/data/ai/quality/operate,
and (e) is retrieval-first/anti-staleness by construction. **We build the server-side peer of the
shipped `frontend` skill** — the Build stage's back half, as one coherent faceted router.

---

## 2. What the research says (the load-bearing signals)

**The hottest stack is a *shape*, not a framework** (X + web + skills all converge independently):
a **TypeScript-first, Postgres-backed, auth-included, edge-or-Vercel-deployed "ship-fast" stack,
increasingly built to be operated by AI agents.** Concretely:

- **Postgres is the unquestioned default** — SO 2025 #1 (58.2%), DB-Engines fastest-growing 6 yrs.
  "Postgres for everything" has hardened *from meme to benchmarked practice*: pgvector, LISTEN/NOTIFY
  (with the disconnect/PgBouncer traps), jobs-table + `FOR UPDATE SKIP LOCKED`, full-text/BM25,
  PG19 native graph. Killer datapoint: **OpenAI serves 800M ChatGPT users on ONE Postgres primary +
  ~50 replicas.**
- **TS edge wave** — Hono (edge-first default, ~38M wk dl), Elysia (Bun, throughput niche), Fastify;
  **Drizzle passed Prisma in weekly downloads late 2025** for greenfield (Prisma 7 TS/WASM is the
  counter-punch; existing apps stay Prisma — teach both, declare no winner).
- **Better-Auth is the new TS auth default** (joined Vercel; Auth.js merged in).
- **Durable execution went mainstream** — Temporal (in OpenAI Agents SDK Feb 2026), Hatchet
  (Postgres-backed); ThoughtWorks Radar flags "Ignoring Durability in Agent Workflows" as Caution.
- **Zanzibar authz mainstreaming** — OpenFGA / SpiceDB / ory-keto (ReBAC beyond RBAC).
- **A "return to fundamentals" counter-narrative** — modular-monolith-first, Go stdlib (1.22 routing),
  SQL-before-ORM, at-least-once + idempotent consumers. *Not* a monolith reversal; "modular monolith
  for core + focused services only where the tradeoff pays."
- **THE META-SIGNAL — agent-native backends** (the 2026 throughline): every major backend tool ships
  an agent interface (Drizzle CLI/SDK, Supabase skills, Better-Auth `npx skills add`, Postgres-MCP,
  Sanity MCP), and the argument that backends should be *agent-legible* (typed contracts, legible
  errors, markdown — Rauch; Effect = "token-efficient for your AI" — Sullivan).

**Crowd-validated scope** (the "beyond CRUD" educator canon, 1.7–3.2K likes each): Caching · Auth &
AuthZ · Rate Limiting · Queues & Workers · Observability · Scaling · Security · communication patterns
(REST/req-resp/sync-async/polling/SSE/pub-sub). This maps almost 1:1 to the job list below — strong
external confirmation.

**Whitespace the skill can OWN** (thin across the whole ecosystem): idempotency, outbox/
transactional-messaging, saga/distributed-tx, connection-pool tuning + isolation-level correctness,
DB observability/slow-query, gRPC outside Go, Elixir/Phoenix + Rails + Rust/Axum depth, time-series/
search/wide-column/graph backend impl.

---

## 3. Proposed skill shape — faceted router (mirrors frontend/architecture)

One primary job × ≤1 base surface (+ agentic additive) + handoff. Thin SKILL.md, on-demand refs
(load 2–3, never preload).

### Primary jobs (pick one) — the engineering task · **9 refs**

| # | Job | Owns (implementation craft) |
|---|---|---|
| 1 | **api-implementation** | The server behind the contract: routing, middleware, request lifecycle, validation, serialization, error shape, pagination, versioning-in-code. REST / GraphQL / gRPC / tRPC **server** impl. Consumes architecture's contract. |
| 2 | **domain-logic** | Business/domain layer: use-cases, service layer, DDD *tactical* (aggregates/entities/VOs), clean/hexagonal layering, anti-anemic-model. (Boundaries = architecture.) |
| 3 | **persistence** | OLTP data access: ORM/query-builder/raw SQL, schema+index+migrations, transactions + isolation levels, connection pooling, N+1, "Postgres-for-everything" craft. |
| 4 | **auth** | AuthN/authZ *implementation*: sessions vs JWT, OAuth2/OIDC (PKCE, RFC 9700), Argon2id hashing, RBAC/ABAC/ReBAC enforcement, Better-Auth/Ory/Keycloak integration. |
| 5 | **async-and-messaging** | Jobs/queues/workers, event-driven, brokers, **outbox · saga · idempotency**, durable execution (Temporal/Hatchet), CDC, delivery semantics. |
| 6 | **caching-and-performance** | Caching strategies + invalidation, rate limiting (token-bucket), query/perf tuning, pooling-as-perf. |
| 7 | **server-security** | Implementing the threat model's controls: input validation, injection prevention, secrets, CORS, security headers, mTLS (OWASP ASVS/Top10). |
| 8 | **service-integration** | Outbound seams: HTTP/gRPC clients, retries+jitter/circuit-breakers, webhooks (in/out), calling an AI service / email / 3rd-party. |
| 9 | **backend-testing** | The skill's *own* verification loop: unit/integration/contract-producer tests, testcontainers, "prove-it-works." (Independent gate = quality.) |

### Surface overlays (add ≤1 base; agentic is additive) — the runtime/system type · **5 refs**

| Surface | Reshapes |
|---|---|
| **surface-rest-crud** (default) | Standard long-running request/response service (modular monolith or service); Node/Bun/Python/Go/JVM idioms. |
| **surface-serverless-edge** | Workers/Lambda/Bun edge: cold-start, **connection pooling / HTTP-native drivers** (the recurring serverless failure mode), D1/KV/R2, statelessness. |
| **surface-realtime** | WS/SSE/pub-sub, stateful connections, presence, fan-out/scaling (Centrifugo/nakama/Phoenix Channels/Durable Objects). |
| **surface-event-driven-microservices** | Distributed/polyglot: gRPC service-to-service, event streaming (Kafka/NATS), service mesh, inter-service contracts, the distributed-data patterns. |
| **surface-agentic** ⭐ **FLAGSHIP** | Backend for LLM/agent systems: durable execution as default, tool/function endpoints, **MCP servers**, agent-legible typed contracts + errors, best-practice skills for the datastore, RAG-retrieval plumbing (calls `ai`'s core, doesn't own it). |

### Handoff · **1 ref**
Consumes `architecture` (contracts, NFR budgets, threat model, ADRs); hands running services +
telemetry hooks → `operate`, own tests → `quality`; the `handoff.yaml` companion.

**Total: 9 + 5 + 1 = 15 references** (family scale). **Swing calls for your input** (see §6).

### Assets (fillable) · ~8
Backend-service scaffold (12-factor) · migration template (expand/contract, zero-downtime) ·
auth-implementation checklist (Argon2id params, cookie flags, OAuth-PKCE) · OWASP backend-controls
checklist (uniform table) · idempotency-key + outbox recipes · API-contract review checklist ·
verification-loop template (✓/✗ report + CI YAML) · `handoff.yaml`.

---

## 4. Universal invariants (the behavioral spine)

1. **Implement inside the decision.** Backend owns the running server; not the boundary/contract-shape/
   NFR-budget/threat-model *decision* (architecture), OLAP/pipelines (data), LLM/agent *core* (ai),
   deploy/CI/monitoring (operate), or the independent load-test/scan/gate (quality). Consume upstream
   artifacts; don't silently override them.
2. **Retrieval-first / anti-staleness — load-bearing, unique to backend's velocity.** Frameworks/ORMs/
   DBs ship breaking changes fast (Prisma 7, Drizzle 1.0, Go 1.22 routing, Kafka 4.0 ZooKeeper-removal,
   Redis licensing). The skill **fetches current framework/ORM/DB docs+types before generating** and
   version-gates idioms to the target release. Strongest structural signal in the corpus (every
   Cloudflare + Hono/Elysia skill opens this way).
3. **Default-stack, then deviate.** Name the current modal default per language (TS: Hono/tRPC +
   managed Postgres + Drizzle + Better-Auth; Python: FastAPI + SQLAlchemy/SQLModel + Alembic; Go:
   net/http + sqlc/pgx; …) — but every recommendation is a **concrete pick + what NOT to build +
   tradeoff**, weighted by reversibility. Anchor fundamentals (SQL-before-ORM, modular-monolith-first,
   at-least-once + idempotent consumers, parameterized queries). Adoption numbers are engagement, not
   gospel — don't trend-chase.
4. **Exhaustive checklists = uniform table + self-audit count, never narrative** (inherited AWS
   coverage-recall lesson) — OWASP controls, API-contract review, 12-factor audit, ASVS mapping.
5. **Prove-it-works before done.** Ship a verification loop (run tests, apply migrations, hit
   endpoints) as the completion gate — distinct from `quality`'s independent gate (django-verification
   donor).
6. **Single-home every canonical fact** — idempotency mechanics, OAuth flows, Postgres-for-everything,
   isolation levels each defined once with pointers elsewhere (avoid the dupes review caught in
   architecture).

---

## 5. Policy rulings

**5.1 License discipline (our copying).** Re-express + cite by default.
- **Verbatim OK (with attribution):** MIT/Apache/BSD/public-domain/CC-BY — Postgres, most frameworks
  (FastAPI/NestJS/Fastify/Hono/Axum/etc.), 12factor.net (MIT), Google AIP + MS REST (CC-BY-4.0),
  `wshobson/agents` (MIT), `samber/cc-skills-golang` (MIT), supabase/prisma/redis/mongo/neon skills
  (MIT/Apache).
- **Paraphrase + attribute (share-alike):** OWASP (CC-BY-SA-4.0).
- **NEVER verbatim — cite/link only:** microservices.io & martinfowler.com (© all-rights-reserved),
  refactoring.guru (© commercial), and **no-LICENSE skill repos** (yusukebe/hono-skill, elysiajs/
  skills, kadajett nestjs — which *falsely* frontmatter-claims MIT, better-auth/skills). We re-express
  everything regardless.

**5.2 Teach the DB *product* licenses (a distinctive value-add, not our copying).** The skill must
state the self-host/resale implications of **SSPL** (MongoDB, Redis-server), **BSL** (CockroachDB,
Memgraph, Timescale-features via TSL — TSL *bans* managed hosting), **AGPL** (Redis 8), and
**source-available** (ScyllaDB, Elastic tri-license, Typesense GPLv3, Encore MPL) — and name the
permissive escape hatches (Valkey, OpenSearch, YugabyteDB, Meilisearch-MIT). No competitor skill does
this.

**5.3 Verify-before-hard-claim.** The build confirms against primary sources or omits: the "42%
microservices rollback" / "40% regret" stats (**drop unless a primary CNCF/ThoughtWorks doc is
found**), InfluxDB 3.0 license, exact Milvus/Weaviate/Drizzle/Prisma LICENSE files, roadmap.sh MIT,
Oso's OSS-vs-Academy status, Auth.js→Better-Auth branding, Turso/Encore vendor-self-published
benchmarks. (All flagged inline in the corpora.)

**5.4 Border discipline** = invariant #1, enforced at review.

---

## 6. Swing calls I want your read on (or I'll take the default)

- **A. `realtime` as surface vs job.** Proposed: **surface** (stateful connections reshape everything).
  Alt: a 10th primary job. *Default if you don't pick: surface.*
- **B. Merge `service-integration` into `async-and-messaging`?** They share reliability patterns
  (retries/idempotency/circuit-breakers). Proposed: **keep separate** (inbound-async vs outbound-calls
  are distinct jobs). *Default: separate — 9 jobs.*
- **C. `surface-event-driven-microservices` — one surface or two?** Proposed: **one** (they co-occur).
  *Default: one.*
- **D. Ref count.** Proposed **15** (9+5+1). If you want leaner, I can fold to ~12 (merge B, drop a
  surface). *Default: 15.*

---

## 7. Composition crew plan (Phase 3 — needs its own go, per tokenomics)

Lean crew, mirroring the architecture build:
- **~6 sonnet writers** (grouped refs), each grounded in the 4 corpora + assigned donor repos already
  cloned in scratchpad.
- **1 opus integrator** — SKILL.md faceted router + plugin.json + SOURCES.md + cross-checks +
  single-home enforcement.
- **1 opus review** (correctness + license + border + single-home) **+ 1 sonnet eval**
  (activation/traversal/output/compression cases — model the samber `evals.json` with-vs-without
  pass-rate methodology).
- Controller does the small work itself (CHANGELOG, plugin bump, cross-ref fixes).

**Estimate: ~9 agents (6 sonnet + 2 opus + 1 sonnet eval), ~$25–50.** I'll ask for an explicit go on
this fan-out *after* you sign off the shape above — two separate gates.

---

## 8. Recommendation

**Build it as proposed** — 15-ref faceted router, `surface-agentic` as the flagship (the agent-native
throughline is the differentiator vs every existing single-purpose backend skill), retrieval-first and
default-stack-then-deviate as the load-bearing invariants, DB-license-teaching as the distinctive
value-add. Sign off §3 + §6, and I'll bring the composition-crew go-gate next.
