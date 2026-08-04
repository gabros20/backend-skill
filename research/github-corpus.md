# Backend Research — GitHub Channel Corpus

**Channel:** GitHub (Lead B) · **Method:** `github-relay` funnel (serial discovery → hydrate → enrich → offline rank) + WebSearch curation lane + 4 parallel sonnet clone-and-read deep-read workers.
**Corpus file:** `scratchpad/backend-research/corpus.json` (714 repos discovered, top 60 enriched + ranked).
**Relay discipline:** all relay calls serialized on the shared account; 0 rate-limit hits; ~7 GraphQL points spent total (36-query batch + 22-id hydrate + enrich of 60).

## 1. Method & keyword sweeps

One serialized `batch` sweep of 36 shards (delay 1.2s), then `hydrate` of 22 human-curated IDs from WebSearch, then `enrich --top 60` (15-signal + real-usage B-chain), then offline `rank` (build-on + dissect profiles). WebSearch was used as a first-class discovery lane (golden rule 3) and does NOT touch the relay account.

**Keyword shards run:** awesome backend · topic:backend · awesome nodejs/go/python · fastapi · django rest framework · nestjs · fastify · hono · elysia bun · axum rust · spring boot · rails api · phoenix elixir · prisma · drizzle orm · sqlalchemy · database migration tool · clean architecture backend · domain driven design · microservices patterns · event sourcing · cqrs · kafka · background job queue · jwt authentication · oauth2 server · rbac authorization · websocket server · pgvector · graphql server · grpc framework · backend roadmap · encore · litestar · trpc.

**WebSearch curation queries:** best backend framework 2025/2026 surveys · awesome backend architecture lists · Drizzle vs Prisma / BullMQ vs pg-boss 2026 · wshobson/agents backend skills + backend roadmap.

License note: `rank` reports the license CLASS only (`permissive` for all top rows below); exact SPDX per repo is captured by the deep-read workers reading each LICENSE file (§4).

## 2. Ranked findings (build-on profile — maintenance × real-usage × star-skeptical)

Filtered to backend-relevant donors (AI/ML-app, transcription, and dashboard noise from the shared corpus dropped). Score `s` is the offline build-on composite; `st` = stars (gate/signal only, never the rank).

