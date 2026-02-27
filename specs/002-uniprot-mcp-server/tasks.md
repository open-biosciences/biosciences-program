# Tasks: UniProt MCP Server

**Input**: Design documents from `/specs/002-uniprot-mcp-server/`
**Prerequisites**: plan.md (required), spec.md (required for user stories)

**Tests**: Explicitly requested in spec.md - "Include a pytest-asyncio test plan" (from standard prompt template)

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

**Note**: Scaffolding already in place via `/scaffold-fastmcp uniprot` Platform Skill:
- Server stub: `src/lifesciences_mcp/servers/uniprot.py`
- Client stub: `src/lifesciences_mcp/client.py` (UniProtClient class)
- Test stub: `tests/integration/test_uniprot_api.py`
- Package exports updated in `src/lifesciences_mcp/__init__.py`

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3, US4)
- Include exact file paths in descriptions

## Path Conventions

Single project structure at repository root:
- Source: `src/lifesciences_mcp/`
- Tests: `tests/`
- Specs: `specs/002-uniprot-mcp-server/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Create protein-specific models structure (scaffolding already exists)

- [X] T001 [P] Create models/protein.py file structure in src/lifesciences_mcp/models/
- [X] T002 [P] Update models/__init__.py to export protein models

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Research and core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Research Tasks (Phase 0 from plan.md)

- [X] T003 [P] [R1] Research UniProt REST API endpoints - document in specs/002-uniprot-mcp-server/research.md
- [X] T004 [P] [R2] Research search API query syntax and ranking - test with "p53", "insulin human", "kinase"
- [X] T005 [P] [R3] Research pagination patterns - test large result sets to identify cursor/offset mechanism
- [X] T006 [P] [R4] Research rate limiting policy - review docs and test burst requests
- [X] T007 [P] [R5] Research cross-reference field mappings - fetch P04637/TP53, P38398/BRCA1 and map to 22-key registry
- [X] T008 [P] [R6] Research CURIE format validation - document regex pattern for UniProt accessions
- [X] T009 [P] [R7] Research error response formats - test invalid queries, rate limits, not-found cases

### Core Models

- [X] T010 [P] Implement Protein model with validation in src/lifesciences_mcp/models/protein.py (per plan.md D1)
- [X] T011 [P] Implement ProteinSearchCandidate model in src/lifesciences_mcp/models/protein.py (per plan.md D1)
- [X] T012 Add CURIE validation regex from R6 to Protein model
- [X] T013 Add score bounds validation (0.0-1.0) to ProteinSearchCandidate model

### Core Client Infrastructure

- [X] T014 Implement UniProtClient base class with httpx AsyncClient in src/lifesciences_mcp/client.py
- [X] T015 Implement rate limiting (10 req/s) with exponential backoff in UniProtClient (per FR-012, lessons from HGNC)
- [X] T016 Add thundering herd prevention in retry loop in UniProtClient (per HGNC review lesson)
- [X] T017 Add context manager support (__aenter__/__aexit__) to UniProtClient (per HGNC review lesson)
- [X] T018 Extract magic numbers to constants in UniProtClient (per HGNC review lesson)

### Cross-Reference Mapping

- [X] T019 Create cross-reference field mapping table from R5 research in specs/002-uniprot-mcp-server/research.md
- [X] T020 Implement UniProt→CrossReferences mapper function in src/lifesciences_mcp/client.py

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Fuzzy Protein Search (Priority: P1) 🎯 MVP

**Goal**: Agents can search for proteins using natural language queries and receive ranked candidates with UniProt CURIEs

**Independent Test**: Search for "insulin human" and verify multiple ranked candidates returned with accessions, scores, and basic metadata

### Tests for User Story 1

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [X] T021 [P] [US1] Write integration test for basic search in tests/integration/test_uniprot_api.py::test_search_proteins_basic
- [X] T022 [P] [US1] Write integration test for pagination in tests/integration/test_uniprot_api.py::test_search_proteins_pagination
- [X] T023 [P] [US1] Write integration test for query length validation in tests/integration/test_uniprot_api.py::test_search_proteins_min_length
- [X] T024 [P] [US1] Write integration test for relevance ranking in tests/integration/test_uniprot_api.py::test_search_proteins_ranking

### Implementation for User Story 1

- [X] T025 [US1] Implement search query construction in UniProtClient.search_proteins() method
- [X] T026 [US1] Implement query validation (min 2 chars) in UniProtClient.search_proteins() (per FR-011)
- [X] T027 [US1] Implement result parsing to ProteinSearchCandidate in UniProtClient.search_proteins()
- [X] T028 [US1] Implement relevance score calculation in UniProtClient.search_proteins() (per FR-013)
- [X] T029 [US1] Implement cursor-based pagination in UniProtClient.search_proteins() (per FR-007)
- [X] T030 [US1] Implement search_proteins MCP tool in src/lifesciences_mcp/servers/uniprot.py
- [X] T031 [US1] Add error handling for AMBIGUOUS_QUERY to search_proteins tool (per FR-009)
- [X] T032 [US1] Verify all US1 tests pass

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently

---

## Phase 4: User Story 2 - Strict Protein Lookup (Priority: P2)

**Goal**: Agents can retrieve complete protein records with cross-references using validated UniProt CURIEs

**Independent Test**: Provide known UniProt ID "UniProtKB:P04637" (human p53) and verify complete protein record returned with all cross-references

### Tests for User Story 2

- [X] T033 [P] [US2] Write integration test for valid CURIE lookup in tests/integration/test_uniprot_api.py::test_get_protein_p53_human
- [X] T034 [P] [US2] Write integration test for invalid CURIE format in tests/integration/test_uniprot_api.py::test_get_protein_invalid_curie
- [X] T035 [P] [US2] Write integration test for not-found CURIE in tests/integration/test_uniprot_api.py::test_get_protein_not_found
- [X] T036 [P] [US2] Write integration test for slim mode in tests/integration/test_uniprot_api.py::test_get_protein_slim

### Implementation for User Story 2

- [X] T037 [US2] Implement CURIE validation in UniProtClient.get_protein() method (per FR-003, FR-004)
- [X] T038 [US2] Implement UniProt API fetch by accession in UniProtClient.get_protein()
- [X] T039 [US2] Implement result parsing to Protein model with cross_references in UniProtClient.get_protein()
- [X] T040 [US2] Implement slim mode (id/name/organism only) in UniProtClient.get_protein() (per FR-008)
- [X] T041 [US2] Implement get_protein MCP tool in src/lifesciences_mcp/servers/uniprot.py
- [X] T042 [US2] Add error handling for UNRESOLVED_ENTITY on fuzzy input to get_protein tool (per FR-004, FR-009)
- [X] T043 [US2] Add error handling for ENTITY_NOT_FOUND to get_protein tool (per FR-009)
- [X] T044 [US2] Verify all US2 tests pass (8/8 UniProt tests passing)

**Checkpoint**: ✅ User Stories 1 AND 2 both work independently (8 tests passing)

---

## Phase 5: User Story 3 - Cross-Database Integration (Priority: P3)

**Goal**: Agents can navigate from UniProt proteins to related entities in HGNC, Ensembl, PDB, and other databases via cross-references

**Independent Test**: Retrieve "UniProtKB:P38398" (BRCA1) and verify cross-references present for HGNC, Ensembl, PDB, and other databases

### Tests for User Story 3

- [X] T045 [P] [US3] Write integration test for cross-reference presence in tests/integration/test_uniprot_api.py::test_protein_cross_references
- [X] T046 [P] [US3] Write integration test for HGNC mapping in tests/integration/test_uniprot_api.py::test_protein_hgnc_mapping
- [X] T047 [P] [US3] Write integration test for omitted keys in tests/integration/test_uniprot_api.py::test_protein_omit_null_refs

### Implementation for User Story 3

- [X] T048 [US3] Implement cross-reference extraction for HGNC in UniProtClient._map_cross_references()
- [X] T049 [US3] Implement cross-reference extraction for Ensembl in UniProtClient._map_cross_references()
- [X] T050 [US3] Implement cross-reference extraction for RefSeq in UniProtClient._map_cross_references()
- [X] T051 [US3] Implement cross-reference extraction for PDB in UniProtClient._map_cross_references()
- [X] T052 [US3] Implement cross-reference extraction for KEGG in UniProtClient._map_cross_references()
- [X] T053 [US3] Implement cross-reference extraction for OMIM in UniProtClient._map_cross_references()
- [X] T054 [US3] Implement omit-if-null pattern for cross_references (per FR-006)
- [X] T055 [US3] Verify all US3 tests pass (12/12 tests passing)

**Checkpoint**: ✅ User Stories 1, 2, AND 3 all work independently (12 tests passing)

---

## Phase 6: User Story 4 - Error Recovery and Guidance (Priority: P3)

**Goal**: Agents receive actionable recovery hints for all error conditions to autonomously correct mistakes

**Independent Test**: Trigger each error condition and verify error response includes recovery hints

### Tests for User Story 4

- [X] T056 [P] [US4] Write unit test for AMBIGUOUS_QUERY recovery hint in tests/unit/test_error_envelopes.py
- [X] T057 [P] [US4] Write unit test for RATE_LIMITED recovery hint in tests/unit/test_error_envelopes.py
- [X] T058 [P] [US4] Write unit test for UNRESOLVED_ENTITY recovery hint in tests/unit/test_error_envelopes.py
- [X] T059 [P] [US4] Write integration test for error recovery workflow in tests/integration/test_error_recovery.py

### Implementation for User Story 4

- [X] T060 [US4] Implement recovery hint for AMBIGUOUS_QUERY in src/lifesciences_mcp/client.py (per FR-010)
- [X] T061 [US4] Implement recovery hint for RATE_LIMITED in src/lifesciences_mcp/client.py (per FR-010)
- [X] T062 [US4] Implement recovery hint for UNRESOLVED_ENTITY in src/lifesciences_mcp/client.py (per FR-010)
- [X] T063 [US4] Implement recovery hint for ENTITY_NOT_FOUND in src/lifesciences_mcp/client.py (per FR-010)
- [X] T064 [US4] Implement recovery hint for UPSTREAM_ERROR in src/lifesciences_mcp/client.py (per FR-010)
- [X] T065 [US4] Verify all US4 tests pass (11 error tests passing: 7 unit + 4 integration)

**Checkpoint**: ✅ All user stories independently functional with comprehensive error guidance (15 error tests passing)

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories and production readiness

- [X] T066 [P] Write concurrency test (100 concurrent requests) in tests/integration/test_concurrency.py (per FR-014, SC-003) - ALREADY EXISTS (3 tests passing)
- [~] T067 [P] Write shutdown cleanup test - SKIPPED (ADR-004: shutdown hooks are antipattern in FastMCP)
- [X] T068 [P] Write performance test (<2s for common queries) - COMPLETE (tests/integration/test_performance.py)
- [~] T069 Add shutdown hook - SKIPPED (ADR-004: FastMCP uses module-level singleton pattern, no shutdown hooks needed)
- [X] T070 Verify rate limiter handles concurrent requests safely - VERIFIED (concurrency tests passing, implementation reviewed)
- [X] T071 [P] Update specs/002-uniprot-mcp-server/quickstart.md - COMPLETE (comprehensive usage guide created)
- [X] T072 [P] Update src/lifesciences_mcp/__init__.py to export Protein models
- [X] T073 [P] Run full test suite and verify all 20+ tests pass (47/47 tests passing)
- [X] T074 [P] Run ruff format and ruff check on all modified files (1 error fixed, 3 files reformatted)
- [X] T075 [P] Run pyright type checking on all modified files (0 errors, 0 warnings)
- [X] T076 Code review against Constitution v1.0.0 checklist (all 6 principles PASS)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
  - Research tasks (T003-T009) can run in parallel
  - Models (T010-T013) can run in parallel
  - Client tasks (T014-T018) are sequential
  - Cross-reference mapping (T019-T020) depends on R5 research
- **User Stories (Phase 3-6)**: All depend on Foundational phase completion
  - User stories CAN proceed in parallel if staffed
  - Or sequentially in priority order: US1 (P1) → US2 (P2) → US3 (P3) → US4 (P3)
- **Polish (Phase 7)**: Depends on all user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Depends only on Foundational - NO dependencies on other stories
- **User Story 2 (P2)**: Depends only on Foundational - May reference US1 for workflow but independently testable
- **User Story 3 (P3)**: Depends on US2 for get_protein implementation - Extends cross-references
- **User Story 4 (P3)**: Depends on US1 and US2 for error scenarios - Adds recovery hints to existing tools

### Within Each User Story

- Tests MUST be written FIRST and FAIL before implementation
- Implementation tasks are generally sequential within a story
- Tests can be verified in parallel once implementation complete

### Parallel Opportunities

- All Setup tasks (T001-T002) can run in parallel
- All Research tasks (T003-T009) can run in parallel
- All Model tasks (T010-T011) can run in parallel
- Tests within a user story can be written in parallel
- User Story 1 and User Story 2 can be worked on in parallel (different team members)
- All Polish tasks marked [P] can run in parallel

---

## Parallel Example: User Story 1

```bash
# Launch all tests for User Story 1 together:
Task: "Write integration test for basic search in tests/integration/test_uniprot_api.py::test_search_proteins_basic"
Task: "Write integration test for pagination in tests/integration/test_uniprot_api.py::test_search_proteins_pagination"
Task: "Write integration test for query length validation in tests/integration/test_uniprot_api.py::test_search_proteins_min_length"
Task: "Write integration test for relevance ranking in tests/integration/test_uniprot_api.py::test_search_proteins_ranking"

