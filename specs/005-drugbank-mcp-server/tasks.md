# Tasks: DrugBank MCP Server

**Feature ID**: 005-drugbank-mcp-server
**Status**: ⛔ Blocked (819 lines written, API access requires paid key)
**Date**: 2025-12-22
**Updated**: 2025-12-23
**Linear**: Pending - child of AGE-65
**Blocker**: Public endpoint (go.drugbank.com) returns Cloudflare challenge. Commercial API (api.drugbank.com) returns 401. Contact: sales@drugbank.com
**Input**: Design documents from `/specs/005-drugbank-mcp-server/`
**Prerequisites**: spec.md, research.md, data-model.md, contracts/

---

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3, US4)
- All paths relative to repository root

---

## Phase 1: Setup

**Purpose**: Verify scaffolding and confirm dependencies

- [X] T001 Verify DrugBank server scaffold exists in src/lifesciences_mcp/servers/drugbank.py
- [X] T002 Verify clients/drugbank.py stub exists from ADR-006 refactor
- [X] T003 [P] Confirm httpx is available in pyproject.toml (already present for HGNC/UniProt)
- [X] T004 Run `uv sync` to confirm all dependencies are installed

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**CRITICAL**: This phase implements API Key Authentication and Rate Limiting (Constitution v1.1.0)

### API Key Authentication (FR-005 - Tiered Access)

- [ ] T005 Define API configuration constants in src/lifesciences_mcp/clients/drugbank.py
  - PUBLIC_BASE_URL: "https://go.drugbank.com"
  - COMMERCIAL_BASE_URL: "https://api.drugbank.com/v1"
  - Define endpoints for search and drug lookup

- [ ] T006 Implement `_get_api_key()` method in src/lifesciences_mcp/clients/drugbank.py
  - Read `DRUGBANK_API_KEY` from environment variable
  - Return None if not set (use public endpoints)
  - Log warning if API key not found (FR-005, SC-007)

- [ ] T007 Implement `_get_headers()` method in src/lifesciences_mcp/clients/drugbank.py
  - Return `Authorization: Bearer {API_KEY}` header if key is set
  - Return empty dict if using public endpoints
  - Include User-Agent for identification

- [ ] T008 Implement `_select_endpoint()` method in src/lifesciences_mcp/clients/drugbank.py
  - Use COMMERCIAL_BASE_URL if API key is set
  - Use PUBLIC_BASE_URL if no API key (public endpoints)
  - Log which mode is being used

### Rate Limiting Implementation (Constitution v1.1.0 MANDATORY)

- [ ] T009 Implement `_rate_limited_get()` method in src/lifesciences_mcp/clients/drugbank.py
  - Add `asyncio.Lock()` for request serialization
  - Add `_last_request_time` tracking (10 req/s = 100ms minimum delay)
  - Thundering herd prevention: re-check timing after acquiring lock
  - Use httpx async client for requests

- [ ] T010 Implement exponential backoff for rate limit errors in src/lifesciences_mcp/clients/drugbank.py
  - Handle 429 (Rate Limited) status codes
  - Handle 401/403 (auth required - may indicate commercial tier needed)
  - Exponential backoff: base_delay=1.0s, max_retries=3, max_delay=60s

### Pagination Implementation

- [ ] T011 Implement cursor encoding/decoding in src/lifesciences_mcp/clients/drugbank.py
  - `_encode_cursor(page, per_page)` → base64 encoded JSON (from research.md R7)
  - `_decode_cursor(cursor)` → (page, per_page) tuple
  - Handle None cursor (default to page 1)

### Lifecycle Management (ADR-004 Compliance)

- [ ] T011a Verify module-level singleton pattern in src/lifesciences_mcp/servers/drugbank.py
  - Confirm `get_client()` uses lazy initialization (already scaffolded correctly)
  - DO NOT add `@mcp.on_event("shutdown")` - causes AttributeError in FastMCP
  - FastMCP manages lifecycle internally - no cleanup hooks needed
  - Reference: docs/adr/accepted/adr-004-v1.0.md

### Pydantic Models

- [ ] T012 [P] Create DrugSearchCandidate model in src/lifesciences_mcp/models/drug.py
  - Fields: id (DrugBank CURIE), name, drug_type, categories, score
  - ID validation: regex `^DrugBank:DB\d{5}$`
  - Score range validation: 0.0-1.0

- [ ] T013 [P] Create Drug model in src/lifesciences_mcp/models/drug.py
  - Fields: id, name, drug_type, categories, description, indication, mechanism_of_action, pharmacodynamics, absorption, metabolism, half_life, cross_references
  - Import CrossReferences from envelopes.py
  - Slim mode support (exclude verbose fields, cross_references)

- [ ] T014 Update src/lifesciences_mcp/models/\_\_init\_\_.py to export Drug, DrugSearchCandidate

