# Server Security Controls

Purpose: Turn a decided threat model (or a bare "harden this backend" request) into the concrete
implementation controls around a request's path — input validation, secrets, CORS, headers, CSRF,
mTLS — mapped to OWASP Top 10:2021/ASVS, as a uniform checklist, not narrative advice.

Read when:
- Request is "secure this API," "add security headers," "fix CORS," "prevent SQL injection," "how do
  we store secrets," "add CSRF protection," or a security-review/audit-prep pass on backend code.
- Implementing controls that a threat model, ASVS level, or compliance requirement already named.

Skip when:
- The threat model itself hasn't been decided (what's in scope, what attacker, what NFR budget) →
  `architecture` — this ref implements controls *inside* that decision, it doesn't produce the decision.
- The ask is identity/token/password/authorization mechanics (sessions, JWT, OAuth, Argon2id, RBAC/ABAC)
  → `auth` (A07 Identification/Authentication Failures lives there; this ref covers A01/A02/A03/A05/
  A06/A08/A09/A10-adjacent controls).
- The ask is rate limiting specifically → `caching-and-performance` (token-bucket algorithm is
  single-homed there; this ref only notes that endpoints need it).
- The ask is an independent security scan/pen-test/compliance gate, not implementation → `quality`.

Inputs:
- Which OWASP Top 10 categories or ASVS level (L1/L2/L3) apply, if already scoped upstream; state
  explicitly when absent — don't assume L3 rigor for an internal tool.
- Whether the deployment is single-service or microservices (changes the mTLS/per-endpoint-authz answer).
- Existing secrets-management tooling (Vault, cloud KMS, none) — changes the "how," not the "never
  hardcode" principle.

Produces: a uniform controls table (control → implementation → verification) covering the request's
scope, plus a self-audit count of controls implemented vs required.

## Contents
- OWASP Top 10:2021 + ASVS framing
- Input validation & injection prevention
- Secrets management
- CORS
- Security response headers
- CSRF
- mTLS / service-to-service

## Procedure

### 1. OWASP Top 10:2021 + ASVS framing
Use the Top 10 as the *what's-at-risk* index and ASVS as the *how-rigorous* audit companion — don't
treat either as exhaustive on its own.
- **A01 Broken Access Control** (now #1, most common real-audit finding) → `auth`'s authorization ladder
  owns the fix; this ref's job is making sure every endpoint actually calls the chokepoint.
- **A02 Cryptographic Failures** → encrypt at rest/in transit with authenticated modes (below); never
  roll custom crypto.
- **A03 Injection** → parameterized queries (below) as the structural fix, not escaping.
- **A05 Security Misconfiguration**, **A06 Vulnerable/Outdated Components** → dependency audit +
  no-default-creds + minimal-surface config as part of the release checklist (`quality` runs the
  independent scan; this ref is where the config gets set correctly the first time).
- **A08 Software/Data Integrity Failures** → verify CI/CD artifact signing and deserialization inputs.
- **A09 Security Logging/Monitoring Failures** → log auth/validation failures with enough context to
  investigate, never log secrets/raw tokens/session IDs (log a salted hash if correlation is needed).
- **A10 SSRF** → allowlist outbound destinations for any backend-initiated fetch of a URL that can be
  influenced by user input; never let a raw user-supplied URL reach an internal-network-capable client.
- **OWASP ASVS v5.0** (several hundred requirements across its chapters, three verification levels
  L1→L3, each a superset of the last) is the audit-checklist companion once a level is scoped — pull the current
  requirement text from asvs.dev at audit time rather than relying on a static excerpt.
Source: OWASP Top 10:2021 + ASVS (both CC BY-SA 4.0, owasp.org) — paraphrased, attribute on reuse.

### 2. Input validation & injection prevention
- **Allowlist over denylist**: define exactly what's permitted (format, length, range, type); denylists
  are bypassable and prone to false positives (blocking `'` breaks legitimate names).
- **Syntactic + semantic validation**: syntactic = correct structure (date format); semantic =
  business-sense correctness (start < end date, price in range) — apply both, server-side only
  (client-side validation is UX sugar, trivially bypassed).
- **Parameterized queries / prepared statements are the primary SQL-injection defense**, not escaping:
  code and data travel to the DB separately at the protocol level, so a malicious `' OR '1'='1` is bound
  as a literal value, never interpreted as SQL. Stored procedures are equally safe only if they don't
  build dynamic SQL internally from unvalidated input. Table/column names can't be bind-parameterized —
  where a dynamic identifier is unavoidable, map it through an allowlist of legal names, never
  interpolate raw.
- **Least privilege for the app's DB account**: never DBA/admin; separate credentials per application.
- **File upload**: extension allowlist, size cap, server-generated random filenames (never trust client
  filename), re-encode uploaded images to strip embedded payloads, explicitly block executable
  (`.php`/`.asp`/`.jsp`) and config (`.htaccess`) extensions.
Source: OWASP SQL Injection Prevention + Input Validation Cheat Sheets (CC BY-SA 4.0) — paraphrased.

### 3. Secrets management
- **Never hardcode secrets** in source or commit them — standardize on one centralized secrets manager
  org-wide (AWS Secrets Manager, Azure Key Vault, GCP Secret Manager, HashiCorp Vault) rather than every
  team inventing its own scheme.
- **Prefer dynamic, short-lived secrets** over static ones: e.g. Vault's DB secrets engine mints a
  fresh, uniquely-scoped DB credential per request/session with a built-in TTL and auto-revoke, bounding
  a leaked credential's blast radius to its TTL window and making access auditable per-lease. Contrast
  with a static secret that lives indefinitely until someone remembers to rotate it manually.
- **Env vars are a pragmatic compromise, not best practice** — readable by any process in scope, prone
  to leaking into logs/crash dumps; prefer a manager's runtime injection (sidecar/API) where the
  deployment supports it.
- **Automate rotation**; low-sensitivity CI/CD secrets can live in the platform's native store (GitHub
  Actions/GitLab CI secrets), high-value secrets should be fetched from the central vault at pipeline
  run time, never committed.
