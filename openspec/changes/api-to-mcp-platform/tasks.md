# Tasks: API-to-MCP Platform

## Review Workload Forecast

Total estimated changed-line range: 11,800–16,170  
Maximum per-unit estimate: 800 (production/tests/migrations/fixtures/config/docs)  
Total work units: 23  
Delivery: `auto-chain`; tracker-accumulates; child-1→tracker; later-child→predecessor; tracker-only→main

Decision needed before apply: No
Chained PRs recommended: Yes
Chain strategy: feature-branch-chain
400-line budget risk: High
Next apply slice: Unit A

Each checkbox is one child-PR/commit/review boundary: RED→minimal-GREEN→verify/runtime→record-results; parentheses=dependencies/lines; final-clause=rollback; no-exceptions.

## Milestones

- **Core parity:** A–P unchanged: deployable shared REST/MCP behavior.
- **Normative MVP:** Q0/Q1/R/S/T mandatory; 23 units; no MUST deferred.

## Phase 1: Identity/Audit

- [ ] Unit A — Harness/identity/audit (-;500–700): migration/audit→`go.mod`,`cmd/`,`internal/{config,postgres,audit}`,`0001_*`; verify/runtime `go test ./...`; PostgreSQL-migrate/audit; rollback binary; empty-only-down/forward-fix.
- [ ] Unit B — Setup/registration (A;650–800): setup/registration/session-race/reset→`internal/{identity,httpapi}`,`0002_*`; verify/runtime `go test ./internal/{identity,httpapi}/...`; migrate/setup/login/reset; rollback routes+`0002_*`.
- [ ] Unit C — Invitations/ownership (B;600–780): invitation-expiry/single-use/promotion/owner-race→`internal/identity`; verify/runtime `go test ./internal/identity/...`; invite/transfer-race; rollback identity-delta.
- [ ] Unit D — Disablement/audit-safe-deletion (C;500–700): disablement/last-owner/anonymized-audit→`internal/{identity,audit}`; verify/runtime `go test ./internal/{identity,audit}/...`; archive/delete-API; rollback handlers; retain-audit.

## Phase 2: Servers/Execution

- [ ] Unit E — OpenAPI-import (A;550–750): OpenAPI3.0/3.1-JSON/YAML/5MiB/500-op/unsupported-constructs-auth→`internal/server`,`0003_*`; verify/runtime `go test ./internal/server/...`; PostgreSQL-import; rollback importer+`0003_*`.
- [ ] Unit F — Publication/catalog-search (E;600–800): publication-race/immutability/deterministic-names/HTTPS/FTS→`internal/server`,`0004_*`; verify/runtime `go test ./internal/server/...`; import/publish/search; rollback publisher+`0004_*`.
- [ ] Unit G — Exact-connections/rotation (A,F;500–700): tenant-denial/encryption/identity-preserving-rotation→`internal/connection`,`0005_*`; verify/runtime `go test ./internal/connection/...`; create/rotate/read; rollback vault+`0005_*`.
- [ ] Unit H — Upstream-transport-safety (G;600–800): explicit-validation/unsafe-DNS-rebinding/redirect/retry/2MiB/5s/30s→`internal/{connection,upstream}`; verify/runtime `go test ./internal/{connection,upstream}/...`; controlled-TLS/DNS; rollback transport/validation.
- [ ] Unit I — Auth/invocation-records (H;600–800): exact-none/API-key-header-query/bearer/Basic/no-fallback/sanitized-metadata-only-invocation→`internal/{executor,invocation}`,`0006_*`; verify/runtime `go test ./internal/{executor,invocation}/...`; TLS-execution; rollback executor+`0006_*`; retain-audit.
- [ ] Unit I2 — Workspace-concurrency (I;350–520): workspace-slot-11/release/reset→`internal/executor/concurrency`; verify/runtime `go test ./internal/executor/...`; 11-call/restart; rollback limiter-wiring.

## Phase 3: Shared Consumer Core

