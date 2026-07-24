# Caching and performance

Purpose: Pick a server-side caching strategy by staleness/loss tolerance (not habit), invalidate
it correctly, implement distributed-correct rate limiting, and tune query/pooling performance —
concrete picks, not a pattern catalogue.

Read when:
- The request adds a cache in front of a DB/service, needs cache invalidation, implements rate
  limiting, or diagnoses a slow endpoint/query.
- The ask is "add Redis" without a stated staleness/loss tolerance — surface that gap first.

Skip when:
- The schema/index/N+1/pooling-depth work itself isn't done yet — → [persistence.md](persistence.md);
  this ref assumes the underlying query is already reasonable.
- The ask is a CDN/edge cache or static-asset caching — → `operate`.
- The ask is an independent load test to *prove* a perf claim — → `quality`.

Inputs:
- Staleness tolerance (how long can a stale read be served) and loss tolerance (can a write be
  lost on crash) for the specific data — state explicitly when unmeasured, never default to
  "cache everything."
- Whether the target is single-instance or distributed (decides rate-limiter implementation).
- The current query/endpoint's actual measured latency, if diagnosing — state "not yet measured"
  rather than guessing a bottleneck.

Produces:
- A cache-strategy pick per data path with its invalidation mechanism named.
- A rate-limiting implementation correct under concurrent/distributed access.
- A named perf bottleneck (query plan, pool exhaustion, missing cache) with its fix, not a
  generic "add caching" recommendation.

## Contents
- Cache strategies — pick by staleness/loss tolerance
- Invalidation
- Rate limiting — token bucket, distributed-correct
- Query and perf tuning
- Pooling as perf

## Procedure

### 1. Cache strategies — pick by staleness/loss tolerance
Four shapes, chosen by what a stale read or a lost write actually costs on this data path:

| Strategy | Mechanism | Staleness | Write-loss risk | Pick when |
|---|---|---|---|---|
| **Cache-aside** (lazy load) | App checks cache; on miss, reads DB, populates cache, returns | Bounded by TTL | None — DB is always the write path | Default for read-heavy, tolerant-of-a-TTL-window data |
| **Write-through** | App writes to cache and DB synchronously, same request | None (cache never stale) | None | Reads must never see stale data after a write; accept the extra write latency |
| **Write-behind** (write-back) | App writes to cache; a background process flushes to DB async | None on cache reads | **Real** — a crash before flush loses the write | High write throughput where eventual DB durability is acceptable (metrics, counters) — never for the write of record on money |
| **Refresh-ahead** | Cache proactively re-fetches hot keys before TTL expiry | Near-zero for hot keys | None | Known hot keys with predictable access, avoiding a stampede at expiry |

Default to **cache-aside** unless a specific staleness/loss constraint rules it out — it's the
simplest failure mode (cache down = fall through to DB, degraded not broken).

### 2. Invalidation
"There are only two hard things in Computer Science: cache invalidation and naming things" is a
cliché because the failure mode is real: a cache that's never invalidated silently serves stale
data forever. Mechanisms, in order of preference: **TTL** (simplest, bounds the staleness window
even if invalidation logic has a bug — set it even alongside explicit invalidation as a backstop);
**explicit invalidation on write** (delete or update the key in the same transaction/request that
changes the source of truth — the correct default for write-through/cache-aside); **event-driven
invalidation** (a downstream consumer invalidates on a change event — needed when the writer and
the cache aren't in the same process/request, see `async-and-messaging` for the delivery
mechanics). **Stampede protection**: when a hot key expires, many concurrent requests miss
simultaneously and all hit the DB at once — mitigate with refresh-ahead for known-hot keys, or a
per-key lock/single-flight so only one request repopulates while others wait on it.

### 3. Rate limiting — token bucket, distributed-correct
**Token bucket** is the default algorithm: each caller has a bucket refilling at a fixed rate up to
a burst capacity; each request consumes a token, rejected when empty — smooths bursts better than
a fixed window while staying simple to reason about (web-canon research corpus, backend security
controls). Expose `X-RateLimit-Limit`/`-Remaining`/`-Reset` so clients can back off correctly
rather than blind-retry.
**Single-instance**: an in-memory bucket per key is correct and cheap.
**Distributed** (multiple app instances behind a load balancer): an in-memory bucket per instance
under-counts total traffic — the limit is only enforced per-instance, not globally. Use
**Redis + a Lua script** so the check-and-decrement is atomic in one round trip (`EVAL`/`EVALSHA`)
— without Lua, a naive `GET`-then-`SET` from the app has a race window where concurrent requests
both read the same remaining count before either writes the decrement, letting the limit be
exceeded under load. This is the correctness reason Redis+Lua is the standard distributed
implementation, not just a performance optimization.
**Throttle by the identity that matters** — account/API-key, not source IP alone (IPs are shared
behind NAT/corporate proxies and spoofable); apply this same "throttle by account, not IP" rule to
auth endpoints specifically (owned in detail by `auth`, restated here because rate limiting is the
enforcement mechanism).

### 4. Query and perf tuning
Diagnose before fixing — `EXPLAIN ANALYZE` (Postgres) or the DB's native equivalent before
guessing at an index or a cache. Common causes in priority order: missing index on a filtered/
joined column (see [persistence.md](persistence.md) §3 for the FK-index trap specifically), N+1
queries (see [persistence.md](persistence.md) §7), an unbounded result set (missing `LIMIT`/
pagination), a large `OFFSET` on deep pagination (scans and discards every prior row — prefer
cursor/keyset pagination for high-write collections), and connection overhead from opening a fresh
connection per request instead of pooling. Add a cache only after the query itself is reasonable —
caching a bad query just moves the same cost to a cache-refill stampede later.

### 5. Pooling as perf
Connection pooling is itself a performance lever, not just a serverless-correctness fix: opening a
raw connection costs real time (TCP + TLS + auth handshake) that a pool amortizes away. Sizing and
mode selection (PgBouncer session/transaction/statement, serverless HTTP-driver split) is owned in
depth by [persistence.md](persistence.md) §6 — this ref points there rather than duplicating it.

## Validation
- Every cache has a named invalidation mechanism (TTL at minimum) — no cache invalidates "never."
- The cache-strategy pick states the staleness/loss tolerance it was chosen against, not asserted
  as default.
- Rate limiting is Redis+Lua-atomic (or equivalent single-round-trip atomic check) whenever more
  than one app instance can see the same key — not a per-instance-only in-memory bucket.
- A perf fix names its measured bottleneck (query plan, N+1, pool exhaustion) before proposing a
  cache.

## Failure modes and handoff
- **"Add a cache" with no stated staleness/loss tolerance** — surface the gap; don't default
  silently to cache-aside without naming the tradeoff.
- **Rate limiting on a multi-instance deployment implemented as in-memory-only** — flag the
  under-counting risk; move to Redis+Lua.
- **The real bottleneck is the schema/index/pooling, not the cache** — route to
  [persistence.md](persistence.md).
- **The ask is CDN/edge/static-asset caching** — route to `operate`.
- **A perf claim needs independent load-test proof** — route to `quality`.
