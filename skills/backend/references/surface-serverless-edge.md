# Surface: Serverless / edge

Purpose: Reshape the 9 job references for a service deployed as short-lived, per-request functions
at the edge or in a managed serverless runtime (Cloudflare Workers, AWS Lambda, Bun-on-edge,
Vercel/Netlify functions) — where the runtime itself is the dominant constraint, not the framework.

Read when:
- The deploy target is Workers/Lambda/edge-functions/Bun-edge, or the request mentions cold start,
  scale-to-zero, or per-invocation billing.
- A database connection error under load traces back to exhausted Postgres connections from many
  short-lived function instances — the single most common serverless-backend failure mode.
- The service needs KV/D1/R2-style edge storage or coordinated in-region state (Durable Objects)
  rather than a conventional always-on database/cache tier.

Skip when:
- The target is a conventional long-running process/container → [surface-rest-crud](surface-rest-crud.md).
- The actual deploy pipeline/CI/monitoring setup for the function itself — that's `operate`'s job;
  this ref covers what the *code* must account for to run correctly on that runtime, not how it
  ships.
- Connection-pooling *mechanics* in depth (pool modes, PgBouncer internals) — owned by
  [persistence](persistence.md); this ref states the failure mode and points there.
- WebSocket/realtime fan-out — `surface-realtime` (Durable Objects also appear there for a
  different reason: stateful coordination, not edge-persistence).

Inputs: the specific runtime (Workers vs Lambda vs Bun-edge — behavior differs enough that "edge"
alone is not sufficient), whether the datastore is already fixed (existing Postgres) or open to a
runtime-native option (D1/KV), current framework version if already chosen.

Produces: the runtime-portability posture (adapter vs core), the pooling/driver decision that
avoids the connection-exhaustion failure mode, and the storage-primitive mapping (which edge
primitive fits which state shape).

## Contents
- Cold start and statelessness
- The connection-pooling failure mode
- Edge persistence primitives (D1/KV/R2)
- Durable Objects for coordinated state
- The adapter-isolates-runtime principle
- Validation
- Failure modes and handoff

## Cold start and statelessness

Every invocation may run in a fresh isolate/container with no guaranteed warm reuse. Two
consequences to build in from the start, not retrofit:
- **No in-process state survives between requests** — no in-memory cache, no session-affinity
  assumption, no "cache it in a module-level variable and hope the instance stays warm." Anything
  that must persist goes to an external store (see below), matching 12-factor's Processes/Disposability
  factors even more strictly than a conventional service.
- **Minimize what runs at module load time.** Heavy imports, connection setup, or crypto key
  derivation at the top of the module cost is paid on every cold start, not once. Keep the
  module-scope surface small; do expensive setup lazily, on first request, when the runtime allows
  a warm instance to amortize it — but never assume it will be warm.

## The connection-pooling failure mode (read this before wiring any relational DB here)

The most repeated serverless-backend failure: a traditional Postgres/MySQL client opens one TCP
connection per function instance. Under bursty serverless concurrency (hundreds of simultaneous
short-lived invocations), this exhausts the database's max-connections ceiling in seconds — a
problem that never shows up in local dev or in a conventional always-on service with a handful of
long-lived pooled connections. Postgres's per-connection process model was never designed for
thousands of concurrent, transient serverless callers.