# After tests written, implement sequentially (dependencies exist)
```

---

## Parallel Example: Foundational Phase

```bash
# Launch all research tasks together:
Task: "[R1] Research UniProt REST API endpoints"
Task: "[R2] Research search API query syntax and ranking"
Task: "[R3] Research pagination patterns"
Task: "[R4] Research rate limiting policy"
Task: "[R5] Research cross-reference field mappings"
Task: "[R6] Research CURIE format validation"
Task: "[R7] Research error response formats"

# Launch all model tasks together:
Task: "Implement Protein model with validation"
Task: "Implement ProteinSearchCandidate model"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001-T002)
2. Complete Phase 2: Foundational (T003-T020) - CRITICAL - blocks all stories
3. Complete Phase 3: User Story 1 (T021-T032)
4. **STOP and VALIDATE**: Test User Story 1 independently - fuzzy search should work end-to-end
5. Deploy/demo if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready (research complete, client infrastructure solid)
2. Add User Story 1 → Test independently → Deploy/Demo (MVP! Agents can now search proteins)
3. Add User Story 2 → Test independently → Deploy/Demo (Agents can now get full protein records)
4. Add User Story 3 → Test independently → Deploy/Demo (Agents can now navigate cross-references)
5. Add User Story 4 → Test independently → Deploy/Demo (Agents get comprehensive error guidance)
6. Each story adds value without breaking previous stories

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together (T001-T020)
2. Once Foundational is done:
   - Developer A: User Story 1 (T021-T032)
   - Developer B: User Story 2 (T033-T044) - can start in parallel with US1
   - Developer C: User Story 3 (T045-T055) - starts after US2 implementation begins
