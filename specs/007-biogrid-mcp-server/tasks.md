# Tasks: BioGRID MCP Server

**Input**: Design documents from `/specs/007-biogrid-mcp-server/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: Integration tests are included per spec.md Test Coverage section

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3, US4)
- Include exact file paths in descriptions

## Path Conventions

- **Single project**: `src/`, `tests/` at repository root
- Paths shown below assume single project structure per plan.md

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 [P] Verify project dependencies in pyproject.toml (fastmcp, httpx, pydantic, pytest-asyncio)
- [ ] T002 [P] Verify BIOGRID_API_KEY environment variable configuration documented in README or .env.example
- [ ] T003 [P] Verify project structure matches plan.md (src/lifesciences_mcp/{clients,models,servers}/biogrid.py)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T004 Verify PaginationEnvelope and ErrorEnvelope exist in src/lifesciences_mcp/models/envelopes.py
- [ ] T005 Verify LifeSciencesClient base class exists in src/lifesciences_mcp/clients/base.py
- [ ] T006 [P] Create BioGridClient skeleton extending LifeSciencesClient in src/lifesciences_mcp/clients/biogrid.py
- [ ] T007 [P] Implement rate limiting (2 req/sec) in BioGridClient using asyncio.Lock and _rate_limited_get method
- [ ] T008 [P] Implement API key validation in BioGridClient.__init__ (check BIOGRID_API_KEY env var)
- [ ] T009 [P] Verify BioGridClient implements async context manager protocol (__aenter__, __aexit__) in src/lifesciences_mcp/clients/biogrid.py

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Gene Symbol Search (Priority: P1) 🎯 MVP

**Goal**: Validate gene symbols for BioGRID interaction queries

**Independent Test**: Can be fully tested by querying "TP53" and verifying gene symbol is valid for human (taxid 9606)

### Integration Tests for User Story 1

- [ ] T010 [P] [US1] Create integration test test_search_genes_tp53 in tests/integration/test_biogrid_api.py
- [ ] T011 [P] [US1] Create integration test test_search_genes_validation in tests/integration/test_biogrid_api.py

### Implementation for User Story 1

- [ ] T012 [P] [US1] Update BioGridSearchCandidate model: remove is_valid, add interaction_count field in src/lifesciences_mcp/models/biogrid.py
- [ ] T013 [US1] Implement search_genes method in BioGridClient using `/interactions?format=count&searchNames=true` API call (src/lifesciences_mcp/clients/biogrid.py)
- [ ] T014 [US1] Add gene symbol regex as fail-fast guard (^[A-Z0-9][A-Z0-9\-_@.]{0,29}$) before API call in search_genes
- [ ] T015 [US1] Add uppercase normalization for gene symbols in search_genes
- [ ] T016 [US1] Add organism parameter support (default: 9606) in search_genes
- [ ] T017 [US1] Wrap search_genes results in PaginationEnvelope with interaction_count
- [ ] T018 [US1] Add AMBIGUOUS_QUERY error for queries < 2 characters and ENTITY_NOT_FOUND for count == 0
- [ ] T019 [US1] Implement search_genes tool in src/lifesciences_mcp/servers/biogrid.py with updated docstring
- [ ] T020 [US1] Add tool description and parameter documentation for search_genes
- [ ] T021 [US1] Verify search_genes integration tests pass (T010, T011, nonexistent gene test)

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently

---

## Phase 4: User Story 2 - Genetic/Protein Interactions (Priority: P2)

**Goal**: Retrieve genetic and protein interactions with experimental evidence types

**Independent Test**: Can be tested by querying "TP53" and verifying interactions with MDM2, ATM including experimental system types

### Integration Tests for User Story 2

- [ ] T022 [P] [US2] Create integration test test_get_interactions_tp53 in tests/integration/test_biogrid_api.py
- [ ] T023 [P] [US2] Create integration test test_get_interactions_evidence in tests/integration/test_biogrid_api.py
- [ ] T024 [P] [US2] Create integration test test_get_interactions_counts in tests/integration/test_biogrid_api.py
- [ ] T025 [P] [US2] Create integration test test_get_interactions_limit in tests/integration/test_biogrid_api.py
- [ ] T026 [P] [US2] Create integration test test_get_interactions_validation in tests/integration/test_biogrid_api.py

### Implementation for User Story 2

- [ ] T027 [P] [US2] Create GeneticInteraction model in src/lifesciences_mcp/models/biogrid.py
- [ ] T028 [P] [US2] Create InteractionResult model in src/lifesciences_mcp/models/biogrid.py
- [ ] T029 [US2] Implement get_interactions method in BioGridClient (src/lifesciences_mcp/clients/biogrid.py)
- [ ] T030 [US2] Add gene_symbol parameter validation (uppercase, regex) in get_interactions
- [ ] T031 [US2] Add organism parameter support (default: 9606) in get_interactions
- [ ] T032 [US2] Add max_results parameter (default: 10000, max: 10000) in get_interactions
- [ ] T033 [US2] Add include_interspecies parameter (default: false) in get_interactions
- [ ] T034 [US2] Map BioGRID API response fields to GeneticInteraction model
- [ ] T035 [US2] Calculate physical_count and genetic_count in InteractionResult
- [ ] T036 [US2] Add ENTITY_NOT_FOUND error for genes with no interactions
- [ ] T037 [US2] Implement get_interactions tool in src/lifesciences_mcp/servers/biogrid.py
- [ ] T038 [US2] Add tool description and parameter documentation for get_interactions
- [ ] T039 [US2] Verify all experimental_system fields map correctly (Affinity Capture-Western, Two-hybrid, etc.)
- [ ] T040 [US2] Verify experimental_system_type is correctly set ("physical" or "genetic")
- [ ] T041 [US2] Verify pubmed_id and throughput optional fields handle None correctly
- [ ] T042 [US2] Verify get_interactions integration tests pass (T022-T026)

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently

---

## Phase 5: User Story 3 - Cross-Database Integration (Priority: P3)

**Goal**: Include cross-references to Entrez Gene IDs for database integration

**Independent Test**: Can be tested by retrieving interactions for TP53 and verifying cross_references contains Entrez Gene ID

### Integration Test for User Story 3

- [ ] T043 [US3] Create integration test verifying cross_references.entrez extraction in tests/integration/test_biogrid_api.py

### Implementation for User Story 3

- [ ] T044 [P] [US3] Create BioGridCrossReferences model in src/lifesciences_mcp/models/biogrid.py
- [ ] T045 [US3] Implement cross_references population logic in get_interactions method (map ENTREZ_GENE_A to cross_references.entrez)
- [ ] T046 [US3] Implement omit-if-null pattern for cross_references.entrez (never set to null)
- [ ] T047 [US3] Verify cross-reference integration test passes (T043)

**Checkpoint**: All user stories 1, 2, AND 3 should now work independently

---

## Phase 6: User Story 4 - Error Recovery and Resilience (Priority: P4)

**Goal**: Provide clear error messages with actionable recovery hints for all error conditions

**Independent Test**: Can be tested by attempting queries without API key, with invalid API key, and verifying error envelopes

### Integration Tests for User Story 4

- [ ] T048 [P] [US4] Create integration test test_api_key_missing in tests/integration/test_biogrid_api.py
- [ ] T049 [P] [US4] Create integration test test_api_key_invalid in tests/integration/test_biogrid_api.py
- [ ] T050 [P] [US4] Create integration test test_error_recovery in tests/integration/test_biogrid_api.py

### Implementation for User Story 4

- [ ] T051 [US4] Implement UPSTREAM_ERROR error envelope for missing API key (recovery hint: obtain key at https://webservice.thebiogrid.org/)
- [ ] T052 [US4] Implement UPSTREAM_ERROR error envelope for invalid API key
- [ ] T053 [US4] Implement RATE_LIMITED error envelope for HTTP 429 or client rate limit exceeded
- [ ] T054 [US4] Implement UPSTREAM_ERROR error envelope for BioGRID API failures (500, timeout)
- [ ] T055 [US4] Add exponential backoff retry logic for 503 errors
- [ ] T056 [US4] Add Retry-After header handling for 429 errors
- [ ] T057 [US4] Verify all error recovery integration tests pass (T048-T050)

**Checkpoint**: All user stories should now be independently functional with robust error handling

---

## Phase 7A: Token Budgeting - slim=True Support (Constitution Principle IV)

**Purpose**: Add `slim=True` parameter to both tools per Constitution Principle IV

- [ ] T069 [P] Add `to_slim()` method to InteractionResult in src/lifesciences_mcp/models/biogrid.py (returns symbol_b + experimental_system_type per interaction, plus counts)
- [ ] T070 Add `slim: bool = False` parameter to `get_interactions` method in src/lifesciences_mcp/clients/biogrid.py; call `to_slim()` when slim=True
- [ ] T071 Add `slim: bool = False` parameter to `get_interactions` tool in src/lifesciences_mcp/servers/biogrid.py
- [ ] T072 Add `slim: bool = False` parameter to `search_genes` tool in src/lifesciences_mcp/servers/biogrid.py (API consistency; no behavior change since candidates are already minimal)
- [ ] T073 Add `slim: bool = False` parameter to `search_genes` method in src/lifesciences_mcp/clients/biogrid.py (pass-through for consistency)
- [ ] T074 Create integration test for slim=True on get_interactions (verify reduced fields) in tests/integration/test_biogrid_api.py
- [ ] T075 Verify all existing integration tests still pass after slim parameter addition

---

## Phase 7B: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] T058 [P] Create unit tests for BioGridSearchCandidate validation in tests/unit/test_biogrid_models.py
- [ ] T059 [P] Create unit tests for GeneticInteraction validation in tests/unit/test_biogrid_models.py
- [ ] T060 [P] Create unit tests for InteractionResult validation in tests/unit/test_biogrid_models.py
- [ ] T061 [P] Create unit tests for BioGridClient rate limiting logic in tests/unit/test_biogrid_client.py
- [ ] T062 [P] Create unit tests for gene symbol validation pattern in tests/unit/test_biogrid_client.py
- [ ] T063 [P] Verify quickstart.md workflows execute successfully (manual validation)
- [ ] T064 [P] Run linting (uv run ruff check --fix . && uv run ruff format .)
- [ ] T065 [P] Run type checking (uv run pyright src/lifesciences_mcp/{models,clients,servers}/biogrid.py)
- [ ] T066 Verify all 12 integration tests pass (uv run pytest tests/integration/test_biogrid_api.py -v)
- [ ] T067 [P] Performance benchmark test validating P95 < 3 seconds for interaction queries (NFR-004) in tests/integration/test_biogrid_performance.py
- [ ] T068 [P] Integration test validating max 10,000 interactions limit (NFR-003) in tests/integration/test_biogrid_api.py

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3+)**: All depend on Foundational phase completion
  - User stories can then proceed in parallel (if staffed)
  - Or sequentially in priority order (P1 → P2 → P3 → P4)
- **Polish (Final Phase)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P2)**: Can start after Foundational (Phase 2) - No dependencies on other stories (independently testable)
- **User Story 3 (P3)**: Can start after Foundational (Phase 2) - Integrates with US2 by adding cross_references to InteractionResult
- **User Story 4 (P4)**: Can start after Foundational (Phase 2) - Enhances all stories with error handling

### Within Each User Story

**User Story 1**:
- Tests T010-T011 can run in parallel (both are integration tests)
- Models T012 independently
- Implementation T013-T021 has some sequential dependencies:
  - T012 (model) → T013-T018 (client methods) → T019-T020 (server tool) → T021 (verify tests)

**User Story 2**:
- Tests T022-T026 can run in parallel (all are integration tests)
- Models T027-T028 can run in parallel (different classes)
- Implementation T029-T042 has some sequential dependencies:
  - T027-T028 (models) → T029-T036 (client methods) → T037-T041 (server tool) → T042 (verify tests)

**User Story 3**:
- Test T043 independently
- Models T044 independently
- Implementation T045-T047 has sequential dependencies:
  - T044 (model) → T045-T046 (population logic) → T047 (verify test)

**User Story 4**:
- Tests T048-T050 can run in parallel (all are integration tests)
- Implementation T051-T057 can mostly run in parallel (different error types)

**Phase 7 (Polish)**:
- All unit test tasks (T058-T062) can run in parallel
- All validation tasks (T063-T068) can run in parallel

---

## Parallel Example: User Story 2 (Interactions)

```bash
# Launch all tests for User Story 2 together (T022-T026):
Task: "Create integration test test_get_interactions_tp53 in tests/integration/test_biogrid_api.py"
Task: "Create integration test test_get_interactions_evidence in tests/integration/test_biogrid_api.py"
Task: "Create integration test test_get_interactions_counts in tests/integration/test_biogrid_api.py"
Task: "Create integration test test_get_interactions_limit in tests/integration/test_biogrid_api.py"
Task: "Create integration test test_get_interactions_validation in tests/integration/test_biogrid_api.py"

