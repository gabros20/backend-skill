# Idempotency-Key and Transactional-Outbox Recipes

Fillable skeletons — copy into the target codebase and fill the bracketed parts. Mechanics and
rationale: `references/async-and-messaging.md` (§2 idempotency, §3 outbox).

## Recipe 1 — Idempotency-Key handler

1. Client generates one unique token per logical operation (UUID or opaque string, ≤255 chars) and
   sends it as `[header name, e.g. Idempotency-Key]`.
2. Server, before executing the operation:
   - Look up `(account_id, idempotency_key)` in the dedupe store.
   - Found + status = completed → return the stored response verbatim; do NOT re-execute.
   - Found + status = in_progress → `[choose: reject 409 "in progress" | block until resolved]`.
   - Not found → insert a row with status=in_progress, then proceed.
3. Execute the operation `[in the same transaction as the dedupe-row insert, if the DB supports it —
   otherwise document the small race window]`.
4. On success: persist the response payload against the key, status=completed,
   TTL = `[24h | 30d | project default]`.
5. On failure: `[choose: delete the row so a retry re-executes | persist the failure and replay it
   too]`.

Dedupe table skeleton:

```sql
create table idempotency_key (
  account_id       [type],
  idempotency_key  varchar(255),
  status           varchar(20) not null,   -- in_progress | completed
  response_body    jsonb,
  created_at       timestamptz not null default now(),
  primary key (account_id, idempotency_key)
);
-- scheduled cleanup: delete where created_at < now() - interval '[window]'
```

## Recipe 2 — Transactional outbox

1. Define the outbox table in the same database as the business tables:

```sql
create table outbox_event (
  id            uuid primary key default gen_random_uuid(),
  aggregatetype varchar(255) not null,   -- e.g. 'order'
  aggregateid   varchar(255) not null,   -- e.g. the order id
  type          varchar(255) not null,   -- e.g. 'OrderCreated'
  payload       jsonb not null,
  created_at    timestamptz not null default now()
);
```

2. In the SAME transaction as the business write, insert one row per event to publish:

```sql
begin;
  insert into [business_table] (...) values (...);
  insert into outbox_event (aggregatetype, aggregateid, type, payload)
    values ('[aggregate]', '[id]', '[EventType]', '[payload_json]');
commit;
```

3. Relay — pick one:
   - **Polling publisher**: a worker runs
     `select * from outbox_event order by created_at limit N for update skip locked`,
     publishes each row to the broker/topic, then deletes it (or marks it published). Simple; adds
     polling latency.
   - **Log tailing / CDC**: a CDC tool reads the WAL and streams inserts directly. Lower latency,
     more moving parts to operate.
4. Consumer side: dedupe by `outbox_event.id` (or a derived message key) using Recipe 1's dedupe
   pattern — the relay can crash mid-publish and redeliver the same row.
5. Cleanup: delete the row once publication is confirmed (`remove-after-insert` default) so the table
   never grows unbounded.

## Fill-in checklist
- [ ] Header/token name for the idempotency key decided
- [ ] TTL window decided and a cleanup job scheduled
- [ ] Outbox relay strategy chosen (polling vs CDC) and its latency budget stated
- [ ] Consumer dedupe key decided (event id vs business key)
