# Declarative Skills Specification

## Purpose

Define immutable skill versions and authorized MCP skill tools; resources, prompts, and semantic/vector search are excluded.

## Requirements

### Requirement: Versioned Declarative Skills

Published skill versions containing guidance and optional stable tool references MUST be immutable. Editing MUST create a version; integrations MUST retain pinned versions until explicit draft activation.

#### Scenario: Integration remains pinned
- GIVEN an integration pins an older skill version
- WHEN a newer version is published
- THEN consumers continue receiving the pinned version

### Requirement: Per-Call Skill Authorization

A skill MUST be visible only when the active revision selects its pinned version and the key has `skills:read`; tool grants MUST NOT authorize it. Every call, including in open sessions, MUST revalidate key status, expiry, integration status, active revision, pinned selection/version, and scope.

#### Scenario: Tool-only relationship
- GIVEN a skill references only granted tools but is unselected
- WHEN search or retrieval occurs
- THEN the skill is not disclosed

#### Scenario: Selection removed during a session
- GIVEN an open session previously listed skill tools
- WHEN its selected skill is removed before the next skill call
- THEN that call returns `authorization_denied` without content

### Requirement: MCP Skill Tool Exposure

`skills_search` and `skills_get` MUST be MCP tools and MUST NOT be resources or prompts. LIST and SEARCH_EXECUTE MUST list both when the key has `skills:read` and the active revision selects at least one skill; otherwise neither MUST be listed. AUTO MUST apply the same rule after mode resolution. Skill tools MUST NOT count toward the AUTO operation threshold.

#### Scenario: LIST exposure
- GIVEN LIST, `skills:read`, and a selected skill
- WHEN `tools/list` is requested
- THEN both skill tools are listed with direct operation tools

#### Scenario: SEARCH_EXECUTE exposure
- GIVEN SEARCH_EXECUTE, `skills:read`, and a selected skill
- WHEN `tools/list` is requested
- THEN both skill tools are listed with `search_tools` and `execute_tool`

#### Scenario: AUTO ignores system tools
- GIVEN AUTO has its threshold count of operations plus both skill tools
- WHEN `tools/list` is requested
- THEN mode resolves from operation count alone and skill tools are listed separately

#### Scenario: No authorized selected skill
- GIVEN scope is absent or no skill is selected
- WHEN `tools/list` is requested
- THEN neither skill tool is listed

### Requirement: Authorized Skill Search

`skills_search` MUST authorize before matching, ranking, or counting and MUST search only selected pinned versions. Unauthorized skills MUST NOT affect results, totals, ranking, or errors. Every authorized matching selected version MUST be eligible for deterministic ranking.

#### Scenario: Matching selected skill
- GIVEN two selected skills and one matches
- WHEN an authorized client calls `skills_search`
- THEN only authorized matching versions are returned

#### Scenario: Unauthorized match
- GIVEN an unselected skill strongly matches
- WHEN search runs
- THEN its existence and count contribution remain hidden

### Requirement: Exact Skill Retrieval

`skills_get` MUST return the selected pinned version, never a newer one. Missing or unauthorized identifiers MUST return sanitized `authorization_denied` without existence disclosure. Tool references outside active grants MUST be omitted.

#### Scenario: Retrieve pinned content
- GIVEN a selected skill has multiple versions
- WHEN an authorized client calls `skills_get`
- THEN the pinned version is returned with only granted tool references

### Requirement: Skill Lifecycle

Archival MUST be blocked while an active integration selects the skill. After removal, archival MUST prevent new selections while immutable history remains subject to `platform-operations`.

#### Scenario: Archive selected skill
- GIVEN an active integration selects a skill
- WHEN archival is requested
- THEN archival is denied until the dependency is removed