- Run pre-commit/CI secret-scanning (e.g. `detect-secrets`, `gitleaks`) to catch accidental commits.
Source: OWASP Secrets Management Cheat Sheet (CC BY-SA 4.0) — paraphrased.

### 4. CORS
- Never `Access-Control-Allow-Origin: *` on any endpoint that accepts credentials/auth — reflect a
  specific allowed origin (or validate against an allowlist) instead; wildcard is fine only for public,
  read-only, credential-free endpoints.
- Answer preflight `OPTIONS` requests with the allowed origin/methods/headers before the browser sends
  the real request; cache the preflight response (`Access-Control-Max-Age`) to cut round-trips.
Source: MDN CORS guide (developer reference for the Fetch/CORS spec).

### 5. Security response headers
Baseline on every production HTTPS endpoint: `Strict-Transport-Security` (long `max-age`,
`includeSubDomains` where safe), `Content-Security-Policy` (restrict script/style/frame/connect
sources to trusted origins), `X-Content-Type-Options: nosniff`, `Referrer-Policy`,
`Permissions-Policy`, and the cross-origin isolation trio `Cross-Origin-Opener-Policy` /
`Cross-Origin-Embedder-Policy` / `Cross-Origin-Resource-Policy`. Also: `Cache-Control: no-store` on any
response carrying a session ID or token; never mirror the client's `Accept` header into the response
`Content-Type`; reject unexpected methods with `405` and unexpected content types with `406`/`415`.
Source: OWASP HTTP Security Response Headers Cheat Sheet / Secure Headers Project (CC BY-SA 4.0).

### 6. CSRF
- **Synchronizer token pattern** (stateful, primary defense): a large, cryptographically random,
  per-session (or per-request) token embedded in forms/headers, checked server-side on every
  state-changing request.
- `SameSite=Strict` (or `Lax`) cookie attribute is a strong **complementary** control, explicitly
  **not a replacement** for the token — use both, defense in depth.
- Stateless alternative for apps that can't hold server-side session state: encrypted/HMAC-based
  double-submit tokens.
Source: OWASP CSRF Prevention Cheat Sheet (CC BY-SA 4.0).

### 7. mTLS / service-to-service
- Both client and server present certificates during the TLS handshake, each proving possession of the
  matching private key — mutual cryptographic identity rather than a bearer credential (API key/token)
  that anyone who copies it can replay. Standard for service meshes (Istio, Linkerd) and zero-trust
  internal networks where "inside the perimeter" is no longer implicitly trusted.
- API gateways commonly terminate public TLS from the external client and re-establish mTLS to internal
  backend services, centralizing certificate/policy management at the gateway.
- **Per-endpoint authorization still applies behind the gateway** — in microservices, each service must
  independently authorize the call rather than assume an upstream gateway already did; mTLS proves
  identity, it does not itself decide authorization.

## Validation — uniform controls table (self-audit)

| # | Control | Implementation | Verified |
|---|---|---|---|
| 1 | Access control | Centralized chokepoint, deny-by-default (see `auth`) | [ ] |
| 2 | Injection | 100% parameterized queries / bound params; no raw string SQL | [ ] |
| 3 | Input validation | Server-side allowlist on every external input | [ ] |
| 4 | Secrets | Zero hardcoded secrets; centralized manager; rotation automated | [ ] |
| 5 | CORS | No wildcard+credentials; preflight answered explicitly | [ ] |
| 6 | Security headers | HSTS, CSP, nosniff, Referrer-Policy, Permissions-Policy present | [ ] |
| 7 | CSRF | Synchronizer token on all state-changing requests | [ ] |
| 8 | Transport | TLS everywhere including pre-auth pages; mTLS on internal calls if microservices | [ ] |
| 9 | Logging | Auth/validation failures logged; no raw secrets/tokens in logs | [ ] |
| 10 | SSRF | Outbound fetch destinations allowlisted | [ ] |

**Self-audit count: __ / 10 controls verified.** Report the count, not a narrative summary, when this
ref's work is handed off.

## Failure modes and handoff
- A control fails because the underlying identity/token/password decision was never made → back to
  `auth` first, this ref implements around that decision, it doesn't substitute for it.
- Rate limiting or DDoS-class throttling comes up mid-audit → route to `caching-and-performance`
  (token-bucket, Redis+Lua for distributed correctness) rather than inventing it here.
- The ask turns into "prove this holds under load/attack independently" → `quality`'s gate, not this
  ref's self-audit.
- ASVS level or Top-10 scope was never assigned → surface the gap explicitly; don't assume L3/all-10
  by default.
