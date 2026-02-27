# Feature Specification: ClinicalTrials.gov MCP Server

**Feature Branch**: `013-clinicaltrials-mcp-server`
**Created**: 2026-01-03
**Status**: Draft
**Input**: Build the ClinicalTrials.gov MCP Server with Fuzzy-to-Fact workflow, Agentic Biolink schema, and Canonical Envelopes.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Fuzzy Trial Search (Priority: P1)

An LLM agent needs to find clinical trials in the ClinicalTrials.gov database using natural language queries and comprehensive filters (condition, intervention, status, location, phase). The agent receives ranked trial candidates with NCT IDs that can be used for strict lookups.

**Why this priority**: Trial search is the primary entry point for clinical research discovery workflows. Without fuzzy search, agents cannot discover NCT identifiers for further investigation.

**Independent Test**: Can be fully tested by searching for "breast cancer immunotherapy" and verifying ranked candidates are returned with valid NCT IDs in the format NCT########.

**Acceptance Scenarios**:

1. **Given** an agent searching for trials by disease, **When** they call `search_trials(query="breast cancer")`, **Then** they receive a PaginationEnvelope with ranked TrialSearchCandidate items containing NCT IDs.
2. **Given** an agent searching with multiple filters, **When** they call `search_trials(condition="diabetes", intervention="insulin", status="recruiting")`, **Then** they receive trials matching all criteria.
3. **Given** an agent searching by location, **When** they call `search_trials(query="heart disease", location="Boston, MA")`, **Then** they receive trials with facilities near Boston.
4. **Given** an agent searching by phase, **When** they call `search_trials(condition="melanoma", phase="Phase 3")`, **Then** they receive only Phase 3 trials.
5. **Given** an agent with an overly broad query returning many results, **When** they use pagination, **Then** they can retrieve all matching trials across multiple pages.

---

### User Story 2 - Strict Trial Lookup (Priority: P2)

An LLM agent needs to retrieve complete trial information using a resolved NCT ID. The agent receives full trial details including protocol, eligibility criteria, outcomes, sponsors, phase, status, enrollment numbers, and start/completion dates.

**Why this priority**: After fuzzy discovery, strict lookup provides the detailed trial data needed for evidence evaluation and clinical reasoning. This completes the Fuzzy-to-Fact protocol for trials.

**Independent Test**: Can be fully tested by calling `get_trial("NCT:01234567")` with a known trial ID and verifying all required fields are populated.

**Acceptance Scenarios**:

1. **Given** an agent with a valid NCT ID, **When** they call `get_trial("NCT:04123456")`, **Then** they receive a complete Trial entity with protocol, eligibility_criteria, outcomes, sponsors, phase, status, enrollment, and dates.
2. **Given** an agent with an invalid ID format, **When** they call `get_trial("breast cancer")` (query string, not NCT CURIE), **Then** they receive an UNRESOLVED_ENTITY error with recovery_hint pointing to search_trials.
3. **Given** an agent with a non-existent NCT ID, **When** they call `get_trial("NCT:99999999")`, **Then** they receive an ENTITY_NOT_FOUND error.
4. **Given** an agent requesting a trial with missing optional fields, **When** they call `get_trial()`, **Then** missing fields are omitted from the response (not set to null).

---

### User Story 3 - Trial Locations and Contacts (Priority: P3)

An LLM agent needs to retrieve geographic facility and contact information for a specific trial. This enables location-based filtering for patient recruitment and investigator discovery.

**Why this priority**: Location data is critical for patient recruitment workflows, trial site identification, and geographic analysis of clinical research activity.

**Independent Test**: Can be fully tested by calling `get_trial_locations("NCT:04123456")` and verifying facility addresses, contact names, and recruitment status are returned.

**Acceptance Scenarios**:

1. **Given** an agent with a valid NCT ID, **When** they call `get_trial_locations("NCT:04123456")`, **Then** they receive a list of facilities with name, city, state, country, zip, contact_name, contact_phone, contact_email, and recruitment_status.
2. **Given** an agent requesting locations for a trial with no facilities, **When** they call `get_trial_locations()`, **Then** they receive an empty list.
3. **Given** an agent with an invalid NCT ID, **When** they call `get_trial_locations("invalid")`, **Then** they receive an UNRESOLVED_ENTITY error.

