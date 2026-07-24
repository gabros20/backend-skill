# API Contract Review Checklist

Fill one row per endpoint/operation before calling an API implementation done. Self-audit count at the
bottom — if any row has an unchecked cell, state why or fix it before handoff.

| Endpoint / operation | Verb + path (or RPC name) | Request schema single-sourced? | Response schema single-sourced? | Error shape matches house standard? | Pagination (cursor if high-write) | Version stated | Auth requirement stated | Status codes correct (2xx/4xx/5xx) |
|---|---|---|---|---|---|---|---|---|
| | | ☐ | ☐ | ☐ | ☐ / N/A | ☐ | ☐ (→ `auth`) | ☐ |
| | | ☐ | ☐ | ☐ | ☐ / N/A | ☐ | ☐ (→ `auth`) | ☐ |
| | | ☐ | ☐ | ☐ | ☐ / N/A | ☐ | ☐ (→ `auth`) | ☐ |

**Self-audit count:** `___ / ___` cells checked. Any unchecked cell needs either a fix or an explicit
stated reason (e.g. "N/A — internal admin-only endpoint, no pagination needed").

## Column definitions
- **Request/response schema single-sourced**: one declaration (Pydantic/TypeBox/JSON-Schema/DTO+
  decorators/Serializer) drives validation + serialization + docs — no hand-duplicated second copy of
  the same shape.
- **Error shape matches house standard**: same `code`/`message`/`status`(/`details`) shape as every
  other endpoint in this service, including on unhandled-exception paths.
- **Pagination**: cursor-based for high-write/high-scale collections (see `api-implementation` §5);
  mark N/A only for single-resource or genuinely small/static lists.
- **Version stated**: URL-path or header version is explicit, not implicit; GraphQL fields use
  `@deprecated` instead of a version bump.
- **Auth requirement stated**: which auth model (session/JWT/OAuth scope) gates this endpoint —
  detail owned by `auth`, this checklist only confirms it's stated, not undefined.
