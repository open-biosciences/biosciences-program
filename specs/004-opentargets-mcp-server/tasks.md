# Tasks: Open Targets MCP Server

**Feature ID**: 004-opentargets-mcp-server
**Status**: In Progress (699 lines, GraphQL API validated, minor test bug)
**Date**: 2025-12-22
**Updated**: 2025-12-23
**Linear**: Pending - child of AGE-65
**Input**: Design documents from `/specs/004-opentargets-mcp-server/`
**Prerequisites**: spec.md, research.md, data-model.md, contracts/

---

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3, US4)
- All paths relative to repository root

---

## Phase 1: Setup

**Purpose**: Verify scaffolding and confirm dependencies

- [X] T001 Verify Open Targets server scaffold exists in src/lifesciences_mcp/servers/opentargets.py
- [X] T002 Verify clients/opentargets.py stub exists from ADR-006 refactor
- [X] T003 [P] Confirm httpx is available in pyproject.toml (already present for HGNC/UniProt)
- [X] T004 Run `uv sync` to confirm all dependencies are installed

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**CRITICAL**: This phase implements the GraphQL client and rate limiting (Constitution v1.1.0)

### GraphQL Client Implementation

- [ ] T005 Define GRAPHQL_ENDPOINT constant in src/lifesciences_mcp/clients/opentargets.py
  - URL: "https://api.platform.opentargets.org/api/v4/graphql"
  - Define default timeout (30 seconds per FR-034)

- [ ] T006 Implement `_execute_graphql()` method in src/lifesciences_mcp/clients/opentargets.py
  - Accept query string and variables dict
  - Use httpx POST to GraphQL endpoint
  - Handle GraphQL response format (data/errors structure)
  - Return parsed response or raise for error handling

- [ ] T007 Define GraphQL query templates in src/lifesciences_mcp/clients/opentargets.py
  - SEARCH_TARGETS_QUERY (from research.md R1)
  - GET_TARGET_QUERY (from research.md R1)
  - GET_ASSOCIATIONS_QUERY (from research.md R1)

### Rate Limiting Implementation (Constitution v1.1.0 MANDATORY)

- [ ] T008 Implement `_rate_limited_graphql()` method in src/lifesciences_mcp/clients/opentargets.py
  - Add `asyncio.Lock()` for request serialization
  - Add `_last_request_time` tracking (10 req/s = 100ms minimum delay)
  - Thundering herd prevention: re-check timing after acquiring lock
  - Wrap `_execute_graphql()` with rate limiting

- [ ] T009 Implement exponential backoff for rate limit errors in src/lifesciences_mcp/clients/opentargets.py
  - Handle 429 (Rate Limited) status codes
  - Handle GraphQL-level rate limit errors
  - Exponential backoff: base_delay=1.0s, max_retries=3, max_delay=60s

### Pagination Implementation

- [ ] T010 Implement cursor encoding/decoding in src/lifesciences_mcp/clients/opentargets.py
  - `_encode_cursor(index, size)` → base64 encoded JSON (from research.md R7)
  - `_decode_cursor(cursor)` → (index, size) tuple
  - Handle None cursor (default to page 0)

### Lifecycle Management (ADR-004 Compliance)

- [ ] T010a Verify module-level singleton pattern in src/lifesciences_mcp/servers/opentargets.py
  - Confirm `get_client()` uses lazy initialization (already scaffolded correctly)
  - DO NOT add `@mcp.on_event("shutdown")` - causes AttributeError in FastMCP
  - FastMCP manages lifecycle internally - no cleanup hooks needed
  - Reference: docs/adr/accepted/adr-004-v1.0.md

### Pydantic Models

- [ ] T011 [P] Create TargetSearchCandidate model in src/lifesciences_mcp/models/target.py
  - Fields: id (Ensembl ID), approved_symbol, approved_name, score
  - ID validation: regex `^ENSG\d{11}$`
  - Score range validation: 0.0-1.0

