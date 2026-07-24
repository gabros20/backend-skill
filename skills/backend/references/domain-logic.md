# Domain Logic

Purpose: Structure the business/domain layer behind an API or job — use-cases/services, DDD tactical
building blocks, clean/hexagonal/onion layering — so business rules live in one testable place instead
of leaking into handlers, ORM models, or infrastructure code.

Read when:
- Request asks to add or refactor business logic, a service layer, use-cases, or domain models.
- Handlers/controllers/views are accumulating business rules directly (a smell this reference
  diagnoses explicitly, see anti-anemic litmus below).
- Request mentions DDD, aggregates, entities, value objects, clean architecture, hexagonal, onion,
  ports and adapters.

Skip when:
- The question is *where the service boundary sits* (which bounded context owns what, how many
  services should exist) — that's a strategic-design decision owned by `architecture`; this reference
  only covers *tactical* patterns inside one service/module.
- The work is the HTTP/RPC layer itself (routing, request lifecycle, error shape) — see
  `api-implementation`.
- The work is persistence/schema/query concerns — see `persistence`.

Inputs: the language/framework in use (state explicitly when absent), whether a bounded-context/service
boundary has already been decided upstream (if not, state that assumption explicitly rather than
inventing one), rough domain complexity (CRUD-shaped vs rule-heavy).

Produces: a layered code structure (domain / application / infrastructure, or the team's existing
equivalent) with business rules expressed as named entity/aggregate methods and use-cases, verified
against the anti-anemic litmus and the dependency rule.

## Contents
- When to use this at all (and when to skip it)
- The dependency rule
- Layers: clean / hexagonal / onion — one model, three names
- DDD tactical building blocks
- The anti-anemic-model litmus test
- Sharp edges: where LLMs place code wrong
- Anti-patterns

## Procedure

### 1. When to use this at all
Full clean/hexagonal/DDD-tactical layering is a cost, not a default. Apply when the domain is complex
enough that business rules genuinely need protecting from framework churn, the codebase is long-lived,
and more than a couple of engineers touch it. **Skip or lighten it** for CRUD-shaped screens, a
prototype, or a solo-maintained small service — "most systems don't need full CQRS or Event Sourcing,"
and most don't need a 4-layer hexagon either. A thin service-layer-over-ORM-models is a legitimate,
smaller default; escalate to full tactical DDD only when that thin layer starts accumulating
conditional business rules that don't belong in a controller.
(Model: a synthesis of DDD/Hexagonal/Clean/Onion/CQRS.)

### 2. The dependency rule
Regardless of which named layering you use, one rule underlies all of them: **dependencies point
inward, toward the domain; the domain depends on nothing outer.** Apply Cockburn's testability litmus
directly: you should be able to run and test your application logic with no UI and no database attached
— if a domain test requires spinning up a real HTTP server or a real database connection, the
dependency rule has been violated somewhere.

### 3. Layers: clean / hexagonal / onion — one model, three names
Treat these as the same idea under different vocabulary, not three competing systems to choose between:
- **Domain** (entities, value objects, domain services, ports/interfaces) — no framework imports, ever.
- **Application** (use-cases/application services) — orchestrates domain objects, owns transaction
  boundaries (begin/commit), calls ports; still framework-agnostic.
- **Infrastructure/Adapters** (ORM repositories, HTTP clients, message publishers) — implements the
  ports the domain/application defined; this is the *only* layer allowed to import framework/ORM code.
There are two equally valid conventions for where ports live: DDD-centered (ports declared beside the
aggregate, inside `domain/`) or stricter Hexagonal (ports under `application/ports/driven/`). Pick one
per codebase and state which — don't silently mix both.

### 4. DDD tactical building blocks
- **Entity** — has identity that persists across state changes (an `Order`, tracked by ID even as its
  fields change).
- **Value Object** — has no identity, defined entirely by its attributes, immutable (a `Money`, an
  `Address`). Decision heuristic: if two instances with identical field values should be
  interchangeable, it's a value object; if identity matters even when fields match, it's an entity.
