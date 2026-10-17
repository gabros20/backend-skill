# Changelog

All notable changes to **backend** are documented here.

The release procedure synchronizes `.codex-plugin/plugin.json`, this changelog, git tag
`v<version>`, and the matching GitHub Release. Runtime `SKILL.md` contains no version metadata.

## [Unreleased]

### Added
- **Operation ledger for external effects** (`async-and-messaging` §2 + Recipe 3 in the
  idempotency/outbox asset): one reserved row per irreversible action, unique on logical action +
  resource scope, same downstream key only on a genuine retry, receipt before advancing,
  post-submit timeout → `unknown` → lookup or park, fencing tokens on expiring leases; points to
  `architecture` for the contract and `automation` for vendor reconciliation.
- **One execution owner per side effect** in durable execution — only one layer (workflow engine,
  graph checkpointer, wrapper) retries a given effect. Backend's Postgres-backed-first default is
  unchanged.
- **Replayable run events** (`surface-realtime` §6): run + outbox in one transaction, sequenced
  persisted events, `Last-Event-ID` resume, snapshot on an expired cursor, provisional partial text,
  disconnect never cancels; the cursor arrives as `Last-Event-ID` or an explicit parameter, and a
  cursorless stream gets the snapshot first.
- **Run control** (`surface-agentic` §7): approval as a paused run re-checked and consumed at
  execution, per-run version pins with draining workers, transactional budget reservation that
  children draw from, four lifecycles linked by reference; **MCP/OAuth token rules** in §6
  (audience validation, no token passthrough, secrets never in model context) with a one-line
  echo in `auth`.
- `persistence`: pgvector filtered-search underfill and the RLS bypass by privileged connections.
- `service-integration`: a model gateway as substrate — feature parity per route, its retries as one
  retry layer, explicit credential fallback, never a permission grant; copy provider outputs before
  their retention window closes.
- `surface-serverless-edge`: route permission, billing, and ledger reads through a cache-disabled
  pooler binding; one Durable Object per coordination unit, named from the verified identity.
  `surface-agentic` §6 also binds tool-returned state handles to the verified user and calls
  downstream with a per-audience token (per-user grant or RFC 8693 exchange).
- `surface-agentic` §7: backend owns the sandbox's environment adapter and its controls; provider
  choice, capacity, and teardown monitoring stay with `operate`.
- Router rows, a frontend handoff seam, and evals: 1 positive + 2 near-miss activation cases
  (`automation` reconciliation, `ai` model routing), 2 traversal cases, 2 output cases.

## [0.1.1] — 2026-07-27

### Fixed
- The `implement inside the decision` invariant named `architecture`, `data`, `ai`, `operate`,
  `quality`, and `automation` but never `product`, leaving a backend request that is really a
  product decision — should this endpoint exist, what should it do, its acceptance criteria — with
  nowhere to route. Adds `product` to the same bullet.
- `backend` ceded nothing to `automation` in writing, despite `async-and-messaging` owning
  saga/idempotency/outbox and `service-integration` owning webhooks and breakers that collide with
  vendor-spanning business processes. Adds the boundary to the invariant and to the Skip-when blocks
  of both colliding references: when the participant is opaque, un-scriptable, and partly human,
  the discipline is `automation`'s.

## [0.1.0] — 2026-07-24

### Added
- Initial release of the `backend` skill — the server-side Build stage of the Digital Product Skills
  family, peer of `frontend`. A faceted router over 9 primary jobs (api-implementation, domain-logic,
  persistence, auth, async-and-messaging, caching-and-performance, server-security,
  service-integration, backend-testing) and 5 runtime/system-type surface overlays (rest-crud,
  serverless-edge, realtime, event-driven-microservices, and the additive agentic overlay for
  LLM/agent backends). 15 references, 8 fillable assets (12-factor scaffold, zero-downtime migration
  checklist, auth + OWASP controls checklists, idempotency/outbox recipes, API-contract review,
  verification-loop template, `handoff.yaml`), and activation/traversal/output/compression evals.
  Retrieval-first, default-stack-then-deviate, implement-inside-the-decision; consumes architecture's
  contracts/budgets/threat-model and hands running services + telemetry hooks to `operate`/`quality`.

[0.1.1]: https://github.com/gabros20/backend-skill/releases/tag/v0.1.1
[0.1.0]: https://github.com/gabros20/backend-skill/releases/tag/v0.1.0
