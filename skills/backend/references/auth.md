# Authentication & Authorization Implementation

Purpose: Turn "add login," "protect this route," "add SSO," or "who can see this resource" into a
concrete authN mechanism (session vs JWT), a concrete OAuth2/OIDC flow, a concrete password-hashing
config, and a concrete authorization model — with a named library/vendor pick, not just theory.

Read when:
- Request involves login, signup, session handling, "protect this endpoint," OAuth/SSO/"sign in with X,"
  service-to-service credentials, password storage, or role/permission checks.
- An existing auth stack needs a security-hardening pass (token validation, cookie flags, rotation).

Skip when:
- The ask is *which* controls an already-chosen auth flow needs at the infra layer (CORS, headers,
  CSRF, secrets storage, mTLS) → `server-security` (this ref owns the identity/token/password/authz
  decision; server-security owns the surrounding hardening).
- The ask is upstream — "should this system even centralize auth," threat modeling, compliance scope →
  `architecture`.
- The ask is fine-grained OLAP/analytics access control on data pipelines → `data`.

Inputs:
- Target framework/language and whether a public client (SPA/mobile) or confidential client (server)
  is involved. State explicitly when absent — don't assume SPA vs server-rendered.
- Multi-tenancy requirement (single-org vs many tenants) — changes the authorization-ladder pick.
- Compliance constraints (FIPS-140, SOC2) if any — changes password-hashing algorithm choice.

Produces: a named session/JWT decision, a named OAuth2/OIDC flow with concrete parameters, a
password-hashing config (algorithm + params), and a named authorization model, wired to one library
or vendor pick — plus what NOT to build at the current scale.

## Contents
- Sessions vs JWT
- OAuth2/OIDC: flows, PKCE, RFC 9700
- Password hashing
- Authorization decision ladder
- Library/vendor pick

## Procedure

### 1. Sessions vs JWT
Default to **short-lived access JWT (5–15 min) + rotating opaque refresh token**, not either extreme
alone:
- Pure server-side session (opaque ID + DB/Redis lookup): instant revocation (delete the row), but
  every request pays a stateful lookup and needs a shared session store across instances.
- Pure stateless JWT: scales horizontally with no shared store (any node verifies the signature
  locally), but a signed JWT is valid until `exp` regardless of server-side state changes (password
  reset, permission downgrade, logout) — genuine early revocation needs a denylist, which reintroduces
  the statefulness JWTs were meant to avoid.
- The compromise, and the 2026 default: short-lived access token (JWT, fast local verify) + long-lived
  refresh token (opaque, stored server-side, revocable) exchanged at a token endpoint. Rotate the
  refresh token on every use; on reuse of an already-rotated-out token, revoke the entire token family
  (not just that token) — this is the RFC 9700 rule, §2 below, and it's what turns "someone stole an old
  refresh token" into a detectable event instead of a silent replay.
- **Storage, both cases**: never `localStorage`/`sessionStorage` (any script, including an injected XSS
  payload, can read them). Use `HttpOnly; Secure; SameSite=Strict` cookies, or a **Backend-for-Frontend
  (BFF)** where the browser never holds the token at all and the BFF proxies calls using a server-held
  token — the only fully XSS-immune option for a SPA.
- Session-cookie shape when a session (not JWT) is the chosen mechanism: `Set-Cookie:
  __Host-SessionID=<64-bit+ random>; Secure; HttpOnly; SameSite=Strict; Path=/`, non-persistent (no
  `Max-Age`), regenerate the ID on login and on any privilege change, idle timeout 15–30 min / absolute
  4–8h enforced server-side.