---

### User Story 4 - Error Recovery (Priority: P4)

An LLM agent encounters errors and needs actionable guidance to self-correct. All errors include recovery hints that guide the agent to the correct tool or action.

**Why this priority**: Autonomous agents must recover from errors without human intervention, especially when confusing query strings with NCT IDs or when encountering rate limits.

**Independent Test**: Can be fully tested by triggering each error type and verifying recovery hints are actionable.

**Acceptance Scenarios**:

1. **Given** an agent receives UNRESOLVED_ENTITY error for passing a query to get_trial, **When** they follow the recovery_hint, **Then** they can call search_trials to resolve the NCT ID.
2. **Given** an agent receives RATE_LIMITED error, **When** they follow the recovery_hint, **Then** they know to retry with exponential backoff.
3. **Given** an agent receives UPSTREAM_ERROR when ClinicalTrials.gov API is unavailable, **When** they follow the recovery_hint, **Then** they know to retry later or check API status.
4. **Given** an agent receives ENTITY_NOT_FOUND for a non-existent NCT ID, **When** they follow the recovery_hint, **Then** they verify the NCT ID format and try search_trials if uncertain.

---

### Edge Cases

- What happens when search returns no results? → Empty items array with total_count: 0
- What happens when ClinicalTrials.gov API is unavailable? → UPSTREAM_ERROR with recovery_hint to retry later
- What happens when rate limit is exceeded (>1 req/sec)? → RATE_LIMITED error with exponential backoff guidance
- What happens when a trial has no locations? → get_trial_locations returns empty list
- What happens when a trial has incomplete eligibility criteria? → Omit missing criteria fields (not null)
- What happens when a trial has no cross-references? → Cross-reference keys are omitted (not null)
- What happens when agent passes "NCT04123456" instead of "NCT:04123456"? → System normalizes to CURIE format or returns UNRESOLVED_ENTITY with hint
- What happens when a trial has withdrawn or terminated status? → Status field accurately reflects current state

## Requirements *(mandatory)*

### Functional Requirements