**Checkpoint**: Foundation ready - DrugBankClient can make authenticated, rate-limited API calls

---

## Phase 3: User Story 1 - Fuzzy Drug Search (Priority: P1) - MVP

**Goal**: Enable researchers to discover drugs using common names, brand names, or therapeutic indications

**Independent Test**: Query "aspirin" and verify ranked candidates include DrugBank:DB00945

### Tests for User Story 1

- [ ] T015 [P] [US1] Create unit test for DrugSearchCandidate validation in tests/unit/test_drugbank_models.py
  - Test valid DrugBank CURIE format
  - Test invalid format rejection
  - Test score bounds validation

- [ ] T016 [P] [US1] Create integration test for search_drugs in tests/integration/test_drugbank_api.py
  - Test: search "aspirin" returns candidates with DrugBank:DB00945
  - Test: search "metformin" returns DrugBank:DB00331 in top results
  - Test: search "Lipitor" returns atorvastatin
  - Test: pagination with cursor parameter
  - Test: slim mode returns minimal fields

### Implementation for User Story 1

- [ ] T017 [US1] Implement `search_drugs()` in DrugBankClient (src/lifesciences_mcp/clients/drugbank.py)
  - Validate query length >= 2 characters (return AMBIGUOUS_QUERY if too short)
  - Use appropriate endpoint based on API key availability
  - Call `_rate_limited_get()` with search URL
  - Parse response (HTML/JSON depending on endpoint)

- [ ] T018 [US1] Implement `_parse_search_response()` helper in src/lifesciences_mcp/clients/drugbank.py
  - Handle JSON response (commercial API)
  - Handle HTML response with embedded JSON (public API)
  - Extract drug ID, name, type, categories

- [ ] T019 [US1] Implement `_transform_search_results()` helper in src/lifesciences_mcp/clients/drugbank.py
  - Map response to list[DrugSearchCandidate]
  - Calculate relevance scores (1.0 for exact, decreasing for partial)
  - Sort by relevance score descending

- [ ] T020 [US1] Implement pagination envelope creation for search results
  - Calculate next cursor based on page + 1
  - Include total_count from response (if available)
  - Return null cursor if no more pages

- [ ] T021 [US1] Wire search_drugs tool in src/lifesciences_mcp/servers/drugbank.py
  - Replace NotImplementedError stub
  - Add slim mode support (FR-023)
  - Return PaginationEnvelope or ErrorEnvelope

**Checkpoint**: User Story 1 complete - Fuzzy search works independently

---

## Phase 4: User Story 2 - Strict Drug Lookup (Priority: P2)

**Goal**: Enable researchers to retrieve complete drug records using DrugBank CURIEs from search results

**Independent Test**: Call get_drug("DrugBank:DB00945") and verify complete aspirin record with cross_references

### Tests for User Story 2

- [ ] T022 [P] [US2] Create unit test for Drug model validation in tests/unit/test_drugbank_models.py
  - Test full mode includes all fields
  - Test slim mode excludes description, indication, mechanism_of_action, cross_references
  - Test cross_references omit-if-null pattern

- [ ] T023 [P] [US2] Create integration test for get_drug in tests/integration/test_drugbank_api.py
  - Test: get_drug("DrugBank:DB00945") returns complete aspirin record
  - Test: verify cross_references contains ChEMBL, UniProt keys
  - Test: slim mode returns minimal fields
  - Test: invalid CURIE returns UNRESOLVED_ENTITY

- [ ] T024 [P] [US2] Create "Junior Dev" ambiguity test in tests/integration/test_drugbank_api.py
  - Test: get_drug("DB00945") (missing prefix) returns UNRESOLVED_ENTITY
  - Test: recovery hint suggests using "DrugBank:DBXXXXX" format
  - This tests FR-010 explicitly

### Implementation for User Story 2

- [ ] T025 [US2] Implement DrugBank ID validation in src/lifesciences_mcp/clients/drugbank.py
  - Create `_validate_drugbank_id()` helper
  - Regex: `^DrugBank:DB\d{5}$` (from research.md R3)
  - Return UNRESOLVED_ENTITY with recovery hint on failure

- [ ] T026 [US2] Implement `get_drug()` in DrugBankClient (src/lifesciences_mcp/clients/drugbank.py)
  - Validate CURIE format first
  - Extract DB ID from CURIE (e.g., "DrugBank:DB00945" -> "DB00945")
  - Use appropriate endpoint based on API key availability
  - Call `_rate_limited_get()` with drug URL

- [ ] T027 [US2] Implement `_parse_drug_response()` helper in src/lifesciences_mcp/clients/drugbank.py
  - Handle JSON response (commercial API)
  - Handle HTML response (public API) - extract data from page
  - Extract all drug fields including pharmacology data

