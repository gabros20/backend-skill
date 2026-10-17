# Sources & attribution

The `backend` skill wraps a published canon of server-side engineering practice — frameworks, ORMs,
datastores, protocols, and security standards — rather than reinventing it. This file attributes that
canon and records the license posture applied to each.

**License policy (applied throughout the pack).** Default posture: **re-express the facts and cite
the source.** Verbatim text is lifted only from permissively licensed sources (MIT, Apache-2.0, BSD,
MIT-0, CC0, public-domain) with attribution here. CC-BY-4.0 sources (Google AIP, Microsoft REST API
Guidelines) are attributed and cited. CC-BY-SA-4.0 sources (the OWASP Top 10, ASVS, and Cheat Sheet
Series) are paraphrased, attributed, and share-alike-aware — the security refs and their checklist
assets carry an inline attribution where the material is used. Restrictive and no-license sources
(microservices.io, martinfowler.com, refactoring.guru, and the no-LICENSE single-framework skill
repos) are **cited/linked only, never re-expressed into shipped text as prose** — facts are drawn
from the primary spec or concept, not from these renderings. Copyrighted books (Evans, Vernon,
Kleppmann, Nygard, Newman) are used as a concept index and cited, never reproduced.

**Datastore product licenses** (SSPL / BSL / AGPL / source-available and the permissive escapes)
appear in [persistence](skills/backend/references/persistence.md) §9 as *skill content* — a taught,
value-add classification of self-host/resale implications, not our copying of any vendor's text.
Verify current terms before a hard claim; product licenses change.

---

## Permissive — re-expressed with attribution (MIT / Apache-2.0 / BSD / MIT-0 / CC0 / public-domain)