- [ ] T012 [P] Create Target model in src/lifesciences_mcp/models/target.py
  - Fields: id, approved_symbol, approved_name, biotype, description, associated_diseases_count, cross_references
  - Import CrossReferences from envelopes.py
  - Slim mode support (exclude description, associated_diseases_count, cross_references)

- [ ] T013 [P] Create Association model in src/lifesciences_mcp/models/target.py
  - Fields: target_id, disease_id, disease_name, score, evidence_count, evidence_sources
  - EFO ID validation: regex `^EFO_\d+$`

- [ ] T014 Update src/lifesciences_mcp/models/\_\_init\_\_.py to export Target, TargetSearchCandidate, Association

**Checkpoint**: Foundation ready - OpenTargetsClient can make rate-limited GraphQL calls

---

## Phase 3: User Story 1 - Fuzzy Target Search (Priority: P1) - MVP

**Goal**: Enable researchers to discover gene targets using common names, symbols, or partial identifiers

**Independent Test**: Search for "BRCA1" or "TP53" and verify ranked candidates with Ensembl IDs

### Tests for User Story 1

- [ ] T015 [P] [US1] Create unit test for TargetSearchCandidate validation in tests/unit/test_opentargets_models.py
  - Test valid Ensembl ID format (ENSG + 11 digits)
  - Test invalid format rejection
  - Test score bounds validation

- [ ] T016 [P] [US1] Create integration test for search_targets in tests/integration/test_opentargets_api.py
  - Test: search "BRCA1" returns candidates with ENSG00000012048
  - Test: search "TP53" returns ENSG00000141510 in top results
  - Test: pagination with cursor parameter
  - Test: slim mode returns minimal fields

### Implementation for User Story 1

- [ ] T017 [US1] Implement `search_targets()` in OpenTargetsClient (src/lifesciences_mcp/clients/opentargets.py)
  - Validate query length >= 2 characters (return AMBIGUOUS_QUERY if too short)
  - Execute SEARCH_TARGETS_QUERY via `_rate_limited_graphql()`
  - Transform results to list[TargetSearchCandidate]
  - Add synthetic relevance scores (1.0 - i*0.05)

- [ ] T018 [US1] Implement `_transform_search_results()` helper in src/lifesciences_mcp/clients/opentargets.py
  - Map GraphQL response fields to TargetSearchCandidate
  - Handle missing approved_symbol/approved_name gracefully

- [ ] T019 [US1] Implement pagination envelope creation for search results
  - Calculate next cursor based on index + 1
  - Include total_count from GraphQL response
  - Return null cursor if no more pages

- [ ] T020 [US1] Wire search_targets tool in src/lifesciences_mcp/servers/opentargets.py
  - Replace NotImplementedError stub
  - Add slim mode support (FR-026)
  - Return PaginationEnvelope or ErrorEnvelope

**Checkpoint**: User Story 1 complete - Fuzzy search works independently

---

## Phase 4: User Story 2 - Strict Target Lookup (Priority: P2)

**Goal**: Enable researchers to retrieve complete target records using Ensembl IDs from search results

**Independent Test**: Call get_target("ENSG00000141510") and verify complete TP53 record with cross_references

### Tests for User Story 2

- [ ] T021 [P] [US2] Create unit test for Target model validation in tests/unit/test_opentargets_models.py
  - Test full mode includes all fields
  - Test slim mode excludes description, associated_diseases_count, cross_references
  - Test cross_references omit-if-null pattern

- [ ] T022 [P] [US2] Create integration test for get_target in tests/integration/test_opentargets_api.py
  - Test: get_target("ENSG00000141510") returns complete TP53 record
  - Test: verify cross_references contains HGNC, UniProt keys
  - Test: slim mode returns minimal fields
  - Test: invalid Ensembl ID returns UNRESOLVED_ENTITY