3. Stories complete and integrate independently

---

## Task Count Summary

- **Phase 1 (Setup)**: 2 tasks ✅ Complete
- **Phase 2 (Foundational)**: 18 tasks ✅ Complete (7 research + 4 models + 5 client + 2 cross-ref)
- **Phase 3 (US1 - Fuzzy Search)**: 12 tasks ✅ Complete (4 tests + 7 implementation + 1 verification)
- **Phase 4 (US2 - Strict Lookup)**: 12 tasks ✅ Complete (4 tests + 7 implementation + 1 verification)
- **Phase 5 (US3 - Cross-DB Integration)**: 11 tasks ✅ Complete (3 tests + 7 implementation + 1 verification)
- **Phase 6 (US4 - Error Recovery)**: 10 tasks ✅ Complete (4 tests + 5 implementation + 1 verification)
- **Phase 7 (Polish)**: 11 tasks - 9 ✅ Complete, 2 ⏭️ Skipped (ADR-004)

**Total**: 76 tasks | **Completed**: 74 tasks (97%) | **Skipped**: 2 tasks (T067, T069 - ADR-004 antipattern)

**Test Coverage**: 50/50 tests passing (29 integration + 21 unit)

**Parallel Opportunities**: 29 tasks marked [P] can run in parallel within their phase

**Independent Test Criteria**:
- US1: Search "insulin human" → verify ranked candidates
- US2: Get "UniProtKB:P04637" → verify full record
- US3: Get "UniProtKB:P38398" → verify cross-references
- US4: Trigger errors → verify recovery hints

**Suggested MVP Scope**: Phase 1 + Phase 2 + Phase 3 (US1 only) = 32 tasks

---

## Notes

- [P] tasks = different files, no dependencies within phase
- [Story] label maps task to specific user story for traceability
- [R1]-[R7] labels map to research tasks from plan.md Phase 0
- Each user story should be independently completable and testable
- Verify tests fail before implementing (TDD approach)
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Code review lessons from HGNC are embedded as functional requirements (FR-012, FR-014, FR-015)
- Constitution v1.0.0 compliance validated at T076
