# Surface: Realtime backend

Purpose: Reshape the job refs for stateful, connection-oriented backends — WebSocket/SSE, pub-sub,
presence-aware sessions. Points to `api-implementation` for the base request/response model this
surface adds a persistent-connection layer on top of.

Read when:
- The request involves a live connection that outlives a single request/response: chat, live
  collaboration, multiplayer/game state, presence indicators, live dashboards, streaming updates.
- The request names WebSocket, SSE, "realtime," "pub-sub," "presence," or "rooms/channels."
- An agent run or long job streams progress to clients that may disconnect, reconnect from another
  device, or resume mid-run.

Skip when:
- The client only needs to be *notified* occasionally and can tolerate a request round-trip
  (webhooks, polling) — that's `service-integration` or plain `api-implementation`, not this surface.
- The connection is stateless request/response even if frequent (standard polling) — no actor/
  presence model needed; stay on `surface-rest-crud`.
- The distributed/multi-service fan-out concern (Kafka/NATS as the backbone between services, not
  to browser clients) is the actual ask → `surface-event-driven-microservices`.

Inputs:
- Expected concurrent-connection scale and whether connections are single-node or must fan out
  across multiple server processes/regions — state explicitly, never assume single-node.
- Whether messages need guaranteed delivery/ordering (reliable) or can be dropped (unreliable,
  e.g. cursor-position updates) — ask per message type, don't default to "everything reliable."

Produces:
- A connection-handling design: transport choice (WS vs SSE), the actor/room model owning each
  connection's mutable state, a presence abstraction, a fan-out strategy sized to the stated
  scale, and — for runs that outlive the connection — a replayable event log.

## Contents
- SSE vs WebSocket: the transport decision
- Actor-per-connection/room model
- Presence as a decoupled abstraction
- Reliable vs unreliable messaging
- Fan-out and scaling
- Replayable run events (a stream over a run that outlives the connection)

## Procedure

### 1. Pick the transport: SSE vs WebSocket
Default to the cheaper primitive that satisfies the requirement, don't reach for WebSocket by
habit. **SSE** (Server-Sent Events): one-directional server→client, works over plain HTTP (no
upgrade handshake), auto-reconnects natively, and is the right default for streaming updates,
token-by-token LLM output, or live dashboards where the client never needs to push mid-stream.
**WebSocket**: full-duplex, required when the client must send frequent, low-latency messages back
(chat input, cursor/game-state updates, collaborative editing) — the added complexity (explicit
reconnect/backoff handling, no native HTTP semantics) is the cost of that duplex requirement. If the
only client→server traffic is occasional (a chat message every few seconds), a plain POST alongside
an SSE stream is often simpler and more debuggable than a WebSocket carrying both directions.

### 2. Model each connection/room as an actor
The recurring, proven shape for authoritative connection state (chat rooms, multiplayer matches,
collaborative documents, live sessions) is: **one actor (goroutine/process/Durable-Object-instance)
owns all mutable state for one session, and every reader/writer talks to it via message passing**,
never by reaching into shared state directly. Nakama's `MatchHandler`/`MatchRegistry` is the
reference shape — each authoritative match is a single-threaded actor with its own tick loop and
mailbox, registered in a lifecycle-tracking registry (create/join/leave/kick/stop). This
generalizes far beyond games: a WS chat room, a collaborative-doc session, or a live-dashboard room
are the same pattern — one owner of truth per session, message-passing at the edges. On Cloudflare
this maps directly onto **Durable Objects** (one object instance per room/connection-group,
co-located storage, no cross-instance race conditions by construction) — reach for that primitive
specifically when you're on the Workers runtime and need this actor guarantee without standing up
your own registry.

### 3. Presence as a decoupled abstraction
Don't build presence bespoke per feature (one presence hack for chat, another for the dashboard).
Nakama's `Tracker` interface is the worked template: a typed, multiplexed stream (mode + subject +
sub-context + label) that many different features — matches, parties, arbitrary channels — all
join/leave against, with separate listener hooks per stream type. The payoff: adding a new
realtime feature (e.g., "who's viewing this document") means registering a new stream type against
the existing tracker, not building a new presence system. Treat presence as its own small service/
module behind an interface, even in a single-process build — it's cheap now and is exactly the
piece that gets duplicated badly later if skipped.

### 4. Reliable vs unreliable, per message
Expose delivery guarantee as a per-message flag, not a transport-wide setting. Nakama's
`MatchDataMessage.Reliable bool` alongside the payload is the concrete shape: cursor-position or
"user is typing" updates can be marked unreliable (fine to drop under load, never worth blocking
on), while chat messages, game-state-mutating actions, or anything the client must not silently
lose are marked reliable (acked, retried, or at minimum logged if delivery fails). Deciding this
per message type up front avoids the common failure mode of either (a) treating everything as
reliable and paying unnecessary backpressure/latency cost, or (b) treating everything as
best-effort and silently dropping messages users actually needed.

