# API Implementation

Purpose: Implement the server behind an API contract — pick the transport (REST/GraphQL/gRPC/tRPC),
wire routing/middleware/validation, and produce a consistent error shape, pagination, and versioning
scheme in code.

Read when:
- Request asks to build, scaffold, or extend server endpoints/handlers/routers/resolvers/RPC methods.
- A contract (OpenAPI/proto/GraphQL SDL) already exists and needs a server implementation behind it.
- Request mentions pagination, API versioning, or a consistent error response shape.

Skip when:
- The transport/boundary *decision* itself (service boundaries, NFR budget, which services talk to
  which) is undecided — that's `architecture`'s contract-design job; consume its output here, don't
  redesign it.
- The work is business-rule/use-case logic behind the handler — see `domain-logic`.
- The work is auth flows/token validation — see `auth`.
- Independent load-testing or an API contract release-gate — see `quality`.

Inputs: target language/framework (state explicitly when absent — do not assume), an existing
contract artifact if one exists (OpenAPI/proto/SDL), expected client(s) (browser/mobile/service-to-
service/public third-party).

Produces: a running server implementation — routes/resolvers/RPC methods, request lifecycle wiring
(validation → handler → serialization → error mapping), a stated pagination and versioning scheme,
and a uniform error-response shape.

## Contents
- Transport choice: REST vs GraphQL vs gRPC vs tRPC
- Server implementation by concern-scoping model
- Request lifecycle and middleware
- Validation and serialization (schema as single source of truth)
- Error shape
- Pagination
- Versioning in code

## Procedure

### 1. Transport choice (decision heuristic)
Pick the transport by *client shape*, not novelty. Concrete pick + what NOT to build:
- **REST + OpenAPI** — default for public/external APIs, many-language clients, long-lived documented
  backward compatibility. Anchor to Richardson Maturity Level 2 (HTTP verbs used semantically, correct
  status codes) as the realistic bar; Level 3 (HATEOAS) is rare in practice — don't build it unless a
  client actually needs discoverable hypermedia.
- **GraphQL** — best when multiple heterogeneous frontends need flexible, aggregated queries and
  over/under-fetching is a real cost (mobile especially). Costs: resolver N+1 (pair with dataloader-
  style batching), weaker HTTP-level caching than REST GETs, query-cost analysis needed for public
  graphs.
- **gRPC** — internal, performance-sensitive service-to-service calls, especially when streaming
  (unary, server-streaming, client-streaming, bidirectional) is needed. Poor fit for public/browser
  clients without a grpc-web proxy.
- **tRPC** — only inside a single-language (TypeScript) full-stack app sharing types directly, zero
  codegen. Collapses the moment a non-TS client (mobile native, third party) needs to call in — switch
  to REST/GraphQL at that point, don't force tRPC across a language boundary.
(Source: Google AIP, Microsoft REST Guidelines, Stripe API practices, graphql.org, grpc.io.)

If architecture already fixed the contract shape (OpenAPI/proto/SDL), implement to it — do not
re-litigate the transport choice.

### 2. Server implementation by concern-scoping model
Three structurally distinct ways frameworks scope cross-cutting concerns (auth, validation, logging)
— pick based on team size/testability need, not familiarity alone:
1. **DI/module graph** (NestJS) — heaviest, most testable at scale; a real dependency-injection
   container resolves providers per module. Request pipeline is 4 explicit stages, always in this
   order: **Guards** (authZ decision, run before the handler) → **Interceptors** (AOP-style
   wrap/transform, can touch request and response) → **Pipes** (per-param transform/validate) →
   **Exception Filters** (catch and normalize errors). Pick when: enterprise team, many modules,
   long-lived codebase, need enforced boundaries.
2. **Encapsulation-context plugin tree** (Fastify) — `register()` creates a child scope that inherits
   parent decorators/hooks but whose own additions are invisible to siblings/parent. No module system,
   zero-boilerplate scoping (e.g. mount auth only under one subtree). Documented lifecycle:
   `onRequest → preParsing → Parsing → preValidation → Validation → preHandler → Handler → Reply →
   preSerialization → onSend → Response → onResponse`. Pick when: Node-only shop, want schema-
   compiled speed without DI ceremony.
