# Service Integration

Purpose: Turn a request to call OUT of the system — HTTP/gRPC clients, inbound/outbound webhooks, a
third-party or AI or transactional-email integration — into a resilient integration built on the
reliability primitives owned by `async-and-messaging`, plus the outbound-specific mechanics owned
here (circuit breakers, webhook verification and delivery).

Read when:
- The request is calling a third-party API, receiving or delivering webhooks, a gRPC/HTTP client to
  another service, tuning a circuit breaker, or integrating a payment/AI/email provider.

Skip when:
- The work is internal background jobs, a broker, CDC, or a saga across the system's own services →
  `async-and-messaging` (owns retry/idempotency/DLQ mechanics — this ref reuses them, doesn't
  redefine them).
- The work is building the LLM/agent's own reasoning core, tool loop, or orchestration → `ai`. A
  single request/response call to a model or an existing agent endpoint is in scope here as "calling
  a third-party service"; the agent's internals are not.
- The work is deploying or monitoring the integration (uptime, alerting on breaker trips) → `operate`.
- The work is the auth handshake for the third-party call (OAuth2 client-credentials, API-key
  rotation) → `auth`.
- The work is a **multi-vendor business process** rather than one integration — reconciling against a
  vendor's own state to catch what a webhook never delivered, managing credentials across a fleet of
  vendors, a human approval node, or cross-system sync — → `automation`. One outbound call and its
  breaker are in scope here; the process wrapped around several of them is not.

Inputs:
- The external API's documented contract, rate limits, and webhook-signing scheme. State explicitly
  when any of these is unverified — never invent endpoint behavior or assume a signing scheme.

Produces:
- A client wrapper with retry + circuit breaker configured, and/or a webhook handler (verified
  inbound processing, retried outbound delivery) with idempotent semantics.