- [ ] T028 [US2] Implement `_transform_drug()` helper in src/lifesciences_mcp/clients/drugbank.py
  - Map response to Drug model
  - Handle optional fields (set to None if missing)
  - Support slim mode (exclude verbose fields)

- [ ] T029 [US2] Wire get_drug tool in src/lifesciences_mcp/servers/drugbank.py
  - Replace NotImplementedError stub
  - Add slim mode support (FR-023, FR-024, FR-025)
  - Return Drug dict or ErrorEnvelope

**Checkpoint**: User Story 2 complete - Fuzzy-to-Fact workflow works end-to-end

---

## Phase 5: User Story 3 - Cross-Database Integration (Priority: P3)

**Goal**: Enable researchers to navigate from DrugBank drugs to ChEMBL, UniProt, KEGG, PubChem

**Independent Test**: Retrieve aspirin and verify cross_references contains chembl, uniprot, kegg, pubchem_compound

### Tests for User Story 3

- [ ] T030 [P] [US3] Create unit test for cross-reference mapping in tests/unit/test_drugbank_models.py
  - Test: ChEMBL xref transformation (add CHEMBL: prefix)
  - Test: UniProt xref transformation (add UniProtKB: prefix)
  - Test: omit-if-null pattern (missing xrefs not included)

- [ ] T031 [P] [US3] Create integration test for cross-references in tests/integration/test_drugbank_api.py
  - Test: aspirin (DrugBank:DB00945) has chembl, uniprot, kegg, pubchem_compound keys
  - Test: verify CURIE formats match 22-key registry patterns
  - Test: drug with no xrefs has empty cross_references

### Implementation for User Story 3

- [ ] T032 [US3] Implement `_build_cross_references()` helper in src/lifesciences_mcp/clients/drugbank.py
  - Map DrugBank external identifiers to flat object
  - Apply CURIE normalization (research.md R4)
  - Omit keys entirely if no xref exists (Constitution Principle III)

- [ ] T033 [US3] Integrate cross_references into get_drug response
  - Call `_build_cross_references()` during drug transformation
  - Include self-reference: drugbank -> current drug ID
  - Only include in full mode (exclude from slim)

**Checkpoint**: User Story 3 complete - Multi-database navigation works

---

## Phase 6: User Story 4 - Error Recovery and Resilience (Priority: P4)

**Goal**: Enable researchers to self-resolve errors with clear messages and actionable recovery hints

**Independent Test**: Trigger each error condition and verify error envelopes contain correct codes and recovery hints

### Tests for User Story 4

- [ ] T034 [P] [US4] Create unit test for error code mapping in tests/unit/test_drugbank_client.py
  - Test: invalid CURIE → UNRESOLVED_ENTITY
  - Test: query too short → AMBIGUOUS_QUERY
  - Test: 404 response → ENTITY_NOT_FOUND
  - Test: 429 response → RATE_LIMITED
  - Test: 401/403 response → UPSTREAM_ERROR (commercial tier messaging)
  - Test: 500/502/503 response → UPSTREAM_ERROR

- [ ] T035 [P] [US4] Create integration test for error recovery in tests/integration/test_drugbank_api.py
  - Test: get_drug("DB00945") (no prefix) returns UNRESOLVED_ENTITY with recovery hint
  - Test: get_drug("DrugBank:DB99999999") returns ENTITY_NOT_FOUND
  - Test: search_drugs("X") returns AMBIGUOUS_QUERY with "minimum 2 characters" hint

- [ ] T036 [P] [US4] Create commercial tier messaging test in tests/integration/test_drugbank_api.py
  - Test: 401/403 returns hint about commercial API access
  - FR-005, SC-007: Gracefully handle tiered access limitations

### Implementation for User Story 4

- [ ] T037 [US4] Implement `_handle_response_error()` in src/lifesciences_mcp/clients/drugbank.py
  - Check HTTP status: 429 → RATE_LIMITED
  - Check HTTP status: 401/403 → UPSTREAM_ERROR with commercial tier hint
  - Check HTTP status: 404 → ENTITY_NOT_FOUND
  - Check HTTP status: 5xx → UPSTREAM_ERROR
  - Return success data or ErrorEnvelope

- [ ] T038 [US4] Add recovery hints to all error responses
  - UNRESOLVED_ENTITY: "Invalid DrugBank CURIE format. Use 'DrugBank:DBXXXXX' (e.g., 'DrugBank:DB00945')"
  - ENTITY_NOT_FOUND: "Drug not found in DrugBank database. Verify at go.drugbank.com"
  - AMBIGUOUS_QUERY: "Minimum query length is 2 characters"
  - RATE_LIMITED: "DrugBank API rate limit exceeded. Wait {seconds}s before retrying"
  - UPSTREAM_ERROR: "DrugBank API temporarily unavailable. Retry in 60 seconds"
  - UPSTREAM_ERROR (401/403): "Commercial API access may be required for this feature. See go.drugbank.com"

