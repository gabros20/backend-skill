# Zero-Downtime Migration Checklist

Fillable checklist + file skeleton for an expand/contract migration against a live table.
Background and the full migration-tooling vocabulary:
[persistence.md](../references/persistence.md) §4. Delete a phase's rows that don't apply (e.g. no
backfill needed).

## Phase 1 — Expand (ship first, additive only)

| Step | Done | Notes |
|---|---|---|
| New column/table added `NULL`-able or with a safe `DEFAULT` | | never `NOT NULL` with no default on a live table |
| Application deployed to **dual-write** old + new shape | | old readers still work unchanged |
| New index created `CONCURRENTLY` (Postgres) if the table is large/live | | avoids the table-lock a plain `CREATE INDEX` takes |
| Verified: old code path still passes with the schema change present | | this deploy must be a no-op for existing readers |

## Phase 2 — Backfill

| Step | Done | Notes |
|---|---|---|
| Backfill run in **batches**, not one statement | | bounds lock time and replication lag |
| Batch size + throttle chosen against measured load | | e.g. 1,000–10,000 rows/batch, paused between batches |
| Backfill is idempotent / resumable | | safe to re-run after a partial failure |
| Verified: new column/table fully populated, spot-checked against source | | |

## Phase 3 — Contract (ship after backfill confirmed complete)

| Step | Done | Notes |
|---|---|---|
| Application deployed to **read from the new shape** | | old shape still present, unused |
| Verified in production for a burn-in period before dropping anything | | length depends on traffic patterns — state it explicitly |
| Old column/table/constraint dropped in a **separate, later deploy** | | never in the same deploy as the read cutover |
| Dropped-column `NOT NULL`/constraint added only after 100% backfill confirmed | | |

**Never** rename or drop a column the running service still reads in the same deploy as the code
change — that is the single failure mode this whole checklist exists to prevent.

---

## Plain-SQL-file + journal shape

Regardless of ORM, keep migrations as reviewable SQL files plus a lightweight manifest — not an
opaque binary/state-only format (Drizzle's `_journal.json` and Prisma's migration folders both
follow this shape; see [persistence.md](../references/persistence.md) §4).

```
migrations/
  0001_create_users/
    migration.sql
  0002_add_users_email_verified/
    migration.sql
  meta/
    _journal.json
```

`migration.sql` — one file per migration, plain SQL, hand-editable:

```sql
-- migration: 0002_add_users_email_verified
-- expand phase — additive only, safe on a live table

ALTER TABLE users ADD COLUMN email_verified boolean NOT NULL DEFAULT false;
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_email_verified ON users (email_verified);
```

`meta/_journal.json` — manifest the runtime migrator reads to know what's applied and in what
order:

```json
{
  "version": "1",
  "entries": [
    { "idx": 0, "tag": "0001_create_users", "breakpoints": true },
    { "idx": 1, "tag": "0002_add_users_email_verified", "breakpoints": true }
  ]
}
```

Fill in per-migration: `idx` (sequence order), `tag` (matches the folder/file name), and whether
the file contains multiple statements needing a `-- > statement-breakpoint` split (each statement
between breakpoints runs as its own executed unit).

## Rollback plan (fill in before applying)

| Question | Answer |
|---|---|
| Can this migration be reverted with a corresponding down-migration? | |
| If not reversible (e.g. a drop), what is the recovery path (backup restore, re-derive from source)? | |
| Who/what is notified if the migration fails mid-batch? | |
