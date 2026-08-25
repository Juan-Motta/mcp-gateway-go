## Exploration: API-to-MCP Platform

### Current State

The repository is an empty application scaffold. CodeGraph found no source symbols, and the filesystem contains no `go.mod`, application source, web application, database migrations, tests, test runner, quality tooling, or CI. The only project artifacts relevant to implementation are the MIT license and the initialized OpenSpec structure. Strict TDD is therefore disabled until a test runner is established.

The following are **accepted product decisions**, not topics to reopen without new evidence:

- Build a simplified multi-tenant API-to-MCP platform as a Go modular monolith with a Next.js App Router control plane, PostgreSQL, Docker Compose, and a reverse proxy.
- Use shared-schema workspaces, multi-workspace users, owner/admin roles, last-owner protection, email/password authentication, Argon2id password hashes, PostgreSQL-backed secure cookie sessions, and invitations.
- Import OpenAPI 3.0/3.1 JSON or YAML; normalize supported path/query/header/body JSON operations; preview drafts; publish immutable versions; defer files, multipart, streaming, callbacks, webhooks, and complex composition.
- Keep generated tool definitions and typed secret-free `AuthSpec` data on server versions. Keep encrypted concrete credentials on workspace-scoped connections.
- Model an integration as explicit server-version/connection/tool-grant bindings. Bind each hashed, workspace-scoped, non-administrative access key to one integration.
- Use PostgreSQL full-text search with authorization applied before ranking or counting. Defer pgvector until evidence justifies it.
- Use the official MCP Go SDK and Streamable HTTP for the negotiated lifecycle, `tools/list`, `tools/call`, `search_tools`, `execute_tool`, `skills_search`, and `skills_get`; exclude MCP resources and prompts.
- Provide JSON logs, correlation IDs, health, readiness, metrics, sanitized invocations, and audit records from the first production slice.
- Run one API replica initially. Keep the sandbox isolated and post-MVP. Do not add Redis, Kubernetes, RabbitMQ, Elasticsearch, warm pools, plugins, a marketplace, or AI schema generation.

The implementation structure, transaction boundaries, identifier formats, draft lifecycle details, invitation policy, integration edge cases, skill visibility rules, access-key lifecycle, and upstream network policy remain open. The supplied size, timeout, retention, rate, and concurrency values are accepted configurable defaults, but their exact enforcement semantics still require proposal/spec decisions.

### Affected Areas

All implementation paths below are **recommended future boundaries**; none exists yet.

- `go.mod` — establish the Go module and dependency baseline.
- `cmd/api/` — compose the HTTP server, MCP transport, lifecycle, configuration, and graceful shutdown.
- `internal/identity/` and `internal/tenancy/` — users, passwords, sessions, invitations, memberships, roles, and last-owner invariants.
- `internal/servers/` — OpenAPI ingestion, normalization, preview, immutable versions, generated tool names, and secret-free auth requirements.
- `internal/connections/` and `internal/execution/` — encrypted credentials, safe request construction, upstream policy, limits, and sanitized invocation records.
- `internal/integrations/` and `internal/accesskeys/` — explicit bindings, grants, scopes, key lifecycle, authorization, rate limits, and concurrency limits.
- `internal/catalog/`, `internal/mcp/`, and `internal/skills/` — authorized FTS, MCP method adapters, and versioned declarative skills.
- `internal/platform/` — PostgreSQL, encryption, configuration, observability, audit, and shared HTTP concerns; this must not become a business-logic dumping ground.
- `db/migrations/` — shared-schema tables, tenant-safe foreign keys, indexes, FTS vectors, retention support, and transactional constraints.
- `web/` — Next.js App Router human control plane; administrative operations use human sessions rather than integration access keys.
- `compose.yaml` and `deploy/proxy/` — core `postgres`, `api`, `web`, and `proxy` services plus optional observability and later sandbox profiles.
- `openspec/changes/api-to-mcp-platform/` — future proposal, delta specs, design, tasks, and verification artifacts after this exploration.

### Approaches

1. **Domain-first modular monolith** — organize Go packages around identity/tenancy, servers, connections/execution, integrations/access keys, catalog/MCP, and skills; expose use cases through explicit application interfaces and keep infrastructure adapters at the edges.
   - Pros: Preserves business boundaries, supports incremental vertical slices, limits accidental coupling, and allows later extraction only where evidence exists.
   - Cons: Requires disciplined dependency direction and some deliberate duplication between domain-specific models.
   - Effort: Medium