- [ ] Unit J — Integration-bindings (D,F,G;650–800): activation-race/exact-pins/one-binding-per-server/grants/promotion-audit/warnings/skill-pins→`internal/integration`,`0007_*`; verify/runtime `go test ./internal/integration/...`; bind/activate/promote; rollback module+`0007_*`; retain-audit.
- [ ] Unit K — Access-key-auth/limits (J;500–700): REST/MCP-scopes/90-day/unlimited/overlap/per-call-revoke-expiry-archive/60rpm→`internal/accesskey`,`0008_*`; verify/runtime `go test ./internal/accesskey/...`; issue/rotate/revoke; rollback module+`0008_*`.
- [ ] Unit L — Shared-REST-catalog/execution (I2,J,K;650–800): shared-REST-list-search-execute/unfiltered-multi-binding/alias-only/403/exact-operation-binding-connection/audit-only-denial/one-invocation→`internal/{catalog,consumer,httpapi}`; verify/runtime `go test ./internal/{catalog,consumer,httpapi}/...`; REST-flow; rollback consumer/catalog-routes.
- [ ] Unit M1 — Skill-lifecycle (J,L;450–650): immutable-skills/explicit-pins/exact-get/grant-filtered-references/lifecycle→`internal/skill`,`0009_*`; verify/runtime `go test ./internal/skill/...`; publish/select/get; rollback module+`0009_*`.
- [ ] Unit M2 — Authorized-skill-ranking (M1;350–520): authorized-FTS/rank-desc/immutable-ID-tie/unauthorized-isolation→`internal/skill/search.go`; verify/runtime `go test ./internal/skill/...`; PostgreSQL-search; rollback search-query/API.
- [ ] Unit N — MCP parity (K,L,M2;650–800): ambiguous-credentials/forged-tenant/per-call-revocation/LIST/SEARCH_EXECUTE/AUTO-20/deterministic-direct-meta-skill-tools/parity→`internal/mcp`; verify/runtime `go test ./internal/mcp/...`; initialize/list/search/execute; rollback `/mcp`-route.

## Phase 4: Deployment/MVP

- [ ] Unit O — Operations-baseline (A,N;400–600): correlated-redaction/protected-metrics/readiness→`internal/operations`; verify/runtime `go test ./internal/operations/...`; health/readiness/metrics; rollback endpoints/logging.
- [ ] Unit P — Deployment (O;500–750): image/proxy/one-replica→`deploy/`,`compose.yaml`; verify/runtime `docker compose config`; `docker compose up --build`; rollback deployment-artifacts.
- [ ] Unit Q0 — UI foundation and approved designs (identity-behavior-specified;150–350 textual): low-fidelity flows→visual foundation/tokens/components→high-fidelity responsive screens→Pencil structural readback (complete proportional design-verification check)→accessibility/state review→explicit human approval in `design/ui-foundation.pen`,`docs/ui-foundation.md`; desktop/mobile scope: setup/registration/login/password-reset/invitation-acceptance/workspace-shell plus loading/empty/validation-error/authorization-error/success only; runtime N/A — passive design artifact; canvas not line-counted; rollback only design/contract artifacts.
- [ ] Unit Q1 — Setup/login web implementation (B,P,Q0-approved;550–750): faithfully implement approved artifacts in `web/`; flow/design change returns to Q0—no improvisation; verify/runtime `npm --prefix=web test:e2e`; browser approved-flow-E2E; rollback only web implementation.
- [ ] Unit R — Live-MCP-notifications (J,N;500–700): advertised-standard-list-change/changed-notify/unchanged-silence/revoked-nondisclosure→`internal/{mcp,integration}/session*`; verify/runtime `go test ./internal/{mcp,integration}/...`; live-activation; rollback registry/hook.
- [ ] Unit S — Retention (A,I,O;450–650): 30-day-invocation/365-day-audit/audit-survival/retry→`internal/operations/retention`,`cmd/api`; verify/runtime `go test ./internal/operations/...`; scheduled-cleanup; rollback scheduler/queries; retain-audit.
- [ ] Unit T — Audit-safe-purge (D,F,G,J,M1,S;550–750): 30-day-grace/references/purge-race→`internal/operations/purge`; shared-parent-`FOR UPDATE`; verify/runtime `go test ./internal/operations/...`; concurrent-cleanup/reference; rollback worker; retain-audit/references.
