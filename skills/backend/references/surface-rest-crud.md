# Surface: REST/CRUD service (default)

Purpose: Reshape the 9 job references for the modal backend shape — a standard long-running
request/response service (modular monolith or a small number of services), reached by HTTP, backed
by a relational store. This is the surface to apply when nothing more specific fits.

Read when:
- The request describes a "normal" API/backend service with no edge/serverless, realtime, or
  distributed-microservices signal (see the other surface refs' own "Read when" for those signals).
- No surface is named explicitly — this is the fallback; apply it by default.
- The system is greenfield and no scale/latency/deployment constraint has pushed it elsewhere yet.

Skip when:
- Deploying to Workers/Lambda/edge with per-request cold starts → [surface-serverless-edge](surface-serverless-edge.md).
- WebSocket/SSE/stateful-connection/presence requirements → `surface-realtime`.
- Already polyglot/multi-service with async event transport between services → `surface-event-driven-microservices`.
- Building tool/function endpoints or an MCP server for an LLM/agent caller → `surface-agentic`.
- The actual job craft (routing/validation, persistence, auth, etc.) — this ref only reshapes; the
  9 job references ([api-implementation](api-implementation.md), [domain-logic](domain-logic.md),
  [persistence](persistence.md), [auth](auth.md), [async-and-messaging](async-and-messaging.md),
  [caching-and-performance](caching-and-performance.md), [server-security](server-security.md),
  [service-integration](service-integration.md), [backend-testing](backend-testing.md)) own it.

Inputs: target language/runtime (state explicitly when absent — do not assume Node), whether this
is greenfield or an existing codebase, any architecture contracts/NFR budgets already handed off.

Produces: a concern-scoping decision (which of the three models below, or a per-language
equivalent) and a 12-factor-compliant service skeleton, before job references are invoked for the
actual routes/persistence/auth content.

## Contents
- Three concern-scoping models (TS/JS ecosystem)
- Per-language idioms
- 12-factor baseline
- Validation
- Failure modes and handoff

## Three concern-scoping models

Every mature backend framework answers the same question — "how do I scope a cross-cutting concern
(auth, validation, DI, a sub-tree of routes) to part of the app instead of the whole thing" — with
one of three structurally distinct mechanisms. This surface's job is **picking** the model that
matches team size and how much structure the framework should enforce; the per-framework
request-lifecycle *mechanics* (Nest's Guards→Interceptors→Pipes→Exception-Filters pipeline,
Fastify's `register()` encapsulation-context hook chain, Hono/Elysia's `compose()` onion + typed
context chaining) are single-homed in [api-implementation](api-implementation.md) §2 — read there
for how each works, decide the pick here. **Do not port one framework's idiom into another's**
(e.g. don't hand-roll a Nest-style DI container inside a Hono app — use Hono's own composition
primitives).

| Model (framework) | Ceremony | Pick when |
|---|---|---|
| **DI/module graph** (NestJS; JVM Spring Boot is the same family) | Heaviest, most enforced layering | Large team, complex domain, want the framework to enforce boundaries over convention; most testable at scale (`Test.createTestingModule()` against real wiring) |
| **Encapsulation-context plugin tree** (Fastify) | Middle | Node-only shop wanting Express-like plugin ergonomics with schema-validated I/O and materially better throughput, without DI ceremony |
| **Flat onion + typed context chaining** (Hono/Elysia) | Lightest | Edge/serverless target where every abstraction has a cold-start cost; don't need full DI |

Cutting across all three: **"adapter isolates the runtime, core stays portable."** Hono's deploy
adapters (`src/adapter/aws-lambda`, `/bun`, `/cloudflare-workers`, …) each translate only the
platform's native event shape to/from a standard `Request`/`Response`; Elysia does the same for
Bun/Cloudflare-Worker targets; Nest applies the identical idea one layer down, swapping the
underlying HTTP engine (Express vs Fastify) behind one `AbstractHttpAdapter` interface. Treat this
as a first-class principle, not per-framework trivia: **write the app against the framework's
web-standard core, push every runtime-specific translation into a thin, swappable adapter.** This is
exactly the seam `surface-serverless-edge` depends on.

## Per-language idioms

State the target language/runtime explicitly before generating; **fetch current official
docs/types for the chosen framework/ORM version before writing code** — backend frameworks ship
breaking changes fast (this surface is idiom-neutral; version-gating is the job references' and the
implementer's responsibility, not a fact to assume stale from training data).

- **Node/Bun (TypeScript)**: pick one of the three models above by team/perf profile. Modal 2026
  default for new greenfield TS services trends toward Hono (flat onion + typed chaining, portable
  across Node/Bun/edge) or Fastify (encapsulation, Node-only throughput); Nest remains the default
  when the org already standardizes on Angular-style DI and multi-module scale.
- **Python**: FastAPI (async, Pydantic-schema-as-contract, dependency-injection via `Depends` — a
  lighter cousin of the DI-module model) is the modal default; layered `api/services/repositories`
  folder convention with a generic typed `BaseRepository[ModelType, CreateSchema, UpdateSchema]`
  is a clean, idiomatic CRUD skeleton to start from.
- **Go**: `net/http` (stdlib router, Go 1.22+ method+wildcard patterns) + `sqlc`/`pgx` for
  compile-time-checked SQL is the lean default; frameworks like GoFr trade that minimalism for a
  batteries-included `Container` (logger, pubsub, multi-datasource clients) injected into every
  handler — pick GoFr-style only when the team explicitly wants Spring-Boot-for-Go conventions over
  net/http's manual wiring.
- **JVM (Java/Kotlin)**: Spring Boot's constructor-injection DI is the same structural family as
  Nest's module graph (heaviest ceremony, most enforced layering) — the default when the org is
  already JVM-native; don't introduce a second DI philosophy into an existing Spring codebase.

## 12-factor baseline

Every REST/CRUD service in this surface should satisfy the 12-factor baseline regardless of
language — config in env, backing services as attachable resources, stateless/share-nothing
processes, build/release/run separation, graceful disposability, logs to stdout. Don't re-derive
this list here: fill out
[assets/backend-service-scaffold.md](../assets/backend-service-scaffold.md) as the starting
checklist for any new service under this surface, and treat deviations (e.g. a typed config/secrets
manager replacing raw env vars) as a documented, deliberate upgrade, not silent drift.

## Validation

- [ ] Concern-scoping model is named explicitly (one of the three, or the language's structural
  equivalent) and matches team size/complexity — not just copied from the last project.
- [ ] The runtime-portability seam (adapter vs core) is respected if there's any chance this service
  moves to serverless/edge later.
- [ ] `assets/backend-service-scaffold.md` is filled, not skipped, for a new service.
- [ ] The actual route/persistence/auth/etc. content was produced by the relevant job reference, not
  invented ad hoc inside this surface pass.

## Failure modes and handoff

- **Porting a framework's idiom across frameworks** (e.g. hand-rolled DI container bolted onto
  Hono) — pick the target framework's own scoping model instead.
- **Treating this surface as the place to write route/persistence/auth logic** — it only reshapes;
  route to the 9 job references for the actual craft.
- **Skipping the 12-factor scaffold "because it's just a small service"** — small services are
  exactly where config-in-code and stateful-process drift creep in unnoticed.
- If the request later reveals cold-start/edge/serverless constraints, hand off to
  [surface-serverless-edge](surface-serverless-edge.md) rather than retrofitting this surface's
  assumptions.
