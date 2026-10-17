# Handoff

Purpose: Package a completed backend build (running service(s), their tests, the contracts they
implement, telemetry hooks) so downstream skills can operate, verify, or extend it without
re-deriving what was decided, plus a compact `handoff.yaml` companion for pipeline routing.

Read when:
- Finishing a backend build that `operate` is expected to deploy/monitor, or `quality` is expected
  to independently verify.
- Deciding whether this run should emit a machine-readable companion at all.
- Writing or checking the `assets/handoff.yaml` template itself.

Skip when:
- The request is standalone with no downstream consumer in view ("implement this endpoint", "fix
  this query") — produce the code/tests and stop; an unread `handoff.yaml` is clutter, not
  diligence.
- You need a specific job's own procedure — that's [api-implementation](api-implementation.md),
  [persistence](persistence.md), [auth](auth.md), or one of the other 6 job references for the
  implementation craft itself. This reference covers what happens after the service runs, not how
  to build it.

Inputs: the completed service(s) and their code paths, the test suite that verifies them, the
upstream artifacts backend built against (architecture's contracts, NFR budgets, threat model,
ADRs — when this run is inside a pipeline), and the decisions/assumptions/risks recorded while
implementing.

Produces: a `handoff.yaml` companion beside the running service and its tests, filled from
[assets/handoff.yaml](../assets/handoff.yaml) — never a duplicate of the code or test output.

## Contents
- Standalone vs pipeline
- What backend consumes
- What backend hands downstream
- The handoff.yaml template
- Per-consumer needs
- The seam discipline
- Validation
- Failure modes

## Standalone vs pipeline

`backend` runs in two modes, read from context, not a flag. **Standalone**: the request is
self-contained ("implement this API", "add this endpoint", "fix this migration") with no downstream
work named — deliver the code, tests, and any inline documentation only. **Pipeline**: the request
or an orchestrating controller names downstream work explicitly (deploy this, hand it to `quality`
for independent verification, `frontend` is waiting on this contract), or a multi-skill run is
already in motion — emit the `handoff.yaml` companion. Default to standalone; only emit the
companion when a downstream consumer is actually expected to read it.

## What backend consumes

When `architecture` has already run, backend implements **inside** the decided shape rather than
re-deriving it — consume these artifacts, don't silently override them:

| Artifact (from `architecture`) | What backend takes from it |
|---|---|
| Solution-architecture doc | the boundaries/shape to build inside — decisions and contracts, not implementation detail it re-litigates |
| API/event contracts | the interface to implement exactly, in parallel with `frontend` coding to the same contract |
| NFR budgets | perf/scale/reliability/cost targets the implementation must meet, not aspire to |
| Threat model | which controls backend is responsible for implementing (vs which `quality` verifies independently — the border must be named, not assumed) |
| ADRs | the durable "why" behind a decision already made — cite it, don't reopen it |

When architecture didn't run (standalone `backend`), these inputs don't exist — say so explicitly
rather than inventing a contract or threat model to fill the gap.

## What backend hands downstream

Backend sits at the Build stage: it hands two distinct things to two distinct consumers — **do not
conflate them.**

| Artifact | Primary consumer | What they take from it |
|---|---|---|
| Running service(s) + telemetry hooks (structured logs, health/readiness endpoints, metrics/trace instrumentation points) | `operate` | what to deploy, and what to wire into dashboards/alerts — backend instruments the hooks, `operate` owns the dashboard/alert/deploy pipeline itself |
| Backend's own tests + the contract they were built against | `quality` | the starting point for *independent* verification — `quality` re-derives its own test/load/scan plan against the contract; it does not simply re-run backend's tests and call that independent |

`operate` and `quality` are consumers, not co-authors: backend ships the hooks and the tests; it
does not run the production monitor or the independent security scan.

## The handoff.yaml template

`assets/handoff.yaml` is the fillable schema. Field meanings follow the same seam discipline as the
sibling `architecture` and `product` skills' handoffs: only `objective` is required; every other
field may be empty, and a consumer must tolerate it absent rather than treat absence as a schema
violation. `artifacts_created` entries use `type` values: `service`, `contract-implementation`,
`tests`, `telemetry-hooks`. `recommended_next` names the actual next skill(s) — `operate`,
`quality`, `frontend`, `data`, `ai` — never a vague "further work needed."

## Per-consumer needs

**`operate`** needs the telemetry hooks (structured log format, health/readiness endpoint paths,
metric/trace instrumentation points) and the NFR budgets backend built to — not a re-explanation of
the code, a pointer to where the hooks live and what they emit. **`quality`** needs the contract
backend implemented against and backend's own test suite as a *starting point*, not a substitute for
independent verification — it re-derives its own acceptance tests against the same contract.
**`frontend`** (when building in parallel against the same contract) needs the contract's actual
implemented shape confirmed — flag any deviation from the architecture-issued contract explicitly,
don't let it surface as a silent runtime mismatch. For an agent backend, that includes the run-event
contract as implemented (event fields, resume cursor, snapshot path, cancel/approve calls —
`surface-realtime` §6). **`data`** needs to know which persistence layer and schema backend actually
built, when downstream analytics/pipelines read from the same store.

## The seam discipline

The running service, its tests, and its telemetry hooks are the real deliverable. `handoff.yaml` is
a routing index into them, never a copy. If `operate` or `quality` needs more than the index gives,
the code/tests are incomplete or under-documented — the yaml should not grow to compensate. Only
`objective` is required; every downstream skill must tolerate every other field absent.

## Validation

Before emitting: `objective` is one sentence and non-empty; every path in `artifacts_created`
resolves to real code/tests/docs; `assumptions` and `risks` trace to entries recorded during
implementation, not invented at handoff time; every `unresolved` item names an owner;
`recommended_next` names an actual sibling skill, not a vague call for more work; if a threat model
was consumed, the border between backend-implemented controls and quality-verified controls is
stated explicitly.

## Failure modes

- **Duplicating the service's code or test output into handoff.yaml** — defeats its purpose as an
  index; keep it a pointer.
- **Emitting handoff.yaml in standalone mode** when nothing downstream will read it — skip it.
- **`status: complete` with a non-trivial `validation.remaining`** — use `partial` and say what's
  outstanding (e.g. "endpoint implemented, load test not yet run").
- **Handing `quality` the same tests backend already ran and calling that independent
  verification** — `quality`'s value is a fresh, adversarial pass against the contract, not a rerun.
- **The threat-model border line is missing** — `quality` doesn't know which controls backend
  already implemented versus which it must verify from scratch; name the border explicitly when a
  threat model was consumed.
- **Silently invoking `operate` or `quality`** instead of naming it in `recommended_next` — the
  never-auto-invoke boundary rule from the main SKILL.md applies at the handoff seam too.
- **An `unresolved` item with no owner** — assign one or drop the item until it has one.

<!-- pattern mirrors the sibling architecture-skill's references/handoff.md (Tamás Gábor, MIT) —
same seam discipline, same handoff.yaml shape, adapted from architecture→frontend/backend/data/ai/
quality to backend→operate/quality/frontend/data/ai. -->