| Source | License | Used in |
|---|---|---|
| [PostgreSQL](https://www.postgresql.org/), pgvector, PgBouncer | PostgreSQL License (permissive) | persistence, caching-and-performance, async-and-messaging, surface-agentic |
| [12factor.net](https://12factor.net/) — Adam Wiggins / Heroku | MIT | surface-rest-crud, backend-service-scaffold asset |
| [honojs/hono](https://github.com/honojs/hono), [elysiajs/elysia](https://github.com/elysiajs/elysia), [fastify/fastify](https://github.com/fastify/fastify) | MIT | api-implementation, surface-rest-crud, backend-testing |
| [nestjs/nest](https://github.com/nestjs/nest) | MIT | api-implementation, surface-rest-crud, backend-testing |
| [fastapi/fastapi](https://github.com/fastapi/fastapi), [litestar-org/litestar](https://github.com/litestar-org/litestar) | MIT | api-implementation, surface-rest-crud |
| [tokio-rs/axum](https://github.com/tokio-rs/axum) (Tower/Hyper) | MIT | api-implementation |
| [encode/django-rest-framework](https://github.com/encode/django-rest-framework) | BSD-3-Clause | api-implementation (pagination tiers) |
| [gofr-dev/gofr](https://github.com/gofr-dev/gofr) | Apache-2.0 | surface-rest-crud |
| [drizzle-team/drizzle-orm](https://github.com/drizzle-team/drizzle-orm) | Apache-2.0 | persistence, migration-template asset, surface-agentic |
| [prisma/prisma](https://github.com/prisma/prisma) — migration-verb taxonomy, outbox notes | Apache-2.0 | persistence, migration-template asset |
| [coleifer/peewee](https://github.com/coleifer/peewee) — imperative-migration fallback | MIT | persistence |
| [sqlalchemy/sqlalchemy](https://github.com/sqlalchemy/sqlalchemy), [sqlc-dev/sqlc](https://github.com/sqlc-dev/sqlc) | MIT | persistence, surface-rest-crud |
| [temporalio/temporal](https://github.com/temporalio/temporal) | MIT | async-and-messaging, surface-agentic |
| [hatchet-dev/hatchet](https://github.com/hatchet-dev/hatchet) — Postgres jobs-table + concurrency keys | MIT | async-and-messaging |
| [debezium/debezium](https://github.com/debezium/debezium) — outbox extension DDL, CDC | Apache-2.0 | async-and-messaging, idempotency-and-outbox-recipes asset |
| [apache/kafka](https://github.com/apache/kafka), [nats-io/nats-server](https://github.com/nats-io/nats-server), RabbitMQ | Apache-2.0 / MPL-2.0 | async-and-messaging, surface-event-driven-microservices |
| [heroiclabs/nakama](https://github.com/heroiclabs/nakama) — MatchHandler/Tracker/reliable-flag shapes | Apache-2.0 | surface-realtime |
| [centrifugal/centrifugo](https://github.com/centrifugal/centrifugo) | MIT/Apache-2.0 | surface-realtime |
| Phoenix Channels ([phoenixframework/phoenix](https://github.com/phoenixframework/phoenix)) | MIT | surface-realtime, surface-event-driven-microservices |
| [openfga/openfga](https://github.com/openfga/openfga) (CNCF), [authzed/spicedb](https://github.com/authzed/spicedb) | Apache-2.0 | auth |
| [ory/kratos](https://github.com/ory/kratos) + [ory/hydra](https://github.com/ory/hydra), [keycloak/keycloak](https://github.com/keycloak/keycloak) | Apache-2.0 | auth |
| [casbin/casbin](https://github.com/casbin/casbin) — PERM metamodel | Apache-2.0 | auth |
| [cedar-policy/cedar](https://github.com/cedar-policy/cedar) (AWS) | Apache-2.0 | auth |
| [pocketbase/pocketbase](https://github.com/pocketbase/pocketbase) — rule-strings authz | MIT | auth |
| [wshobson/agents](https://github.com/wshobson/agents) — backend-development plugin (structural precedent) | MIT | router shape, api-implementation, persistence |
| [samber/cc-skills-golang](https://github.com/samber) — Go one-concern skills, evals methodology | MIT | surface-rest-crud, evals |
| [mehdihadeli/awesome-software-architecture](https://github.com/mehdihadeli/awesome-software-architecture) | CC0-1.0 | domain-logic, async-and-messaging |
| [wiremock](https://github.com/wiremock/wiremock), testcontainers | Apache-2.0 / MIT | backend-testing |

## CC-BY-4.0 — attributed, cited

| Source | Used in |
|---|---|
| [Google AIP — API Improvement Proposals](https://google.aip.dev/) | api-implementation, api-contract-review-checklist |
| [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines) | api-implementation |

## CC-BY-SA-4.0 — paraphrased, attributed, share-alike-aware

The security references and their checklist assets that paraphrase these carry an inline attribution.

| Source | Used in |
|---|---|
| [OWASP Top 10:2021](https://owasp.org/Top10/) | server-security, owasp-backend-controls-checklist |
| [OWASP ASVS v5.0](https://asvs.dev/) | server-security |
| [OWASP AI Agent Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/AI_Agent_Security_Cheat_Sheet.html) — atomic check-and-consume of an approval before execution | surface-agentic |
| OWASP Cheat Sheet Series — [Password Storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html), SQL Injection Prevention, Input Validation, Secrets Management, [HTTP Security Response Headers](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html), CSRF Prevention | auth, server-security, auth-implementation-checklist, owasp-backend-controls-checklist |

## Cited, not copied — restrictive / no-license, standards, and concepts

Named and linked as canon; **not** re-expressed from a restrictively licensed rendering. Facts are
drawn from the open standard or the primary concept, not from these renderings' prose.

- **Restrictive / all-rights-reserved (cite/link only):** [microservices.io](https://microservices.io)
  (Chris Richardson — API Gateway/BFF, CQRS, Transactional Outbox, Saga); [martinfowler.com](https://martinfowler.com)
  bliki (DDD Aggregate, MonolithFirst, CQRS "risky complexity" caution); [refactoring.guru](https://refactoring.guru)
  (commercial — patterns index).
- **No LICENSE on file (re-expressed regardless, never lifted):** single-framework community skill
  repos — `yusukebe/hono-skill`, `elysiajs/skills`, a NestJS community skill, `better-auth/skills`.
  Facts re-derived from each framework's own permissively licensed docs, not these skill repos' text.
- **Open standards & specs:** OAuth 2.0 Security BCP [RFC 9700](https://www.rfc-editor.org/rfc/rfc9700),
  [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749), JWT [RFC 7519](https://www.rfc-editor.org/rfc/rfc7519),
  problem+json [RFC 7807](https://www.rfc-editor.org/rfc/rfc7807), OAuth 2.1 (consolidation draft),
  [OpenAPI](https://spec.openapis.org/), [AsyncAPI](https://www.asyncapi.com), [gRPC/proto3](https://grpc.io),
  [GraphQL](https://graphql.org) + the Relay cursor-connection spec, SQL-92 isolation levels,
  [PostgreSQL transaction isolation docs](https://www.postgresql.org/docs/current/transaction-iso.html),
  [PgBouncer features](https://www.pgbouncer.org/features.html),
  [PostgreSQL row security policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
  (persistence — who bypasses RLS), the WHATWG HTML
  [Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html) section
  (`Last-Event-ID` resume — surface-realtime), [RFC 8707](https://www.rfc-editor.org/rfc/rfc8707)
  resource indicators, OAuth 2.0 Token Exchange [RFC 8693](https://www.rfc-editor.org/rfc/rfc8693), and the [Model Context Protocol](https://modelcontextprotocol.io) — its
  [Authorization spec](https://modelcontextprotocol.io/specification/latest/basic/authorization)
  (token audience validation, no token transit) and
  [Security Best Practices](https://modelcontextprotocol.io/docs/draft/tutorials/security/security_best_practices)
  (token passthrough, confused deputy, state-handle hijacking) — surface-agentic, auth.
- **Concepts / books (concept index only, never reproduced):** Eric Evans, *Domain-Driven Design*;
  Vaughn Vernon, *Implementing Domain-Driven Design*; Alistair Cockburn, *Hexagonal Architecture*;
  Robert C. Martin, *Clean Architecture* (the Dependency Rule); Martin Kleppmann, *Designing
  Data-Intensive Applications*; Michael Nygard, *Release It!* (circuit breaker, stability patterns);
  Sam Newman, *Building Microservices*.
- **Cited essays / articles / practitioner discourse (not copied):** Stripe API practices
  (idempotency-key, cursor pagination, dated versioning); AWS Builders' Library — "Timeouts, retries,
  and backoff with jitter" (Full/Equal/Decorrelated Jitter); [postgresforeverything.com](https://postgresforeverything.com/);
  [encore.dev — Drizzle vs Prisma](https://encore.dev/articles/drizzle-vs-prisma); ThoughtWorks
  Technology Radar ("Ignoring Durability in Agent Workflows"); OpenAI Agents SDK (Temporal durable
  execution); Guillermo Rauch (agent-legible backends), Rhys Sullivan (Effect token-efficiency),
  `arpit_bhayani` (OpenAI's single-primary Postgres), DrizzleORM and CodeHagen (Drizzle/Prisma
  discourse) on X.
- **Cited documentation and engineering accounts — read 2026-10-17, re-verify before a hard claim:**
  Temporal docs — [Activity definition](https://docs.temporal.io/activity-definition) (idempotency;
  run ID + activity ID as key; worker crash before reporting), [Retry policies](https://docs.temporal.io/encyclopedia/retry-policies)
  (activities retry by default, unlimited attempts), [Worker Versioning](https://docs.temporal.io/worker-versioning)
  (pinned and draining versions); LangGraph docs — [Fault tolerance](https://docs.langchain.com/oss/javascript/langgraph/fault-tolerance)
  (retries opt-in) and [Checkpointers](https://docs.langchain.com/oss/javascript/langgraph/checkpointers)
  (replay re-executes later nodes' API calls); AWS Builders' Library — "Making retries safe with
  idempotent APIs" (parameter mismatch on a reused token); Martin Kleppmann, "How to do distributed
  locking" (2016, fencing tokens); Cursor, ["What we've learned building cloud agents"](https://cursor.com/blog/cloud-agent-lessons)
  (Jun 2026 — decoupled agent loop / machine / conversation state, client stream rewind);
  [pgvector README](https://github.com/pgvector/pgvector) *Filtering* and *Iterative Index Scans*;
  Supabase [Row Level Security guide](https://supabase.com/docs/guides/database/postgres/row-level-security)
  (`service_role` bypasses RLS); LiteLLM docs — [Drop unsupported params](https://docs.litellm.ai/docs/completion/drop_params)
  and [Router](https://docs.litellm.ai/docs/routing) (gateway-level retries, cooldowns, fallbacks);
  Vercel AI Gateway [BYOK](https://vercel.com/docs/ai-gateway/authentication-and-byok/byok)
  (failed BYOK request falls back to system credentials, billed to credits — a dated example);
  Cloudflare Hyperdrive [Query caching](https://developers.cloudflare.com/hyperdrive/concepts/query-caching/)
  (on by default, 60 s `max_age` + 15 s `stale_while_revalidate`, disabled per configuration —
  surface-serverless-edge); Cloudflare [Rules of Durable Objects](https://developers.cloudflare.com/durable-objects/best-practices/rules-of-durable-objects/)
  (one object per coordination unit, no single global object — surface-serverless-edge); Replicate
  [Data retention](https://replicate.com/docs/topics/predictions/data-retention) (API prediction data
  removed after an hour by default — service-integration); Twilio [Media Streams WebSocket messages](https://www.twilio.com/docs/voice/media-streams/websocket-messages)
  (μ-law 8 kHz mono, bidirectional only under `<Connect><Stream>`, `clear`/`mark`/`dtmf`), OpenAI
  [Realtime conversations](https://developers.openai.com/api/docs/guides/realtime-conversations)
  (voice fixed per session, server vs client truncation by transport) and
  Google [Live API session management](https://ai.google.dev/gemini-api/docs/live-session)
  (`GoAway`, resumption tokens) — surface-realtime voice transport,
  dated examples only. The
  three-way gateway taxonomy (marketplace / proxy-policy / hosted-inference catalog) in
  service-integration is this pack's classification of those products' own docs, not a quoted source.
- **Verify-before-hard-claim (stated as uncertain or omitted in the pack):** the "42% microservices
  rollback" / "40% regret" statistics (omitted — no primary CNCF/ThoughtWorks source confirmed);
  exact current datastore product-license terms (taught as *shape*, verify before asserting);
  vendor-self-published performance benchmarks (Turso/Encore) — cited as vendor claims, not as
  independent fact.

---

*Every framework and standard above is wrapped, not reproduced. If an attribution is missing or a
license posture is wrong, open an issue — see [CONTRIBUTING.md](CONTRIBUTING.md).*
