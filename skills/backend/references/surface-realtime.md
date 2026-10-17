# Surface: Realtime backend

Purpose: Reshape the job refs for stateful, connection-oriented backends — WebSocket/SSE, pub-sub,
presence-aware sessions, and the media transport under a voice agent. Points to `api-implementation`
for the base request/response model this surface adds a persistent-connection layer on top of.

Read when:
- The request involves a live connection that outlives a single request/response: chat, live
  collaboration, multiplayer/game state, presence indicators, live dashboards, streaming updates.
- The request names WebSocket, SSE, "realtime," "pub-sub," "presence," or "rooms/channels."
- An agent run or long job streams progress to clients that may disconnect, reconnect from another
  device, or resume mid-run.
- A voice agent needs its audio path (WebRTC, a WebSocket to a speech model, SIP, a media stream).

Skip when:
- The client only needs to be *notified* occasionally and can tolerate a request round-trip
  (webhooks, polling) — that's `service-integration` or plain `api-implementation`, not this surface.
- The connection is stateless request/response even if frequent (standard polling) — no actor/
  presence model needed; stay on `surface-rest-crud`.
- The distributed/multi-service fan-out concern (Kafka/NATS as the backbone between services, not
  to browser clients) is the actual ask → `surface-event-driven-microservices`.
- Turn-taking, speech components, or the voice architecture is the ask → `ai` (`surface-voice.md`
  "Surface: Voice (additive)"); this surface carries only the audio.

Inputs:
- Expected concurrent-connection scale and whether connections are single-node or must fan out
  across multiple server processes/regions — state explicitly, never assume single-node.
- Whether messages need guaranteed delivery/ordering (reliable) or can be dropped (unreliable,
  e.g. cursor-position updates) — ask per message type, don't default to "everything reliable."

Produces:
- A connection-handling design: transport choice (WS vs SSE), the actor/room model owning each
  connection's mutable state, a presence abstraction, a fan-out strategy sized to the stated
  scale, a replayable event log for runs that outlive the connection, and a voice agent's media path.

## Contents
- SSE vs WebSocket: the transport decision
- Actor-per-connection/room model
- Presence as a decoupled abstraction
- Reliable vs unreliable messaging
- Fan-out and scaling
- Replayable run events (a stream over a run that outlives the connection)
- Voice media transport (the substrate under a voice agent)

## Procedure

### 1. Pick the transport: SSE vs WebSocket
Default to the cheaper primitive that satisfies the requirement. **SSE**: one-directional
server→client over plain HTTP, auto-reconnects natively — the default for streaming updates,
token-by-token LLM output, or live dashboards where the client never pushes mid-stream.
**WebSocket**: full-duplex, required when the client sends frequent, low-latency messages back (chat
input, cursor/game state, collaborative editing); explicit reconnect/backoff and no native HTTP
semantics are the cost. If client→server traffic is occasional, a plain POST beside an SSE stream is
simpler and more debuggable than a WebSocket carrying both directions.

### 2. Model each connection/room as an actor
The proven shape for authoritative connection state (chat rooms, matches, collaborative documents,
live sessions, calls): **one actor (goroutine/process/Durable-Object instance) owns all mutable state
for one session, and every reader/writer talks to it by message passing**, never through shared
state. Nakama's `MatchHandler`/`MatchRegistry` is the reference shape — each match a single-threaded
actor with its own tick loop and mailbox, in a lifecycle-tracking registry
(create/join/leave/kick/stop). On Cloudflare Workers, **Durable Objects** give this guarantee
directly (one instance per room, co-located storage, no cross-instance races) without your own
registry.