- [ ] T039 [US4] Add invalid_input field to all error envelopes
  - Include the exact input that caused the error
  - Helps users identify their mistake

- [ ] T040 [US4] Add comprehensive try/except wrapping to all client methods
  - search_drugs: wrap API call and parsing
  - get_drug: wrap API call and parsing
  - All return ErrorEnvelope on failure (never raise exceptions to MCP layer)

**Checkpoint**: User Story 4 complete - Production-ready error handling

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Performance validation, documentation, and code quality

### Performance Tests (FR-028, SC-001, SC-002, SC-004)

- [ ] T041 [P] Create performance test for SC-001 in tests/integration/test_performance.py
  - Test: 95% of search queries complete in <2 seconds
  - Use known drugs: aspirin, metformin, atorvastatin

- [ ] T042 [P] Create performance test for SC-002 in tests/integration/test_performance.py
  - Test: 100% of valid CURIE lookups complete in <1 second
  - Test with DrugBank:DB00945, DrugBank:DB00331, DrugBank:DB01076

- [ ] T043 [P] Create concurrency test for SC-004 in tests/integration/test_performance.py
  - Test: 100 concurrent requests across all tools without degradation
  - Verify rate limiting prevents overwhelming API

### Documentation Updates

- [ ] T044 [P] Update src/lifesciences_mcp/\_\_init\_\_.py exports
  - Add Drug, DrugSearchCandidate to \_\_all\_\_
  - Verify DrugBankClient is exported

- [ ] T045 [P] Update CLAUDE.md with DrugBank server documentation
  - Add DrugBank to "Implemented Tools" section
  - Document search_drugs, get_drug
  - Include usage examples and API key configuration
  - Document commercial vs public tier differences

- [ ] T046 Run quickstart.md validation scenarios manually
  - Execute all example queries from quickstart.md
  - Verify expected outputs match

### Code Quality

- [ ] T047 Run `uv run ruff check --fix src/lifesciences_mcp/` for linting
- [ ] T048 Run `uv run ruff format src/lifesciences_mcp/` for formatting
- [ ] T049 Run `uv run pyright src/lifesciences_mcp/` for type checking
- [ ] T050 Run full test suite: `uv run pytest tests/ -v`
  - Target: 50+ tests (matching UniProt/ChEMBL/Open Targets standard)
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
- **US3 (Cross-References)**: Extends US2's Drug model - can run in parallel with US2 core
- **US4 (Error Recovery)**: Applies to all tools - can run in parallel with US2/US3

### Critical Path (Serial)

T001 → T004 → T005 → T006 → T007 → T008 → T009 → T010 → T017 → T018 → T019 → T021 (MVP complete)

### Parallel Opportunities

**Phase 2 (after T011)**:
- T012, T013 can run in parallel (different model classes)

**Phase 3-6 (after Foundational)**:
- All test tasks marked [P] within each phase
- US3 and US4 can start once US2 core (T025-T028) is complete

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001-T004)
2. Complete Phase 2: Foundational (T005-T014) - **CRITICAL: includes API key auth + rate limiting**
3. Complete Phase 3: User Story 1 (T015-T021)
4. **STOP and VALIDATE**: Test fuzzy search independently
5. Deploy/demo if ready

### Incremental Delivery

1. **MVP (US1)**: Fuzzy search works → researchers can discover drugs
2. **+US2**: Strict lookup works → complete Fuzzy-to-Fact workflow
3. **+US3**: Cross-references work → multi-database navigation
4. **+US4**: Error handling polished → production-ready

### Estimated Task Counts

| Phase | Tasks | Parallel Opportunities |
|-------|-------|----------------------|
| Phase 1: Setup | 4 | 1 (T003) |
| Phase 2: Foundational | 10 | 2 (T012, T013) |
| Phase 3: US1 | 7 | 2 (T015, T016) |
| Phase 4: US2 | 8 | 3 (T022, T023, T024) |
| Phase 5: US3 | 4 | 2 (T030, T031) |
| Phase 6: US4 | 7 | 3 (T034, T035, T036) |
| Phase 7: Polish | 10 | 4 (T041-T045) |
| **Total** | **50** | **17 parallel** |

---

## Notes

- [P] tasks can run in parallel if staffed by multiple agents
- Each user story is independently testable after Foundational phase
- Constitution v1.1.0 MANDATES rate limiting in Phase 2 - do not skip T009, T010
- API Key Authentication (T005-T008) supports tiered access per FR-005
- DrugBank has tiered API access - public endpoints for basic data, commercial API for enhanced access
- All error handling must return ErrorEnvelope (never raise exceptions to MCP layer)
- Commercial tier messaging (T036, T038) ensures graceful handling of access limitations (SC-007)