2. **Layer-first monolith** — organize globally by handlers, services, repositories, and models.
   - Pros: Familiar, quick to scaffold, and initially simple for a very small team.
   - Cons: Cross-domain changes spread across global layers, tenant authorization becomes easier to bypass, and the monolith tends toward a shared service/model core.
   - Effort: Low initially, High as the product grows

3. **Selector/verifier PostgreSQL sessions** — place a random selector and verifier in the secure cookie, index the selector, store only a verifier hash, rotate on login and privilege-sensitive events, and revoke server-side.
   - Pros: Efficient lookup, database-backed revocation, no recoverable bearer token at rest, and clean support for session expiry and device lists later.
   - Cons: More lifecycle logic than storing a single opaque token hash, and rotation/race behavior must be specified.
   - Effort: Medium

4. **Single opaque-token PostgreSQL sessions** — place one random token in the cookie and store its hash with session metadata.
   - Pros: Small model, straightforward revocation, and still consistent with PostgreSQL-backed sessions.
   - Cons: Requires a lookup strategy such as a separate identifier or keyed hash; naïve hashing can make indexed lookup or breach resistance awkward.
   - Effort: Low to Medium

5. **Source-plus-canonical OpenAPI versioning** — retain the uploaded source for audit/debugging, normalize once into a canonical draft, preview that model, and transactionally publish an immutable snapshot containing frozen operations, auth requirements, and tool names.
   - Pros: Reproducible execution, stable tool identity, fast reads, inspectable import diagnostics, and no behavior drift after parser upgrades.
   - Cons: Requires a versioned canonical model and explicit re-import/migration behavior when normalization evolves.
   - Effort: High

6. **Source-only with on-demand normalization** — retain the uploaded document and regenerate operations for previews and runtime reads.
   - Pros: Less initial persistence modeling and easier parser replacement.
   - Cons: Parser upgrades can silently change published behavior and names, runtime work increases, and immutable publication becomes difficult to prove.
   - Effort: Medium initially, High operationally

7. **Explicit integration bindings** — persist each integration entry as a workspace-safe tuple of server, immutable version, connection, and `all_tools` or selected-tool grants; resolve access keys only through those bindings.
   - Pros: Deterministic behavior, auditable authorization, safe version pinning, and support for multiple servers or credentials without inference.
   - Cons: Publication and connection replacement need explicit rebinding workflows, and duplicate-server semantics must be decided.
   - Effort: Medium

8. **Implicit latest-version/current-connection resolution** — store server references and infer the latest published version and current connection at execution time.
   - Pros: Fewer binding records and automatic uptake of changes.
   - Cons: Changes behavior without integration review, weakens auditability, complicates selected-tool grants, and risks credential mix-ups.
   - Effort: Low initially, High risk

9. **Application authorization plus database tenant invariants** — require workspace context in every use case and repository method, include `workspace_id` in tenant tables and composite foreign keys, and authorize inside the same transaction as mutations and search.
   - Pros: Explicit behavior in Go, portable migrations, testable authorization paths, and strong protection against cross-workspace references.
   - Cons: Every query must follow the convention; missing predicates remain a serious review risk.
   - Effort: Medium

10. **PostgreSQL RLS from the first slice** — set transaction-local workspace/user context and enforce tenant visibility primarily through row-level security policies.
   - Pros: Adds database-level defense in depth and can protect ad hoc query paths.
   - Cons: Connection-pool context, background jobs, migrations, owner bypass, and policy testing add substantial complexity to an otherwise empty scaffold.
   - Effort: High

### Recommendation

Use the **domain-first modular monolith**, **selector/verifier PostgreSQL sessions**, **source-plus-canonical OpenAPI versioning**, **explicit integration bindings**, and **application authorization backed by composite tenant invariants**. Treat PostgreSQL RLS as a later defense-in-depth decision after repository boundaries and tenant tests are mature; do not use its absence to weaken SQL-level workspace scoping.

Build one Go process with separate adapters for human REST, access-key REST, and MCP Streamable HTTP over the same application use cases. The Next.js application should remain a control plane, not a second business-logic authority. Centralize credential encryption/decryption behind a narrow service using the accepted base64 32-byte key, `*_FILE` loading, AES-256-GCM, authenticated metadata, and production fail-fast behavior. Never return decrypted credentials to the web tier.