Two independent fixes, often combined — full mechanics owned by [persistence](persistence.md):
- **A pooler in front of the database, in transaction mode.** PgBouncer transaction-mode (or a
  managed equivalent — Neon's built-in pooler, Prisma Accelerate, or **Cloudflare Hyperdrive** for an
  existing Postgres/MySQL reached from Workers) holds a connection only for the
  duration of one transaction, then returns it to the pool immediately — decoupling "number of
  serverless callers" from "number of real database connections." Know the tradeoff before adopting
  it blind: transaction-mode pooling breaks session-scoped features (LISTEN/NOTIFY, advisory locks,
  some `SET`/temp-table usage) unless the pooler version specifically restores them (PgBouncer 1.21+
  added prepared-statement support in transaction mode — verify the deployed version, don't assume).
  A pooler can also *cache*: Hyperdrive caches eligible read-only query results by default
  (`max_age` 60 s plus 15 s `stale_while_revalidate`; Cloudflare docs, *Query caching*, read
  2026-10-17), and the switch is per configuration, not per query — route reads of permissions,
  billing, or an operation ledger through a second, cache-disabled binding.
- **An HTTP-native driver that skips persistent TCP connections entirely** — e.g. Neon's HTTP driver
  or Turso/libSQL's HTTP protocol issue each query as a stateless HTTP request instead of holding a
  socket open. This is a clean fit for edge runtimes where holding long-lived TCP sockets is awkward
  (Cloudflare Workers historically had no raw TCP; it now supports outbound TCP via `connect()` from
  `cloudflare:sockets` — which is what Hyperdrive itself uses to pool a conventional database), at the
  cost of losing connection-level features (multi-statement transactions across round-trips, session
  state) that a pooled TCP connection would give you.

Default rule for this surface: **if the datastore is a conventional Postgres/MySQL and the deploy
target is edge-native (Workers) or high-concurrency serverless (Lambda), do not reach for a plain
TCP client** — pick transaction-mode pooling or an HTTP-native driver explicitly, and say which one
and why, before writing persistence code.

## Edge persistence primitives (D1/KV/R2)

When the datastore is open to a runtime-native option rather than fixed to existing Postgres, match
the storage primitive to the state shape rather than defaulting to "a database" out of habit:
- **KV** — eventually-consistent, globally-replicated key-value; right for read-heavy, rarely-changed
  data (config, feature flags, cached lookups) where up to ~60s of propagation lag across regions
  is acceptable. Wrong fit for anything needing strong consistency or frequent writes to the same key.
- **D1** — SQLite-based, edge-native relational store; right for structured, per-tenant, or
  moderate-scale relational data that wants SQL semantics without standing up a full Postgres
  instance reachable from the edge. Treat it as a lighter relational tier, not a Postgres
  replacement at scale.
- **R2** — S3-compatible object storage; right for blobs/files/large payloads, not structured
  queryable data.

State explicitly which primitive backs which piece of state — don't let "it's serverless so just
use KV for everything" become the default when the data is actually relational.

## Durable Objects for coordinated state

When a piece of state needs **single-writer coordination** (a single source of truth for one
logical entity — a chat room, a booking/seat-lock, a rate-limit counter, a WebSocket
session-coordinator) rather than globally-distributed eventually-consistent access, that's a
different problem than KV/D1 solve: Durable Objects give one object instance per logical ID, with
its own private storage and in-memory state, reachable from anywhere but always routed to the same
instance — the mechanism for strong consistency and coordination at the edge where a conventional
edge-cache primitive would give you race conditions instead. Full realtime/fan-out treatment lives
in `surface-realtime`; here, use them whenever a piece of *non-realtime* state (a lock, a counter,
a per-tenant coordinator) needs single-writer semantics that KV/D1 can't provide. One object per
coordination unit (user, room, tenant), never one global object for every request — Cloudflare's
*Rules of Durable Objects* call that a chokepoint. A name always maps to the same object, so derive
it server-side from the verified identity, never from a client-supplied ID.

## The adapter-isolates-runtime principle

This surface is the sharpest test of the principle stated in
[surface-rest-crud](surface-rest-crud.md#three-concern-scoping-models): write the application
against the framework's web-standard core (`Request`/`Response`), and push every
platform-specific translation (the Lambda event shape, the Workers `fetch` handler signature, the
Bun server API) into a thin adapter at the edge of the app. Hono and Elysia both ship this as a
first-party pattern — the app/router code is identical whether it runs on Cloudflare Workers, Bun,
or Node; only the adapter file changes. Adopt this even outside those two frameworks: **never let
Lambda-event-shape or Workers-`fetch`-signature details leak into route/service/domain code** — that
coupling is what makes a serverless-first codebase expensive to ever move off one specific vendor's
event model.

## Validation

- [ ] No in-process state assumed to survive across invocations.
- [ ] The connection-pooling failure mode is explicitly addressed (pooler-in-transaction-mode or
  HTTP-native driver named), not left as a plain TCP client against Postgres/MySQL.
- [ ] Each piece of state maps to the storage primitive suited to its consistency/access shape
  (KV/D1/R2/Durable-Object), stated explicitly rather than defaulted.
- [ ] Runtime-specific event/handler shapes are isolated in an adapter layer, not spread through
  app code.
- [ ] Deploy/CI/monitoring for the function itself is left to `operate` — not designed here.

## Failure modes and handoff

- **A plain TCP Postgres client wired into a Workers/Lambda handler** — the single most common
  failure this ref exists to prevent; fix before shipping, not after the first connection-exhaustion
  incident.
- **Using KV for data that's actually relational/transactional** — reach for D1 (or fixed
  Postgres via pooling) instead once queries need joins or multi-row consistency.
- **Coupling route/service code to one vendor's event shape** — retrofit an adapter layer before it
  spreads further.
- **Treating this surface as owning deploy config** — hand that decision to `operate`; this ref only
  covers what the application code must account for.
- If the workload turns out to need long-lived stateful connections (WebSockets, presence,
  fan-out) beyond a single coordinating object, hand off to `surface-realtime`.