3. **Flat onion middleware + typed context chaining** (Hono/Elysia) — lightest; `compose()` funnels
   requests through a middleware stack via recursive `next()`, all thrown errors caught by one
   `onError`. Elysia additionally accumulates types across `.state()/.derive()/.get()` chains so a
   later handler's context type reflects everything chained before it. Pick when: edge/serverless
   target, want minimal cold-start overhead, one runtime-portable core with thin per-target adapters
   (Hono's deploy-target adapters, Elysia's Bun/CF-worker adapters — same "adapter isolates the
   runtime, core stays portable" idea, worth keeping regardless of framework).

Python equivalents: **FastAPI** — DI-as-function-signature (`Depends(...)`, composable, `yield`-based
setup/teardown, cache the `Annotated[Type, Depends(fn)]` alias to avoid repeating `Depends()`); default
recommendation for new Python APIs. **Litestar** — same DI shape but explicit app/router/route-level
layering plus a built-in generic `AbstractAsyncRepository` + `FilterTypes` (pagination/ordering/search)
worth adopting as the CRUD-standardization shape across many entities even outside Litestar itself.

Rust: **axum** — no framework-specific middleware; middleware is `tower::Service`, shared with the
whole Tokio/Hyper/Tower ecosystem. Extractors (`Path`, `Query`, `Json<T>`, `State<T>`) parse-and-
validate *as handler parameters* — if extraction fails the handler never runs and a typed rejection is
returned automatically ("parse, don't validate, as params"). Services must resolve to `Infallible`
error type — a handler never drops a connection unhandled, `E: IntoResponse` forces every error path to
produce a deliberate typed response.

**Retrieval-first**: frameworks ship breaking changes fast (framework major-version routing changes,
new DI syntax). Fetch current official docs/types for the target framework version before generating
route/handler code — do not rely on memorized APIs.

### 3. Validation and serialization: schema as single source of truth
Define the contract **once**, derive validation + serialization + docs/types from that one
declaration — never hand-maintain a separate DTO class, a separate validator, and a separate OpenAPI
annotation for the same shape. Concrete shapes, pick one per stack:
- TypeBox/Zod schema (Elysia/Hono) → validation + serialization + OpenAPI + TS type from one object.
- JSON Schema (Fastify) → Ajv-compiled validator **and** fast-json-stringify-compiled serializer, both
  directions — the schema also governs what fields *leave* the server, not just what's accepted in.
  Security note: schemas compile to `new Function()` — never accept user-supplied schemas at runtime.
- Pydantic model (FastAPI) → request body schema + response schema (`response_model`) + OpenAPI, one
  declaration, three uses.
- DTO class + decorators (NestJS `class-validator`/`class-transformer` via `ValidationPipe`) — the
  standard enterprise-Node shape; contrast honestly: extra artifact vs Elysia/Fastify's single-schema
  model, but broader tooling/ecosystem maturity.
- `Serializer`/`ModelSerializer` (DRF) — one object owns "shape of data in and out," auto-derived from
  the model where possible.

Whichever shape is chosen, do not let a second, hand-written source of truth for the same field list
drift out of sync (e.g., a manually-kept OpenAPI YAML alongside a Pydantic model) — regenerate the
docs artifact from the code schema, not the reverse.

### 4. Error shape (single-home: THIS reference owns it)
Normalize every error — expected (validation, 404, 409) and unexpected (unhandled exception) — into
one consistent response shape before it reaches the client. Minimum fields: `type`/`code` (machine-
readable, stable), `message` (human-readable), `status` (HTTP code), and for validation errors a
`details`/`errors` array of per-field problems. RFC 7807 (`application/problem+json`: `type`, `title`,
`status`, `detail`, `instance`) is a reasonable default shape for REST if no house style exists yet —
state this as a default, not a mandate, if the repo already has an established error contract.
Mechanism per framework: NestJS `ExceptionFilter` (`BaseExceptionFilter` normalizes both `HttpException`
and unknown errors via the `HttpAdapterHost`, independent of Express/Fastify underneath), Hono/Elysia's
single `onError` funnel, axum's `IntoResponse`-typed error enum, FastAPI's `HTTPException` + registered
exception handlers. Principle underneath all of them: **never let an unhandled error escape as a
broken/dropped connection** — always answer the client with a typed, deliberate response.

### 5. Pagination (single-home: THIS reference owns it)
Default to **cursor-based pagination** for any collection at scale or under concurrent writes — opaque
cursor tokens don't drift when rows are inserted/deleted mid-page, unlike offset pagination. Offset
pagination (`?page=2&size=20`) is acceptable only for small, low-write, admin-style lists where simple
page-number UI matters more than correctness under concurrent writes. Reference implementations:
DRF ships three tiers (`PageNumberPagination`, `LimitOffsetPagination`, `CursorPagination`) — teach
picking cursor for high-write collections specifically. GraphQL: use the Relay cursor-connection spec
(`edges`/`node`/`cursor`/`pageInfo`) rather than inventing an ad hoc shape. Stripe's convention (cursor
= previous page's last object ID) is a simple, battle-tested cursor design when no stronger opinion
exists.

### 6. Versioning in code (single-home: THIS reference owns it)
State a versioning strategy explicitly rather than leaving it implicit:
- **REST**: version in the URL path (`/v1/...`) or a header (`Accept: application/vnd.api+json;
  version=1`) — pick URL-path versioning as the simpler default unless the codebase already uses
  header versioning. Never break a shipped version silently; add a new version instead of mutating one
  in place. Stripe's model (pinned per-integration version, dated public changelog) is a good pattern
  for how to *announce* changes even inside a single version if truly additive.
- **GraphQL**: avoid versioning entirely — evolve the schema by additive change only (new nullable
  fields/types); because clients declare exactly the fields they want, additive changes are non-
  breaking by construction. Deprecate fields with `@deprecated(reason: ...)` instead of removing them
  outright.
- **gRPC/protobuf**: evolve via field numbers — never reuse or renumber an existing field number, add
  new fields with new numbers, mark removed fields `reserved`.

## Validation
- [ ] Transport choice stated explicitly with the tradeoff, not just asserted.
- [ ] One schema/contract declaration drives validation, serialization, and docs/types — no
      hand-duplicated shape.
- [ ] Every error path (expected and unexpected) returns the same normalized shape; nothing throws
      an unhandled exception past the framework boundary.
- [ ] Pagination strategy stated (cursor default at scale) and consistent across the API surface.
- [ ] Versioning strategy stated in code, not left implicit.
- [ ] Current official framework docs/types were checked for the target version before generating
      framework-specific code (routing/DI syntax changes fast).

## Failure modes and handoff
- Contract undecided (boundaries, which services exist, NFR budget) → route to `architecture`; do not
  invent a boundary decision here.
- Business-rule logic creeping into the handler/controller → route to `domain-logic` (use-case/service
  layer, anti-anemic-model check).
- Auth/token/session handling needed inline → route to `auth`.
- Need to verify the implementation actually works (tests, migrations, hitting endpoints) →
  `backend-testing`.
- Need an independent load test, security scan, or release gate on this API → `quality`.
