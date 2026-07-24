# Surface: Event-driven microservices

Purpose: Reshape the job refs for a distributed, polyglot runtime — multiple independently
deployed services communicating over gRPC and an event-streaming backbone. Gates on whether the
system actually needs this surface before applying it; the default remains a single deployable.

Read when:
- Services are (or are being split into) independently deployable units with their own data store,
  communicating over the network rather than in-process.
- The request names gRPC, Kafka/NATS, service mesh, or "microservices" explicitly, or describes
  multiple teams/services needing independent deploy cadence.

Skip when:
- There's no named, concrete trigger for splitting (team-ownership boundary, independent scaling
  need, independent deploy cadence, a proven bounded context) — default to `surface-rest-crud` on a
  modular monolith instead; see step 1 before reading further.
- The event/async need is *within* one service (background jobs, one service's own queue) — that's
  `async-and-messaging` directly, not this surface.
- The *decision* of where the service boundary goes is still open — that's `architecture`'s call;
  this surface implements a boundary already decided, it doesn't decide one.

Inputs:
- The stated trigger for the split (if any) — team ownership, scaling, deploy independence, a
  bounded context from `architecture`. State explicitly when none exists yet.
- Which services are in scope, and whether the event backbone (Kafka/NATS/other) is already chosen
  or must be picked.

Produces:
- Inter-service contracts (versioned, with an explicit compatibility policy), a choice of
  synchronous (gRPC) vs asynchronous (event stream) communication per interaction, and — if
  database-per-service — a stated Saga/CQRS strategy for the consistency this gives up.

## Contents
- Gate: do you actually need this surface?
- Database-per-service and its consequences
- Inter-service contracts
- gRPC for service-to-service calls
- Event streaming backbone
- Service mesh: when it earns its keep

## Procedure

### 1. Gate: do you actually need this surface?
The dominant 2025-2026 counter-narrative to reflexive microservices is **modular-monolith-first**:
start with one deployable, organized into clearly bounded modules, and split into separate services
only when a concrete, named trigger justifies the operational cost (independent scaling, independent
deploy cadence, a genuinely separate team, a bounded context proven stable over time) — not
preemptively, and not because microservices are the perceived industry default. A production case
worth citing directly: a team migrated a SaaS **from a Python/React microservices architecture back
to a single Elixir/Phoenix monolith** — 3 apps→1, 4 languages→1, 5 frameworks→1 — as a real,
production consolidation, not a thought experiment. Treat this gate as mandatory: if there's no
named trigger in the request, say so and default to a modular monolith rather than silently
building out this surface.

### 2. Database-per-service and its consequences
Once services are genuinely separate, **database-per-service** (each service owns its private data,
no shared schema/cross-service joins) is the accepted default — but it is a package deal, not a free
lunch. It forces two consequences you must plan for together, not bolt on after:
- **Saga** — a multi-step business transaction that used to be one local DB transaction becomes a
  sequence of local transactions with compensating actions on failure (choreography: each service
  reacts to events; orchestration: a coordinator drives the steps). Saga gives up cross-service
  isolation — accept eventual consistency deliberately, don't discover it as a surprise. The
  mechanics (compensation design, orchestrator-vs-choreography tradeoff, idempotent compensations)
  live in `async-and-messaging` — this surface only tells you Saga is *required* the moment you
  commit to database-per-service, it doesn't re-derive how to build one.
- **CQRS** — separating the read model from the write model becomes necessary once a query needs
  data that now lives across multiple services' private databases (API composition or a dedicated
  read-side projection). Martin Fowler's own framing is a caution, not an endorsement: CQRS is
  "risky complexity" appropriate only for a narrow bounded context with a genuine read/write
  asymmetry — don't apply it system-wide by default. Mechanics again live in `async-and-messaging`.