Freeze canonical operation identifiers and `<server_slug>__<operation_slug>` tool names at publication. Use normalized source fields for deterministic fallback and collision suffixes rather than database insertion order. Bind integrations to immutable published versions and concrete connections so `tools/list`, search, and execution all derive from the same authorization graph. Apply workspace, integration, grant, scope, revocation, and expiry predicates before FTS ranking/counting or tool lookup.

For the one-replica MVP, in-process per-workspace semaphores and token buckets are coherent for the accepted concurrency and access-key rate defaults if restart resets and single-replica semantics are explicitly accepted. Keep the interfaces replaceable before horizontal scaling. Persist invocation and audit events independently of log output, with size-bounded sanitized fields and retention cleanup.

The broad MVP direction is proposal-ready as an **umbrella plan**, but it is too large for one implementation/review unit. With an 800 changed-line review budget and `ask-on-risk`, the first deliverable should be a tenancy foundation: project/tooling bootstrap, configuration and observability skeleton, PostgreSQL migrations, user/session/workspace/membership/invitation rules, last-owner protection, and a minimal authenticated control-plane boundary. OpenAPI and MCP behavior should follow in later independently reviewable slices. The proposal round must confirm whether `api-to-mcp-platform` is the umbrella change or whether it should be narrowed to that first slice.

### Risks

- Tenant isolation is the primary systemic risk. Cross-workspace references must be prevented by schema constraints as well as authorization checks, including search counts and audit access.
- User-supplied upstream URLs create SSRF, DNS-rebinding, private-network, redirect, and TLS-policy risks. Disabling redirects alone is insufficient.
- OpenAPI 3.0/3.1 normalization can drift across parser versions; unsupported composition and media types need explicit diagnostics rather than partial silent imports.
- Tool names become public API once published. Slug changes, duplicate operation IDs, path collisions, and server renames need immutable deterministic behavior.
- Last-owner checks, invite acceptance, role changes, key revocation, and integration rebinding are race-prone and require transactional locking/idempotency rules.
- Encrypted credentials need key identification and an eventual rotation/re-encryption story; production fail-fast prevents missing-key startup but does not solve rotation.
- Sanitized invocation/audit data can still leak secrets through URLs, bodies, headers, or upstream errors unless redaction and truncation are schema-aware and deny-by-default.
- In-memory limits and MCP session state fit one replica but are not horizontally safe. The proposal must label that boundary rather than imply scale-out support.
- The complete MVP will greatly exceed 800 changed lines, so later task planning will require explicit review slicing before apply.

### Ready for Proposal

Yes. The accepted direction is coherent, and the recommended architecture provides a safe path from an empty scaffold. The interactive proposal round should ask the following **one at a time**, starting with scope because it changes every later answer:

1. Is `api-to-mcp-platform` an umbrella change that documents the full MVP while delivery is split into sub-changes, or must this named change itself stop after the tenancy-foundation first slice?
2. What creates the first workspace: automatic creation at signup, explicit post-signup creation, or invitation-only onboarding; and who may invite, choose roles, revoke invites, and transfer/delete ownership?
3. May an integration bind the same server more than once through different connections, and what should happen to existing bindings and selected-tool grants when a new server version is published?
4. Is one mutable draft per server sufficient, and are published versions permanently retained even after a server or integration is archived?
5. Which upstream hosts are allowed in MVP: public HTTPS only, explicit workspace allowlists, or private-network targets; who accepts the SSRF implications?
6. Are skills explicitly attached/granted per integration, or are all workspace skills visible when their linked tools are granted; can skills link across multiple server bindings?
7. How many active access keys may an integration have, what is the default/max expiry, is rotation overlap allowed, and is the 60 requests/minute limit per key, integration, or workspace?
8. Are in-memory rate/concurrency enforcement and restart resets acceptable for the declared one-replica MVP, or must these limits be durable from the first release?
9. What are the archive/delete semantics and retention exceptions for users, workspaces, servers, connections, integrations, invocations, and audit records, especially when immutable published versions remain referenced?

The proposal should record answers as business rules and explicit out-of-scope boundaries. It should not choose test commands, PR mechanics, or detailed library APIs during that question round.
