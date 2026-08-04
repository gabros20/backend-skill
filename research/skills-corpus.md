# Backend Research — Agent-Skills Discovery Corpus

**Channel:** agent-skills ecosystem discovery (via `find-skills` / `npx skills find` + skills.sh leaderboard) and local `~/.claude/skills` assessment.
**Lead:** Opus Lead A. **Method:** serial batched keyword sweeps (find-skills is NOT the throttled relay account, but sweeps were run serially and batched anyway per HARD RULE discipline), then 4 parallel sonnet deep-read workers (git clone + Read — not throttled) on the highest-value donor repos.
**Scope reminder:** we collect backend *implementation* craft (API/endpoint impl, domain logic, OLTP persistence + migrations, authN/Z impl, async/messaging/workers, caching + server perf, server-side security controls, service integration, backend testing). Upstream *decisions* (boundaries, contract shape, NFR budgets, threat model) belong to `architecture`; OLAP/ETL to `data`; LLM/agent core to `ai`; deploy/CI/CD/IaC/monitoring to `operate`; independent load-test/scan/gate to `quality`.

> NOTE: This doc's deep-read synthesis (donor STRUCTURE + "what to lift" detail) is being enriched by 4 sonnet workers whose raw reports live in
> `.../scratchpad/backend-research/reports/skills-w{1..4}-*.md`. Sections marked **[worker-enriched]** are folded in from those.

---

## 1. Keyword sweeps run

38 sweeps, serial/batched:
`backend, api, rest api, graphql, database, postgres, orm, prisma, auth, jwt` ·
`fastapi, django, express, nestjs, fastify, hono, spring boot, rails, golang, rust backend` ·
`microservice, kafka, message queue, redis, caching, websocket, serverless, migration, rbac, oauth` ·
`drizzle, sqlalchemy, grpc, trpc, celery background jobs, idempotency, clean architecture, ddd, backend testing, bun elysia` ·
`event-driven, vector database, connection pooling, supabase, convex, api design, sqlmodel litestar, durable execution workflow`.

Install counts below are from the skills.sh leaderboard at query time (2026-07-24) — a strong battle-tested/adoption signal.

---

## 2. Ranked findings (donor skills, source-linked)

Ranking blends install count (adoption), source authority (official vendor / framework-creator > known aggregator > unknown), and backend relevance/liftability. License column: repos not yet byte-verified are marked "verify" and confirmed by the deep-read workers (see §6 flags).

### Tier S — flagship donors (high install + high authority + directly liftable)

| Skill | Source (skills.sh / GitHub owner/repo) | Installs | Why it matters / what backend can lift |
|---|---|---|---|
| supabase-postgres-best-practices | supabase/agent-skills | **304.6K** | The single most-installed backend skill. Postgres schema/index/RLS/security best-practices from the vendor. Lift: OLTP schema + RLS + query-perf canon. **[w3]** |
| supabase | supabase/agent-skills | 181.1K | Full Supabase backend (auth, storage, edge functions, RLS). **[w3]** |
| convex-* (quickstart, performance-audit, setup-auth, migration-helper, create-component) | get-convex/agent-skills | ~90K each | Reactive backend-as-db. `performance-audit` + `migration-helper` are model "audit/migrate" skill shapes. **[borderline: proprietary runtime, but audit/migration STRUCTURE is liftable]** |
| better-auth-best-practices | better-auth/skills | **78.7K** | Modern TS auth (sessions, providers, 2FA). `create-auth-skill` shows a meta "build-an-auth-skill" recipe. Lift: authN/Z impl canon + "concept vs vendor" split. **[w4]** |
| neon-postgres | neondatabase/agent-skills | 55.7K | Serverless Postgres + driver adapters + connection pooling for edge. Lift: serverless-Postgres + pooling craft. **[w3]** |
| claude-api | anthropics/skills | 52.2K | (Adjacent) service-integration exemplar — how to call an external API well. |
| prisma-* (client-api 45.9K, database-setup 46.8K, cli 44.3K, postgres 41.1K, driver-adapter-implementation 37.3K, upgrade-v7 37.3K) | prisma/skills | ~40–47K each | Official ORM vendor; a whole *family* of ORM skills (setup / client / cli / postgres / driver-adapter / version-upgrade). Lift: ORM canon + the family-decomposition pattern + a `*-upgrade-v7` migration-skill shape. **[w3]** |
| nodejs-backend-patterns | wshobson/agents | **40.6K** | Top "backend" hit. Node service patterns. Part of the mega-repo (see §4). **[w1]** |
| golang-* (code-style, error-handling, testing, performance, design-patterns, security, grpc, graphql, database — ~34–36K each) | samber/cc-skills-golang | ~35K each | Comprehensive Go backend family, one concern per skill. Model decomposition for a language's backend craft. **[w1]** |

