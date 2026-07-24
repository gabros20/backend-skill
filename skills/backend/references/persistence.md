# Persistence

Purpose: Decide and implement the OLTP data-access layer — SQL vs ORM, schema/index design,
migrations, transaction/isolation correctness, connection pooling, and how far to push a single
Postgres before reaching for a specialized store — concrete picks with named tradeoffs.

Read when:
- The request touches schema design, an ORM/query-builder choice, a migration, a transaction that
  must not corrupt money/inventory, pool sizing, N+1 symptoms, or "can Postgres do X instead of a
  new store."
- A DB product choice needs its license implications named (managed-hosting/SaaS context).

Skip when:
- The store/consistency-model *decision* (CQRS/ES, CAP/PACELC, polyglot-persistence restraint,
  identifier scheme) isn't settled — → `architecture`'s `data-architecture.md`.
- The ask is caching, rate limiting, or perf tuning once the schema is right — →
  [caching-and-performance.md](caching-and-performance.md).
- The ask is an OLAP pipeline/warehouse/dbt — → `data`.
- The ask is idempotency keys, outbox, saga, or durable execution for cross-service writes — →
  `async-and-messaging`.

Inputs:
- Target DB engine/version and the ORM already in use — state explicitly when greenfield, never
  assume Postgres+Drizzle without confirming.
- The operation's money/uniqueness sensitivity (decides isolation strategy) — state explicitly
  when unknown rather than guessing "low-stakes."
- Deployment target (long-running server vs serverless/edge) — decides pooling strategy.

Produces:
- A schema + index design, or a migration (or
  [migration-template.md](../assets/migration-template.md) filled in) reviewable as plain SQL.
- A transaction/isolation-level decision named for the specific code path.
- A pooling configuration matched to the deployment target.
- Any DB-product license flag surfaced before a hosting/managed-service decision.

## Contents
- SQL-before-ORM
- Drizzle vs Prisma — teach both
- Schema + index design
- Migrations: the complete vocabulary
- Transactions + isolation levels
- Connection pooling
- N+1
- "Postgres for everything" — and its traps
- DB product licenses

## Procedure