### 2. OAuth2/OIDC: flows, PKCE, RFC 9700
Two grants cover essentially all backend needs; treat the other two RFC 6749 grants as retired:
- **Authorization Code + PKCE (S256)** — any flow with a browser/redirect leg (web app login, "sign in
  with Google," mobile/native). PKCE binds the code-redemption step to the client that started the
  flow, closing the code-interception window.
- **Client Credentials** — service-to-service / machine-to-machine, no user in the loop. This is the
  correct grant for backend-to-backend calls; never use Resource Owner Password Credentials (posting a
  raw username/password to the token endpoint) for this — it's explicitly forbidden below.
- Implicit grant and ROPC: **do not implement either** in a new system — both are removed from OAuth 2.1
  and forbidden by RFC 9700 (implicit leaks tokens via URL fragment/browser history; ROPC defeats
  OAuth's premise of never exposing credentials to the client).
- **RFC 9700 (OAuth 2.0 Security BCP, Jan 2025)** is the load-bearing 2026 document — treat it as more
  authoritative than reading RFC 6749 alone for what to configure:
  - PKCE is **mandatory for public clients**, **recommended even for confidential clients**; use S256,
    never `plain`.
  - Authorization servers **MUST use exact string matching** on redirect URIs (the only carve-out:
    native apps may vary the `localhost` port).
  - Prefer **sender-constrained access tokens** (mTLS or DPoP) over plain bearer tokens where feasible —
    a stolen bearer token alone is otherwise sufficient to replay it from anywhere.
  - Refresh tokens for public clients: sender-constrained, or rotation-on-use + reuse-detection with
    whole-family revocation (§1 above).
- **OAuth 2.1** is a consolidation, not a new numbered RFC — it folds these RFC 9700 rules into one spec.
  Teach it as the target shape (Auth Code+PKCE, Client Credentials, no Implicit/ROPC, exact redirect
  match) rather than teaching the legacy grants as live options.
- **Don't conflate OAuth and OIDC.** OAuth's access token answers "can this client access this
  resource" (an opaque or JWT bearer credential, scoped by `scope`). OIDC's **ID token** (always a JWT,
  RFC 7519) answers "who is this user," requested via the `openid` scope, carrying `iss`/`sub`/`aud`/
  `exp` claims. Authorize API calls off the access token/its scopes; authenticate "who is this" off the
  ID token's signature and claims — using one where the other belongs is a recurring implementation bug.
- JWT verification, either token type: hardcode/pin the expected algorithm at verify time (don't trust
  the token's own `alg` header — this is what lets an attacker flip `RS256`→`HS256` or `alg:none` and
  fool a verifier that treats a public key as an HMAC secret), reject `alg:none` explicitly, and always
  validate `exp`, `nbf`, `iss`, `aud` — not just signature validity. `aud` must name *this* service:
  never forward an inbound bearer token to another service — call it with your own credential (the
  MCP rule, in `surface-agentic` §6).

### 3. Password hashing
**Argon2id, minimum `m=19MiB, t=2, p=1`** (memory-hard, GPU/ASIC-resistant, PHC-competition winner) is
the default for any new system. Equal-security alternates trading memory for CPU: `m=46MiB,t=1,p=1` /
`m=12MiB,t=3,p=1` / `m=7MiB,t=5,p=1`.
- **bcrypt** (legacy systems only, not new builds): cost factor ≥10; has a hard **72-byte input cap** —
  longer inputs silently truncate unless pre-hashed (`bcrypt(base64(hmac-sha384(password)))`), a common
  source of the "password shucking" vulnerability class.
- **scrypt** (`N=2^17,r=8,p=1`) if Argon2id isn't available in the stack; **PBKDF2**
  (`HMAC-SHA256@600k` or `HMAC-SHA512@210k` iterations) only when FIPS-140 compliance forces it.
- Regardless of algorithm: unique random salt per password, target ~1s server-side hash time, store
  algorithm+params in PHC string format so multiple versions coexist during migration, and
  **rehash-on-next-login** when upgrading parameters rather than forcing a mass reset.
- Never roll a custom timing-unsafe comparison — use the library's constant-time verify function.
Source: OWASP Password Storage Cheat Sheet (CC BY-SA 4.0, cheatsheetseries.owasp.org) — paraphrased.

### 4. Authorization decision ladder
Pick the cheapest model that fits current scale; don't default to the heaviest option.
1. **Rule-strings** (simplest, single-app scale): a boolean filter expression per operation per
   resource type, evaluated against the request/record context — e.g. PocketBase's per-collection
   `ListRule`/`ViewRule`/`CreateRule`/`UpdateRule`/`DeleteRule`. Good enough when there's no cross-tenant
   sharing graph.
2. **RBAC**: permissions attach to roles, users inherit roles. Simplest mental model; suffers **role
   explosion** as org/resource combinations multiply and is weak for cross-tenant/relationship cases.
   Multi-tenant RBAC needs a domain/tenant parameter on role grouping, not a flat global role table.
3. **ABAC**: decisions computed from subject + resource + environment attributes against policy logic.
   Handles multi-tenant and fine-grained cases RBAC can't express. **Cedar** (AWS, open policy language,
   formally verifiable, embeddable independent of AWS) is the reference pick when rules are mostly
   attribute/policy-expression driven.