**Search Capabilities**
- **FR-001**: System MUST provide fuzzy trial search via `search_trials` tool accepting query, condition, intervention, status, location, and phase parameters
- **FR-002**: System MUST return ranked TrialSearchCandidate results with NCT IDs in CURIE format (NCT:########)
- **FR-003**: System MUST support pagination using cursor-based navigation from ClinicalTrials.gov API v2
- **FR-004**: System MUST validate all search parameters and return INVALID_INPUT error for malformed inputs
- **FR-005**: System MUST support multiple simultaneous filter combinations (e.g., condition + phase + status)

**Strict Lookup**
- **FR-006**: System MUST provide strict trial lookup via `get_trial` tool accepting ONLY NCT CURIEs
- **FR-007**: System MUST return complete Trial entities including protocol, eligibility_criteria, primary_outcomes, secondary_outcomes, sponsors, phase, status, enrollment, start_date, completion_date, and cross_references
- **FR-008**: System MUST validate NCT ID format (NCT:########) and return UNRESOLVED_ENTITY error for invalid formats with recovery_hint pointing to search_trials

**Location Data**
- **FR-009**: System MUST provide trial location lookup via `get_trial_locations` tool accepting NCT CURIEs
- **FR-010**: System MUST return facility data including name, city, state, country, zip, contact_name, contact_phone, contact_email, and recruitment_status
- **FR-011**: System MUST return empty list for trials with no facility data

**Schema Compliance**
- **FR-012**: All entities MUST use the Agentic Biolink schema with flattened JSON structure
- **FR-013**: All entities MUST include cross_references object mapping to external databases (PubMed, ClinicalTrials.gov registry links, etc.)
- **FR-014**: System MUST omit fields with no data rather than returning null values

**Envelopes**
- **FR-015**: All list responses MUST use PaginationEnvelope with items, cursor, total_count, and page_size fields
- **FR-016**: All errors MUST use ErrorEnvelope with success: false, error.code, error.message, error.recovery_hint, and error.invalid_input fields
- **FR-017**: System MUST return standardized error codes: UNRESOLVED_ENTITY, ENTITY_NOT_FOUND, AMBIGUOUS_QUERY, RATE_LIMITED, UPSTREAM_ERROR, INVALID_INPUT

**API Integration**
- **FR-018**: System MUST use ClinicalTrials.gov API v2 base URL: https://clinicaltrials.gov/api/v2/
- **FR-019**: System MUST implement httpx-based async client extending LifeSciencesClient
- **FR-020**: System MUST enforce conservative 1 req/sec rate limit with exponential backoff
- **FR-021**: System MUST handle API errors gracefully and map to appropriate error envelopes

**Testing**
- **FR-022**: System MUST include pytest-asyncio test suite covering all user stories
- **FR-023**: System MUST include integration tests for "Junior Dev" ambiguity cases (e.g., passing query strings to strict lookup tools)
- **FR-024**: System MUST validate error recovery workflows with actionable recovery hints

### Key Entities

- **Trial**: Represents a clinical trial with protocol details, enrollment data, and administrative information
  - Attributes: nct_id (CURIE), title, brief_summary, detailed_description, protocol, eligibility_criteria, primary_outcomes, secondary_outcomes, sponsors, phase, status, enrollment, start_date, completion_date, last_update_date, cross_references
  - Relationships: Has multiple facilities/locations, belongs to sponsor organizations

- **TrialSearchCandidate**: Lightweight trial representation for fuzzy search results
  - Attributes: nct_id (CURIE), title, brief_summary, phase, status, conditions, interventions
  - Purpose: Ranked search results for discovery phase

- **TrialLocation**: Geographic facility information for a trial
  - Attributes: facility_name, city, state, country, zip, contact_name, contact_phone, contact_email, recruitment_status
  - Relationships: Belongs to a Trial (via NCT ID)

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Agents can discover relevant clinical trials in under 2 seconds for 95% of queries
- **SC-002**: Agents can complete Fuzzy-to-Fact workflow (search → get_trial) in under 3 seconds for 90% of cases
- **SC-003**: System successfully recovers from 100% of error scenarios using recovery hints without human intervention
- **SC-004**: All trial lookups return accurate phase, status, and enrollment data matching ClinicalTrials.gov registry
- **SC-005**: Search results are ranked by relevance with most pertinent trials in top 10 results for 80% of queries
- **SC-006**: System handles concurrent requests from multiple agents without rate limit violations
- **SC-007**: All responses comply with Agentic Biolink schema with no null values for missing data (fields omitted instead)
- **SC-008**: Integration tests validate all "Junior Dev" ambiguity scenarios (query strings vs NCT IDs, missing parameters, malformed inputs)

## Assumptions

1. **API Stability**: ClinicalTrials.gov API v2 is stable and maintains backward compatibility
2. **Rate Limiting**: 1 req/sec is sufficient for agent workflows without triggering API throttling
3. **Data Completeness**: Not all trials have complete data; missing fields will be omitted per ADR-001
4. **CURIE Format**: NCT IDs will be normalized to NCT:######## format internally if users provide raw NCT numbers
5. **Search Relevance**: ClinicalTrials.gov API provides relevance ranking; we do not re-rank results
6. **Location Data**: Some trials may have no facility/location data (e.g., observational studies, registry studies)
7. **Cross-References**: Limited cross-reference data available from ClinicalTrials.gov API; primarily PubMed links and registry URLs
8. **Pagination**: API v2 supports cursor-based pagination for search results exceeding page_size

## Dependencies

- **External APIs**: ClinicalTrials.gov API v2 (public, no authentication)
- **Internal Libraries**: LifeSciencesClient base class, httpx, pydantic, FastMCP
- **Schema**: Agentic Biolink schema, PaginationEnvelope, ErrorEnvelope (from src/lifesciences_mcp/models/)
- **Architecture**: ADR-001 compliance for Fuzzy-to-Fact protocol and async client patterns

## Out of Scope

- Real-time trial status updates (data is refreshed per ClinicalTrials.gov update schedule)
- Advanced analytics or aggregation across trials (agents can implement this using returned data)
- Patient matching algorithms (eligibility criteria are provided, but matching logic is agent-specific)
- Trial registration or submission (read-only API)
- Historical trial data versioning (returns current state only)