### 1. SQL-before-ORM
Know the SQL an ORM would generate before hiding behind it — a bad join plan or N+1 is invisible
until you can read `EXPLAIN ANALYZE`. Raw SQL (or a thin builder — sqlc, Drizzle's `.select()`)
for hand-tuned/hot-path queries; the ORM's convenience layer for standard CRUD. This ordering, not
"ORM vs no ORM," is the actual decision.

### 2. Drizzle vs Prisma — teach both, no winner
Both Apache-2.0. Genuine, unresolved practitioner split
([encore.dev](https://encore.dev/articles/drizzle-vs-prisma); discourse incl.
[DrizzleORM](https://x.com/DrizzleORM/status/2071716332924465239),
[CodeHagen](https://x.com/CodeHagen/status/1815684730597114027)):

| | Drizzle | Prisma |
|---|---|---|
| Architecture | Thin SQL builder, direct driver calls, zero engine process | Schema DSL → generated client → engine (TS/WASM in **Prisma 7**, was a Rust binary) |
| Type safety | Inferred from TS table defs, no codegen | `prisma generate` codegen; polished autocomplete + Prisma Studio |
| Query style | `db.select().from(t).where(eq(...))` — SQL-shaped | `prisma.user.findMany({ where, include })` — higher abstraction |
| Bundle/cold-start | ~7.4kB gzip, 0 deps — edge/serverless-best | Prisma 7 cut ~14MB→~1.6MB; driver adapters (`adapter-pg`/`-neon`/`-d1`) close the gap |
| Migrations | Plain SQL + `_journal.json`, hand-editable | Managed: interactive prompts, shadow-DB diffing |
| Pick when | SQL-literate team, edge target, greenfield 2026 | Max abstraction, rapid CRUD, existing Prisma codebase |

TypeORM: maintenance mode — no reason to pick new, legacy risk only. Python: Django ORM if already
on Django, else SQLAlchemy+Alembic. Go: GORM (convention) vs sqlc (compile-generated from hand SQL,
best type-safety on hot paths). **Retrieval-first**: fetch current docs before generating — Prisma
7's `prisma.config.ts` replaces `.env`-driven config; Drizzle's 0.x→1.0 `relations()`→
`defineRelations()` fork is **incompatible, do not mix**.

### 3. Schema + index design
- `BIGINT GENERATED ALWAYS AS IDENTITY` for single-writer tables (best index locality); UUIDv7
  when IDs must be client-generated without coordination (timestamp-prefixed, keeps B-tree
  locality UUIDv4 destroys); UUIDv4 only for real opacity needs. Full scheme comparison lives in
  `architecture`'s `data-architecture.md` — this ref applies the pick.
- **Postgres does not auto-index foreign keys** — the single most-repeated trap in this research;
  add the index explicitly.
- `timestamptz`, always — naive timestamps are a silent timezone bug.
- JSONB: `jsonb_ops` (default GIN) vs `jsonb_path_ops` (smaller/faster, containment-only, no
  `?`/`?|`/`?&`) — pick by which operators the queries use.
- Range-partition past ~100M rows when old data drops wholesale (`DROP TABLE` beats row `DELETE`).

### 4. Migrations: the complete vocabulary
Teach **Prisma's verb taxonomy** regardless of ORM — the most complete public vocabulary for what
migration tooling needs to do (Apache-2.0, [prisma/prisma](https://github.com/prisma/prisma)):

| Verb | Does | Use |
|---|---|---|
| `migrate dev` | Generate+apply+regen client, interactive | Local dev only |
| `migrate deploy` | Apply-only, non-interactive | CI/CD, production |
| `migrate diff` | Diff schemas without applying | Review before merge |
| `migrate reset` | Drop+recreate+reseed | Local/test only |
| `migrate resolve` | Mark applied/rolled-back without running | Hotfix a broken history |
| `migrate status` | Report pending/applied state | CI gate |
| `db pull`/`push`/`seed` | Introspect / apply directly, no file / seed | Early-stage, one-off |

**Drizzle's journal**: `drizzle-kit generate` writes timestamped SQL files + a `meta/_journal.json`
manifest (idx/timestamp/tag/breakpoints); the runtime migrator replays them, splitting on
`-- > statement-breakpoint` — reviewable, no proprietary format ([drizzle-team/drizzle-
orm](https://github.com/drizzle-team/drizzle-orm), Apache-2.0). Same generate/push/migrate/
introspect vocabulary, ORM-agnostic. **Peewee's imperative fallback**: no diffing tool → hand-
compose ops (`add_column`, `rename_column`, `drop_column`, `add_not_null`) via a `migrator`,
wrapped in `db.transaction()` ([coleifer/peewee](https://github.com/coleifer/peewee), MIT) — the
model for any stack without mature schema-diffing.

**Zero-downtime = expand/contract, always**: (1) expand — add the new column/table nullable or
defaulted, dual-write old+new; (2) backfill in batches; (3) contract — flip reads, then drop the
old column/constraint in a later deploy. Never rename/drop a column the running service still
reads in the same deploy as the code change. Never run `push`/`db push` against production — no
migration file, no history, no rollback. Fillable checklist:
[migration-template.md](../assets/migration-template.md).

### 5. Transactions + isolation levels
SQL-92, strictest→loosest: **Serializable → Repeatable Read → Read Committed → Read
Uncommitted**. Postgres (and most DBs) default to **Read Committed** — each statement sees only
what committed before *that statement* started; non-repeatable reads across statements in the same
transaction are possible. Default: Read Committed for general CRUD; escalate around money/
inventory/uniqueness paths — **Serializable** (no dirty/non-repeatable/phantom reads, throughput
cost; Postgres and CockroachDB both support it), **`SELECT ... FOR UPDATE`** (pessimistic row
lock, simplest to reason about), or an **optimistic version column** (`WHERE version = $expected`,
retry on 0-row update — no lock held, needs a retry loop) ([postgresql.org transaction-
iso](https://www.postgresql.org/docs/current/transaction-iso.html)).
**Portability trap**: MySQL/InnoDB defaults to **Repeatable Read**, not Read Committed — code
assuming Postgres's default behaves differently on MySQL; state the target DB's default
explicitly, never assume.

### 6. Connection pooling
**PgBouncer** modes: session (one connection per client session — safest, preserves prepared
statements/temp tables/`LISTEN`/advisory locks/`SET`), transaction (returned to the pool between
transactions — 1.21+ added prepared-statement support here), statement (rarest, most restrictive)
([pgbouncer.org](https://www.pgbouncer.org/features.html)). `pool_size ≈ cores × 2 + spindles` as a
starting formula, tuned against measured load. **Serverless**: a persistent TCP pool doesn't
survive isolated Lambda-style invocations — use an HTTP-native driver (`@neondatabase/serverless`,
Prisma's `adapter-pg` over an HTTP-capable driver) for cold/isolated runtimes; for warm/shared runtimes (Fluid compute,
long-lived containers) a pool created **once at module scope** is correct — this warm-vs-cold
split is the sharpest recurring serverless-Postgres failure mode across vendor docs. `-pooler`
hostname suffix (Neon/Supabase) or provider-native pooling is close to mandatory once instance
count exceeds what raw per-connection Postgres processes can hold.

### 7. N+1
Symptom: one query for a list, then one query per row for related data. Fix by loading the
relation in the same round trip — a `JOIN`/`include`/`with: { relation: true }`, or one batched
`WHERE id IN (...)`, never a loop issuing one query per parent row. Ecto (Elixir) avoids this class
by design (no hidden lazy loading); ActiveRecord and DataLoader-less GraphQL resolvers are the
most common places it ships silently — check any loop over a collection for a query call inside.

### 8. "Postgres for everything" — and its traps
Thesis ([postgresforeverything.com](https://postgresforeverything.com/)): one well-run Postgres
replaces a separate vector DB, queue, search engine, and time-series DB for most apps — trading a
poly-DB consistency/ops burden for one system. Corroboration: OpenAI runs ChatGPT (~800M users) on
one Postgres primary + ~50 read replicas (OpenAI eng blog, via
[arpit_bhayani](https://x.com/arpit_bhayani/status/2074509051686445362)).
- **pgvector**: HNSW/IVFFlat, production-stable; start here under ~500K vectors — one SQL
  statement joins vectors and source rows, no dual-write problem. Graduate to Qdrant/Milvus only at
  genuine ANN scale or rich payload filtering at low latency.
- **Jobs-as-queue**: `SELECT ... FOR UPDATE SKIP LOCKED` on a jobs table for atomic claim-and-
  process — the production-safe pattern, not `LISTEN/NOTIFY` alone.
- **`LISTEN`/`NOTIFY` traps**: notifications are lost if a listener is disconnected when sent;
  every `NOTIFY`-carrying commit takes a lock that serializes commits (a fix is expected in a
  future Postgres release — verify against the target version); it **breaks under PgBouncer
  transaction-mode pooling**. Use only to wake idle workers
  polling the SKIP LOCKED table, never as the durable delivery mechanism itself.
- **Full-text**: built-in `tsvector`/`tsquery`/GIN is often enough; an extension like ParadeDB's
  `pg_search` (formerly `pg_bm25`) adds BM25 ranking inside Postgres for closer-to-dedicated-search
  relevance without a second system.
- Graduate off Postgres-for-everything only against a named failure (see `architecture`'s
  polyglot-persistence restraint rule) — not novelty.

### 9. DB product licenses
Skill content, a value-add — not copied text. Verify current terms before a hard claim; teach the
*shape*:

| Model | Meaning | Examples |
|---|---|---|
| **SSPL** | Forces any cloud provider offering it "as a service" to open-source their whole surrounding stack — not OSI-approved | MongoDB (since 2018), one of Redis 8's license options |
| **BSL** | Source-available, free to self-host, converts to Apache 2.0 after ~3–4 years; stops competitors reselling as managed service | CockroachDB, Memgraph; TimescaleDB's advanced features (TSL — **bans managed hosting**, self-host free) |
| **AGPLv3** | OSI-approved copyleft; network use triggers source-disclosure | Redis 8 (one of three options) |
| **Source-available (other)** | Vendor-specific, not OSI-approved | ScyllaDB, Elastic (one of three post-2024 options), Typesense (GPLv3 — copyleft) |

**Permissive escapes**: **Valkey** (BSD-3, Linux Foundation fork of Redis 7.2.4, AWS/Google/
Oracle-backed) for Redis's SSPL; **OpenSearch** (Apache 2.0, Amazon fork of Elasticsearch 7.10.2)
for Elasticsearch's SSPL/ELv2; **YugabyteDB** (Apache 2.0) for CockroachDB's BSL when Postgres-
wire-compatible NewSQL is needed without the constraint; **Meilisearch** (MIT) vs Typesense's
GPLv3. Flag the license explicitly for any hosted/managed-service or SaaS-resale context — exactly
what these clauses target. **PostgreSQL, MySQL Community, MariaDB core, DuckDB, SQLite** are
permissive/public-domain, no such trap.

## Validation
- The ORM/query-builder pick names its tradeoff against the alternative, not just "we use X."
- Every migration is expand/contract-safe for a live table, or explicitly scoped to pre-launch/
  local/test.
- The isolation level for any money/inventory/uniqueness path is named explicitly, not left at
  the driver default.
- Pooling strategy matches the actual deployment target (warm long-running vs cold serverless).
- Any DB product recommended for a hosted/managed-service context carries its license flag.
- No N+1 pattern (query inside a loop over a collection) ships unreviewed.

## Failure modes and handoff
- **Schema/consistency-model decision still open** — route to `architecture`'s `data-
  architecture.md` before writing DDL.
- **Real ask is cache invalidation or rate limiting, not the schema** — route to
  [caching-and-performance.md](caching-and-performance.md).
- **Cross-service write needs atomicity with an event publish** — route to `async-and-messaging`
  for outbox/saga; this ref owns only the local transaction.
- **A DB product license conflicts with the deployment plan** (e.g., reselling MongoDB/
  TimescaleDB-TSL as a managed service) — stop and surface the conflict rather than proceeding.
- **Version-sensitive ORM/DB syntax** (Prisma 7, Drizzle 1.0, Postgres 19) — fetch current docs
  before generating; don't rely on a stale idiom.
