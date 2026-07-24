# OWASP Backend Controls Checklist (fillable)

Uniform-table format per skill convention — fill "Implemented" and "Evidence," never narrate. Source:
`references/server-security.md` (OWASP Top 10:2021 + Cheat Sheet Series, CC BY-SA 4.0, paraphrased).

| # | OWASP category / control | Implementation required | Implemented (Y/N) | Evidence / location |
|---|---|---|---|---|
| A01 | Broken Access Control | Centralized authz chokepoint, deny-by-default, per-object recheck | | |
| A02 | Cryptographic Failures | Authenticated encryption (GCM/CCM) at rest+transit; no custom crypto | | |
| A03 | Injection | 100% parameterized queries; identifier allowlist where params can't bind | | |
| A04 | Insecure Design | Threat model / NFR budget confirmed upstream (`architecture`) before build | | |
| A05 | Security Misconfiguration | No default creds; verbose errors disabled in prod; minimal enabled features | | |
| A06 | Vulnerable/Outdated Components | Dependency audit run; no known-EOL packages in prod | | |
| A07 | Auth Failures | See `auth.md` — session/JWT + password-hash config verified separately | | |
| A08 | Software/Data Integrity Failures | CI/CD artifacts signed/verified; deserialization inputs validated | | |
| A09 | Logging/Monitoring Failures | Auth+validation failures logged; secrets/tokens never logged raw | | |
| A10 | SSRF | Outbound fetch destinations allowlisted; no raw user URL reaches internal net | | |
| — | Input validation | Server-side allowlist, syntactic+semantic, on every external input | | |
| — | Secrets management | Zero hardcoded secrets; centralized manager; rotation automated; dynamic/short-lived preferred | | |
| — | CORS | No wildcard `*` + credentials; preflight explicit; allowlisted origins | | |
| — | Security headers | HSTS, CSP, `X-Content-Type-Options: nosniff`, Referrer-Policy, Permissions-Policy | | |
| — | CSRF | Synchronizer token on all state-changing requests; `SameSite` as complement | | |
| — | Transport / mTLS | TLS on every page incl. pre-auth; mTLS internal if microservices | | |

**Self-audit count: ____ / 16 controls implemented and evidenced.**

Anything below 16/16 must be either fixed before ship or explicitly waived with a stated reason and
owner — do not ship a silent gap. ASVS level applied (if scoped upstream): L____.
