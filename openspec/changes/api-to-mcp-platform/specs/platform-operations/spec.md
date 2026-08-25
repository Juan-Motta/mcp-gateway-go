# Platform Operations Specification

## Purpose

Define operational visibility, audit, retention, purge, and the supported single-replica deployment boundary.

## Requirements

### Requirement: Safe Operational Telemetry

The platform MUST emit structured logs with correlation identifiers and MUST expose health, readiness, and aggregate metrics. Telemetry MUST NOT contain credentials, full request arguments, upstream response payloads, session tokens, or access keys.

#### Scenario: Correlated failure
- GIVEN a request fails
- WHEN operators inspect its safe log and consumer error
- THEN both share a trace identifier without exposing sensitive payloads

#### Scenario: Metrics access
- GIVEN metrics are requested through the operational boundary
- WHEN the requester is unauthorized
- THEN access is denied without tenant or integration details

### Requirement: Health and Readiness

Health MUST report whether the process is running. Readiness MUST fail while required dependencies or startup configuration prevent safe service, and MUST recover when service can safely accept requests.

#### Scenario: Dependency unavailable
- GIVEN a required dependency cannot serve requests
- WHEN readiness is checked
- THEN readiness reports unavailable while health MAY remain healthy

#### Scenario: Dependency recovers
- GIVEN required dependencies become usable
- WHEN readiness is checked again
- THEN readiness reports available

### Requirement: Audit History

Security-sensitive authentication, authorization, membership, invitation, publication, activation, grant, key, archive, purge, and anonymization outcomes MUST create workspace-scoped audit records. Audit records MUST preserve actor and outcome accountability without secret or payload capture and MUST be retained for 365 days.

#### Scenario: Denied privilege change
- GIVEN an admin attempts an owner-only action
- WHEN authorization denies it
- THEN a sanitized audit outcome is retained for the workspace

#### Scenario: Anonymized user history
- GIVEN a user is anonymized
- WHEN prior audit records are viewed by an authorized operator
- THEN required accountability remains without recoverable user profile data

### Requirement: Archive and Purge

Archival MUST immediately disable consumption. Purge MUST wait at least 30 days after archival, satisfy capability-specific reference checks, and preserve records under longer retention. Eligibility MUST be rechecked atomically when cleanup races with a new reference.

#### Scenario: Grace period
- GIVEN a resource was archived fewer than 30 days ago
- WHEN purge runs
- THEN the resource is retained

#### Scenario: Purge-reference race
- GIVEN an archived resource reaches purge age
- WHEN purge races with creation of a valid retained reference
- THEN purge does not leave a dangling reference

### Requirement: Invocation Retention

Invocation metadata and sanitized errors MUST be retained for 30 days and then removed. Full arguments, credentials, and upstream response payloads MUST NOT be captured.

#### Scenario: Invocation expires
- GIVEN invocation metadata is older than 30 days
- WHEN retention cleanup runs
- THEN it is removed while required audit history remains

### Requirement: Supported Deployment Boundary

The MVP MUST provide a reproducible deployment containing the data service, one API replica, human control plane, and edge routing. Restart resets for in-memory rate, concurrency, and open MCP transport state MUST be accepted and observable. Sandbox execution, horizontal MCP scaling, RLS, and heavy observability MUST NOT be claimed as supported.

#### Scenario: API restart
- GIVEN active in-memory limits or MCP transport sessions
- WHEN the sole API replica restarts
- THEN those states MAY reset while persisted business records remain

#### Scenario: Unsupported scale-out
- GIVEN an operator attempts multiple API replicas
- WHEN validating the MVP deployment
- THEN the configuration is rejected or clearly reported as unsupported