4. **ReBAC / Zanzibar model**: access granted by a **relationship chain** between subject and resource
   (`object#relation@user` tuples — "user is a member of a group that edits a folder containing this
   doc"). Suits ownership/sharing-graph resources (Drive/Notion/Figma-style permission trees). Reference
   open implementations: **OpenFGA** (CNCF, Apache-2.0, HTTP/gRPC + first-party SDKs, models RBAC as a
   subset, adds ABAC-like `Conditions`) or **SpiceDB** (Apache-2.0, adds "caveats" — conditional
   relationships evaluated against runtime context — and per-request consistency tokens for the
   staleness/latency tradeoff). Pick OpenFGA for simplicity/SDK breadth, SpiceDB if ABAC-on-relationships
   is needed.
5. **Casbin** (Apache-2.0, embeddable library, not a service) when any of the above is wanted **in-process**
   without standing up a separate authz service. Its "PERM metamodel" — `request_definition` (shape of a
   check, e.g. `sub, obj, act`), `policy_definition` (shape of a stored rule), `policy_effect` (how
   matching rules combine, e.g. deny-override `some(where(p.eft==allow)) && !some(where(p.eft==deny))`,
   or priority/first-match), `matchers` (the boolean expression linking request to policy) — expresses
   RBAC, ABAC (a matcher can be as simple as `r.sub == r.obj.Owner`), multi-tenant RBAC (add a `dom`
   field to `r`/`p` and `g`), and deny/priority composition as swappable `.conf`+policy pairs instead of
   hardcoded `if` chains. Casbin explicitly does **not** do authentication or user/role management —
   that stays in the host app.
Deny-by-default and centralized enforcement (one chokepoint — middleware/interceptor, not scattered
per-handler checks) apply regardless of which rung is chosen; re-check the caller's permission on the
*specific object*, never trust object-type-level access as a stand-in for per-object IDOR checks.

### 5. Library/vendor pick
**Better-Auth is the 2026 TypeScript default** — self-hosted, framework-agnostic, having overtaken
Auth.js/NextAuth in mindshare as the TS ecosystem consolidated around it; ships an agent-installable skill
(`npx skills add better-auth/skills`) and joined Vercel to build "OSS auth for apps and agents." Default
session priority is secondaryStorage → DB → stateless cookie; default hash is scrypt, with Argon2id as
an opt-in via `@node-rs/argon2`. Deviate when:
- **Clerk** — fastest DX/pre-built UI for a pre-PMF app shipping auth same-day (not for teams wanting
  full backend ownership).
- **Auth0/Okta** — buyer specifically needs enterprise SSO/SCIM breadth, can absorb per-MAU pricing.
- **Ory (Kratos+Hydra)** — self-hostable at internet scale; Kratos = headless identity, Hydra = a
  separate OAuth2/OIDC token-issuance server that deliberately delegates "who is this user" out via
  redirect. Pick for data residency or when per-MAU pricing becomes prohibitive.
- **Keycloak** — self-hosted enterprise IAM (SSO/OIDC/SAML/LDAP) without vendor lock-in.
Retrieval-first: fetch current docs for whichever is picked before generating config/route code — this
cluster breaks fast (plugin import paths, cookie-cache strategy names, adapter model-vs-table-name
mismatches).

## Validation
- [ ] Access-token lifetime ≤15 min; refresh token rotates on use with reuse-triggered family revocation.
- [ ] PKCE(S256) present on every Authorization Code flow; redirect URI matched by exact string.
- [ ] JWT verification pins the algorithm and validates `exp`/`nbf`/`iss`/`aud`; `alg:none` rejected.
- [ ] Password hash is Argon2id (or a stated, justified alternate) at or above minimum params; unique
      salt per password.
- [ ] Authorization enforced at one centralized chokepoint, deny-by-default, checked per-object (no IDOR).
- [ ] No bearer token (JWT or opaque) ever written to `localStorage`/`sessionStorage`.

## Failure modes and handoff
- Picking ReBAC (OpenFGA/SpiceDB) before RBAC/ABAC is exhausted → recursive relationship checks need
  explicit resolution-depth/breadth limits or they DoS themselves on deeply nested groups; if that's not
  budgeted, step back down the ladder.
- "Which controls surround this token/session at the transport layer" (CORS, CSRF, security headers,
  secrets storage, mTLS) → `server-security`.
- Rate-limiting login/token endpoints → `caching-and-performance` (token-bucket is single-homed there).
- A threat model or compliance boundary hasn't been decided yet (which data needs encryption at rest,
  what counts as "sensitive") → back to `architecture`; implement inside that decision, don't invent it.