# Launch both models for User Story 2 together (T027-T028):
Task: "Create GeneticInteraction model in src/lifesciences_mcp/models/biogrid.py"
Task: "Create InteractionResult model in src/lifesciences_mcp/models/biogrid.py"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: Test User Story 1 independently
5. Deploy/demo if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 → Test independently → Deploy/Demo (MVP!)
3. Add User Story 2 → Test independently → Deploy/Demo
4. Add User Story 3 → Test independently → Deploy/Demo
5. Add User Story 4 → Test independently → Deploy/Demo
6. Each story adds value without breaking previous stories

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: User Story 1
   - Developer B: User Story 2
   - Developer C: User Story 3
   - Developer D: User Story 4
3. Stories complete and integrate independently

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Verify tests fail before implementing (TDD approach)
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence

---

## Task Summary

| Phase | Tasks | Parallel | Story |
|-------|-------|----------|-------|
| Phase 1: Setup | 3 | 3 | N/A |
| Phase 2: Foundational | 6 | 4 | N/A |
| Phase 3: User Story 1 (P1) | 12 | 3 | US1 |
| Phase 4: User Story 2 (P2) | 21 | 7 | US2 |
| Phase 5: User Story 3 (P3) | 5 | 1 | US3 |
| Phase 6: User Story 4 (P4) | 10 | 3 | US4 |
| Phase 7A: Token Budgeting | 7 | 2 | N/A |
| Phase 7B: Polish | 11 | 11 | N/A |
| **TOTAL** | **75** | **34** | **4 stories** |

**Parallel efficiency**: 45% of tasks can run in parallel (34/75)

**Independent test criteria**:
- US1: Query "TP53" → Get validated gene symbol
- US2: Use gene symbol → Get interaction network with experimental evidence
- US3: Retrieve interactions → Verify cross_references contains Entrez Gene ID
- US4: Provide invalid input → Get helpful error with recovery hint

**Suggested MVP scope**: Phase 1 + Phase 2 + Phase 3 (User Story 1) = 21 tasks