### 3. Presence as a decoupled abstraction
Don't build presence bespoke per feature. Nakama's `Tracker` interface is the worked template: a
typed, multiplexed stream (mode + subject + sub-context + label) that matches, parties, and arbitrary
channels all join/leave against, with listener hooks per stream type — a new feature ("who's viewing
this document") registers a stream type instead of building a presence system. Keep presence its own
module behind an interface even in a single-process build; it is the piece that gets duplicated
badly when skipped.

### 4. Reliable vs unreliable, per message
Expose delivery guarantee as a per-message flag, not a transport-wide setting. Nakama's
`MatchDataMessage.Reliable bool` alongside the payload is the concrete shape: cursor-position or
"user is typing" updates can be marked unreliable (fine to drop under load, never worth blocking
on), while chat messages, game-state-mutating actions, or anything the client must not silently
lose are marked reliable (acked, retried, or at minimum logged if delivery fails). Deciding per
message type avoids both failures: everything reliable pays needless backpressure, everything
best-effort silently drops messages users needed.

### 5. Fan-out and scaling
Single-node connection handling works until connections or horizontal scaling outgrow one process —
plan the fan-out seam before you need it. Options, cheapest to most operationally heavy:
- **In-process pub-sub** — fine for a single server instance; simplest, no extra infra.
- **Redis pub-sub / a dedicated realtime broker (Centrifugo)** — when multiple server processes
  must share room state/broadcast across instances; Centrifugo is purpose-built WS/SSE fan-out
  across a cluster, instead of hand-built pub-sub bridging.
- **Framework-native clustering (Phoenix Channels)** — Elixir/OTP's Channels are supervised
  processes per connection with BEAM's distribution built in; a crash isolates to one connection
  without taking down others — a strong option when the base stack is already Elixir/Phoenix.
- **Platform-native (Cloudflare Durable Objects)** — on Workers the actor-per-room model (step 2)
  *is* the fan-out story: each object is addressable and the platform routes to it, no broker.
Pick from what's already in the stack (no Elixir just for Channels, no Redis beside Durable
Objects), not the most-cited option.

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
The event contract's shape is `architecture`'s (`contracts.md` "Contract-first design"); rendering
and client-side resume are `frontend`'s.

### 7. Voice media transport — the substrate under a voice agent
- **Where the caller is picks the transport.** Browser or app → WebRTC; server to a speech model →
  WebSocket; phone → a SIP trunk into your media server or the model's SIP endpoint, or a
  programmable-voice media stream forking call audio to your WebSocket.
- **Telephony media is a fixed codec.** Twilio Media Streams carry base64 μ-law, 8 kHz, mono, both
  ways, no header bytes; a 16/24 kHz PCM model needs transcoding and resampling at the edge, and the
  band limit stays. Only a `<Connect><Stream>` stream is bidirectional and delivers `dtmf` (Twilio,
  *Media Streams WebSocket messages*).
- **Barge-in is a playback-control sequence, with one owner.** On caller speech: flush buffered
  audio (`clear`, which returns the pending `mark`s), cancel in-flight LLM and TTS work, and record
  from the marks how much played, so history keeps only what the caller heard. Whoever plays the
  audio truncates: over WebRTC and SIP the provider's server; over a WebSocket the client, with
  `conversation.item.truncate` and the played `audio_end_ms` (OpenAI, *Realtime conversations*).
  Never run a native and a manual interruption handler together — both act, and playback overlaps.
- **Capped model sessions reconnect from day one.** Speech-to-speech sessions end on hard caps (dated
  caps: `ai` `surface-voice.md` "Architecture: cascade, native speech-to-speech, or hybrid"); Gemini
  Live sends a `GoAway` before the close and issues resumption tokens (Google, *Live API session
  management*). The call's actor (§2) owns the call; the model connection is a
  replaceable child: on `GoAway` or close, resume with the latest handle or re-seed a
  session from the persisted transcript with the same voice (fixed once a session has spoken), never
  dropping the phone leg.

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
- [ ] A voice path picks its transport by caller location, converts the codec at the edge, has one
      barge-in owner that records what played, and survives a model-session cap without dropping
      the call.

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
- Voice operations → `operate`: the latency SLI (`surface-ai-system.md` "Time-to-first-token is a
  distinct SLI from total duration"), call cost and concurrency (`capacity-and-cost.md` "The
  2020s-specific version: agent and LLM spend scales with activity, not users"), carrier outages
  (`surface-ai-system.md` "Provider outages are correlated across everyone using them"), recording
  retention (`observability.md` "AI-system observability — same discipline, one new signal").
  Verifying the call at the caller's ear → `quality` (`surface-ai-system.md` "Voice: evidence at the
  caller's ear").