## Contents
- Retries at the client boundary (reuse, don't redefine)
- Circuit breaker (closed/open/half-open)
- Webhooks inbound (verification, idempotent handling)
- Webhooks outbound (signing, delivery, retry)
- Calling a third-party / AI / email service (incl. a model gateway)

## Procedure

### 1. Retries — reuse `async-and-messaging`, add one rule
Backoff+jitter mechanics (Full Jitter as default) and DLQ/idempotency-key mechanics are owned by
`async-and-messaging` — apply them here rather than re-deriving them. The one rule specific to
outbound calls: only retry idempotent/safe operations (GET, or a POST guarded by an idempotency key
honored on *both* sides). Retrying a non-idempotent POST without a shared idempotency key risks a
duplicate side effect on the remote system that you cannot undo from your side.

### 2. Circuit breaker — **owned here**
A stateful wrapper around a remote call with three states:
- **Closed** — calls pass through normally; failures are counted.
- **Open** — after a failure threshold, calls fail fast without hitting the remote at all, protecting
  the caller's own resources (threads/connections) from a slow or down downstream.
- **Half-Open** — after a cooldown, a trial request checks recovery: success closes the breaker,
  failure reopens it.

Practical shape: a split-phase API — `allow() -> bool` then `report(success: bool)` — so the call
site decides whether a given attempt counts toward the breaker's state, isolating one unhealthy
downstream from starving an entire outbound queue or worker pool. A production reference trips after
5 consecutive failures and leaves the failed task queued for later retry rather than dropping it.

Tuning is genuinely hard, not a set-and-forget default: too aggressive → false trips on transient
blips; too lax → cascading latency/failure propagates through every caller of the breaker. Start
around a 5-consecutive-failure threshold with a 10-60s cooldown, then tune from observed incident
data. Always combine with retry+jitter: the breaker's job is to stop retries entirely once a
dependency is confirmed down, rather than continuing to jitter-retry against a fully offline service.

### 3. Webhooks — inbound (receiving)
- Verify the signature (HMAC over the raw request body + shared secret, with a timestamp window to
  reject replays) before trusting the payload at all — never process an unverified webhook body.
- Treat every inbound delivery as at-least-once: providers retry on non-2xx responses or timeouts, so
  the handler MUST be idempotent — dedupe by the provider's event ID using the idempotency-key
  mechanics from `async-and-messaging` §2 (persist processed event IDs with a TTL, short-circuit
  replays).
- Respond 2xx fast; do the actual work asynchronously (enqueue a job) rather than processing inline
  and risking a provider-side timeout that triggers a retry storm on top of work already in flight.

### 4. Webhooks — outbound (delivering to someone else's endpoint)
- Sign your own payloads (HMAC) so receivers can verify authenticity — the same discipline expected
  as a receiver in §3.
- Deliver with the same retry+jitter+DLQ discipline as any external call (§1): a customer endpoint
  being down should never block your process — queue the delivery attempt and back off instead of
  blocking or dropping it.
- Version the payload shape explicitly (a field or header) since receivers cannot be forced to deploy
  in lockstep with changes on your side.

### 5. Calling a third-party / AI / transactional-email service
- Wrap as any outbound client (§1-2). Respect the provider's documented rate limits: honor
  `Retry-After`/rate-limit-remaining headers, prefer webhooks over polling where offered, and use
  conditional requests (`ETag`/`If-None-Match`) for free "not modified" responses that don't count
  against quota — general consumer-side hygiene for any well-behaved external API client, not tied to
  one vendor.
- Calling an LLM or an existing agent endpoint is in scope here exactly like calling any other
  third-party service (retry/circuit-breaker/rate-limit discipline applies identically); building
  that agent's reasoning core or tool-orchestration loop is `ai`'s job — point there, don't absorb it.
- **A model gateway is substrate, not policy.** Three products wear the name — a multi-provider
  marketplace (one account and bill over many providers), a proxy/policy gateway (routing, keys,
  quotas, logs over your own provider accounts), a hosted-inference catalog (models served on the
  platform) — name which one you are wiring. OpenAI-compatible is not feature parity: tool calling,
  structured output, caching, streaming events, and cancellation differ per provider, and a
  portability layer may reject or silently drop an unsupported parameter (LiteLLM raises by default,
  drops with `drop_params`) — test the features you use, per route. Gateways retry, cool down, and
  fall back on their own (LiteLLM Router): that is a retry layer, counted in
  `async-and-messaging` §6's budget, with exactly one layer owning retries. Set credential fallback
  explicitly — e.g. as of 2026-10-17 Vercel AI Gateway retries a failed bring-your-own-key request
  on its system credentials and bills your gateway credits, a cost and data-policy choice rather
  than a default to inherit. A gateway picks an inference route; it never grants tool permission,
  tenant access, or action authorization. Which model a request should get is `ai`'s
  (`model-selection-and-routing.md`).
- Transactional email/SMS providers get the same client-resilience treatment; most also webhook-notify
  on delivery/bounce/complaint — that inbound notification goes back through §3.

## Validation
- [ ] Every outbound client call has a bounded, jittered retry — no unbounded retry loops
- [ ] A circuit breaker guards any outbound call that could exhaust a shared pool/thread budget on
      caller failure
- [ ] Inbound webhook signatures are verified before the payload is trusted
- [ ] Inbound webhook handlers are idempotent (dedupe by provider event ID) and ack fast, deferring
      real work to a queue
- [ ] Outbound webhook deliveries are queued and retried, not fired synchronously and dropped on
      first failure
- [ ] Rate-limit headers from the external API are respected proactively, not just caught as errors
      after the fact
- [ ] A model gateway's own retries/fallbacks are counted as one retry layer, the features in use are
      tested per route, and its credential-fallback behavior is set explicitly

## Failure modes and handoff
- The request is actually about internal jobs/broker/CDC/saga, not an external system →
  `async-and-messaging` (owns the retry/idempotency/DLQ mechanics this ref reuses).
- The request is building the LLM/agent's own reasoning/tool-orchestration core → `ai`.
- The request is deploying or monitoring the integration (uptime, alerting on breaker trips) →
  `operate`.
- The request needs the auth handshake for the third-party call itself → `auth`.
