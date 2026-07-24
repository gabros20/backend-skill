# Surface: Agentic backend

Purpose: Reshape the 9 primary-job refs for a backend that serves LLM/agent systems — durable
execution as the control plane, tool/MCP endpoints, agent-legible contracts, RAG plumbing,
datastore best-practice skills, and agent-scoped auth. This surface does not redefine the job refs;
it tells you which knobs in each to turn when the caller is an agent, not a human client.

Read when:
- The system under-build calls an LLM in a loop (tool-calling agent, multi-step agent workflow,
  RAG pipeline) and you're building the server around/under it.
- The request mentions "agent," "MCP server," "tool endpoint," "AI SDK backend," or "expose this
  to an agent/Claude/GPT."
- A human-facing API also needs to be callable by an agent (dual-surface: same backend, two
  clients).

Skip when:
- The request is about the agent's reasoning/prompts/eval loop itself — that's `ai`'s core; this
  surface owns the server that *hosts and serves* the agent, not the agent's brain.
- No agent/LLM caller exists — use `surface-rest-crud` (default) or another surface.
- The realtime transport question (WS/SSE for streaming tokens) is the concern — apply
  `surface-realtime` alongside this one; they compose.

Inputs:
- Which job ref(s) this surface is layered onto (api-implementation, async-and-messaging, auth,
  persistence are the most commonly touched). State explicitly which are in scope; never assume.
- Whether a durable-execution engine is already chosen or must be picked.
- Whether the agent core (model, prompts, tool-selection logic) is owned by this codebase or a
  separate `ai`-scoped service — state explicitly when unclear, don't guess the border.

Produces:
- A backend that (a) survives agent-loop crashes/restarts without hand-rolled state machines,
  (b) exposes typed, agent-legible tool/MCP endpoints, (c) wires retrieval plumbing without owning
  the retrieval logic, and (d) scopes what an agent can do with short-lived, auditable tokens.

## Contents
- Durable execution as the default control plane
- Tool/function endpoints and MCP servers
- Agent-legible contracts and errors
- RAG-retrieval plumbing — own the plumbing, not the core
- Best-practice skills for the datastore
- Agent-scoped auth

## Procedure

### 1. Durable execution as the default control plane, not an afterthought
Treat "the agent loop must survive a crash/restart/deploy" as a default requirement, not an edge
case — this is the single loudest 2025-2026 shift in agent-backend architecture. OpenAI's Agents
SDK integrates **Temporal** for durable execution specifically because long-running,
tool-calling agent loops map onto the workflow/activity split naturally (deterministic
orchestration code that can replay, vs. side-effecting tool calls that must be idempotent or
non-retryable). ThoughtWorks Technology Radar flags **"Ignoring Durability in Agent Workflows"**
as a **Caution** — i.e., shipping an agent loop as an in-memory process with no crash-recovery
story is a named, called-out anti-pattern, not a hypothetical risk.
Do NOT re-derive the workflow/activity split, idempotency-key mechanics, or the Postgres-backed
alternative (Hatchet) here — those live in `async-and-messaging`; this surface only tells you
*when* to reach for them: reach for durable execution whenever an agent loop (a) spans more than
one tool call, (b) can take longer than a single request/response cycle, or (c) has side effects
that must not double-fire on retry. A single-turn, single-tool-call request/response doesn't need
it — don't over-engineer a chatbot FAQ answer into a durable workflow.

### 2. Tool/function endpoints and MCP servers
Two distinct exposure shapes for the same underlying capability, and the request usually tells you
which is needed:
- **Function/tool endpoints** — a typed HTTP (or RPC) endpoint the agent's tool-calling loop
  invokes directly, following the same request-lifecycle discipline as any endpoint in
  `api-implementation` (validation, error shape, idempotency where mutating). The only agentic
  delta: the schema the agent sees (its tool definition) must match the server's actual input
  validation byte-for-byte, or the agent will generate calls that 400 unpredictably.
- **MCP servers** — a standardized protocol surface that exposes a *set* of backend capabilities
  (tools, resources, prompts) to any MCP-capable agent client, not just one caller. Reach for this
  when the backend capability should be reusable across multiple agent clients/products rather than
  wired to one agent loop. Vendors are shipping this as the default integration path for their own
  products: Sanity ships an MCP server to stand up a content backend from a description; Postgres-
  MCP exists specifically for migrations. If the request is "let an agent use this backend," ask
  whether it needs to be MCP-shaped (multi-client, protocol-standard) or a bespoke tool endpoint
  (single caller, simpler) before building either.

### 3. Agent-legible contracts and errors
An agent is a client that reads your API surface as text, not through a hand-built UI — the
contract quality bar goes up, not down. Two converging arguments from the practitioner discourse:
- **Guillermo Rauch's framing**: 2026-era backends should be *agent-legible* — well-typed contracts,
  Unix-style CLIs, `curl`-able endpoints, and documentation in markdown that an agent can read and
  act on directly, rather than requiring a human to translate a Swagger UI into behavior.
