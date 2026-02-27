# Tasks: ChEMBL MCP Server

**Feature ID**: 003-chembl-mcp-server
**Status**: In Progress (112 tests passing, SDK approach working)
**Date**: 2025-12-22
**Updated**: 2025-12-23
**Linear**: Pending - child of AGE-65
**Input**: Design documents from `/specs/003-chembl-mcp-server/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/

---

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3, US4)
- All paths relative to repository root

---

## Phase 1: Setup

**Purpose**: Verify scaffolding and add ChEMBL SDK dependency

- [X] T001 Verify ChEMBL server scaffold exists in src/lifesciences_mcp/servers/chembl.py
- [X] T002 Add `chembl-webresource-client` dependency to pyproject.toml
- [X] T003 [P] Verify clients/chembl.py stub exists from ADR-006 refactor
- [X] T004 Run `uv sync` to install ChEMBL SDK and verify import works

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**CRITICAL**: This phase implements the `run_in_executor` pattern (ADR-001 §2 exception) and rate limiting (Constitution v1.1.0)

### Async Wrapper Implementation (run_in_executor)

- [X] T005 Implement ChEMBLClient.\_\_init\_\_ with SDK initialization in src/lifesciences_mcp/clients/chembl.py
  - Initialize `chembl_webresource_client.new_client.molecule` (synchronous SDK)
  - Set base_url to "https://www.ebi.ac.uk/chembl/api/data"
  - Lazy-init ThreadPoolExecutor (use Python defaults)

- [X] T006 Implement `_run_in_executor()` helper method in src/lifesciences_mcp/clients/chembl.py
  - Wrap synchronous SDK calls with `asyncio.get_event_loop().run_in_executor()`
  - Use self._executor (ThreadPoolExecutor) for thread pool
  - Return awaitable result

### Rate Limiting Implementation (Constitution v1.1.0 MANDATORY)

- [X] T007 Implement `_rate_limited_sdk_call()` method in src/lifesciences_mcp/clients/chembl.py
  - Add `asyncio.Lock()` for request serialization
  - Add `_last_request_time` tracking (10 req/s = 100ms minimum delay)
  - Thundering herd prevention: re-check timing after acquiring lock
  - Combine with run_in_executor wrapping

- [X] T008 Implement exponential backoff for rate limit errors in src/lifesciences_mcp/clients/chembl.py
  - Handle 429 (Rate Limited) and 503 (Service Unavailable) status codes
  - Exponential backoff: base_delay=1.0s, max_retries=3, max_delay=60s
  - Log backoff attempts for debugging

### Lifecycle Management (ADR-004 Compliance)

- [X] T008a Verify module-level singleton pattern in src/lifesciences_mcp/servers/chembl.py
  - Confirm `get_client()` uses lazy initialization (already scaffolded correctly)
  - DO NOT add `@mcp.on_event("shutdown")` - causes AttributeError in FastMCP
  - FastMCP manages lifecycle internally - no cleanup hooks needed
  - Reference: docs/adr/accepted/adr-004-v1.0.md

### Pydantic Models

- [X] T009 [P] Create CompoundSearchCandidate model in src/lifesciences_mcp/models/compound.py
  - Fields: id (CHEMBL CURIE), name, molecular_formula, score
  - CURIE validation: regex `^CHEMBL:[0-9]+$`
  - Score range validation: 0.0-1.0

- [X] T010 [P] Create Compound model in src/lifesciences_mcp/models/compound.py
  - Fields: id, name, molecular_formula, molecular_weight, smiles, inchi, canonical_name, synonyms, cross_references
  - Import CrossReferences from envelopes.py
  - Slim mode support (exclude cross_references, synonyms, structural data)

- [X] T011 Update src/lifesciences_mcp/models/\_\_init\_\_.py to export Compound, CompoundSearchCandidate

**Checkpoint**: Foundation ready - ChEMBLClient can make rate-limited async SDK calls

---

## Phase 3: User Story 1 - Fuzzy Compound Search (Priority: P1) - MVP

**Goal**: Enable researchers to discover compounds using common names, trade names, or partial identifiers

**Independent Test**: Query "aspirin" and verify ranked candidates include CHEMBL:25 with molecular properties

### Tests for User Story 1

- [X] T012 [P] [US1] Create unit test for CompoundSearchCandidate validation in tests/unit/test_chembl_models.py
  - Test valid CURIE format
  - Test invalid CURIE rejection
  - Test score bounds validation

- [X] T013 [P] [US1] Create integration test for search_compounds in tests/integration/test_chembl_api.py
  - Test: search "aspirin" returns candidates with CHEMBL:25
  - Test: search "imatinib" returns CHEMBL:941 in top results
  - Test: pagination with cursor parameter
  - Test: slim mode returns minimal fields

### Implementation for User Story 1

- [X] T014 [US1] Implement `search_compounds()` in ChEMBLClient (src/lifesciences_mcp/clients/chembl.py)
  - Validate query length >= 2 characters (return AMBIGUOUS_QUERY if too short)
  - Call SDK's `molecule.search()` via `_rate_limited_sdk_call()`
  - Transform results to list[CompoundSearchCandidate]
  - Add synthetic relevance scores (1.0 - i*0.05)

- [X] T015 [US1] Implement result transformation to PaginationEnvelope in src/lifesciences_mcp/clients/chembl.py
  - Create `_transform_search_results()` helper
  - Handle pagination with cursor (offset-based)
  - Default page_size=50, max=100

- [X] T016 [US1] Wire search_compounds tool in src/lifesciences_mcp/servers/chembl.py
  - Replace NotImplementedError stub
  - Add slim mode support (FR-025, FR-026)
  - Return PaginationEnvelope or ErrorEnvelope

**Checkpoint**: User Story 1 complete - Fuzzy search works independently

---

## Phase 4: User Story 2 - Strict Compound Lookup (Priority: P2)

**Goal**: Enable researchers to retrieve complete compound records using ChEMBL CURIEs from search results

**Independent Test**: Call get_compound("CHEMBL:25") and verify complete record with SMILES, InChI, and cross_references

### Tests for User Story 2

- [X] T017 [P] [US2] Create unit test for Compound model validation in tests/unit/test_chembl_models.py
  - Test full mode includes all fields
  - Test slim mode excludes cross_references, synonyms
  - Test cross_references omit-if-null pattern

- [X] T018 [P] [US2] Create integration test for get_compound in tests/integration/test_chembl_api.py
  - Test: get_compound("CHEMBL:25") returns complete aspirin record
  - Test: verify SMILES, InChI, molecular_weight present
  - Test: slim mode returns minimal fields
  - Test: invalid CURIE returns UNRESOLVED_ENTITY

- [X] T019 [P] [US2] Create integration test for get_compounds_batch in tests/integration/test_chembl_api.py
  - Test: batch lookup ["CHEMBL:25", "CHEMBL:941"] returns 2 compounds
  - Test: batch with mixed valid/invalid CURIEs handles gracefully
  - Test: batch defaults to slim=True

### Implementation for User Story 2

- [X] T020 [US2] Implement CURIE validation in src/lifesciences_mcp/clients/chembl.py
  - Create `_validate_chembl_curie()` helper
  - Regex: `^CHEMBL:[0-9]+$`
  - Return UNRESOLVED_ENTITY with recovery hint on failure

- [X] T021 [US2] Implement `get_compound()` in ChEMBLClient (src/lifesciences_mcp/clients/chembl.py)
  - Validate CURIE format first
  - Extract numeric ID from CURIE (e.g., "CHEMBL:25" -> 25)
  - Call SDK's `molecule.get()` via `_rate_limited_sdk_call()`
  - Transform result to Compound model

- [X] T022 [US2] Implement `_transform_compound()` helper in src/lifesciences_mcp/clients/chembl.py
  - Map SDK response fields to Compound model
  - Handle optional fields (set to None if missing)
  - Support slim mode (exclude verbose fields)

- [X] T023 [US2] Implement `get_compounds_batch()` in ChEMBLClient (src/lifesciences_mcp/clients/chembl.py)
  - Validate all CURIEs first
  - Use SDK's `molecule.filter()` for batch retrieval (single API call)
  - Default slim=True (FR-012)
  - Handle mixed valid/invalid (FR-013): return successes + error list

- [X] T024 [US2] Wire get_compound tool in src/lifesciences_mcp/servers/chembl.py
  - Replace NotImplementedError stub
  - Add slim mode support
  - Return Compound dict or ErrorEnvelope

- [X] T025 [US2] Wire get_compounds_batch tool in src/lifesciences_mcp/servers/chembl.py
  - Replace NotImplementedError stub
  - Return list[dict] or ErrorEnvelope

**Checkpoint**: User Story 2 complete - Fuzzy-to-Fact workflow works end-to-end

---

## Phase 5: User Story 3 - Cross-Database Integration (Priority: P3)

**Goal**: Enable researchers to navigate from ChEMBL compounds to UniProt, PDB, PubChem, DrugBank, KEGG

**Independent Test**: Retrieve aspirin and verify cross_references contains UniProt targets, PDB structures, and PubChem CID

### Tests for User Story 3

- [X] T026 [P] [US3] Create unit test for cross-reference mapping in tests/unit/test_chembl_models.py
  - Test: UniProt xref transformation (add UniProtKB: prefix)
  - Test: DrugBank xref transformation (add DB: prefix)
  - Test: omit-if-null pattern (missing xrefs not included)

- [X] T027 [P] [US3] Create integration test for cross-references in tests/integration/test_chembl_api.py
  - Test: aspirin (CHEMBL:25) has uniprot, pdb, pubchem_compound keys
  - Test: verify CURIE formats match 22-key registry patterns
  - Test: compound with no xrefs has empty cross_references

### Implementation for User Story 3

- [X] T028 [US3] Implement `_build_cross_references()` helper in src/lifesciences_mcp/clients/chembl.py
  - Map ChEMBL's `cross_references` array to flat object
  - Transform xref_name to our registry key (UniProt->uniprot, PDBe->pdb, etc.)
  - Apply CURIE normalization (research.md R3)
  - Omit keys entirely if no xref exists (Constitution Principle III)

- [X] T029 [US3] Integrate cross_references into get_compound response
  - Call `_build_cross_references()` during compound transformation
  - Include self-reference: chembl -> current compound ID
  - Only include in full mode (exclude from slim)

- [X] T030 [US3] Integrate cross_references into get_compounds_batch response
  - Apply same logic as get_compound
  - Respect slim parameter (default True excludes xrefs)

**Checkpoint**: User Story 3 complete - Multi-database navigation works

---

## Phase 6: User Story 4 - Error Recovery and Resilience (Priority: P4)

**Goal**: Enable researchers to self-resolve errors with clear messages and actionable recovery hints

**Independent Test**: Trigger each error condition and verify error envelopes contain correct codes and recovery hints

### Tests for User Story 4

- [X] T031 [P] [US4] Create unit test for error code mapping in tests/unit/test_chembl_client.py
  - Test: invalid CURIE -> UNRESOLVED_ENTITY
  - Test: query too short -> AMBIGUOUS_QUERY
  - Test: 404 response -> ENTITY_NOT_FOUND
  - Test: 429 response -> RATE_LIMITED
  - Test: 500/502/503 response -> UPSTREAM_ERROR

- [X] T032 [P] [US4] Create integration test for error recovery in tests/integration/test_chembl_api.py
  - Test: get_compound("CHEMBL123") (no colon) returns UNRESOLVED_ENTITY with recovery hint
  - Test: get_compound("CHEMBL:99999999999") returns ENTITY_NOT_FOUND
  - Test: search_compounds("X") returns AMBIGUOUS_QUERY with "minimum 2 characters" hint

### Implementation for User Story 4

- [X] T033 [US4] Implement `_map_sdk_error()` helper in src/lifesciences_mcp/clients/chembl.py
  - Map HTTPError(404) -> ENTITY_NOT_FOUND
  - Map HTTPError(429) -> RATE_LIMITED (with backoff info)
  - Map HTTPError(500/502/503) -> UPSTREAM_ERROR
  - Map Timeout -> UPSTREAM_ERROR
  - Map ValueError -> UNRESOLVED_ENTITY

- [X] T034 [US4] Add recovery hints to all error responses
  - UNRESOLVED_ENTITY: "Invalid ChEMBL CURIE format. Use 'CHEMBL:NNNNN' (e.g., 'CHEMBL:25')"
  - ENTITY_NOT_FOUND: "ChEMBL ID not found. Verify the compound exists at www.ebi.ac.uk/chembl"
  - AMBIGUOUS_QUERY: "Minimum query length is 2 characters"
  - RATE_LIMITED: "ChEMBL API rate limit exceeded. Wait {seconds}s before retrying"
  - UPSTREAM_ERROR: "ChEMBL API temporarily unavailable. Retry in 60 seconds"

- [X] T035 [US4] Add invalid_input field to all error envelopes
  - Include the exact input that caused the error
  - Helps users identify their mistake

- [X] T036 [US4] Add comprehensive try/except wrapping to all client methods
  - search_compounds: wrap SDK search call
  - get_compound: wrap SDK get call
  - get_compounds_batch: wrap SDK filter call
  - All return ErrorEnvelope on failure (never raise exceptions to MCP layer)

**Checkpoint**: User Story 4 complete - Production-ready error handling

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Performance validation, documentation, and code quality

### Performance Tests (FR-030, SC-001, SC-002, SC-004, SC-007)

- [X] T037 [P] Create performance test for SC-001 in tests/integration/test_performance.py
  - Test: 95% of search queries complete in <2 seconds
  - Use known compounds: aspirin, imatinib, metformin

- [X] T038 [P] Create performance test for SC-002 in tests/integration/test_performance.py
  - Test: 100% of valid CURIE lookups complete in <1 second
  - Test with CHEMBL:25, CHEMBL:941, CHEMBL:1201583

- [X] T039 [P] Create performance test for SC-007 in tests/integration/test_performance.py
  - Test: Batch of 10 compounds completes in <3s
  - Compare to 10 sequential calls (should be >70% faster)

### Documentation Updates

- [X] T040 [P] Update src/lifesciences_mcp/\_\_init\_\_.py exports
  - Add Compound, CompoundSearchCandidate to \_\_all\_\_
  - Verify ChEMBLClient is exported

- [X] T041 [P] Update CLAUDE.md with ChEMBL server documentation
  - Add ChEMBL to "Implemented Tools" section
  - Document search_compounds, get_compound, get_compounds_batch
  - Include usage example

- [X] T042 Run quickstart.md validation scenarios manually
  - Execute all example queries from quickstart.md
  - Verify expected outputs match

### Code Quality

- [X] T043 Run `uv run ruff check --fix src/lifesciences_mcp/` for linting
- [X] T044 Run `uv run ruff format src/lifesciences_mcp/` for formatting
- [X] T045 Run `uv run pyright src/lifesciences_mcp/` for type checking
- [X] T046 Run full test suite: `uv run pytest tests/ -v`
  - Target: 50+ tests (matching UniProt standard)
  - All tests must pass (109 passed)

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
- **US2 (Strict Lookup)**: Uses CompoundSearchCandidate from US1 in end-to-end workflow
- **US3 (Cross-References)**: Extends US2's Compound model - can run in parallel with US2 core
- **US4 (Error Recovery)**: Applies to all tools - can run in parallel with US2/US3

### Critical Path (Serial)

T001 → T002 → T004 → T005 → T006 → T007 → T008 → T014 → T015 → T016 (MVP complete)

### Parallel Opportunities

**Phase 2 (after T008)**:
- T009, T010 can run in parallel (different model classes)

**Phase 3-6 (after Foundational)**:
- All test tasks marked [P] within each phase
- US3 and US4 can start once US2 core (T020-T022) is complete

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001-T004)
2. Complete Phase 2: Foundational (T005-T011) - **CRITICAL: includes run_in_executor + rate limiting**
3. Complete Phase 3: User Story 1 (T012-T016)
4. **STOP and VALIDATE**: Test fuzzy search independently
5. Deploy/demo if ready

### Incremental Delivery

1. **MVP (US1)**: Fuzzy search works → researchers can discover compounds
2. **+US2**: Strict lookup works → complete Fuzzy-to-Fact workflow
3. **+US3**: Cross-references work → multi-database navigation
4. **+US4**: Error handling polished → production-ready

### Estimated Task Counts

| Phase | Tasks | Parallel Opportunities |
|-------|-------|----------------------|
| Phase 1: Setup | 4 | 1 (T003) |
| Phase 2: Foundational | 7 | 2 (T009, T010) |
| Phase 3: US1 | 5 | 2 (T012, T013) |
| Phase 4: US2 | 9 | 3 (T017, T018, T019) |
| Phase 5: US3 | 5 | 2 (T026, T027) |
| Phase 6: US4 | 6 | 2 (T031, T032) |
| Phase 7: Polish | 10 | 4 (T037-T041) |
| **Total** | **46** | **16 parallel** |

---

## Notes

- [P] tasks can run in parallel if staffed by multiple agents
- Each user story is independently testable after Foundational phase
- Constitution v1.1.0 MANDATES rate limiting in Phase 2 - do not skip T007, T008
- ADR-001 §2 requires run_in_executor for ChEMBL SDK - do not skip T005, T006
- Batch tool (T023, T025) prevents thread pool exhaustion during bulk operations
- All error handling must return ErrorEnvelope (never raise exceptions to MCP layer)
