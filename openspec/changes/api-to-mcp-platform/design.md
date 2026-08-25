# Design: API-to-MCP Platform

## Technical Approach

Go modular monolith: shared REST/MCP consumer pipeline, PostgreSQL invariants/cleanup, and an approved UI contract before Next.js implementation.

## Architecture Decisions

| Option | Tradeoff | Decision and rationale |
|---|---|---|
| Shared pipeline | Coupling | One authorization/resolution/execution/audit path preserves parity. |
| Canonical publication | Split workflow | Retain diagnostics; freeze operations; reject >5 MiB/>500; support none, API-key header/query, bearer, Basic. |
| Fixed egress | No recovery | Public HTTPS; no redirects/retries; 2 MiB bodies; 5 s connect/30 s total. |
| In-memory limits | Reset | Single-replica limiters avoid coordination; resets remain observable. |
| Approved UI foundation | Human gate | Prevents improvised flows and makes responsive/accessibility acceptance reviewable. |

## Data Flow and Contracts

    REST/MCP → key revalidation → authorized graph → resolve → workspace limiter → HTTPS → invocation
                              activation → session registry → reauthorize/recalculate → list_changed
    cleanup tick → transaction/parent lock → cutoff+references recheck → delete or retain

Denials: audit only; no invocation/upstream. Search: authorized bindings; aliases narrow; unauthorized aliases disclose nothing. Execution: one metadata-only invocation. LIST: operations. SEARCH_EXECUTE: `search_tools`/`execute_tool`. AUTO: operation count, default 20. Skill tools: `skills:read`+pins, excluded from AUTO. Skill FTS: rank DESC, `skill_version_id` ASC; unauthorized rows affect no matches/totals/rank.

## Append-Only Schema Plan

Migrations `0001`–`0009` own identity/audit, sessions, drafts, publications, connections, invocations, integrations, keys, skills. Corrections append. `0001` down requires unused/audit-empty deployment; otherwise forward-correct preserving records.

## Review Work Units

Twenty-three autonomous units include code/tests/fixtures/docs/config/migrations.

- **A (none), 500–700:** Harness+identity/audit: `go.mod`, `cmd/`, `internal/{config,postgres,audit}`, `0001_*`; migration/audit tests; revert binary; down only empty, else audit-safe forward-fix.
- **B (A), 650–800:** Setup/registration/sessions: `internal/{identity,httpapi}`, `0002_*`; setup/session races; remove routes+0002.
- **C (B), 600–780:** Membership/invitations/owner locks: `internal/identity`; ownership races; revert delta.
- **D (C), 500–700:** Archive/anonymize: `internal/{identity,audit}`; disablement/anonymity; revert handlers.
- **E (A), 550–750:** OpenAPI normalize/diagnose: `internal/server`, `0003_*`; limit/auth fixtures; remove importer+0003.
- **F (E), 600–800:** Publish/stable names/FTS: `internal/server`, `0004_*`; collision/races; remove publisher+0004.
- **G (A,F), 500–700:** Connection vault/rotation: `internal/connection`, `0005_*`; crypto/tenancy; remove vault+0005.
- **H (G), 600–800:** Validation/one-attempt transport: `internal/{connection,upstream}`; RED DNS/redirect/body/deadline; remove transport.
- **I (H), 600–800:** Exact-auth executor/invocations: `internal/{executor,invocation}`, `0006_*`; auth/one-record; remove executor+0006.
- **I2 (I), 350–520:** Ten-slot limiter: `internal/executor/concurrency`; RED 11th rejects pre-transport, release on success/error/panic/cancel, restart reset; remove wiring.
- **J (D,F,G), 650–800:** Revisions/bindings/grants/activation: `internal/integration`, `0007_*`; activation races; remove module+0007.
- **K (J), 500–700:** Scoped keys/rotation/rate-limit: `internal/accesskey`, `0008_*`; expiry/per-request; remove module+0008.
- **L (I2,J,K), 650–800:** Authorized FTS/REST parity: `internal/{catalog,consumer,httpapi}`; alias/multi-binding/parity; remove service.
- **M1 (J,L), 450–650:** Skill versions/pins/get: `internal/skill`, `0009_*`; pin/disclosure; remove module+0009.
- **M2 (M1), 350–520:** Authorized skill FTS: `internal/skill/search.go`; RED rank/tie stability and unauthorized-match isolation; remove search API/index query.
- **N (K,L,M2), 650–800:** `/mcp` modes/direct/meta/skill tools: `internal/mcp`; mode/revocation parity; remove route.
- **O (A,N), 400–600:** Safe logs/health/readiness/metrics: `internal/operations`; dependency/redaction; remove endpoints.
- **P (O), 500–750:** Images/proxy/one-replica Compose: `deploy/`, `compose.yaml`; Compose/scale; remove deployment.
- **Q0 (identity specified):** Planned editable `design/ui-foundation.pen`, `docs/ui-foundation.md`; readback/review/approval; remove only both artifacts. Markdown/support 150–350; canvas bounded below.
- **Q1 (B,P,Q0 approved), 550–750:** Faithful approved control plane: `web/`; contract Playwright; remove only web.
- **R (J,N), 500–700:** List-change sessions: `internal/{mcp,integration}/session*`; RED changed emits standard notification, unchanged does not, revoked discloses nothing; remove registry/hook.
- **S (A,I,O), 450–650:** 30/365-day retention: `internal/operations/retention`, `cmd/api`; RED cutoffs, audit survives invocation deletion, transaction retry; disable scheduler/revert queries.
- **T (D,F,G,J,M1,S), 550–750:** 30-day purge: `internal/operations/purge`; RED grace/reference/race; transaction and reference creation lock parent `FOR UPDATE`; disable worker.

### Q0 UI Approval Gate

Design-only workflow: requirements → low-fidelity flows → visual foundation → high-fidelity responsive screens/components → Pencil structural readback plus accessibility/state review → explicit human approval. `docs/ui-foundation.md` defines information architecture, route/flow map, design tokens, component inventory, responsive breakpoints, accessibility requirements, and state matrix.

Desktop/mobile approval covers setup, registration, login, password reset, invitation acceptance, workspace shell, loading, empty, validation-error, authorization-error, and success states only. Canvas complexity is not line-counted; named screens/states cap it. Markdown/support forecasts 150–350 changed lines.

Q0 starts after identity behavior is specified/available and may parallel unrelated backend units. Q1 requires Q0 approval plus B/P. Its visual/accessibility acceptance source is the approved artifacts; drift returns to Q0—no improvised flows. Q0 rollback removes only design/contract artifacts; Q1 removes only web implementation.

## Testing and Threat Matrix

Unit A establishes `go test ./...`, `httptest`, Docker PostgreSQL; implementation units begin RED.

HTTP/MCP routing — **Applicable:** ambiguous credentials/forged tenant fail pre-capability; RED in N.  
Outbound HTTP — **Applicable:** unsafe DNS, redirects, retries, sizes, deadlines fail in one attempt; RED in H.  
Documentation paths — **N/A:** no executable-file classification.  
Git selection — **N/A:** no VCS automation.  
Commit state — **N/A:** no commit automation.  
Push state — **N/A:** no push automation.  
PR commands — **N/A:** no PR automation.

## Migration / Rollout

Feature-branch-chain: no-merge tracker; child 1 targets it, later children their predecessor; only tracker merges. A–P delivers core/deployment; Q0 gates Q1; unrelated R–T follow existing dependencies. Q1/R–T complete MVP. Rollbacks preserve audit.
