# Integration Access Control Specification

## Purpose

Define integration revisions, bindings, grants, selected skills, and non-administrative access keys.

## Requirements

### Requirement: Draft and Activation

Changes MUST remain in a draft until explicit activation. Consumers MUST use one active revision, and concurrent activation from one base MUST have exactly one winner. An unverified bound connection MUST allow activation and return one `connection_unverified` warning per binding. An invalid or archived bound connection MUST reject activation and preserve the active revision.

#### Scenario: Concurrent activation
- GIVEN two drafts share one active base
- WHEN both activate concurrently
- THEN one succeeds and one returns a stale-revision conflict

#### Scenario: Unverified connection
- GIVEN a valid draft binds an unverified connection
- WHEN activation succeeds
- THEN that binding is reported with `connection_unverified`

#### Scenario: Invalid connection
- GIVEN a draft binds an invalid or archived connection
- WHEN activation is requested
- THEN activation fails and the prior revision remains active

### Requirement: Explicit Pinned Bindings

An active integration MUST pin immutable server versions and exact connections, with at most one binding per server. New server or skill versions MUST require manual draft activation; latest-version resolution MUST NOT occur.

#### Scenario: Duplicate server binding
- GIVEN a draft already binds a server
- WHEN another connection for it is added
- THEN the draft is rejected without replacement

### Requirement: Tool Grants

Each binding MUST grant all operations or an explicit set. Read and mutating operations MUST follow identical grants without destructive-action confirmation. Promotion MUST preview, remove, and audit selected grants absent from the target version.

#### Scenario: Version promotion removes a grant
- GIVEN a selected operation is absent from a target version
- WHEN promotion activates
- THEN the grant is removed and its removal is visible and audited

### Requirement: Explicit Skill Selection

An integration MUST explicitly select and pin each skill version. Tool grants MUST NOT imply skill visibility.

#### Scenario: Unselected skill
- GIVEN a skill references only granted tools but is unselected
- WHEN skill search or retrieval occurs
- THEN the skill is not disclosed

### Requirement: Canonical Access-Key Scopes

Keys MUST belong to one workspace and integration, MUST be non-administrative, and MUST use `catalog:read` for catalog list/search, `tools:execute` for execution, and `skills:read` for skill search/get. REST and MCP MUST apply these scopes identically. Execute-by-name MUST NOT grant catalog list/search. Integrations MAY have unlimited active keys; new keys MUST default to 90-day expiry, and rotation overlap MUST be allowed.

#### Scenario: Execute without catalog scope
- GIVEN a key has `tools:execute`, the required grant, and no `catalog:read`
- WHEN it executes a known public name and then searches
- THEN execution succeeds and search returns `authorization_denied`

#### Scenario: Missing skill scope
- GIVEN a key lacks `skills:read`
- WHEN REST or MCP skill search/get is requested
- THEN `authorization_denied` is returned without skill disclosure

### Requirement: Per-Request Key Enforcement

Every REST and MCP request, including calls in open sessions, MUST revalidate key status, expiry, integration-active status, active revision, and required scope. Revocation, expiry, or archival MUST immediately return `authorization_denied`. Each key MUST allow at most 60 requests per minute, subject to accepted single-replica restart resets.

#### Scenario: Revoke open-session key
- GIVEN an MCP session uses an active key
- WHEN the key is revoked before its next call
- THEN that call returns `authorization_denied`

#### Scenario: Rate limit reached
- GIVEN a key has consumed 60 requests in its window
- WHEN another request arrives
- THEN it is rejected without reaching capability behavior
