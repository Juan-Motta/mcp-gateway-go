# Proposal: API-to-MCP Platform

## Intent

Define the umbrella MVP for turning OpenAPI services into secure, workspace-scoped MCP integrations, delivered in independent slices under the 800-line budget; tenancy/authentication comes first.

## Scope

### In Scope
- Go domain-first modular monolith, Next.js control plane, PostgreSQL, reverse proxy, and Compose baseline.
- Single `/mcp`; one API replica with accepted in-memory limits and restart resets.

### Out of Scope
- Sandbox runner, OAuth2 upstream, semantic/vector search, horizontal MCP scaling, RLS, and heavy observability.
- Files/multipart, streaming APIs, callbacks, webhooks, resources, prompts, plugins, marketplace, and AI schema generation.

## Capabilities

### New Capabilities
- `identity-tenancy-lifecycle`: Setup, email/password sessions, registration (off by default), automatic personal workspace, memberships, invitations, ownership safeguards, archival, and anonymization; no email verification.
- `openapi-server-versioning`: OpenAPI 3.0/3.1 import into one canonical draft and immutable published versions, deterministic stable server/tool slugs, and public-HTTPS-only policy metadata.
- `connection-execution-security`: Workspace connections, encrypted secrets, explicit validation, no credential fallback, safe HTTP execution, sanitized errors, and metadata-only invocations.
- `integration-access-control`: Draft/activation, pinned versions, one connection per server, all/selected grants for read and mutating tools, explicit skill selection, and scoped keys with revocation, rotation overlap, unlimited active count, and 90-day default expiry.
- `mcp-tool-catalog`: Authorization-before-PostgreSQL-FTS; LIST, SEARCH_EXECUTE, and AUTO (default configurable threshold 20); Streamable HTTP `tools/list`, `tools/call`, `search_tools`, `execute_tool`, and live list-change notifications.
- `declarative-skills`: Versioned skills pinned by integrations and exposed through authorized `skills_search` and `skills_get`.
- `platform-operations`: JSON logs, health/readiness/metrics, audit, retention, archive/purge, and Compose operations; 30-day purge grace, 30-day invocations, and 365-day audit.

### Modified Capabilities
None.

## Approach

Use shared-schema scoping with composite tenant invariants, selector/verifier PostgreSQL sessions, source-plus-canonical publication, explicit bindings, AES-256-GCM credentials, and the official MCP Go SDK. Deliver vertical slices.

## Affected Areas

| Area | Impact | Description |
|---|---|---|
| `cmd/api/`, `internal/`, `db/migrations/` | New | Domain, transport, persistence, security |
| `web/` | New | Human control plane |
| `compose.yaml`, `deploy/proxy/` | New | Operational baseline |

## Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| Tenant leakage or SSRF | High | Transactional authorization, tenant-safe constraints, public HTTPS enforcement |
| Umbrella scope overwhelms review | High | Independent vertical slices; ask on budget risk |

## Rollback Plan

Roll back each slice independently; disable routes/workers and revert binaries and migrations with paired down migrations or forward corrections, retaining audit records.

## Dependencies

- Go, Next.js, PostgreSQL, official MCP Go SDK, Docker Compose, reverse proxy.

## Success Criteria

- [ ] Every capability has downstream normative specs and slice-level verification.
- [ ] Cross-workspace access, credential fallback, private/non-HTTPS execution, and revoked-key use fail closed.
- [ ] Published tools remain deterministic; authorized MCP discovery and execution reflect activated catalog changes.