### 5. Fan-out and scaling
Single-process/single-node connection handling works until concurrent connections or horizontal
scaling requirements outgrow one process — plan the fan-out seam before you need it, don't retrofit
it under load. Options, cheapest-to-most-operationally-heavy:
- **In-process pub-sub** — fine for a single server instance; simplest, no extra infra.
- **Redis pub-sub / a dedicated realtime broker (Centrifugo)** — when multiple server processes
  must share room state/broadcast across instances; Centrifugo is a purpose-built scaling layer for
  exactly this (WS/SSE fan-out across a cluster) rather than building pub-sub bridging by hand.
- **Framework-native clustering (Phoenix Channels)** — Elixir/OTP's Channels are supervised
  processes per connection with BEAM's distribution built in; a crash isolates to one connection
  without taking down others — a strong option when the base stack is already Elixir/Phoenix.
- **Platform-native (Cloudflare Durable Objects)** — when already on Workers, the actor-per-room
  model (step 2) *is* the fan-out story: each object is independently addressable and the platform
  handles routing to the right instance globally, no separate broker layer needed.
Pick based on what's already in the stack (don't introduce Elixir just for Channels; don't add
Redis if Durable Objects are already available) rather than defaulting to the most-cited option.

### 6. Replayable run events — when the stream reports a run that outlives the connection
For an agent run or long job, the stream is a *view over a persisted log*, not the run itself:
- Create the run and an outbox event in one transaction (`async-and-messaging` §3), return the run
  ID; never acknowledge work that exists only in process memory.
- Persist each meaningful event: schema version, run ID, event ID, monotonic per-run sequence,
  step/attempt ID, type, and the payload or an artifact reference (large outputs by reference).
- Resume by cursor, from either source: an existing `EventSource` reconnecting sends the last
  `id:` back as the `Last-Event-ID` header — but only then (WHATWG HTML, *Server-sent events*); a
  reloaded tab, a second device, or a fetch-based client sends none, so also accept the cursor as an
  explicit parameter (e.g. `?after=<sequence>`, or in a fetch stream's request). Replay committed
  events after the cursor, then tail. No cursor, or one aged out of retention → send the current
  snapshot with its sequence first, then tail. The replay read is authorized like any other read.
- Mark partial model text provisional, keyed by step/attempt, so a retried step's committed output
  replaces it — Cursor's cloud agents rewind the client stream on exactly this case (Cursor, *What
  we've learned building cloud agents*, Jun 2026).
- Cancel, approve, and other mutations are authenticated API calls, not stream messages; a closed
  stream is a connectivity event, never an implicit cancel.
The event contract's shape is `architecture`'s; rendering and client-side resume are `frontend`'s.

## Validation
- [ ] Transport choice (SSE vs WS) is justified by whether the client needs to push mid-stream, not
      chosen by default/habit.
- [ ] Each session/room's mutable state has exactly one owner (actor/object instance); no shared
      mutable state reachable from multiple readers/writers.
- [ ] Presence is implemented as a reusable abstraction, not duplicated per feature.
- [ ] Every message type has an explicit reliable/unreliable classification, not a blanket setting.
- [ ] The fan-out strategy is sized to the stated concurrent-connection/multi-node requirement —
      not single-node-only if multi-node was in scope, and not over-built if it wasn't.
- [ ] Reconnect/backoff handling is implemented client-side for both SSE and WS paths (SSE's native
      auto-reconnect doesn't cover application-level state resync).
- [ ] A run that outlives its connection streams from a persisted, sequenced event log: replay after
      the client's cursor (`Last-Event-ID` or an explicit parameter), snapshot + sequence when there
      is no cursor or it expired, mutations as authenticated calls, and a disconnect never cancels.

## Failure modes and handoff
- The base request-lifecycle question (validation, error shape, auth on the initial handshake) is
  the actual gap → that's `api-implementation`; this surface only adds the persistent-connection
  layer on top of it.
- The realtime channel is actually service-to-service (not browser-facing) fan-out over Kafka/NATS
  → route to `surface-event-driven-microservices`, not this surface.
- Delivery guarantees need to survive a process crash, not just a dropped packet → that's durable-
  execution/outbox territory in `async-and-messaging`, distinct from the unreliable-message flag
  here (crash-durability vs. best-effort-delivery are different problems).
- Presence/auth needs per-connection authorization decisions (who can join which room) → implement
  against `auth`'s session/token model; don't invent a parallel authz path for sockets.
