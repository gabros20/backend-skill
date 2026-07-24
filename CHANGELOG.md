# Changelog

All notable changes to **backend** are documented here.

The release procedure synchronizes `.codex-plugin/plugin.json`, this changelog, git tag
`v<version>`, and the matching GitHub Release. Runtime `SKILL.md` contains no version metadata.

## [Unreleased]

### Added
- Queue unreleased changes here.

## [0.1.0] — 2026-07-24

### Added
- Initial release of the `backend` skill — the server-side Build stage of the Digital Product Skills
  family, peer of `frontend`. A faceted router over 9 primary jobs (api-implementation, domain-logic,
  persistence, auth, async-and-messaging, caching-and-performance, server-security,
  service-integration, backend-testing) and 5 runtime/system-type surface overlays (rest-crud,
  serverless-edge, realtime, event-driven-microservices, and the additive agentic overlay for
  LLM/agent backends). 15 references, 8 fillable assets (12-factor scaffold, zero-downtime migration
  checklist, auth + OWASP controls checklists, idempotency/outbox recipes, API-contract review,
  verification-loop template, `handoff.yaml`), and activation/traversal/output/compression evals.
  Retrieval-first, default-stack-then-deviate, implement-inside-the-decision; consumes architecture's
  contracts/budgets/threat-model and hands running services + telemetry hooks to `operate`/`quality`.

[0.1.0]: https://github.com/gabros20/backend-skill/releases/tag/v0.1.0