- **The Effect argument (Rhys Sullivan)**: a typed-effect system is "more token-efficient for your
  AI" specifically because the type system encodes what errors *can* happen — the agent doesn't
  need to speculatively probe failure modes, they're declared in the contract. Drizzle now ships
  native Effect drivers on the same premise.
Practical rule for this surface: every tool/MCP-exposed endpoint needs (a) a schema that is the
single source of truth (generate the agent's tool definition from it, don't hand-maintain both),
and (b) error responses that name the failure mode in a field the agent can branch on (not just an
HTTP status + free-text message) — legible errors are what let an agent retry correctly instead of
retrying blindly or giving up.

### 4. RAG-retrieval plumbing — own the plumbing, not the core
Hold the border precisely here: retrieval-augmented generation splits into a *core* (prompt
construction, chunking strategy, re-ranking logic, eval of retrieval quality — owned by `ai`) and
*plumbing* (the backend that serves it — owned by this surface). Plumbing means: the vector-store
wiring (pgvector table + index, connection pooling to it, upsert/query endpoints), the ingestion
pipeline's operational path (not its chunking strategy), and the retrieval endpoint's latency/
error-handling as a normal backend service. Default: **pgvector inside the Postgres you already
run** — consistent with the "Postgres-for-everything" default this skill teaches elsewhere (see
`persistence`); reach for a dedicated vector DB only when pgvector's ANN performance or index types
are a confirmed bottleneck, not preemptively. If the request is "build the RAG pipeline," ask
explicitly which half — prompt/chunking/eval (`ai`) or the serving backend (this surface) — you're
being asked to build; state the split back if it's ambiguous rather than assuming both.

### 5. Best-practice skills for the datastore
Every major backend tool is now shipping an agent-facing interface to its own best practices — a
2026 throughline, not a one-off. Drizzle ships an LLM-agent CLI/SDK; Supabase ships installable
"Postgres best-practices" agent skills; Better-Auth ships `npx skills add` for its own auth setup.
When this surface's build touches a datastore/framework that has a maintained agent-skill or
agent-CLI of its own, prefer installing/consulting it over hand-writing the same guidance — it's
maintained by the vendor and version-tracks their releases (reinforces this skill's retrieval-first
invariant: fetch current docs/tools rather than trusting pretrained knowledge of a fast-moving
target).

### 6. Agent-scoped auth
An agent acting on a user's behalf is a distinct principal from the human — don't reuse the human's
long-lived session/JWT for agent tool calls. Scope agent actions to short-lived, narrowly-permissioned
tokens (the same primitive as OAuth2 scoped access tokens, applied to an agent-as-client rather than
a third-party app — see `auth` for the OAuth2/OIDC mechanics this borrows from), and log which tool
calls an agent token authorized, since agent actions need the same auditability as any other
privileged automation. If the agent needs elevated or destructive actions (writes, deletes, spend),
require an explicit scope grant per action class rather than a single broad "agent" role.

## Validation
- [ ] Every agent-loop path spanning >1 tool call or >1 request cycle has a stated durability
      story (durable-execution engine, or an explicit justification for why it's out of scope).
- [ ] Every tool/MCP endpoint's exposed schema is generated from (or contract-tested against) the
      server's actual validation — not hand-duplicated.
- [ ] Error responses on agent-facing endpoints carry a machine-branchable failure-mode field, not
      status-code-only.
- [ ] RAG endpoints are confirmed as plumbing-only (chunking/prompt/eval logic lives in `ai`); the
      border was stated explicitly, not assumed.
- [ ] Agent-issued tokens are scoped and short-lived, distinct from the human session they act on
      behalf of; privileged actions require explicit scope grants.
- [ ] Retrieval-first check: current docs/CLI/SDK for any agent-facing vendor tool (Drizzle, Better-
      Auth, MCP SDK) were fetched, not assumed from training data.

## Failure modes and handoff
- Agent loop has no crash-recovery story and spans multiple tool calls → don't hand-roll a state
  machine; route the durability mechanics to `async-and-messaging` (workflow/activity split,
  idempotency keys, Temporal/Hatchet choice).
- The ask is really about prompt quality, tool-selection accuracy, or retrieval eval → that's `ai`'s
  core; state the border and hand off rather than silently absorbing it.
- The ask is about the threat model for what an agent is allowed to do (not just how to scope a
  token) → that boundary decision belongs to `architecture`; this surface implements the scoping
  once the decision is made.
- WS/SSE streaming of agent tokens to a client is the actual ask → compose with `surface-realtime`
  (SSE-vs-WS choice, connection lifecycle) rather than re-deriving transport mechanics here.
- Vector index tuning/ANN algorithm choice becomes a real bottleneck, not a hypothetical one →
  that's a `persistence` (or `data`, if it crosses into OLAP-scale retrieval) escalation, not
  something to solve inside this surface ref.
