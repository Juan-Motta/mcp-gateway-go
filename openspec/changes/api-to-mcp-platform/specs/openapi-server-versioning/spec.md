# OpenAPI Server Versioning Specification

## Purpose

Define deterministic import, publication, naming, policy metadata, and server lifecycle.

## Requirements

### Requirement: Canonical Import

The platform MUST accept OpenAPI 3.0/3.1 JSON or YAML, retain source, and normalize compatible path, query, header, and JSON-body operations into one mutable draft. Encoded documents MUST NOT exceed 5 MiB or declare over 500 operations; either excess MUST reject the entire import without changing the draft. Unsupported constructs MUST produce visible per-operation diagnostics and no hidden behavior.

#### Scenario: Compatible import
- GIVEN a supported document within both limits
- WHEN it is imported
- THEN every compatible operation appears in the draft

#### Scenario: Oversize document
- GIVEN a document exceeds 5 MiB
- WHEN import is requested
- THEN `document_too_large` rejects the entire import and preserves the draft

#### Scenario: Excess operations
- GIVEN a document declares over 500 operations
- WHEN import is requested
- THEN `operation_limit_exceeded` rejects the entire import and preserves the draft

#### Scenario: Unsupported construct
- GIVEN an operation requires files, multipart, streaming, callbacks, or webhooks
- WHEN import completes
- THEN that operation is excluded with a visible diagnostic

### Requirement: Immutable Publication

Publication MUST freeze canonical operations, secret-free authentication requirements, and public identities into immutable history. A server MUST have one mutable draft.

#### Scenario: Concurrent publication
- GIVEN two requests publish one draft revision
- WHEN they race
- THEN one version is created and one returns a conflict

### Requirement: Stable Public Names

Server and operation slugs MUST be deterministic; the server slug MUST freeze after first publication. Tool names MUST use `<server_slug>__<operation_slug>`. Collision resolution MUST be repeatable and creation-order independent. Display names MAY change without public-name changes.

#### Scenario: Colliding operations
- GIVEN two operations normalize to one slug
- WHEN published
- THEN both receive distinct repeatable public names

### Requirement: Endpoint and Authentication Eligibility

Only public HTTPS endpoints MAY be executable. Published authentication metadata MUST exclude credentials. Supported schemes are none, API key header/query, bearer, and Basic; OAuth2 and all others MUST be unsupported. Import MUST diagnose each unsupported alternative. An operation with no fully supported alternative MUST be excluded from executable operations and tool identities with `unsupported_auth`; partial hidden execution MUST NOT occur.

#### Scenario: Unsupported-only authentication
- GIVEN an operation declares only OAuth2 or another unsupported scheme
- WHEN imported and published
- THEN `unsupported_auth` identifies it and no executable tool is created

#### Scenario: Supported alternative
- GIVEN an operation has one unsupported and one complete supported alternative
- WHEN imported
- THEN the first is diagnosed and eligibility uses only the supported alternative

#### Scenario: Ineligible endpoint
- GIVEN a draft declares HTTP or private destination metadata
- WHEN publication is attempted
- THEN it is marked ineligible with a safe diagnostic

### Requirement: Server Lifecycle and Promotion

New versions MUST NOT replace versions pinned by active integrations. Archival MUST be blocked while an active integration depends on the server; history MUST remain until `platform-operations` permits purge.

#### Scenario: New version preserves pin
- GIVEN an integration pins an older version
- WHEN a newer version is published
- THEN the integration continues using its pinned version
