# Backend Skill — Web-Canon Corpus (Channel D)

_Channel: authoritative web canon via WebSearch/WebFetch. Research only — no skill content. Assembled 2026-07-24 from 5 parallel sonnet workers._

## 1. Channel, method, keyword sweeps

- **Channel:** Web canon — official framework docs, standards bodies (IETF RFCs, OWASP, OpenID), survey/radar data (Stack Overflow, State of JS, JetBrains, DB-Engines, ThoughtWorks), and authoritative pattern sites (12factor.net, microservices.io, martinfowler.com). Via WebSearch + WebFetch. **Jina key not used (compromised).**
- **Method:** 5 sonnet workers fanned out in parallel (web fetches don't hit the throttled relay, so parallel is safe), one per cluster: (1) frameworks by language, (2) databases + when-to-reach-for-each, (3) patterns canon + API design, (4) auth + security implementation, (5) hottest-stack landscape 2025–2026.
- **Raw worker reports (nothing lost):** `scratchpad/backend-research/reports/web-1-frameworks.md`, `web-2-databases.md`, `web-3-patterns.md`, `web-4-authsec.md`, `web-5-landscape.md`.
- **Sweeps run:** per-framework "official docs + best practices" (all languages); per-DB "when to use vs alternatives + license"; pattern-site canonical pages (12-factor, microservices.io patterns, Fowler bliki, refactoring.guru, roadmap.sh, Google AIP, Zalando, Richardson maturity, GraphQL/gRPC); OWASP ASVS/Top10/Cheat-Sheets + OAuth/OIDC RFCs + authz-model landscape; radar/survey/DB-Engines + named hot-movement verification.

## 2. Ranked findings

### 2.1 Frameworks by language

Cross-cutting signal first: **only Symfony has a first-party, prescriptive "Best Practices" doc** (https://symfony.com/doc/current/best_practices.html). For almost every other framework "best practices" is community consensus, not vendor doctrine (FastAPI's widely-cited community repo https://github.com/zhanymkanov/fastapi-best-practices is the notable near-exception). Useful framing for the skill. All frameworks below are **MIT or Apache-2.0/BSD** unless flagged — permissive, re-express-and-cite.

**Python**
- **FastAPI** (https://fastapi.tiangolo.com/, MIT) — async-first, OpenAPI/JSON-Schema auto-generated from type hints; Pydantic models for validation/serialization; `Depends()` function-based DI incl. `yield`-teardown; `async def` on the loop, `def` auto-offloaded to a threadpool (so sync DB drivers don't stall the loop). No bundled ORM (pairs w/ SQLAlchemy or SQLModel). **The default new-Python-API pick;** Stack Overflow 2025 shows it +5pts, the biggest framework mover. **Lift:** the reference model for "type-hints → contract → validation."
- **Django + DRF** (https://docs.djangoproject.com/, https://www.django-rest-framework.org/, BSD-3) — batteries-included; `ModelSerializer` + `ViewSet`+`Router` for CRUD; guidance = push business logic into a service layer, side-effects into signals/Celery, keep views thin. Best for admin-heavy/relational-first monoliths. Mature, not hot.
- **Litestar** (https://docs.litestar.dev/, MIT) — structured/faster FastAPI alternative; explicit DI + dedicated **DTO layer** (`PydanticDTO`/`MsgspecDTO`/`SqlAlchemyDTO`); **msgspec** in hot paths (~12x faster than Pydantic v2 per migration write-ups). **Rising challenger.** FastAPI→Litestar migration guide doubles as idioms ref.
- **Flask** (https://flask.palletsprojects.com/, BSD-3) — WSGI micro-framework; Blueprints = modularity primitive; everything opt-in. Stable, ceded greenfield-async to FastAPI/Litestar.
- **SQLModel** (https://sqlmodel.tiangolo.com/, MIT) — one class = Pydantic model + SQLAlchemy table (kills dup model/schema). Niche to FastAPI stack; **flagged declining in favor** (polymorphic-model gaps, stale PRs per landscape worker).

**Node/Bun/TS**
- **NestJS** (https://docs.nestjs.com/, MIT) — opinionated enterprise framework; IoC/DI container, modules/providers, request pipeline middleware → **guards** (authz) → interceptors → **pipes** (validate/transform, class-validator or Zod); runs on Express or Fastify adapter. ~75k stars, **the enterprise Node default.**
- **Fastify** (https://fastify.dev/, MIT) — JSON-Schema validation **compiled at startup** (Ajv) into fast validator/serializer fns = its speed edge; plugin-encapsulation architecture; 45–70k+ req/s cited. **Hot pick.**
- **Hono** (https://hono.dev/, MIT) — ultra-light, zero-dep, **Web-standards (Fetch API)** framework running identically on Workers/Deno/Bun/Node/Lambda; `hono/client` RPC gives end-to-end types with **zero codegen**. **The edge/serverless default;** ~38M weekly downloads. **Hot pick.**
- **Elysia** (https://elysiajs.com/, MIT) — Bun-native; **Eden Treaty** e2e type safety, zero codegen; up to 2.5M req/s raw. Locked to Bun. ~18k stars, ~460k weekly dl (80:1 download gap vs Hono). In prod at X, CS.Money. **Hot pick, throughput-first niche.**
- **Express** (https://expressjs.com/, MIT) — original minimal framework, still the largest install base (~40–55%), but framed as the safe-legacy option, not the new-project default. v5 current.
- **tRPC** (https://trpc.io/, MIT) — type-safety layer for TS monorepos (not REST); procedures + Zod, inferred client types, no codegen. TS-both-ends only. Spread via T3 stack (create-t3-app 100k+ projects). **Layered alongside REST, not replacing it.**
- **Nitro** (https://nitro.build/, MIT) — universal server toolkit (built on h3), "deploy anywhere" presets; powers Nuxt + SolidStart/Analog/TanStack Start. Rising as shared substrate. Secondary for standalone APIs.
- **effect-ts / @effect/platform** (https://effect.website/, MIT) — ZIO-style functional-effect runtime w/ HTTP layer; typed errors, Layer-based DI, runtime-agnostic. **High-commitment, contested** ("wants to own your control flow," all-or-nothing). Niche but growing (fintech/dev-tools).

**Rust** (all dual MIT/Apache-2.0)
- **Axum** (https://docs.rs/axum/, https://github.com/tokio-rs/axum) — Tokio team's framework; **no bespoke middleware** — built on `tower::Service`, inherits the whole tower/tower-http ecosystem; `FromRequest` extractors. Pairs w/ sqlx/sea-orm. **The new pragmatic Rust default;** crates.io + Lichess run on it. **Hot pick.**
- **Actix-web** (https://actix.rs/) — longest-established, highest raw throughput (N single-threaded runtimes, one per core; actor model). ~10–15% more req/s than Axum but Axum is winning mindshare.

**Go**
- **net/http stdlib** (https://pkg.go.dev/net/http, BSD-3) — **Go 1.22 added method matching (`GET /items/{id}`) + wildcard path params (`r.PathValue`)** to ServeMux, removing most reasons to reach for a router. **"Stdlib is all you need" shift** — a real build-vs-buy change. (https://go.dev/blog/routing-enhancements)
- **Gin** (https://gin-gonic.com/, MIT) — most-adopted Go framework; onion middleware (`c.Next()`); struct-tag binding+validation (`binding:"required"`, go-playground/validator). The "Express of Go."
- **Echo** (https://echo.labstack.com/, MIT) — zero-alloc router, `Group()`, centralized error handling. Smaller mindshare than Gin.
- **Fiber** (https://docs.gofiber.io/, MIT) — Express-style, built on **fasthttp (not net/http)** = raw speed but loses stdlib compatibility. **v3 is beta / not prod-recommended;** use v2.x.
- **Encore.go** (https://encore.dev/docs/go, **MPL-2.0 OSS core** + proprietary Encore Cloud) — **infra-from-code**: databases/Pub-Sub/cron/caches declared as typed Go objects, provisioned to real AWS/GCP. Distinctive pitch vs plain routers. ~11k stars, Groupon cited. **Hot/niche.** (License flag: MPL-2.0 file-level copyleft on the SDK.)
- **Chi** (https://go-chi.io/, MIT) — **100% net/http-compatible** composable router (<1000 LOC core), sub-router mounting. In prod at Cloudflare/Heroku. The stdlib-fidelity pick.

**Java/Kotlin** (all Apache-2.0)
- **Spring Boot** (https://docs.spring.io/spring-boot/) — dominant enterprise default; **constructor injection** preferred; layering = controllers inject services (not repositories); Spring Data JPA repos auto-detected. Blocking MVC vs reactive **WebFlux** (Reactor) is a real decision point (**gap: WebFlux not deep-researched**).
- **Quarkus** (https://quarkus.io/) — "Supersonic Subatomic," **GraalVM native-image** = ms startup + 4–10x lower memory; CDI-based ArC DI; built on Jakarta standards (JAX-RS/JPA). The pick when JVM startup/footprint is a dealbreaker (K8s/serverless).
- **Ktor** (https://ktor.io/) — JetBrains' coroutine-native Kotlin framework; plugin-based (routing itself is a plugin). The Kotlin-first pick vs Java-lineage Spring.

**Ruby / Elixir / PHP**
- **Rails API mode** (https://guides.rubyonrails.org/api_app.html, MIT) — `rails new --api` strips browser middleware, keeps ActiveRecord + generators. Steady; the Hotwire "resurgence" is about full-stack mode, not API mode.
- **Phoenix** (https://hexdocs.pm/phoenix/, MIT) — Elixir/BEAM/OTP; **Contexts** = domain-boundary organizing convention; **Channels** = supervised OTP processes per WS connection (a crash isolates to one connection); LiveView pushes DOM diffs. Strong for real-time/high-concurrency.
- **Laravel** (https://laravel.com/docs/, MIT) — dominant PHP framework; **service container** central to DI; Eloquent ORM (watch N+1 → eager-load); thin controllers, service/repository layers.
- **Symfony** (https://symfony.com/doc/, MIT) — component-based; autowiring + autoconfiguration; **make services private** (forces constructor DI). Only framework here with an **official Best Practices doc.**

### 2.2 Databases + persistence

_License-heavy cluster — flags are load-bearing. DB-Engines full top-20 not fetched (TLS/cert error); ranks from press releases + SO survey._

**Relational / OLTP**
- **PostgreSQL** (https://www.postgresql.org/, **PostgreSQL License** — permissive BSD/MIT-like) — the default for new backends; ACID, MVCC, JSONB, extensions. **#1 in Stack Overflow 2025 (58.2%), tops admired (65%) + desired (46%); DB-Engines fastest-growing 6 yrs straight (+21.97 pts H1 2026), on track to reach #3 by late 2026.** Wins on complex SQL/analytics/extensibility vs MySQL. **Top donor.**
- **MySQL/MariaDB** (https://www.mysql.com/ GPL + Oracle commercial dual-license; https://mariadb.org/ GPLv2/LGPL community-governed) — MySQL = broadest managed-service ubiquity/ecosystem; MariaDB = BSL-free, ColumnStore, faster simple reads. Win on raw read throughput + hosting ubiquity.
- **SQLite / libSQL / Turso** (https://www.sqlite.org/ **public domain**; libSQL https://turso.tech/libsql **MIT**; Turso service) — SQLite = embedded/local/test; libSQL = SQLite + network protocol; Turso = distributed libSQL w/ **embedded replicas** (local-disk read speed, writes to primary) across 35+ regions. For edge/high-read-ratio/multi-tenant. Younger ecosystem tradeoff.

**NewSQL / distributed**
- **CockroachDB** (https://www.cockroachlabs.com/, **BSL 1.1** → Apache after ~3yr) — Postgres-wire, auto-shard/replicate, serializable across regions. For multi-region active-active + data residency.
- **TiDB** (https://www.pingcap.com/, Apache-2.0) — MySQL-compatible, **HTAP via TiFlash** columnar replicas. For MySQL migration + analytics-on-OLTP. Weak under high WAN latency.
- **YugabyteDB** (https://www.yugabyte.com/, **Apache-2.0**) — reuses Postgres query layer on Spanner-style storage. The open-license Spanner-like pick; lacks native OLAP engine.
- **Google Spanner** (https://cloud.google.com/spanner, proprietary GCP) — TrueTime external consistency; strongest global consistency, full lock-in.

**Key-value / cache**
- **Redis / Valkey** — **the licensing saga (precise):** Redis ≤7.2.4 BSD-3 → **Mar 2024 SSPL/RSAL (source-available)** → Linux Foundation forked **Valkey** (https://valkey.io/, BSD-3, backed by AWS/Google/Oracle) within days → **Redis 8.0 (May 2025) added AGPLv3**, now tri-licensed. **Valkey** = permissive, full protocol compat, AWS recommends it (cheaper on ElastiCache). **Redis 8** = if AGPLv3 OK. (https://redis.io/) Redis rising +8pts in SO 2025 (#5 DB).
- **DynamoDB** (https://aws.amazon.com/dynamodb/, proprietary AWS) — managed KV+doc, single-table design, single-digit-ms (pair w/ DAX for cache tier). For AWS + zero-ops + spiky traffic.

**Document**
- **MongoDB** (https://www.mongodb.com/, **SSPL — not OSI-approved**; forces cloud providers to open-source their whole hosting stack, which is why AWS/GCP/Azure built DocumentDB/Firestore/Cosmos instead). BSON, flexible schema, native sharding + time-series collections. **2026 pragmatic default for new apps is Postgres+JSONB** (benchmarks within ~18%, sometimes faster); reserve Mongo for write-heavy horizontal-shard or IoT. **Flag SSPL loudly if building a hosted product.**

**Wide-column**
- **Cassandra** (https://cassandra.apache.org/, **Apache-2.0**) — masterless, multi-DC, tunable consistency; 5.0 adds vector search. The safe open-license choice for billions of writes/day.
- **ScyllaDB** (https://www.scylladb.com/, **ScyllaDB Source-Available License**, was AGPL) — C++ shard-per-core Cassandra-compatible rewrite, 5–10x throughput/node. Perf-per-dollar over license permissiveness.

**Graph**
- **Neo4j** (https://neo4j.com/) — native property graph, Cypher, broadest ecosystem, GraphRAG. **Memgraph** (https://memgraph.com/, **BSL**) — in-memory streaming graph. **Neptune** (https://aws.amazon.com/neptune/, proprietary) — managed, Gremlin/openCypher + SPARQL. **Dgraph** (https://dgraph.io/, Apache-2.0) — distributed native-GraphQL, the OSS distributed pick.

**Vector**
- **pgvector** (https://github.com/pgvector/pgvector, permissive) — Postgres extension, HNSW/IVFFlat, no separate service. **Start here if on Postgres + under ~500k vectors.** **Qdrant** (https://qdrant.tech/, Apache-2.0, Rust) — graduate here for scale/payload filtering. **Weaviate** (https://weaviate.io/, BSD-3) — hybrid vector+keyword. **Milvus** (https://milvus.io/, Apache-2.0, LF AI&Data) — billion-scale, needs ops team. **Pinecone** (https://www.pinecone.io/, proprietary) — zero-ops managed.

**Time-series / search / embedded**
- **TimescaleDB** (https://www.tigerdata.com/, **split license** — Apache-2.0 core, **Timescale License (TSL, source-available)** on the features you actually want (columnar compression, continuous aggregates, retention); **TSL prohibits offering it as a managed service**). Time-series inside Postgres/SQL. **InfluxDB** (https://www.influxdata.com/, **mixed license per component/version — verify**) — purpose-built TSDB, 3.0 on Arrow/Parquet/DataFusion.
- **Elasticsearch** (https://www.elastic.co/, **tri-licensed AGPLv3/ELv2/SSPL since Aug 2024**, was Apache until 2021) vs **OpenSearch** (https://opensearch.org/, **Apache-2.0**, AWS fork, LF-governed) — OpenSearch = hard OSS guarantee + free security features. **Meilisearch** (https://www.meilisearch.com/, **MIT**, DX-first, strong multi-language) vs **Typesense** (https://typesense.org/, **GPLv3**, lower RAM, all features free). Note the MIT-vs-GPLv3 obligation difference.
- **DuckDB** (https://duckdb.org/, **MIT**) — "SQLite of OLAP," in-process columnar analytics. Local ETL/ad-hoc SQL, not OLTP. (Borders `data` skill — implementation-adjacent only.)

**ORMs / migrations (the Drizzle-vs-Prisma verdict)**
- **TS:** **Prisma** (https://www.prisma.io/) — schema-first DSL + generated client; **Prisma 7 (Nov 2025) replaced the Rust engine with TS/WASM**, bundle ~14MB→~1.6MB, native edge — a direct answer to Drizzle. Still most-installed (~2.5M weekly). **Drizzle** (https://orm.drizzle.team/) — code-first TS schema, readable SQL, zero-dep, best cold-start; **v1 stable 2026, passed Prisma in weekly downloads late 2025**, ~2x new-project rate. Prod: Replit/Sentry/Figma/Astro. **Flow is one-directional Prisma→Drizzle for greenfield; Prisma stays for existing/enterprise.** **TypeORM = maintenance mode, no reason for new projects.** Verdict: Drizzle for SQL-literate/edge; Prisma for max abstraction/tooling.
- **Python:** SQLAlchemy + **Alembic** (framework-agnostic migrations) vs Django ORM (built-in, auto conflict resolution). **Go:** GORM (convenience) vs **sqlc** (type-safe codegen from hand-written SQL). **Rust:** Diesel (compile-checked) / SeaORM (ActiveRecord-style, v2.0 Jan 2026) / SQLx (raw + compile-time checked). **Elixir:** Ecto (explicit changesets, no N+1 by design). **Ruby:** ActiveRecord (convenient, N+1-prone).

**"Postgres for everything"** (https://postgresforeverything.com/) — **rising from meme to benchmarked practice.** One Postgres replaces Redis/queue/search/vector for most apps: pgvector (embeddings next to rows, kills dual-write), **LISTEN/NOTIFY** pub-sub (but lost on disconnect, global commit lock until PG19, **breaks under PgBouncer transaction-mode** — prod pattern is a jobs table + `SELECT ... FOR UPDATE SKIP LOCKED` or **pgmq**), UNLOGGED-tables-as-cache, full-text search, **PostgREST** (https://postgrest.org, auto REST from schema respecting RLS). Tradeoff: less infra vs deeper PG ops expertise.

**Serverless Postgres:** **Neon** (https://neon.tech, CoW branching, scale-to-zero; **Databricks acquired ~$1B 2025**; **Vercel killed its own Postgres June 2025, migrated all to Neon**) / **Supabase** (https://supabase.com, full BaaS: auth/storage/realtime + PostgREST + pg_graphql) / **PlanetScale** (https://planetscale.com, Vitess/MySQL + Postgres GA Sep 2025).

**Cross-cutting:** **Connection pooling** — **PgBouncer** (https://www.pgbouncer.org/): session mode (safest) vs transaction mode (**1.21+ now supports prepared statements**) vs statement mode. **Near-mandatory in serverless** (Postgres process-per-connection doesn't scale to thousands of function invocations) — or use HTTP-native drivers (Neon HTTP, Turso) to sidestep TCP-per-invocation. **Isolation levels** (https://www.postgresql.org/docs/current/transaction-iso.html): Serializable→Repeatable-Read→Read-Committed→Read-Uncommitted. **Skill rule to teach:** Read Committed default for CRUD; Serializable / `SELECT FOR UPDATE` / optimistic-versioning around money/inventory/uniqueness; **MySQL default is Repeatable Read, Postgres is Read Committed — a cross-DB portability trap.**

### 2.3 Patterns canon + API design

**The canon backbone (reference-only / cite-don't-copy):** **microservices.io** (Chris Richardson, © all rights reserved) + **martinfowler.com** (© Fowler). **The re-usable MIT/CC-BY sources:** **12factor.net** (MIT), **Google API Design Guide / AIPs** (CC-BY-4.0 + Apache code), **Microsoft REST guidelines** (CC-BY-4.0), **roadmap.sh** (MIT, unverified this pass).

- **12-Factor App** (https://12factor.net/, **MIT**) — codebase / dependencies / **config-in-env** / backing-services-as-attached-resources / **build-release-run separation** / stateless processes / port binding / concurrency via process model / **disposability (fast start, graceful SIGTERM)** / dev-prod parity / logs-as-stdout-streams / admin-as-one-off-processes. **Lift:** the shared baseline vocabulary. Layer container/K8s + secrets-manager nuance on top (env-var config critique).
- **microservices.io pattern catalog** (https://microservices.io/patterns/microservices.html) — **API Gateway/BFF**, **Database-per-Service** (private data, forces Saga + CQRS/API-composition), **Saga** (choreography vs orchestration; local txns + compensations, gives up isolation), **Transactional Outbox** (write message in same txn as data, relay publishes; polling vs log-tailing; consumers must dedupe), **Circuit Breaker** (closed/open/half-open; tuning is hard), **Event Sourcing** (append-only events, needs CQRS for reads). **Top donor for distributed-backend patterns.**
- **Fowler bliki** — **CQRS** (separate read/write models; Fowler cautionary: "risky complexity," narrow bounded context only), **Strangler Fig** (incremental legacy replacement), **microservices tradeoffs** ("microservices premium" — **start with a modular monolith**, split only when boundaries proven), **DDD Aggregate** (root as sole access point; never span a txn across aggregates → that's what sagas are for), **Bounded Context** (consistency boundary; service-boundary precursor).
- **refactoring.guru** (https://refactoring.guru/design-patterns, © commercial, cite-only) — GoF catalog. Backend eng reaches for **Repository** (PoEAA, not GoF), **Factory**, **Strategy** constantly; rest is OOP hygiene.
- **roadmap.sh/backend** (https://roadmap.sh/backend, MIT) — use as a **completeness checklist** (link, don't quote): internet fundamentals, language, git, SQL+NoSQL, APIs (REST/GraphQL/gRPC), caching, security, testing, CI/CD, Docker, patterns/DDD, message brokers, observability.

**API design authorities**
- **Google API Design Guide / AIPs** (https://google.aip.dev + https://docs.cloud.google.com/apis/design, **CC-BY-4.0**) — resource-oriented design, numbered/addressable AIPs (naming AIP-122, methods, custom methods AIP-136, errors AIP-193, versioning AIP-185). Most systematic; good for pinpoint citation.
- **Microsoft REST guidelines** (https://github.com/microsoft/api-guidelines, **CC-BY-4.0**) — strong on versioning + structured errors.
- **Zalando RESTful guidelines** (https://opensource.zalando.com/restful-api-guidelines/, OSS) — most exhaustive single house-style (API-First, OpenAPI 3.1, kebab URLs, snake_case JSON, eventing section). "If you read one style guide."
- **Stripe** (https://stripe.com/blog/idempotency, https://stripe.com/blog/api-versioning, cite-only) — **type-prefixed IDs** (`cus_`,`ch_`), **`Idempotency-Key` header** (server stores response keyed by account+key, replays on retry — prevents double-charge; v2 = key+endpoint+account within 30-day window), **pinned per-integration versioning** w/ dated upgrade log, **cursor pagination**.
- **GitHub REST best-practices** (https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api, cite-only) — **consumer-side**: authenticate, respect `Retry-After`/`X-RateLimit-*`, webhooks-over-polling, conditional requests (ETag→304), Link-header pagination, don't hammer 4xx/5xx (ban risk).
- **Richardson Maturity Model** (https://martinfowler.com/articles/richardsonMaturityModel.html) — L0 (HTTP tunnel) → L1 (resources) → L2 (verbs+status codes; **where most "REST" actually sits**) → L3 (HATEOAS). **Per Fielding, L3 is the precondition for "true REST"** — most self-described REST APIs are "HTTP APIs." State this precisely.
- **GraphQL** (https://graphql.org/learn/best-practices/, LF/GraphQL Foundation) — fields **nullable by default**; **Relay cursor-connection** pagination; **avoid versioning entirely** (additive evolution, clients pick fields). **gRPC** (https://grpc.io/docs/, Apache-2.0 code) — proto3 IDL + codegen stubs; 4 modes (unary/server-stream/client-stream/bidi). **Decision heuristic:** REST+OpenAPI for public/multi-language/backward-compat; GraphQL for many heterogeneous frontends + over/under-fetch cost (pays: resolver N+1, weak HTTP caching); gRPC for internal perf-sensitive + streaming (poor for browsers w/o grpc-web); tRPC only single-language TS full-stack.

**Reliability / correctness patterns**
- **Idempotency keys** (Stripe, §above; IETF has a draft `Idempotency-Key` header) — client token → server persists (key→result) w/ TTL, replays on retry. For any client-retriable mutation.
- **Retry with backoff + jitter** (https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) — **Full Jitter** best measured; Equal Jitter worst; Decorrelated adapts. Combine w/ cap + circuit breaker.
- **DLQ** (https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) — quarantine poison messages after max-receive-count instead of infinite retry / head-of-line blocking. (Kafka has no native DLQ — consumer-group level.)
- **At-least-once vs exactly-once** (https://docs.confluent.io/kafka/design/delivery-semantics.html) — **industry consensus: prefer at-least-once + idempotent consumers** over paying the exactly-once coordination tax (~10–20% throughput in Kafka). True exactly-once for narrow financial paths only.
- **Saga vs 2PC** — 2PC gives strong atomicity but holds locks + blocks under partition; avoided in modern distributed systems. Saga (local txns + compensations + eventual consistency) is the accepted replacement.
- **Optimistic vs pessimistic locking** — pessimistic under high contention (seat booking); optimistic (version check at commit) for low-contention CRUD (higher concurrency, retry cost on conflict). **Gap:** no single canonical URL — cite first-party DB engine MVCC/locking docs (Postgres/MySQL) rather than blogs.

### 2.4 Auth + security implementation

_This is the backend's controls, not the threat model (→ architecture skill). OWASP = **CC-BY-SA-4.0** (attribute + share-alike; paraphrase). IETF RFCs open._

**OWASP (single densest source for this cluster — nearly every concrete default traces here)**
- **ASVS v5.0.0** (https://asvs.dev/, https://owasp.org/www-project-application-security-verification-standard/) — ~350 requirements, 17 chapters, `chapter.section.requirement` IDs, **L1/L2/L3** verification levels. The audit-checklist "how do I know I'm done" companion. (Full chapter enumeration not retrieved — pull PDF at authoring.)
- **Top 10 (2021)** (https://owasp.org/Top10/2021/) — A01 **Broken Access Control (now #1)**, A02 Crypto Failures, A03 Injection, A04 Insecure Design, A05 Misconfig, A06 Vulnerable Components, A07 Auth Failures, A08 Integrity Failures, A09 Logging/Monitoring Failures, A10 SSRF.
- **Cheat Sheets** (https://cheatsheetseries.owasp.org/) — the implementation-concrete defaults:
  - **Password Storage:** **Argon2id m=19MiB/t=2/p=1 minimum** (prefer m=46MiB/t=1/p=1); scrypt N=2^17/r=8/p=1 fallback; bcrypt cost≥10 legacy-only (**72-byte cap** → pre-hash); PBKDF2 for FIPS (HMAC-SHA256 @600k). Unique salt, ~1s target, **rehash-on-login upgrade**, PHC string format.
  - **Authentication:** min 8 chars w/ MFA else 15, max ≥64, **no composition rules, no forced rotation**, block breached passwords, throttle **by account not IP**, generic error messages, TLS on the login page itself, step-up re-auth for sensitive actions. MFA ≈99.9% compromise prevention.
  - **Session Management:** ≥64-bit random ID, `Set-Cookie: __Host-Session=…; Secure; HttpOnly; SameSite=Strict; Path=/`, **regenerate ID on auth + privilege change**, idle 2–5min (high-value)/15–30min, absolute 4–8h. **Avoid localStorage for tokens** (XSS) — httpOnly cookies or BFF.
  - **Authorization:** **deny-by-default, centralize enforcement** (one chokepoint), least privilege, **IDOR** = re-check permission on *that specific object*, server-side every request. OWASP favors ABAC/ReBAC over RBAC at scale.
  - **SQL Injection:** **parameterized queries/prepared statements** = structural fix; escaping discouraged; dynamic identifiers via allowlist; least-privilege DB accounts.
  - **Input Validation:** allowlist > denylist, syntactic + semantic, server-side mandatory, canonicalize first, file-upload (extension allowlist/size cap/random server filename/re-encode). It's a contributing control, not the primary XSS/SQLi defense.
  - **Secrets Management:** never hardcode; dedicated manager (Vault/AWS SM/Azure KV/GCP SM); automate rotation; **prefer dynamic short-lived secrets**; env vars = pragmatic compromise not best practice; pre-commit/CI scanning (detect-secrets).
  - **Crypto Storage:** AEAD (GCM/CCM) first; never reversibly encrypt passwords; keys separate from ciphertext (KMS/HSM).
  - **CSRF:** synchronizer token (primary); `SameSite` complementary **not a replacement**.
  - **Secure Headers** (https://owasp.org/www-project-secure-headers/): HSTS, CSP, X-Content-Type-Options nosniff, Referrer-Policy, Permissions-Policy, COOP/COEP/CORP.
  - **JWT:** (canonical sheet URL unstable this pass — cross-checked w/ RFC 7519 + REST Security sheet) — **pin the expected `alg` server-side** (reject `none`, block RS256→HS256 confusion), validate `exp`/`nbf`/`iss`/`aud` not just signature, never localStorage, no sensitive data in payload (base64 ≠ encrypted), revocation needs denylist/short-TTL+refresh.

**OAuth / OIDC**
- **RFC 6749** (https://www.rfc-editor.org/rfc/rfc6749) — grants: **Authorization Code** (confidential clients), Implicit (**dead/removed in 2.1**), ROPC (**MUST NOT**), **Client Credentials (correct flow for backend-to-backend)**.
- **RFC 7636 PKCE** (https://www.rfc-editor.org/rfc/rfc7636.html) — `code_verifier`/`code_challenge` (use **S256**), binds redemption to initiator; for public clients (SPA/native).
- **RFC 9700 OAuth Security BCP (Jan 2025)** (https://datatracker.ietf.org/doc/rfc9700/) — **the single most load-bearing OAuth doc for 2026.** PKCE MUST for public clients (recommended for all), **exact-string redirect matching**, no implicit, no ROPC, **sender-constrained tokens** (mTLS/DPoP), **refresh-token rotation-on-use with reuse detection** (replay → revoke whole family).
- **OAuth 2.1** (https://oauth.net/2.1/) — consolidates the above into one spec; teach as the default target. **OIDC Core** (https://openid.net/specs/openid-connect-core-1_0.html) — identity layer; **don't conflate access tokens (authz) with ID tokens (authn)**.

**JWT vs sessions:** stateless JWT scales horizontally (any node verifies locally) but **revocation is the cost** (valid until exp regardless of state); server-side sessions give instant revocation at a per-request lookup cost. Compromise = short access JWT + long opaque revocable refresh token (rotated per RFC 9700). Storage: httpOnly+Secure+SameSite cookies or **BFF** (browser never holds token).

**Authorization models:** RBAC (simple, **role explosion** at scale) → **ABAC** (subject+resource+env attributes vs policy; **Cedar** https://docs.cedarpolicy.com/ = formally verifiable, backs AWS Verified Permissions) → **ReBAC / Google Zanzibar** (relationship tuples `object#relation@user`; encodes RBAC+ABAC as subsets; Zanzibar = trillions of ACLs, <10ms). **OpenFGA** (https://openfga.dev/, CNCF Incubating) + **SpiceDB** = open Zanzibar implementations. **Casbin / Oso** = embeddable in-process libraries (Oso status shifting toward "Authorization Academy" — verify before citing as active lib).

**Auth libs/services:** Auth.js/NextAuth (folding into Better Auth — verify), Passport (Express middleware/strategies), **Keycloak** (self-hosted IdP), **Ory Kratos+Hydra** (headless identity + OAuth server), Clerk (managed, fast DX), Auth0/Okta (enterprise breadth/compliance), Supabase Auth (bundled), WorkOS (enterprise SSO/SCIM). Heuristic: self-host for control/scale, managed for speed, WorkOS/Auth0 for enterprise-buyer SSO/SCIM.

**Controls:** rate limiting (**token bucket** default; Redis-backed w/ Lua for distributed correctness; expose `X-RateLimit-*`), **CORS** (never wildcard `*` with credentials — allowlist origins; cache preflight), **mTLS** (mutual cert identity, service-mesh/zero-trust; gateway terminates public TLS, re-establishes mTLS internally), **HashiCorp Vault dynamic secrets** (per-request TTL'd DB creds, bounds blast radius).

## 3. Hottest-stack signals (2025–2026)

**Headline:** The 2025–26 backend center of gravity is **(a) TypeScript's edge-native wave — Bun/Hono/Elysia + Drizzle overtaking Prisma for greenfield + tRPC-alongside-REST**, **(b) "Postgres for everything" hardening from meme to benchmarked practice with Postgres #1 and fastest-growing across every survey**, and **(c) durable execution (Temporal) becoming standard infra for production + AI-agent workflows** — all against a maturing **modular-monolith / "return to fundamentals"** counter-narrative to reflexive microservices.

**Top named tools + momentum:**
1. **PostgreSQL** — rising + sticky. SO 2025 #1 (58.2%), DB-Engines fastest-growing 6 yrs, on track for #3 by late 2026. The unquestioned default.
2. **Hono** — rising, the edge/serverless default (~38M weekly dl, runs everywhere). Top-3 State-of-JS satisfaction.
3. **Drizzle** — rising, **passed Prisma in weekly downloads late 2025**, ~2x greenfield rate; Prisma 7 (TS/WASM engine) is the counter-punch. Existing apps stay Prisma.
4. **Temporal** — the standout. **OpenAI put it in the Agents SDK (Feb 2026)**, **$300M @ $5B valuation**, corroborated by ThoughtWorks Radar v34 Caution "Ignoring Durability in Agent Workflows."
5. **Axum** — rising, now the default Rust web answer (crates.io/Lichess run on it), reaching 1.0 "boring/durable" status.
6. **Elysia** — rising in the Bun-throughput niche (2.5M req/s), 80:1 download gap behind Hono; satisfaction-high, usage-low.
7. **Go net/http stdlib** — consolidating; Go 1.22 method+wildcard routing displaces third-party routers.
8. **Litestar** — rising Python FastAPI-challenger (msgspec perf); FastAPI still the greenfield default (+5pts SO 2025).

**Survey/radar anchors:** Stack Overflow 2025 (49k+ resp) — Node 49.1%/React 46.9%/Next 21.5%; Postgres #1, Redis +8pts, Docker +17pts (largest single-year jump), Python +7pts. State of JS 2025 — backend section returned after 3yr absence; Express dominant usage but Hono/Nitro/Elysia top-3 satisfaction; teams use only ~1.9 backend frameworks (choice is sticky). JetBrains 2025 — TS/Rust/Go highest growth potential; Go 11% plan-to-adopt; Rust broad interest but only 26% professional use. ThoughtWorks Radar v34 (Apr 2026) — theme is AI "cognitive debt → return to fundamentals"; Adopt: Passkeys, Zero-Trust, DORA metrics, Structured LLM Output; Caution: "Ignoring Durability in Agent Workflows," "MCP by Default."

**Infra facts (hard, not hype):** Kafka 4.0 (Mar 2025) removed ZooKeeper (KRaft-only); Redis licensing → Valkey fork; Vercel killed its Postgres → Neon (Databricks bought Neon ~$1B); serverless-Postgres consolidating to Neon/Supabase/PlanetScale; **connection pooling in serverless is the recurring failure mode** (PgBouncer transaction-mode / Prisma Accelerate / HTTP-native drivers). Message queues: Kafka steady/dominant (event streaming), NATS JetStream rising (lightweight/low-latency), RabbitMQ steady enterprise standard, SQS background AWS default.

**Contested / hype-flagged (don't overstate):** effect-ts (all-or-nothing adoption cost, contested); Encore (comparison content is vendor-self-published); Turso/D1 sub-10ms claims (SEO-blog sourced); the **"42% microservices rollback / CNCF survey"** and **"40% regret microservices / ThoughtWorks"** figures (**unverified — trace to primary or drop**); "monolith renaissance" (weakest-sourced section — only the Amazon Prime Video 90%-cost rollback is genuinely primary). Consensus is **not** a reversal to monoliths but **"modular monolith for core + focused services only where the tradeoff pays."**

## 4. Gaps / uncertainties

- **DB-Engines full top-20 not fetched** — WebFetch TLS/cert error ("unable to get local issuer certificate"); ranks are from Red-Gate press releases + SO survey snippets. Re-fetch db-engines.com when network allows.
- **License re-verification recommended before publishing as hard claims:** InfluxDB 3.0 (mixed/conflicting signals per component), Prisma-client + Drizzle exact license files (Apache-2.0 / MIT per repos, not independently pulled), Milvus/Weaviate license pages (not directly fetched), roadmap.sh MIT (repo license file not re-verified), Nitro MIT (inferred from UnJS convention).
- **OWASP JWT Cheat Sheet URL unstable** — 404'd this pass (renamed/merged in CheatSheetSeries repo); JWT guidance cross-checked via RFC 7519 + REST Security sheet. Re-check the cheatsheetseries index at authoring.
- **ASVS v5.0 full 17-chapter enumeration + exact L1/L2/L3 criteria** not retrieved — pull the PDF/GitHub repo if the skill needs chapter-by-chapter mapping.
- **Spring WebFlux (reactive Spring)** not researched in depth — the blocking-vs-reactive decision deserves its own pass if the skill goes deep on Java. **Quarkus best-practices doc** — no canonical URL distinct from quarkus.io/guides.
- **Unverified adoption/figure claims (flagged inline):** effect-ts named adopters (Vercel/Prisma/fintech — secondary aggregator), Encore vs-X benchmarks (vendor self-published), Turso/D1 latency numbers (SEO blogs), the two microservices-regret percentages (no primary CNCF/ThoughtWorks publication traced).
- **Oso OSS/commercial status** — appears pivoted toward educational "Authorization Academy"; verify before citing as an active embeddable library. **Auth.js→Better Auth** and **NuxtLabs→Vercel** rebrands — verify current branding.
- **Rate-limiting + mTLS** guidance rests on practitioner-blog consensus, not standards-body authority — fine for a "practical guidance" tier, not the same weight as OWASP/IETF.
- **No standalone "State of DB" survey** exists distinct from DB-Engines + the SO survey database section. No dedicated InfoQ *backend* trends report (only the AI-centric Software Architecture trends report).
- **Border reminders for the skill authors:** OLAP/DuckDB, streaming analytics → `data`; monolith-vs-micro *decision* + NFR *budgets* + threat *model* → `architecture`; LLM/RAG *core* → `ai`; deploy/CI/CD/runtime-monitoring → `operate`. We captured the backend *implementation* craft of these seams only.
