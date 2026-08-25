# Connection Execution Security Specification

## Purpose

Define workspace credentials and safe upstream execution; sandboxing and payload capture are excluded.

## Requirements

### Requirement: Workspace-Scoped Connections

Connections MUST belong to one workspace, protect secrets, and never expose decrypted credentials. Multiple connections per server MAY exist. Secret-only rotation MUST preserve identity; account or endpoint changes MUST create a new connection.

#### Scenario: Cross-workspace connection
- GIVEN a connection belongs to another workspace
- WHEN read, bind, validate, or archive is attempted
- THEN access returns `authorization_denied` without details

### Requirement: Explicit Validation

New connections MUST start unverified and MUST be tested only by explicit validation. Activation MUST follow `integration-access-control`; invalid or archived connections MUST fail execution.

#### Scenario: Invalid connection
- GIVEN the exact bound connection is invalid
- WHEN execution is requested
- THEN execution fails before upstream contact

### Requirement: Exact Authentication Satisfaction

The MVP MUST support none, API key header, API key query, bearer, and Basic authentication; OAuth2 and all others MUST be unsupported. Execution MUST use the exact connection bound to the pinned server version. One declared security alternative is satisfied only when it is fully supported and that connection supplies every credential in it. No shared, most-recent, other-connection, or fallback credentials MAY be used.

#### Scenario: Exact bound credentials
- GIVEN one complete supported alternative is present on the bound connection
- WHEN an authorized operation executes
- THEN only those bound credentials are applied

#### Scenario: Incomplete credentials
- GIVEN an alternative requires two credentials and the bound connection has one
- WHEN execution is requested
- THEN `incomplete_credentials` is returned before upstream contact

#### Scenario: Unsupported authentication
- GIVEN no declared alternative is fully supported
- WHEN execution is requested
- THEN `unsupported_auth` is returned before upstream contact

### Requirement: Single Public HTTPS Attempt

Before the sole outbound attempt, the platform MUST reject non-HTTPS, private, loopback, link-local, or other non-public destinations and limit construction to the published contract. Redirect following and automatic retries MUST be disabled.

#### Scenario: Non-public resolution
- GIVEN an eligible hostname resolves non-publicly at execution time
- WHEN execution is attempted
- THEN it is denied before network contact

#### Scenario: Redirect response
- GIVEN the upstream returns a redirect
- WHEN the response is received
- THEN no redirect is followed and `upstream_redirect` is returned

#### Scenario: Retryable failure
- GIVEN the sole attempt returns a retryable failure
- WHEN the failure is handled
- THEN no retry occurs and a sanitized upstream error is returned

### Requirement: Safe Results and Fixed Limits

Errors MUST contain only allowed status, safe message, category, and trace identifier. Invocations MUST contain metadata and sanitized errors only. Request and response bodies MUST each be limited to 2 MiB. Connect timeout MUST be 5 seconds; total timeout MUST be 30 seconds. At most ten workspace executions MAY run concurrently.

#### Scenario: Oversize input
- GIVEN serialization would produce a body over 2 MiB
- WHEN execution is requested
- THEN `input_too_large` is returned before upstream contact

#### Scenario: Oversize output
- GIVEN an upstream body exceeds 2 MiB
- WHEN it is consumed
- THEN consumption stops and `upstream_response_too_large` is returned

#### Scenario: Timeout
- GIVEN connection exceeds 5 seconds or execution exceeds 30 seconds
- WHEN the deadline expires
- THEN the attempt is canceled with `upstream_timeout` and no retry

#### Scenario: Concurrent limit
- GIVEN ten workspace executions are active
- WHEN another requests capacity
- THEN it is rejected without upstream contact

### Requirement: Connection Archival

Archival MUST be blocked while an active integration binds the connection. After dependencies are removed, archival MUST make it unusable immediately; purge follows `platform-operations`.

#### Scenario: Archive bound connection
- GIVEN an active integration binds a connection
- WHEN archival is requested
- THEN archival is denied until the dependency is removed
