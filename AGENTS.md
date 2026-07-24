# Agent guide — backend-skill

This repository is the released `backend` skill: the server-side half of the Build stage of the
Digital Product Skills family, peer of `frontend`. The runtime skill lives under `skills/backend/`;
repository and plugin names use the `-skill` suffix. Version metadata lives in
`.codex-plugin/plugin.json`, `CHANGELOG.md`, and tags — never in `SKILL.md`.

## What this skill owns

The *running server* behind the contract: API/handler implementation (REST/GraphQL/gRPC/tRPC),
domain/service-layer logic, OLTP persistence (schema, migrations, transactions, pooling), auth
implementation (sessions/JWT, OAuth2/OIDC, RBAC/ABAC/ReBAC), async work and messaging
(queues/outbox/saga/idempotency/durable execution), caching and rate limiting, implementing a threat
model's controls (OWASP), outbound service integration (clients/webhooks), and the tests that prove
it runs — across REST-CRUD, serverless/edge, realtime, event-driven-microservices, and agentic
backends in Node/Python/Go/Rust/JVM. It sits between `architecture` (which decides the boundaries,
contract shape, NFR budgets, and threat model) and `operate`/`quality` (which deploy, monitor, and
independently verify). It owns the implementation craft, not the decision behind it. It recommends
siblings (`architecture`, `frontend`, `data`, `ai`, `quality`, `operate`) for work it does not own
and never silently invokes them.

## Maintainer rules

1. Run `scripts/check-sync` before any release; it validates packaging, routes, references,
   metadata, and eval fixtures. Run `scripts/lint-skill` and `scripts/count-skill-tokens` too.
2. `SKILL.md` is the direct router: a faceted table of one primary job × at most one base surface
   (the agentic overlay stacks additively). References are flat under `references/`, directly
   linked, self-describing, and loaded only when their observable condition applies. No required
   second-hop navigation.
3. Retrieval-first is load-bearing, not a nit: never generate framework/ORM/DB-specific code without
   fetching the current official docs/types and version-gating idioms to the target release —
   backends ship breaking changes fast (Prisma 7, Drizzle 1.0, Go 1.22 routing, Kafka 4.0
   KRaft-only).
4. Every method must be grounded in its named source with a citation in `SOURCES.md`. Fix a wrong
   formula, a misattribution, or a license-posture error (including a datastore product's
   self-host/resale terms) as a correctness bug, not a nit.
5. Keep the boundary discipline: reference sibling territory by backtick name and recommend it;
   never absorb architecture/frontend/data/ai/quality/operate work.
6. Keep the eval contract current: `evals/{activation,traversal,output,compression-ablation}` must
   reflect the router. Add near-miss activation cases when a new sibling overlap appears.
7. Treat README as the primary storefront. The `site/` and `remotion/` profiles follow the
   `site/README.md` design brief — designed for this skill's faceted-router model inside the shared
   family chrome and its own emerald accent, never a filled-in skeleton.
8. Release through matching plugin version + changelog entry + `v<version>` tag + GitHub Release,
   and keep the GitHub About area configured (family-style description, canonical site alias as
   homepage, base + domain topics).

Do not mark a change complete merely because lint passes. Activation, traversal, and output
evaluations must show the skill still changes agent behavior usefully.