- **Aggregate** — a cluster of entities/value-objects treated as one consistency/transaction boundary,
  with a single **Aggregate Root** as the only external access point (an `Order` aggregate root owning
  its `LineItem` entities). Rule: load and save the whole aggregate atomically; never let one
  transaction span multiple aggregates. Aggregate-boundary heuristic: more than ~10 entities in one
  aggregate is a signal to split it; cross-aggregate consistency is handled via domain events, not a
  shared transaction. (Source: Fowler's DDD Aggregate note, Evans 2003, Vernon 2013.)
- **Repository** — a collection-like interface over an aggregate's persistence, declared as a port in
  the domain/application layer and implemented in infrastructure. One repository per aggregate root,
  not per entity (repository-per-entity is a listed anti-pattern below).
- **Domain Events** — "something happened" facts published by an aggregate, named in the past tense
  (`OrderPlaced`, not `PlaceOrder`), used to coordinate across aggregate/service boundaries without a
  shared transaction. Pair with the Transactional Outbox pattern: saving an aggregate and publishing its
  event are two separate writes — a crash between them silently drops the event unless the event is
  written to an outbox table in the *same* local transaction as the aggregate save (mechanics owned by
  `async-and-messaging` — point there, don't reimplement the outbox here).
- **Use Case / Application Service** — one method per user-facing operation, orchestrates
  load-aggregate → call domain method → save-aggregate → publish events, and owns the transaction
  boundary. Thin by design — the business rule itself lives on the entity/aggregate, not here.

### 5. The anti-anemic-model litmus test
An **anemic domain model** is entities that are just field bags (getters/setters, no behavior) with all
business rules living in a separate service that reads state, decides, and writes state back — this
defeats the point of having a domain model at all. Litmus test: **if a service method reads an entity's
state, makes a decision based on it, and writes the result back onto the entity, that logic belongs
inside a named method on the entity itself**, not the service. Example: instead of
`if order.status == 'pending' and order.total > 0: order.status = 'confirmed'` sitting in a service,
give `Order` a `confirm()` method that enforces its own invariant and raises a domain error if called
in an invalid state. The service/use-case then just calls `order.confirm()`.

### 6. Sharp edges — where LLMs place code wrong
A concrete "where does this go" table for the mistakes that recur most:
| Concern | Wrong place | Right place |
|---|---|---|
| A business invariant/rule | Application service | Entity/aggregate method |
| Input-format validation (types, required fields) | Domain entity | Adapter/DTO at the boundary |
| Transaction begin/commit | Domain | Application/use-case layer |
| An ORM model | Used as the domain entity directly | Infrastructure only — map to/from a separate domain entity, never let the ORM model *be* the domain model |
| Authorization ("can this user do this?") | Domain (unless it's a true business rule, e.g. "only the owner can cancel") | Application-layer policy |
| Clock / UUID generation | Called directly in domain code | Declared as a port in the domain, implemented as an adapter in infrastructure (so tests can inject a fixed clock/ID) |

### 7. Anti-patterns
Uniform checklist — self-audit each item before calling domain-layer work done:
- [ ] Anemic Domain Model (see litmus test above)
- [ ] Repository-per-entity instead of per-aggregate-root
- [ ] Leaking Infrastructure (framework/ORM imports inside `domain/`)
- [ ] God Aggregate (one aggregate absorbing unrelated entities/rules — split by transaction boundary)
- [ ] Skipping Use Cases (handler calls repositories directly, no application-layer orchestration)
- [ ] CRUD Thinking (modeling everything as create/read/update/delete instead of named domain
      operations like `confirm()`/`cancel()`/`ship()`)
- [ ] Premature CQRS/Event Sourcing (adopted without a proven need — see §1)
- [ ] Cross-Aggregate Transaction (one DB transaction spanning two aggregate roots — use domain events
      + eventual consistency instead)

## Validation
- [ ] Domain layer has zero framework/ORM imports (grep-checkable: e.g.
      `grep -r "@Service\|@Autowired\|import express\|import fastapi" domain/` should return nothing).
- [ ] Every business rule found while reviewing lives on an entity/aggregate method, not floating in a
      service that only reads-decides-writes.
- [ ] One repository per aggregate root, not per entity.
- [ ] Aggregate boundaries match transaction boundaries — no code path commits across two aggregates
      in one transaction.
- [ ] Chosen port-placement convention (domain-centered vs strict-hexagonal `application/ports/`) is
      stated and applied consistently, not mixed.

## Failure modes and handoff
- Question is really "should this be two services" or "what's the bounded context boundary" → route to
  `architecture`; this reference only covers tactical patterns inside an already-decided boundary.
- Cross-aggregate coordination needs reliable event delivery / an outbox → point to
  `async-and-messaging` for the delivery mechanics; keep only the domain-event *naming/raising*
  convention here.
- Need to verify the layering itself holds (dependency-rule compliance test, not just business
  correctness) → `backend-testing` (structural/architecture-compliance check).
- Persistence mapping (how the ORM model maps to the domain entity, migrations) → `persistence`.