### Tier A — strong donors (framework-creator authored, or high install + focused)

| Skill | Source | Installs | Lift |
|---|---|---|---|
| fastify-best-practices | mcollina/skills | 29K | **Authored by Matteo Collina (Fastify co-creator)** — highest-authority Node framework skill. **[w2]** |
| hono | yusukebe/hono-skill | 10.3K | **Authored by the Hono creator.** Edge-first framework canon. **[w2]** |
| elysiajs | elysiajs/skills | 8K | Official ElysiaJS/Bun skill — the hot Bun runtime stack. **[w2]** |
| nestjs-best-practices | kadajett/agent-nestjs-skills | 23K | Most-installed NestJS skill (DI, modules, guards, interceptors). **[w2]** |
| api-design-principles | wshobson/agents | 25.1K | REST/versioning/pagination/errors canon — backend consumes the contract. **[w1]** |
| postgresql-table-design | wshobson/agents | 22.2K | Schema/normalization/index design. **[w1]** |
| fastapi-templates | wshobson/agents | 21.8K | FastAPI scaffolds. **[w1]** |
| java-springboot / kotlin-springboot / create-spring-boot-*-project | github/awesome-copilot | 8–18K | Official-ish GitHub collection; Spring Boot canon + project scaffolding. |
| database-migration | wshobson/agents | 14.4K | Migration workflow (expand/contract, zero-downtime). **[w1]** |
| apollographql/* (rust-best-practices 14.3K, graphql-schema, graphql-operations, apollo-client) | apollographql/skills | 2.8–14.3K | Official GraphQL vendor; schema + operations canon. |
| redis-* (development 3.2K, core, connections, security, observability) | redis/agent-skills | 0.8–3.2K | Official Redis; caching + connection + observability decomposition. **[w3]** |
| microservices-patterns | wshobson/agents | 10.4K | Service decomposition/impl patterns. **[w1]** |
| python-background-jobs | wshobson/agents | 8.4K | Celery/RQ/async worker patterns. **[w1]** |
| backend-patterns / django-* / fastapi-patterns / nestjs-patterns / springboot-patterns | affaan-m/everything-claude-code | 2–8.9K | Broad multi-framework family; **django-tdd + django-verification model a "prove-it-works" loop**. **[w4]** |
| dotnet-backend-patterns | wshobson/agents | 15.7K | .NET backend (underrepresented elsewhere). **[w1]** |
| clerk-backend-api | clerk/skills | 24.9K | Vendor auth backend integration. |
| mongodb-connection | mongodb/agent-skills | 3K | Official Mongo connection/pooling. **[w3]** |

### Tier B — useful/breadth or community (cite; verify quality)

| Skill(s) | Source | Installs | Note |
|---|---|---|---|
| fastapi-python, express-typescript, hono-typescript, fastify-typescript, trpc, grpc-development, kafka-development, rabbitmq-development, websocket-development, microservices, go-backend-microservices, clean-architecture, serverless, fastapi-microservices-serverless | mindrally/skills | 0.5–11.5K | Aggregator with the **widest backend breadth** — good gap-map, variable depth. **[w2]** |
| spring-boot-rest-api-standards, spring-boot-security-jwt, spring-boot-event-driven-patterns, clean-architecture, drizzle-orm-patterns, nestjs-drizzle-crud-generator, qdrant-vector-database-integration, unit-test-wiremock-rest-api | giuseppe-trisciuoglio/developer-kit | 0.1–2.4K | Java/Spring + Drizzle + event-driven + vector; CRUD-generator + wiremock-test shapes. **[w3][w4]** |
| clean-ddd-hexagonal, postgres-drizzle | ccheney/robust-skills | 1.2–3.9K | DDD/hexagonal + Drizzle-on-Postgres. **[w4]** |
| fastapi-expert, django-expert, nestjs-expert, rails-expert, api-designer, microservices-architect, websocket-engineer | jeffallan/claude-skills | 3–5.2K | "Expert persona" family across frameworks. **[w4]** |
| drizzle-orm, sqlalchemy-orm, trpc-type-safety, golang-grpc, hono-cloudflare | bobmatnyc/claude-mpm-skills | 0.1–4.4K | Broad community catalog. |
| sqlalchemy-alembic-expert-best-practices-code-review | wispbit-ai/skills | 1.4K | SQLAlchemy+Alembic as a **code-review** skill shape. |
| fastapi (official) | fastapi/fastapi | 6.1K | Official FastAPI repo skill (MIT). **[w4]** |
| trpc-router (official) | trpc/trpc | 0.2K | Official tRPC. |
| litestar / advanced-alchemy-litestar | litestar-org/litestar-skills, alti3/litestar-skills | <30 | Litestar (hot Python ASGI) — low install, official-adjacent. |
| sqlmodel (official) | fastapi/sqlmodel | 18 | Official SQLModel. |
| redis-development | upstash/skills (upstash-redis-js) | 1.9K | Edge/serverless Redis (Upstash). |
| golang-graphql, golang-database | samber/cc-skills-golang | ~33K | (also in Tier S family) |
| express-oauth2-jwt-bearer, auth0-fastify-api, auth0-fastify | auth0/agent-skills | 0.5K | Vendor OAuth/JWT middleware integration. **[w4]** |
| jwt-security | mindrally/skills | 1.2K | Concept-level JWT hardening. |

### Security-oriented (backend controls; borderline `quality`, but impl-side is ours)
- **yaklang/hack-skills** — `api-auth-and-jwt-abuse`, `jwt-oauth-token-attacks`, `oauth-oidc-misconfiguration`, `expression-language-injection`, `websocket-security` (~2K each). Attacker's-eye checklists → invert into backend *hardening* rules. Cite/verify license (offensive-security repo).
- **giuseppe-trisciuoglio** `spring-boot-security-jwt`; **mindrally** `jwt-security`. Defensive impl.

---

## 3. What backend can LIFT (structure & content patterns) — [worker-enriched, provisional]

From local skills (byte-read; see §5) + discovery signals; deep-read workers confirm/extend:

1. **Retrieval-first / anti-staleness meta-pattern.** The Cloudflare skill family (workers-best-practices, durable-objects, agents-sdk, wrangler) all open with a "Retrieval Sources / FIRST: Fetch Latest References" block and explicitly *bias toward live docs over pretrained knowledge*. For a fast-moving backend (frameworks/ORMs ship breaking changes), this is the single most important structural lift — the skill should fetch current framework/ORM types/docs before generating.
2. **"When to Use / Do NOT Use For" gating** (durable-objects) — maps directly to backend's runtime/pattern selection (when a queue vs cron, DO vs Redis, WS vs SSE), while keeping the *decision* deferring to `architecture`.
3. **Rules Quick Reference → Anti-Patterns to Flag → Review Workflow** (workers-best-practices) — a three-part shape: positive rules, a named anti-pattern list ("floating promises, global state, secrets in code, missing observability"), and a review pass. Ideal skeleton for backend's per-concern sections.
4. **One-concern-per-skill decomposition** (samber Go family: code-style / errors / testing / perf / security / grpc / graphql / database; prisma family; redis family). A backend *family* should split by concern, not cram one megafile.
5. **Family-with-migration-skill** (prisma-upgrade-v7, convex-migration-helper) — ship a dedicated version-upgrade/migration skill alongside the core.
6. **"Prove-it-works" verification loop** (affaan-m django-tdd + django-verification) — a backend skill needs an explicit verification layer (run tests, hit endpoints, check migrations apply) even though independent gating lives in `quality`.
7. **Phased build workflow + per-phase reference loading** (mcp-builder: Research→Implement→Review→Evals, with reference files loaded per phase) — model for backend's service-integration / agentic build path.
8. **Concept-vs-vendor split** (better-auth concept + auth0 vendor middleware) — teach the portable concept, then the vendor impl, separately.

---

## 4. The mega-repo pattern: wshobson/agents — [w1, pending]

`wshobson/agents` is the single largest backend donor (nodejs-backend-patterns 40.6K, api-design-principles 25.1K, postgresql-table-design 22.2K, fastapi-templates, database-migration, microservices-patterns, python-background-jobs, dotnet-backend-patterns, signed-audit-trails-recipe, +more). Its dir layout / naming / shared conventions are themselves a liftable *organization* model for a multi-surface backend skill. Worker w1 is capturing the meta-structure.

---

## 5. Local `~/.claude/skills` assessment (byte-read frontmatter + structure)

Directly backend-relevant, already on disk — highest-trust donors we own:

- **workers-best-practices** — STRUCTURE: `Retrieval Sources` → `FIRST: Fetch Latest References` (fetches current Workers types) → `Rules Quick Reference` (Configuration / Request & Response / Architecture / Observability / Code Patterns / Security) → `Anti-Patterns to Flag` → `Review Workflow` → `Scope` → `Principles`, plus a `references/` dir. **Lift: the whole anti-pattern-review skeleton + retrieval-first opener.** Best single structural donor we own.
- **durable-objects** — `When to Use` + `Use For / Do NOT Use For` + `Core Principles` + `Critical Rules` + `Anti-Patterns (NEVER)` + Storage/Alarms/Testing (Vitest) + `references/`. **Lift: stateful/real-time backend (WS, coordination, SQLite-at-edge) + the "when/when-not" gating + testing-quickstart.**
- **agents-sdk** — Capabilities + retrieval-first + references grouped (Core / Chat&Streaming / Background Processing / Integrations / Experimental). **Lift: async/background-processing + durable-workflow + queues/retries/observability framing; reference categorization.**
- **mcp-builder** — phased workflow (Research→Implement→Review→Evals) + a per-phase Documentation Library + `scripts/` + `reference/` + LICENSE. **Lift: service-integration build path; the "create evaluations" phase as a verification model.** (Python FastMCP + Node SDK dual-track = concept-vs-language split.)
- **sandbox-sdk** — secure untrusted-code execution (lifecycle, commands, files, preview URLs). **Lift: sandboxing / code-exec backend niche.**
- **cloudflare-email-service** — transactional email integration (Workers binding + REST + SPF/DKIM/DMARC). **Lift: service-integration exemplar (email), incl. deliverability controls.**
- **cloudflare** (umbrella) — KV/D1/R2/Vectorize/Hyperdrive/Queues/Workflows. **Lift: edge/serverless persistence + queue/workflow vocabulary.** Border: deploy/IaC parts → `operate`.
- **wrangler** — CLI for Queues/Workflows/D1/R2/Hyperdrive/Secrets. Mostly `operate` border, but Queues/Workflows/D1 usage is backend.
- **cloudflare-one / cloudflare-one-migrations** — Zero-Trust/SASE. Mostly out of scope (network/security ops → `operate`/`quality`).
- **github-code-search** — research tool, not backend content.

**Meta-takeaway from the CF family:** every one opens retrieval-first and biases to live docs — strongest confirmation that a backend skill must be anti-staleness by construction.

---

## 6. Hottest-stack signals (2025–2026) — [worker-enriched, provisional]

From skill *existence + install momentum* (a real adoption proxy) and creator-authored skills:

- **Edge/Bun runtimes are real, not hype.** Framework-*creator*-authored skills exist for **Hono** (yusukebe, 10.3K) and **ElysiaJS/Bun** (official elysiajs, 8K); Fastify's own creator ships `fastify-best-practices` (mcollina, 29K). Signal: Hono (edge-first) and Elysia (Bun) have crossed into "the maintainers write agent skills for them" territory.
- **Postgres-as-everything.** Postgres dominates the DB sweeps by an order of magnitude (supabase-postgres-best-practices 304.6K, neon-postgres 55.7K, prisma-postgres 41.1K, postgresql-table-design 22.2K). MySQL/Mongo/others trail hard. Vector lives *inside* Postgres/pgvector + Qdrant as the standalone.
- **Drizzle vs Prisma is the live ORM debate.** Prisma ships a 6-skill official family (~40K each) incl. `driver-adapter-implementation` (edge/serverless). Drizzle shows up across giuseppe (drizzle-orm-patterns), ccheney (postgres-drizzle), bobmatnyc, lobehub, jezweb (d1-drizzle). Prisma = install king; Drizzle = community-momentum challenger, especially on edge/serverless + SQLite/D1/Turso.
- **Serverless-Postgres + driver adapters + connection pooling** recur as an explicit edge concern (Neon, Prisma driver-adapter, Upstash Redis-over-HTTP). This is a distinct backend sub-skill.
- **tRPC** present but modest (bobmatnyc trpc-type-safety 1.1K, mindrally trpc, official trpc-router 0.2K) — type-safe RPC is known but nowhere near Postgres/auth adoption.
- **Reactive/BaaS backends** (Convex ~90K family, Supabase 300K+) are a major adoption story — "backend without a backend." Their *audit/migration/perf* skill shapes are liftable even where the runtime is proprietary.
- **Go** = the strongest *traditional* backend-language signal (samber family ~35K/skill across the full concern matrix). Rust backend is thin (windmill rust-backend 0.8K; apollo rust-best-practices 14.3K is GraphQL-router-specific). Litestar/SQLModel (hot Python ASGI) exist but low-install.
- **Auth is consolidating on Better Auth** (78.7K) for TS + vendor stacks (Clerk 24.9K, Auth0, Firebase 112.7K, Supabase auth). Session-vs-JWT + OAuth/OIDC hardening is a mature, well-covered area.

*(Deep-read workers add named tradeoffs + who's-using-it detail to each of the above.)*

---

## 7. Gaps & uncertainties

- **License hygiene not yet byte-verified** for most external repos — confirmed by workers in their reports; flags collected in the return summary. Community aggregators (mindrally, affaan-m, bobmatnyc, jeffallan, giuseppe, ccheney) especially need LICENSE checks before any verbatim lift (we re-express by default regardless).
- **Thin coverage** in the ecosystem for: idempotency (only prompt/community skills), outbox/transactional-messaging, saga/distributed-tx, connection-pool tuning depth, isolation-level correctness, DB observability/slow-query, gRPC outside Go, Elixir/Phoenix, Rails depth, Rust/Axum depth, time-series (Timescale/Influx), search (Elastic/Meili/Typesense), wide-column (Cassandra/Scylla), graph DB backend impl. These are **whitespace** the `backend` skill can own rather than lift.
- **Border discipline risk:** many "microservices/clean-architecture/event-driven/DDD" skills mix *decision* (architecture's) with *impl* (ours). We lift only the impl craft.
- **Install count ≠ quality.** Some high-install skills are thin; some framework-creator skills (Hono/Fastify/Elysia) are lower-install but far denser/more trustworthy — weight authority over raw installs.
- find-skills surfaces top ~6 per query; the long tail (Phoenix, Axum, Timescale, etc.) is under-sampled here and largely confirms the whitespace above.

---

*Deep-read raw reports: `.../scratchpad/backend-research/reports/skills-w{1..4}-*.md`. This corpus is source-linked per HARD RULE 2; verbatim lifts require the per-repo license confirmed in §6/worker reports (we re-express by default).*