- [ ] T023 [P] [US2] Create "Junior Dev" ambiguity test in tests/integration/test_opentargets_api.py
  - Test: get_target("TP53") (gene symbol, not Ensembl ID) returns UNRESOLVED_ENTITY
  - Test: recovery hint suggests using search_targets first
  - This tests FR-037 explicitly

### Implementation for User Story 2

- [ ] T024 [US2] Implement Ensembl ID validation in src/lifesciences_mcp/clients/opentargets.py
  - Create `_validate_ensembl_id()` helper
  - Regex: `^ENSG\d{11}$` (from research.md R3)
  - Return UNRESOLVED_ENTITY with recovery hint on failure

- [ ] T025 [US2] Implement `get_target()` in OpenTargetsClient (src/lifesciences_mcp/clients/opentargets.py)
  - Validate Ensembl ID format first
  - Execute GET_TARGET_QUERY via `_rate_limited_graphql()`
  - Handle null result → ENTITY_NOT_FOUND
  - Transform result to Target model

- [ ] T026 [US2] Implement `_build_cross_references()` helper in src/lifesciences_mcp/clients/opentargets.py
  - Map Open Targets `crossReferences` array to flat object
  - Apply CURIE normalization (research.md R4)
  - Omit keys entirely if no cross-reference exists (Constitution Principle III)

- [ ] T027 [US2] Implement `_transform_target()` helper in src/lifesciences_mcp/clients/opentargets.py
  - Map GraphQL response fields to Target model
  - Call `_build_cross_references()` for full mode
  - Support slim mode (exclude description, cross_references)

- [ ] T028 [US2] Wire get_target tool in src/lifesciences_mcp/servers/opentargets.py
  - Replace NotImplementedError stub
  - Add slim mode support (FR-027, FR-028)
  - Return Target dict or ErrorEnvelope

**Checkpoint**: User Story 2 complete - Fuzzy-to-Fact workflow works end-to-end

---

## Phase 5: User Story 3 - Target-Disease Association Discovery (Priority: P3)

**Goal**: Enable researchers to query target-disease associations and evidence scores

**Independent Test**: Call get_associations("ENSG00000141510") and verify cancer-related diseases with evidence scores

### Tests for User Story 3

- [ ] T029 [P] [US3] Create unit test for Association model validation in tests/unit/test_opentargets_models.py
  - Test valid Ensembl ID and EFO ID formats
  - Test score bounds validation
  - Test evidence_sources defaults to empty list

- [ ] T030 [P] [US3] Create integration test for get_associations in tests/integration/test_opentargets_api.py
  - Test: get_associations("ENSG00000141510") returns cancer-related diseases
  - Test: pagination works for targets with many associations
  - Test: filtering by disease_id works
  - Test: target with no associations returns empty items array

### Implementation for User Story 3

- [ ] T031 [US3] Implement `get_associations()` in OpenTargetsClient (src/lifesciences_mcp/clients/opentargets.py)
  - Validate target Ensembl ID format
  - Optional disease_id filter (FR-011)
  - Execute GET_ASSOCIATIONS_QUERY via `_rate_limited_graphql()`
  - Transform results to list[Association]

- [ ] T032 [US3] Implement `_transform_associations()` helper in src/lifesciences_mcp/clients/opentargets.py
  - Map GraphQL associatedDiseases response to Association models
  - Extract evidence_sources from datatypeScores
  - Handle empty associations (return empty list, not error)

- [ ] T033 [US3] Implement pagination for associations
  - Use same cursor encoding as search_targets
  - Include count from GraphQL response as total_count
  - Handle edge case: target exists but has zero associations

- [ ] T034 [US3] Wire get_associations tool in src/lifesciences_mcp/servers/opentargets.py
  - Create new tool function (may not exist in scaffold)
  - Return PaginationEnvelope or ErrorEnvelope

**Checkpoint**: User Story 3 complete - Disease association discovery works

---

## Phase 6: User Story 4 - Error Recovery & Validation (Priority: P4)