### API frameworks & runtimes
- **[tokio-rs/axum](https://github.com/tokio-rs/axum)** — s92.2, 26.6k — Rust HTTP routing/handler lib on tower/hyper; ergonomic extractors, tower middleware ecosystem. Hottest Rust backend pick.
- **[honojs/hono](https://github.com/honojs/hono)** — s90.3, 31.5k — Web-Standards framework, multi-runtime (Workers/Deno/Bun/Node/Lambda). **The edge-first default of 2026.**
- **[elysiajs/elysia](https://github.com/elysiajs/elysia)** — s92.5, 18.8k — Bun-native, end-to-end type safety; 2-3x Express throughput on simple JSON.
- **[gofr-dev/gofr](https://github.com/gofr-dev/gofr)** — s92.2, 21.1k — opinionated Go microservice framework (observability/DB/pubsub batteries-included).
- **[fastapi/fastapi](https://github.com/fastapi/fastapi)** — s91.4, 100.8k — Python ASGI, Pydantic validation + DI; the pure-API Python default.
- **[litestar-org/litestar](https://github.com/litestar-org/litestar)** — s91.7, 8.4k — Python ASGI, DI-first, plugin architecture; FastAPI challenger.
- **[wemake-services/django-modern-rest](https://github.com/wemake-services/django-modern-rest)** — s91.5, 1.3k — typed async modern REST for Django.
- **[platformatic/platformatic](https://github.com/platformatic/platformatic)** — s90.2, 2.0k — Node app platform (auto REST/GraphQL over DB, on Fastify).
- **[cloudwego/kitex](https://github.com/cloudwego/kitex)** — s90.2, 8.0k — high-perf Go RPC/microservice framework (ByteDance).

### ORM / DB tooling / migrations
- **[drizzle-team/drizzle-orm](https://github.com/drizzle-team/drizzle-orm)** — TS SQL-like ORM, edge/serverless, smallest bundle. **Challenging Prisma's default status (~25k stars).**
- **[prisma/prisma](https://github.com/prisma/prisma)** — TS ORM, schema-first + migration engine + Accelerate pooling.
- **[coleifer/peewee](https://github.com/coleifer/peewee)** — s91.3, 12.0k — small expressive Python ORM w/ migrations.
- **[koxudaxi/datamodel-code-generator](https://github.com/koxudaxi/datamodel-code-generator)** — s90.6, 4.0k — generate Pydantic v2 / dataclasses / TypedDict from schemas (contract→code).
- **[liam-hq/liam](https://github.com/liam-hq/liam)** — s92.9, 4.9k — auto ER diagrams from schema (schema comprehension/docs).

### AuthN / AuthZ
- **[ory/hydra](https://github.com/ory/hydra)** — s93.1, 17.4k — certified OpenID Connect + OAuth2.1 provider.
- **[apache/casbin](https://github.com/apache/casbin)** — s93.5, 20.3k — access-control library, pluggable RBAC/ABAC/ReBAC models (multi-language).
- **[openfga/openfga](https://github.com/openfga/openfga)** — s93.3, 5.5k — Google-Zanzibar fine-grained authz engine (CNCF).
- **[authzed/spicedb](https://github.com/authzed/spicedb)** — s92.4, 6.9k — Zanzibar-inspired authz database (relationship tuples).
- **[ory/keto](https://github.com/ory/keto)** — s89.4, 5.4k — Zanzibar permission server.
- **[clerk/javascript](https://github.com/clerk/javascript)** — s88.9 — managed auth SDK (borders into frontend, but backend session/JWT patterns).

### Async / messaging / jobs / CDC / streaming
- **[temporalio/temporal](https://github.com/temporalio/temporal)** — s93.6, 21.8k — durable execution / workflow engine. **The durable-workflow pattern donor.**
- **[hatchet-dev/hatchet](https://github.com/hatchet-dev/hatchet)** — s93.3, 7.6k — Postgres-backed background tasks + durable orchestration (Postgres-as-queue signal).
- **[debezium/debezium](https://github.com/debezium/debezium)** — s93.1, 13.0k — change data capture / outbox pattern donor.
- **[AutoMQ/automq](https://github.com/AutoMQ/automq)** — s92.0, 10.3k — diskless Kafka on S3 (modern streaming infra).
- **[ag2ai/faststream](https://github.com/ag2ai/faststream)** — s89.4, 5.3k — async Python framework for event-driven services (Kafka/RabbitMQ/NATS/Redis).
- **[risingwavelabs/risingwave](https://github.com/risingwavelabs/risingwave)** — s92.5, 9.2k — streaming DB / event processing (borders data, but stream-consumer patterns).
- **[hookdeck/outpost](https://github.com/hookdeck/outpost)** — s90.4, 1.0k — outbound webhooks + event-destination infra (webhook delivery/retries).
- **[IBM/sarama](https://github.com/IBM/sarama)** / **[dpkp/kafka-python](https://github.com/dpkp/kafka-python)** — Go/Python Kafka clients.
- **[strimzi/strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator)** — Kafka on K8s (borders operate).

### Realtime (WS/SSE/pub-sub)
- **[centrifugal/centrifugo](https://github.com/centrifugal/centrifugo)** — s90.0, 10.5k — language-agnostic scalable realtime messaging server.
- **[heroiclabs/nakama](https://github.com/heroiclabs/nakama)** — s91.8, 12.9k — realtime game backend (WS, presence, matchmaking, leaderboards).
- **[juggleim/im-server](https://github.com/juggleim/im-server)** — s91.1, 3.6k — high-perf self-hosted IM server in Go.

### GraphQL servers
- **[apollographql/apollo-server](https://github.com/apollographql/apollo-server)** — s89.3, 13.9k — spec-compliant JS GraphQL server.
- **[ChilliCream/graphql-platform](https://github.com/ChilliCream/graphql-platform)** — s89.9, 5.7k — Hot Chocolate GraphQL for .NET.

### Backend-as-a-Service / batteries-included backends (pattern donors)
- **[parse-community/parse-server](https://github.com/parse-community/parse-server)** — s93.0, 21.4k — Node/Express BaaS.
- **[nhost/nhost](https://github.com/nhost/nhost)** — s93.1, 9.2k — OSS Firebase alt w/ GraphQL.
- **[supabase/supabase](https://github.com/supabase/supabase)** (+ supabase-js) — Postgres-centric OSS Firebase alt (**Postgres-as-everything** signal).
- **[pocketbase/pocketbase](https://github.com/pocketbase/pocketbase)** — Go single-file backend (auth + realtime + DB).
- **[InsForge/InsForge](https://github.com/InsForge/InsForge)** — s89.5, 12.4k — OSS backend platform for agentic coding.

### Backend agent-skills (closest structural precedent)
- **[wshobson/agents](https://github.com/wshobson/agents)** — multi-harness agent-skill marketplace; `plugins/backend-development/` holds skills like nodejs-backend-patterns, fastapi-templates, postgresql-table-design, api-design-principles, plus a backend-architect agent. **Direct donor for how to structure a backend skill.**
- **[VoltAgent/awesome-agent-skills](https://github.com/VoltAgent/awesome-agent-skills)** · **[rohitg00/awesome-claude-code-toolkit](https://github.com/rohitg00/awesome-claude-code-toolkit)** — skill/agent catalogs (backend-developer agents).

### Architecture / pattern indexes (taxonomy donors — cite, not framework code)
- **[mehdihadeli/awesome-software-architecture](https://github.com/mehdihadeli/awesome-software-architecture)** — curated pattern taxonomy (DDD, CQRS, event-sourcing, clean architecture, microservices).
- **[mfornos/awesome-microservices](https://github.com/mfornos/awesome-microservices)** · **[binhnguyennus/awesome-scalability](https://github.com/binhnguyennus/awesome-scalability)** · **[karanpratapsingh/system-design](https://github.com/karanpratapsingh/system-design)** · **[ashishps1/awesome-system-design-resources](https://github.com/ashishps1/awesome-system-design-resources)** — system-design/scalability references (border with `architecture`).

## 3. Hottest-stack signals (2025–2026)

- **Edge-first TS APIs are the new greenfield default.** State-of-JS 2025: Express still most-used, but the top-3 *satisfaction* are all new — **Hono, Nitro, ElysiaJS**. New APIs increasingly built on **Hono running on Bun or Cloudflare Workers**. (Source: 2025.stateofjs.com back-end-frameworks; encore.dev/articles/elysia-vs-hono.)
- **Bun as a runtime matured** — Elysia (Bun-only) handles 2-3x Express req/s on simple JSON; Bun adoption is the tailwind.
- **Drizzle is eating Prisma's default slot** — ~25k stars, production-at-scale; picked for edge/serverless, smallest bundle, SQL-like TS. Prisma still wins on auto-migrations + gentler DX + Accelerate pooling + Mongo/CockroachDB/SQLServer. (makerkit.dev, encore.dev, designrevision.com.)
- **Postgres-as-everything** — pg-boss (Postgres job queue, SKIP LOCKED, ACID, no Redis) vs BullMQ (Redis, full-featured: priorities/delays/rate-limit/flows). Hatchet = Postgres-backed durable orchestration. Supabase = Postgres-centric BaaS. (pkgpulse.com BullMQ-vs-pg-boss.)
- **Durable execution rising** — Temporal + Hatchet normalizing workflow/saga/retry/idempotency as first-class backend primitives (esp. for AI-agent backends).
- **Zanzibar authz mainstreaming** — OpenFGA + SpiceDB + ory/keto all implement Google-Zanzibar relationship-tuple fine-grained authz; a real backend pattern to teach beyond RBAC.
- **FastAPI remains the Python API default**, with Litestar the credible DI-first challenger; **Pydantic** validation is the shared Python idiom.
- **tRPC** for end-to-end typed RPC in the TS monorepo world; **gRPC/Kitex/Connect** for polyglot service-to-service.

## 4. Deep-read donor profiles

*(Folded in from the 4 parallel clone-and-read workers as they land — each with exact license + liftable craft. Pending at time of skeleton write.)*

- **github-1 — API frameworks** (hono, elysia, nest, fastify, axum, gofr): _pending_
- **github-2 — Python + ORM/migrations** (fastapi, litestar, DRF, drizzle, prisma, peewee): _pending_
- **github-3 — async/messaging/jobs/CDC/realtime** (temporal, hatchet, debezium, automq, nakama, casbin): _pending_
- **github-4 — auth + patterns + backend agent-skills** (ory/hydra, openfga, spicedb, wshobson/agents, mehdihadeli/awesome-software-architecture, pocketbase): _pending_

## 5. Gaps & uncertainties

- **ossinsight trending lane was down** (timeout at discovery) — trending-repo signal not captured; ranked signal came from keyword + curation lanes only.
- **starredAt restricted** on the shared token — fake-star forensics (F-group burstiness) degraded to ClickHouse-only; star counts above are directional, ranking used maintenance/usage not raw stars.
- **Language coverage skew:** discovery surfaced strong Go/TS/Python/Rust/.NET donors; **Elixir/Phoenix, Ruby/Rails, Java/Spring, Kotlin** under-represented in the ranked top (Spring appears via spring-boot-admin/demos not core). A targeted follow-up sweep on rails/phoenix/spring core + Ecto/ActiveRecord/Hibernate migration craft would balance it.
- **Caching (Redis/Valkey) and rate-limiting** as first-class donors thin in the ranked set — present as infra, light on "how-to" pattern repos; lean on framework middleware (§4) + hottest-stack notes.
- **Serverless/edge** captured via Hono; dedicated serverless-framework donors (SST, Nitro, Encore) noted but not deep-read — candidate follow-up.
- License SPDX per repo pending worker LICENSE reads; all top-ranked rows are license-CLASS `permissive` (no NC/ND/no-LICENSE flags seen in the ranked set).