### 3. Inter-service contracts
Every service boundary is a contract that must survive independent deploys on both sides — version
it explicitly rather than relying on "we'll coordinate deploys." Concretely: additive-only schema
evolution where the protocol allows it (gRPC/protobuf fields are optional and numbered; adding a
field is safe, renumbering or removing one is not), a stated deprecation window before removing any
field/RPC a consumer depends on, and consumer-side tolerance for unknown fields (never fail hard on
an unrecognized field from a producer that shipped first). The contract's *shape* (what fields, what
guarantees) is `architecture`'s call if it's part of the initial boundary design — this surface
owns keeping the contract's *evolution* disciplined once implementation starts.

### 4. gRPC for service-to-service calls
Default choice for internal, performance-sensitive, or streaming synchronous calls between services
in this surface: proto3 IDL with generated stubs, four call modes (unary, server-streaming,
client-streaming, bidirectional). Reach for it over REST internally when you want strict typed
contracts and streaming; avoid it for anything a browser calls directly (`grpc-web` support is
weak) — REST+OpenAPI stays the right choice for public/multi-language/browser-facing surfaces (that
decision lives in `api-implementation`). Within this surface, gRPC is the synchronous complement to
the event-streaming backbone below: use gRPC when the caller needs an immediate answer, events when
it doesn't.

### 5. Event streaming backbone
Kafka (or NATS for lighter-weight/simpler ops) is the default asynchronous backbone connecting
services in this surface — the broker choice, delivery-semantics mechanics (at-least-once +
idempotent consumers, partition/ordering guarantees, consumer-group rebalancing), and CDC/outbox
wiring into the stream are owned by `async-and-messaging`; this surface's job is deciding *which*
inter-service interactions should be event-driven (fire-and-forget notifications, fan-out to
multiple consumers, anything where the producer shouldn't block on every consumer's availability)
versus synchronous gRPC (the caller needs the result now). Don't re-derive broker internals here —
point there.

### 6. Service mesh: when it earns its keep
A service mesh (mTLS between services, retries/circuit-breaking at the network layer, observability
without app-code changes) earns its operational cost once the service count and cross-team surface
area make hand-wiring those concerns per-service unmanageable — it is not a default for a
first split of 2-3 services. Until then, implement retries/circuit-breakers/mTLS directly in each
service (see `service-integration` for the client-side retry/circuit-breaker mechanics, `server-
security` for mTLS) — introducing a mesh earlier adds an operational layer (control plane, sidecar
proxies) that isn't paying for itself yet.

## Validation
- [ ] A concrete, named trigger for the service split is stated (not "microservices by default");
      absent one, the recommendation is a modular monolith instead.
- [ ] If database-per-service: both Saga (for cross-service transactions) and, where a cross-service
      read is needed, CQRS/API-composition are explicitly planned — not discovered later as bugs.
- [ ] Every inter-service contract has a stated evolution policy (additive-only, deprecation window,
      unknown-field tolerance).
- [ ] Each inter-service interaction is explicitly classified sync (gRPC) or async (event stream),
      not left ambiguous.
- [ ] Service mesh is justified by stated service-count/cross-team scale, not adopted by default.

## Failure modes and handoff
- No named trigger for splitting services exists → stop, recommend `surface-rest-crud` on a modular
  monolith, and say so plainly rather than building out this surface speculatively.
- The service-boundary *decision* (where exactly to cut, what the bounded context is) is still open
  → that's `architecture`'s call; this surface implements a boundary already decided.
- Saga/CQRS/outbox mechanics, broker choice, or delivery-semantics detail is the actual gap → hand
  off to `async-and-messaging`; don't re-derive broker/consumer internals inside this surface.
- The async need doesn't cross a service boundary at all (one service's own background jobs) → this
  isn't the right surface; go straight to `async-and-messaging`.
- Retry/circuit-breaker/mTLS implementation detail (not the "do we need a mesh" decision) is the
  ask → route to `service-integration` and `server-security` respectively.