**Goal**: Enable researchers to self-resolve errors with clear messages and actionable recovery hints

**Independent Test**: Trigger each error condition and verify error envelopes contain correct codes and recovery hints

### Tests for User Story 4

- [ ] T035 [P] [US4] Create unit test for error code mapping in tests/unit/test_opentargets_client.py
  - Test: invalid Ensembl ID → UNRESOLVED_ENTITY
  - Test: query too short → AMBIGUOUS_QUERY
  - Test: GraphQL target=null → ENTITY_NOT_FOUND
  - Test: 429 response → RATE_LIMITED
  - Test: 500/502/503 response → UPSTREAM_ERROR

- [ ] T036 [P] [US4] Create integration test for error recovery in tests/integration/test_opentargets_api.py
  - Test: get_target("TP53") (wrong format) returns UNRESOLVED_ENTITY with recovery hint
  - Test: get_target("ENSG00000000000") returns ENTITY_NOT_FOUND
  - Test: search_targets("X") returns AMBIGUOUS_QUERY with "minimum 2 characters" hint

- [ ] T037 [P] [US4] Create all 6 error code recovery tests in tests/integration/test_error_recovery.py
  - FR-038: Test each error code returns actionable recovery hint
  - Verify recovery hint enables user to self-correct

### Implementation for User Story 4

- [ ] T038 [US4] Implement `_handle_graphql_response()` in src/lifesciences_mcp/clients/opentargets.py
  - Check HTTP status first (429 → RATE_LIMITED, 5xx → UPSTREAM_ERROR)
  - Check GraphQL errors array → map to appropriate error code
  - Check null data → ENTITY_NOT_FOUND
  - Return success data or ErrorEnvelope

- [ ] T039 [US4] Add recovery hints to all error responses
  - UNRESOLVED_ENTITY: "Ensembl ID must match format ENSG[11 digits]. Use search_targets to find the correct ID."
  - ENTITY_NOT_FOUND: "Ensembl ID not in Open Targets database. Verify at platform.opentargets.org"
  - AMBIGUOUS_QUERY: "Minimum query length is 2 characters"
  - RATE_LIMITED: "Open Targets API rate limit exceeded. Wait {seconds}s before retrying"
  - UPSTREAM_ERROR: "Open Targets API temporarily unavailable. Retry in 60 seconds"

- [ ] T040 [US4] Add invalid_input field to all error envelopes
  - Include the exact input that caused the error
  - Helps users identify their mistake

- [ ] T041 [US4] Add comprehensive try/except wrapping to all client methods
  - search_targets: wrap GraphQL execution
  - get_target: wrap GraphQL execution
  - get_associations: wrap GraphQL execution
  - All return ErrorEnvelope on failure (never raise exceptions to MCP layer)

**Checkpoint**: User Story 4 complete - Production-ready error handling

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Performance validation, documentation, and code quality

### Performance Tests (FR-039, FR-040, SC-001, SC-002, SC-004)

- [ ] T042 [P] Create performance test for SC-001 in tests/integration/test_performance.py
  - Test: 95% of search queries complete in <2 seconds
  - Use known targets: BRCA1, TP53, EGFR

- [ ] T043 [P] Create performance test for SC-002 in tests/integration/test_performance.py
  - Test: 100% of valid Ensembl ID lookups complete in <1 second
  - Test with ENSG00000141510, ENSG00000012048, ENSG00000146648

- [ ] T044 [P] Create concurrency test for SC-004 in tests/integration/test_performance.py
  - Test: 100 concurrent requests across all tools without degradation
  - Verify rate limiting prevents overwhelming API

### Documentation Updates

- [ ] T045 [P] Update src/lifesciences_mcp/\_\_init\_\_.py exports
  - Add Target, TargetSearchCandidate, Association to \_\_all\_\_
  - Verify OpenTargetsClient is exported

