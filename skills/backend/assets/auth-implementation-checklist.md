# Auth Implementation Checklist (fillable)

Fill in brackets. Source: `references/auth.md`. Use alongside `assets/owasp-backend-controls-checklist.md`
for the surrounding transport-layer controls.

## System
- Client type: [ ] confidential (server) [ ] public (SPA/mobile) [ ] both
- Mechanism chosen: [ ] session cookie [ ] short JWT + refresh [ ] BFF (SPA never holds token)
- Multi-tenant: [ ] yes [ ] no

## Password hashing (if passwords stored)
- Algorithm: [ ] Argon2id (default) [ ] scrypt (fallback) [ ] bcrypt (legacy only) [ ] PBKDF2 (FIPS only)
- Params used: m=____ MiB, t=____, p=____  (Argon2id min: m=19, t=2, p=1)
  bcrypt cost=____ (≥10; note 72-byte cap pre-hash if inputs may exceed it)
  scrypt N=____ r=____ p=____ (fallback default: N=2^17, r=8, p=1)
  PBKDF2 HMAC-____ @ ____ iterations (SHA256:600k / SHA512:210k)
- [ ] Unique random salt per password
- [ ] Constant-time verify (library function, not custom comparison)
- [ ] Stored in PHC string format (algorithm/params travel with hash)
- [ ] Rehash-on-login path exists for future param upgrades

## Session cookie (if session-based, or refresh-token cookie)
- [ ] `Secure` set
- [ ] `HttpOnly` set
- [ ] `SameSite=Strict` (or `Lax` with justification: __________)
- [ ] Non-persistent (no `Max-Age`/`Expires`) unless "remember me" explicitly required
- [ ] `Path` narrowed to app scope; `Domain` omitted unless subdomain sharing required
- [ ] Session ID ≥64 bits random, regenerated on login and on privilege change
- [ ] Idle timeout: ____ min · Absolute timeout: ____ h
- [ ] Logout invalidates server-side + emits `Clear-Site-Data`

## OAuth2/OIDC (if implementing a redirect/browser flow)
- Grant: [ ] Authorization Code + PKCE (browser/native) [ ] Client Credentials (service-to-service)
- [ ] PKCE method = S256 (never `plain`)
- [ ] Redirect URI checked by exact string match
- [ ] Implicit and ROPC grants NOT implemented
- [ ] Refresh token rotates on every use
- [ ] Reuse of a rotated-out refresh token revokes the whole token family
- [ ] Sender-constrained tokens (mTLS/DPoP) used where feasible
- OIDC ID token vs OAuth access token kept separate: [ ] authentication decisions use ID token only
  [ ] authorization decisions use access token/scopes only

## JWT validation (any JWT — access token or ID token)
- [ ] Algorithm pinned server-side at verify time (not read from the token's own `alg` header)
- [ ] `alg: none` explicitly rejected
- [ ] `exp` validated
- [ ] `nbf` validated
- [ ] `iss` validated against expected issuer
- [ ] `aud` validated against expected audience
- [ ] No sensitive data in the payload (JWT payload is base64, not encrypted)
- [ ] Token never written to `localStorage`/`sessionStorage`

## Authorization
- Model chosen: [ ] rule-strings [ ] RBAC [ ] RBAC+domain(tenant) [ ] ABAC (Cedar) [ ] ReBAC (OpenFGA/SpiceDB) [ ] Casbin (embedded)
- [ ] Enforcement is one centralized chokepoint (not scattered per-handler checks)
- [ ] Deny-by-default confirmed (missing rule = deny)
- [ ] Per-object recheck on every access (no IDOR — object-type access ≠ this-object access)

## Vendor/library
- Pick: [ ] Better-Auth (TS default) [ ] Clerk [ ] Auth0/Okta [ ] Ory (Kratos+Hydra) [ ] Keycloak [ ] other: ____
- [ ] Current official docs fetched before generating config/route code (staleness check)
