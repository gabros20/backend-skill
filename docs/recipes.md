# Recipes

Use the invocation form supported by your client. The examples below use Codex's explicit
`$backend` form.

## Build the endpoints behind a contract

Use when: `architecture` has handed off contracts (or you have an OpenAPI/AsyncAPI spec) and you
need the server implementation, not the design decision.

```text
Use $backend to build the REST endpoints behind this contract [paste OpenAPI spec or handoff.yaml]:
routing, validation, error shape, cursor pagination, and in-code versioning.
```

Route: primary job `api-implementation` (transport pick, request-lifecycle wiring, uniform error
shape) + surface overlay `surface-rest-crud` (default, unless the target is serverless/edge,
realtime, or event-driven). Retrieval-first: fetches the current framework docs for the target
version before generating handler code.

## Add authentication with OAuth2/OIDC

Use when: a product needs login/signup, SSO, or route protection and the auth model itself
(RBAC/ABAC/ReBAC, session vs JWT) hasn't been decided yet.

```text
Use $backend to add login with OAuth2/OIDC and PKCE for [provider]: decide sessions vs JWT, wire
password hashing, and pick the authorization model.
```

Route: primary job `auth` only (no surface overlay needed unless the target is serverless/edge or
realtime, which changes session storage). Produces the session-vs-JWT decision, the OAuth2/OIDC+PKCE
flow (RFC 9700), Argon2id hashing config, and an RBAC/ABAC/ReBAC model + library pick — a concrete
choice plus what NOT to build, not a menu.

## Set up an outbox and saga for a background workflow

Use when: a multi-step workflow needs at-least-once delivery with idempotent consumers, not a bare
queue that can silently double-process or drop a step.

```text
Use $backend to set up the outbox and saga for [workflow, e.g. order fulfillment]: idempotency-key
path, compensations, and the delivery-semantics stance.
```

Route: primary job `async-and-messaging` (broker/queue pick, transactional outbox, saga +
compensations, durable-execution decision — Temporal/Hatchet when the workflow is long-running). Add
`surface-event-driven-microservices` only when the workflow spans independently deployed services.

## Harden the API against the OWASP Top 10

Use when: a threat model's controls (from `architecture`) need implementing, or an API needs a
security pass before shipping.

```text
Use $backend to harden this API against the OWASP Top 10 and ASVS: injection, auth, CORS, secrets,
and security headers — with a self-audit count.
```

Route: primary job `server-security` — produces a uniform control→implementation→verification table
mapped to OWASP Top 10:2021 / ASVS, with every applicable control audited as a row and a self-audit
count shown, never a narrative pass.

## Expose an agentic/MCP backend

Use when: the backend serves an LLM/agent system — tool calls, MCP endpoints, or agent-legible
contracts — on top of whatever base surface the service already runs on.

```text
Use $backend to expose an MCP/tool endpoint for our agent backend on [serverless/realtime/rest]:
durable execution, agent-legible contracts and errors, and short-lived agent-scoped tokens.
```

Route: primary job `api-implementation` (or `async-and-messaging` if the workflow is long-running) +
the matching base surface + `surface-agentic` stacked additively — durable execution as the default
control plane, tool/MCP endpoint shape, agent-legible typed contracts and errors, RAG-retrieval
plumbing that calls `ai`'s core without owning it, and agent-scoped short-lived tokens.