- [ ] T046 [P] Update CLAUDE.md with Open Targets server documentation
  - Add Open Targets to "Implemented Tools" section
  - Document search_targets, get_target, get_associations
  - Include usage examples and Fuzzy-to-Fact workflow

- [ ] T047 Run quickstart.md validation scenarios manually
  - Execute all example queries from quickstart.md
  - Verify expected outputs match

### Code Quality

- [ ] T048 Run `uv run ruff check --fix src/lifesciences_mcp/` for linting
- [ ] T049 Run `uv run ruff format src/lifesciences_mcp/` for formatting
- [ ] T050 Run `uv run pyright src/lifesciences_mcp/` for type checking
- [ ] T051 Run full test suite: `uv run pytest tests/ -v`
  - Target: 50+ tests (matching UniProt/ChEMBL standard)
  - All tests must pass

---

## Dependencies & Execution Order

### Phase Dependencies

```
Phase 1 (Setup)
     ↓
Phase 2 (Foundational) ← BLOCKS ALL USER STORIES
     ↓
┌────┴────┬────────┬────────┐
↓         ↓        ↓        ↓
Phase 3   Phase 4  Phase 5  Phase 6
(US1)     (US2)    (US3)    (US4)
     ↓         ↓        ↓        ↓
     └────────┴────────┴────────┘
                  ↓
             Phase 7 (Polish)
```

### User Story Dependencies

- **US1 (Fuzzy Search)**: Independent - no dependencies on other stories
- **US2 (Strict Lookup)**: Uses search results from US1 in end-to-end workflow
- **US3 (Associations)**: Requires target from US2 (conceptually), but can be tested independently with known Ensembl IDs
- **US4 (Error Recovery)**: Applies to all tools - can run in parallel with US2/US3

### Critical Path (Serial)

T001 → T004 → T005 → T006 → T007 → T008 → T009 → T010 → T017 → T018 → T019 → T020 (MVP complete)

### Parallel Opportunities

**Phase 2 (after T010)**:
- T011, T012, T013 can run in parallel (different model classes)

**Phase 3-6 (after Foundational)**:
- All test tasks marked [P] within each phase
- US3 and US4 can start once US2 core (T024-T027) is complete

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001-T004)
2. Complete Phase 2: Foundational (T005-T014) - **CRITICAL: includes GraphQL client + rate limiting**
3. Complete Phase 3: User Story 1 (T015-T020)
4. **STOP and VALIDATE**: Test fuzzy search independently
5. Deploy/demo if ready

### Incremental Delivery

1. **MVP (US1)**: Fuzzy search works → researchers can discover targets
2. **+US2**: Strict lookup works → complete Fuzzy-to-Fact workflow
3. **+US3**: Associations work → disease relevance discovery
4. **+US4**: Error handling polished → production-ready

### Estimated Task Counts

| Phase | Tasks | Parallel Opportunities |
|-------|-------|----------------------|
| Phase 1: Setup | 4 | 1 (T003) |
| Phase 2: Foundational | 10 | 3 (T011, T012, T013) |
| Phase 3: US1 | 6 | 2 (T015, T016) |
| Phase 4: US2 | 8 | 3 (T021, T022, T023) |
| Phase 5: US3 | 6 | 2 (T029, T030) |
| Phase 6: US4 | 7 | 3 (T035, T036, T037) |
| Phase 7: Polish | 10 | 4 (T042-T046) |
| **Total** | **51** | **18 parallel** |

---

## Notes

- [P] tasks can run in parallel if staffed by multiple agents
- Each user story is independently testable after Foundational phase
- Constitution v1.1.0 MANDATES rate limiting in Phase 2 - do not skip T008, T009
- GraphQL client implementation (T005-T007) is unique to Open Targets - no run_in_executor needed (async-native)
- "Junior Dev" ambiguity test (T023) explicitly validates FR-037
- All error handling must return ErrorEnvelope (never raise exceptions to MCP layer)
