# Identity Tenancy Lifecycle Specification

## Purpose

Define human identity, workspace membership, and destructive lifecycle rules. Email verification is outside this MVP.

## Requirements

### Requirement: Setup and Registration

The platform MUST allow setup exactly once, create the first user as owner of an automatic personal workspace, and keep public registration disabled by default. Registration MUST be environment-configurable after setup. When enabled, each direct signup MUST receive a personal workspace; invitation onboarding MUST remain available while registration is disabled.

#### Scenario: First setup succeeds
- GIVEN no user exists
- WHEN one valid setup request completes
- THEN the user becomes owner of a personal workspace

#### Scenario: Concurrent setup is closed
- GIVEN no user exists
- WHEN concurrent valid setup requests race
- THEN exactly one succeeds and every other request is rejected

#### Scenario: Disabled registration
- GIVEN setup is complete and public registration is disabled
- WHEN a person signs up without a valid invitation
- THEN registration is denied without revealing existing accounts

### Requirement: Human Authentication and Recovery

The platform MUST authenticate humans with email and password sessions, MUST NOT represent an email as verified, and MUST revoke all human sessions after a password reset.

#### Scenario: Valid login
- GIVEN an active user supplies valid credentials
- WHEN login completes
- THEN a human session is established

#### Scenario: Password reset revokes sessions
- GIVEN a user has multiple active sessions
- WHEN that user's password is reset
- THEN every prior human session becomes unusable

### Requirement: Membership and Invitations

Membership MUST be workspace-scoped. Owners and admins MAY issue invitations, including admin invitations; only an owner MAY promote another member to owner. Invitations MUST expire after seven days and MUST be accepted at most once.

#### Scenario: Invitation acceptance
- GIVEN public registration is disabled and an unexpired workspace invitation exists
- WHEN its intended recipient accepts it
- THEN the specified non-owner membership is created once

#### Scenario: Unauthorized owner promotion
- GIVEN an admin is not an owner
- WHEN the admin attempts to promote a member to owner
- THEN the request is denied without changing membership

### Requirement: Last-Owner Protection

The platform MUST NOT remove, demote, or delete a workspace's last owner. Competing membership changes MUST preserve at least one owner.

#### Scenario: Owner transfer
- GIVEN a workspace has two owners
- WHEN one owner removes their own membership
- THEN the other owner remains and removal succeeds

#### Scenario: Concurrent owner removals
- GIVEN a workspace has two owners
- WHEN both ownership removals race
- THEN at most one succeeds and one owner remains

### Requirement: Workspace and User Lifecycle

Archiving a workspace MUST immediately disable its sessions, invitations, keys, integrations, and consumption. User deletion MUST revoke sessions, remove memberships, archive and anonymize the user, and preserve non-identifying audit history; purge timing is governed by `platform-operations`.

#### Scenario: Workspace archival
- GIVEN an active workspace
- WHEN an authorized owner archives it
- THEN subsequent human and machine access for it is denied

#### Scenario: Last-owner deletion
- GIVEN a user is the last owner of a workspace
- WHEN deletion is requested
- THEN deletion is blocked until ownership is resolved
