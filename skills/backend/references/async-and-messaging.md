# Async and Messaging

Purpose: Turn a request for background jobs, event-driven communication, or a multi-step process
that must survive failure into a concrete queue/broker pick plus the reliability patterns
(idempotency, outbox, saga, delivery semantics, durable execution) that make it correct.

Read when:
- The request mentions background jobs, workers, queues, pub/sub, event-driven services, CDC,
  Kafka/NATS/RabbitMQ/SQS, "process this asynchronously," saga, outbox, "at-least-once," durable
  workflows, or Temporal/Hatchet.
- A multi-step business process (payments, onboarding, an agent tool-loop) must survive a crash or
  deploy mid-flight.

Skip when:
- The question is whether the system should be async/distributed at all (a boundary decision) →
  `architecture`.
- The work is calling OUT to a third-party/webhook/AI/email endpoint → `service-integration`
  (reuses the retry/idempotency mechanics defined here — don't redefine them there).
- The work is streaming analytics/OLAP off an event stream → `data`.
- The work is deploying, scaling, or monitoring the broker/queue itself → `operate`.
- SQL mechanics for a jobs table (indexes, isolation level, connection pooling) → `persistence`.
- The saga spans **vendors you cannot deploy to** — no compensating action exists on the other side
  (no un-send, no un-charge), or a step is a human approval → `automation`. The mechanics here
  still apply; the discipline for running them against an opaque third party is `automation`'s.

Inputs:
- Current stack (already Postgres-centric? existing broker?), rough throughput, and whether the
  process spans multiple services/databases. State explicitly when unknown — never assume Kafka-scale
  volume without evidence.

Produces:
- A broker/queue pick with rationale, plus the reliability patterns applied: idempotency-key path,
  operation ledger (if a worker calls out to an irreversible effect), outbox (if a DB write must
  reliably notify elsewhere), saga with compensations (if multi-service), an explicit
  delivery-semantics stance, and a durable-execution decision if the process is long-lived.

## Contents
- Choosing the queue/broker (Postgres-as-queue vs Kafka/NATS/RabbitMQ/SQS)
- Idempotency keys + the operation ledger for external effects
- Transactional outbox
- Saga (choreography vs orchestration)
- Delivery semantics
- Retry, backoff, jitter, DLQ
- Durable execution (Temporal vs Hatchet)

## Procedure

### 1. Choose the queue/broker
Default: if the system is already Postgres-centric and volume is modest, don't add a broker. Use a
jobs table with `SELECT ... FOR UPDATE SKIP LOCKED` as the safe-concurrent-dequeue primitive (via
pg-boss, Hatchet, or hand-rolled) — Hatchet's own SQL schema uses exactly this and is load-tested to
10k tasks/sec on it. Tradeoff, per Hatchet's README: more DB resources than a Redis/RabbitMQ-backed
queue, in exchange for full execution history for free (no separate replay/inspect tooling). SQL,
pooling, and isolation mechanics → `persistence`; this ref owns the *shape* decision.

Beyond single-process jobs, pick by what's actually needed:
- **Kafka** — steady/dominant default for high-throughput, ordered event streaming; 4.0 (Mar 2025)
  dropped ZooKeeper (KRaft-only).
- **NATS JetStream** — rising pick when you want pub/sub without Kafka's operational weight.
- **RabbitMQ** — steady enterprise standard; strong routing topology (exchanges/queues) when message
  routing complexity matters more than raw throughput.
- **SQS** — AWS-managed default when already on AWS; native DLQ via redrive policy.
- Cost is an architectural axis: shared-storage designs (e.g. diskless Kafka on S3) trade local-disk
  throughput for lower cross-AZ egress cost — a real 2025-2026 line item. For an open-core
  diskless-Kafka product, verify whether a recommendation covers the OSS core or the commercial
  control-plane/HA tier.
- Anti-pattern: Kafka/RabbitMQ "because it's the standard" for what is a handful of background
  jobs — the Postgres jobs table is simpler ops there.

### 2. Idempotency keys — **owned here**
A client-supplied unique token per logical operation; the server persists `(key → result)` with a
TTL and replays the stored result on retry instead of re-executing (Stripe's pattern: an
`Idempotency-Key` header, response stored keyed by account+key, replayed within a bounded window).
Apply it at two boundaries: the message-consumer boundary (dedupe by message/event ID) and the API
boundary (`service-integration` applies the same primitive to inbound webhooks — point there, don't
redefine). Two equivalent storage shapes, worth teaching as one concept: TTL-based (dedupe within a
time window) and status-based (dedupe by run status) — Hatchet ships both as SDK config, not just a
raw hash-the-payload trick. See `assets/idempotency-and-outbox-recipes.md` for a fillable handler.

**Operation ledger for external effects — owned here.** The idempotency key dedupes retries coming
*in*; a call *out* to an effect you cannot roll back (send, charge, provision) needs the mirror
image — one ledger row per consequential action, reserved in a transaction before the call, **unique
on logical action + resource scope** (DDL and states: the recipes asset, Recipe 3). A genuine retry
resends the **same** downstream key (Temporal: workflow run ID + activity ID is stable across
retries); a new user-authorized action gets a new logical ID; same key with different arguments is a
validation error, not a replay (AWS Builders' Library, *Making retries safe with idempotent APIs*).
Persist the receipt before advancing. A post-submit timeout writes `unknown` and runs the reconcile
path (Recipe 3, step 4; the rule is `architecture`'s Idempotency row). The ledger alone does not
close the window between downstream success and the local receipt write (Temporal documents the
worker-crash-before-report rerun) — only a downstream key or lookup does. An expiring lease needs a
**fencing token** the store checks, so a stalled worker cannot write after ownership moved
(Kleppmann, *How to do distributed locking*). Cancel stops future steps; it undoes nothing. The
contract is `architecture`'s (`contracts.md`, *API-contract review checklist*, Idempotency and
Error-handling rows); reconciling against a vendor's own records is `automation`'s
(`reconciliation.md` "Compare the ID set before the fields, then classify — never binary").

### 3. Transactional outbox — **owned here**
The dual-writes problem, precisely: an app that commits its DB write, then separately writes to a
broker/search-index/cache, can crash between the two writes and go permanently inconsistent — this is
the reason the outbox pattern exists, not an arbitrary complexity add.

Mechanism: write the outgoing event to an `outbox_event` table in the SAME local transaction as the
business write; a relay reads that table and publishes to the broker; consumers dedupe by event ID
(§2). Ready-to-teach DDL (Debezium's outbox extension):

```sql
create table outbox_event (
  id                  uuid primary key,
  aggregatetype       varchar(255) not null,
  aggregateid         varchar(255) not null,
  type                varchar(255) not null,
  timestamp           timestamptz not null default now(),
  payload             jsonb not null,
  tracingspancontext  varchar(512)   -- optional: propagate a distributed-tracing context
);
```

Two relay strategies: **polling publisher** (simple, adds DB polling load/latency) vs **transaction
log tailing / CDC** (lower latency, DB-specific, more operational surface — Debezium is the reference
implementation). A router/transform step between the CDC log and consumer topics re-keys the message
by the business aggregate ID (not the outbox row's UUID) and pushes the outbox UUID into a header —
don't pass the raw CDC message straight to consumers. Default cleanup: delete the row once published
(`remove-after-insert`) so the table doesn't grow unbounded.

### 4. Saga — **owned here**
When a business transaction spans multiple services/databases (private data each; cross-service 2PC
would hold every participant's locks for the whole round — fragile under partition, rarely
justified), replace the ACID transaction with a saga: a sequence of local transactions, each
publishing an event/command that triggers the next; failures are undone with compensating
transactions, not automatic rollback.

Two coordination styles:
- **Choreography** — services react to each other's domain events, no central coordinator. Less
  coupling, but the overall flow is only visible by tracing events across services.
- **Orchestration** — a central orchestrator issues commands and tracks state. Easier to observe and
  reason about, at the cost of reintroducing a coordination point. `on_failure_task()`-style hooks (a
  step that runs only after the workflow fails, with the failing task's error) are a minimal shape
  for orchestrated compensation — simpler than a bespoke saga coordinator.

Constraint from domain design: sagas exist precisely because a transaction must never span aggregate
boundaries — each local step commits one aggregate; the saga coordinates across many. Default: prefer
choreography for 2-3 step flows with few participants; move to orchestration once compensation logic
or participant count grows enough to need central visibility. Always pair with outbox (§3) so each
step's local write and next-step publish are atomic.

### 5. Delivery semantics — **owned here**
**At-least-once** (ack after processing — a crash before ack causes reprocessing, no data loss, but
duplicates are possible) vs **exactly-once** (needs transactional coordination across the broker,
costs roughly 10-20% throughput per Kafka's own transactional-producer numbers). Industry-consensus
default: **prefer at-least-once + idempotent consumers** over paying the exactly-once tax; reserve
true exactly-once for narrow, high-value paths (financial ledgers). The same lesson recurs as a
workflow engine's "idempotent or non-retryable activity" contract and a CDC tool's "delivery
semantics is a per-consumer choice" — teach it once here.

### 6. Retry, backoff, jitter, DLQ
Exponential backoff alone causes lockstep retry storms under contention. Use jitter — three
strategies: **Full Jitter** (`sleep = random(0, min(cap, base * 2^attempt))`) has the best measured
performance and is the default; **Equal Jitter** (half fixed + half random) measures worst of the
three; **Decorrelated Jitter** (next bound derived from the previous sleep) adapts dynamically, close
to Full Jitter. Cap retry-of-retries: don't let independent retry layers (caller, client wrapper,
server-side interceptor) stack unbounded — cap the innermost layer to one extra attempt and let the
outermost layer own the real retry budget.

DLQ: a secondary queue receiving messages after they exceed a max-receive-count — quarantines
poison/malformed messages instead of infinite retry (head-of-line blocking) or silent drop. Kafka has
no native DLQ concept; implement it at the consumer-group level (a dedicated error topic + manual or
automated replay). Combine retry+jitter with a circuit breaker (mechanics owned by
`service-integration`) so retries stop entirely once a dependency is confirmed down.

### 7. Durable execution
When a multi-step process must survive crashes/deploys without a hand-rolled state machine,
durable execution is the current default — rising with agent orchestration, since a tool-calling
loop maps directly onto workflow/activity.

Core mental model: every execution is an append-only event History; current state is derived by
*replaying* History, not stored as a blob. Two task types on one poll loop — Workflow Tasks (resume
deterministic user code until it blocks on a timer/activity/completion) vs Activity Tasks (execute
side-effecting I/O). The determinism contract, teach verbatim: **workflow code must be deterministic
and side-effect-free (replay-safe); activity code must be idempotent OR non-retryable** — this is the
idempotency/determinism boundary from §2, applied at the engine level. Durable sleep (suspend a
workflow on a timer, or in a durable wait for an external signal such as an approval, without
holding a worker process alive) is the clearest single illustration of
the category for someone new to it.

Two runtimes, two tradeoffs:
- **Temporal** — a dedicated event-sourced replay cluster; mature ecosystem, MIT-licensed server.
  Reach for it when the system needs a dedicated durable-execution platform independent of the OLTP
  store, or cross-language worker fleets at scale.
- **Hatchet** — Postgres-backed (the same `FOR UPDATE SKIP LOCKED` primitive from §1) durable task
  queue with full execution history, no separate broker or cluster to run. Ships concurrency control
  as declarative per-key expressions (e.g. max concurrent runs per tenant with a named
  cancel-in-progress/queue strategy) instead of manual semaphores — a direct template for per-tenant
  fair scheduling.
- Default: reach for a Postgres-backed durable-execution engine first if already Postgres-centric and
  scale is moderate; graduate to Temporal once a dedicated cluster/ecosystem or large cross-language
  worker fleets are actually needed.
- **One execution owner per side effect.** When a graph runtime's checkpointer runs inside a
  workflow engine (LangGraph in Temporal), the engine owns the business lifecycle, the graph the
  agent state, and exactly one layer retries each side effect — Temporal retries activities by
  default and a LangGraph replay re-executes later nodes' calls, so stacked layers multiply the
  effect. Same for a durability wrapper around a framework that persists its own session. Engine
  task dedup does not make the actions inside a task replay-safe — that is the ledger's job (§2).

## Validation
- [ ] Every client-retriable mutation has an idempotency-key path (persisted key→result, TTL set)
- [ ] Every DB write that must notify elsewhere uses outbox or CDC — never a bare dual write
- [ ] Any multi-service business transaction has an explicit saga with compensations per step, not an
      implicit ACID assumption
- [ ] Delivery-semantics stance is stated explicitly (at-least-once + idempotent is the default;
      exactly-once is justified in writing if chosen)
- [ ] Retries use jittered backoff (Full Jitter default) with a cap, plus a DLQ or equivalent
      quarantine for poison messages
- [ ] If durable execution is used, workflow code is verified deterministic/side-effect-free and
      activities are idempotent-or-non-retryable, and exactly one layer retries each side effect
- [ ] Every irreversible external effect runs through an operation-ledger row (unique logical
      action, receipt persisted before advancing, post-submit timeout → `unknown` → lookup or park)

## Failure modes and handoff
- The real question is whether this should be async/distributed at all → `architecture` owns that
  boundary decision; this ref implements once the decision is made.
- The work is calling OUT to a third party/webhook/AI/email service → `service-integration` (reuses
  the retry/idempotency/DLQ mechanics defined above — point, don't duplicate).
- The work is streaming analytics/OLAP off the event stream → `data`.
- The work is deploying/scaling/monitoring the broker or queue → `operate`.
- SQL mechanics for the jobs/outbox table (indexes, isolation level, pooling) → `persistence`.
