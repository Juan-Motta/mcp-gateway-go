# MCP Tool Catalog Specification

## Purpose

Define authorized discovery and execution at `/mcp`; resources, prompts, vector search, and horizontal MCP scaling are excluded.

## Requirements

### Requirement: Key-Derived Context

The endpoint MUST derive workspace, integration, active revision, capabilities, and scopes solely from the presented key and MUST NOT accept client-selected tenant context. It MUST support Streamable HTTP, `tools/list`, and `tools/call`.

#### Scenario: Forged tenant context
- GIVEN a client supplies another workspace identifier
- WHEN it makes an MCP request
- THEN the request fails with sanitized `authorization_denied` without disclosure

### Requirement: Catalog Modes

Integrations MUST support LIST, SEARCH_EXECUTE, and AUTO. LIST MUST expose authorized direct operation tools; SEARCH_EXECUTE MUST expose lowercase `search_tools` and `execute_tool`. AUTO MUST select LIST at or below its configurable operation threshold and SEARCH_EXECUTE above it; the environment default MUST be 20 and an integration MAY override it. Mode MUST NOT change authorization.

#### Scenario: AUTO threshold
- GIVEN AUTO has exactly its threshold count of authorized operations
- WHEN `tools/list` is requested
- THEN LIST presentation is returned

#### Scenario: AUTO above threshold
- GIVEN AUTO exceeds its authorized-operation threshold
- WHEN `tools/list` is requested
- THEN `search_tools` and `execute_tool` replace direct operation tools

### Requirement: Authorization Before Search

Search MUST authorize workspace, integration, active revision, bindings, grants, `catalog:read`, expiry, revocation, and integration status before matching, ranking, or counting. Unfiltered search MUST span every authorized binding. An authorized alias MUST only narrow that set. An unauthorized alias MUST return HTTP 403 for REST or MCP category `authorization_denied`, without existence or count disclosure.

#### Scenario: Unfiltered multi-binding search
- GIVEN matching operations exist across every authorized binding
- WHEN REST search or `search_tools` runs without an alias
- THEN all authorized bindings contribute eligible results

#### Scenario: Authorized alias narrows
- GIVEN a key authorizes two bindings
- WHEN search uses one authorized alias
- THEN only that binding contributes results

#### Scenario: Unauthorized alias
- GIVEN an alias is outside the key's authorization graph
- WHEN search uses that alias
- THEN `authorization_denied` is returned without confirming existence

### Requirement: REST and MCP Core Parity

REST list, search, and execute-by-name and MCP direct calls, `search_tools`, and `execute_tool` MUST use the same request-time active revision, grants, scopes, operation resolution, exact binding and connection, executor, safe errors, audit outcomes, and invocation policy. Each authorized execution MUST resolve exactly one pinned `<server_slug>__<operation_slug>`, binding, and connection and create exactly one metadata-only invocation. Authorization failure MUST create a sanitized audit outcome, no invocation, and no upstream contact.

#### Scenario: Exact parity resolution
- GIVEN a public name is authorized through one active binding
- WHEN REST execute-by-name, direct `tools/call`, or `execute_tool` calls it
- THEN every path uses the same operation, binding, connection, and executor

#### Scenario: Direct and meta-tool denial
- GIVEN the active revision, grant, or `tools:execute` denies an operation
- WHEN direct `tools/call` or `execute_tool` calls it
- THEN both return `authorization_denied` without upstream contact

### Requirement: Live Catalog Changes

The endpoint MUST advertise list-change support. Activation MUST update open sessions and emit the standard notification when their visible list changes. Every request MUST reauthorize; a revoked key MUST receive no catalog data.

#### Scenario: Activation changes list
- GIVEN an open session and a draft changes visible tools or mode
- WHEN the draft activates
- THEN a list-change notification is emitted and later lists use the new revision

#### Scenario: Revoked session
- GIVEN an open session's key is revoked
- WHEN its next request occurs
- THEN `authorization_denied` is returned without catalog data
