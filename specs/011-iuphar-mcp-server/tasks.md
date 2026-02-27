# Tasks: IUPHAR/GtoPdb MCP Server

**Input**: Design documents from `/specs/011-iuphar-mcp-server/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/iuphar.openapi.yaml

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3, US4, US5)
- Include exact file paths in descriptions

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 [P] Create `src/lifesciences_mcp/models/pharmacology.py` with imports and module docstring
- [ ] T002 [P] Add `IUPHAR_CURIE_PATTERN` regex constant to `src/lifesciences_mcp/models/pharmacology.py`
- [ ] T003 [P] Update `src/lifesciences_mcp/__init__.py` to export IUPHARClient
- [ ] T004 [P] Update `src/lifesciences_mcp/models/__init__.py` to export Ligand, Target, LigandSearchCandidate, TargetSearchCandidate
- [ ] T005 [P] Create `src/lifesciences_mcp/clients/iuphar.py` with imports and module docstring
- [ ] T006 [P] Create `src/lifesciences_mcp/servers/iuphar.py` with imports and module docstring
- [ ] T007 [P] Create `tests/unit/test_pharmacology_models.py` with test fixtures
- [ ] T008 [P] Create `tests/integration/test_iuphar_api.py` with test fixtures
- [ ] T009 [P] Update `tests/conftest.py` to add IUPHARClient fixtures

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Foundational Models (can run in parallel)

- [ ] T010 [P] [FOUND] Implement `LigandSearchCandidate` base class structure in `src/lifesciences_mcp/models/pharmacology.py`
- [ ] T011 [P] [FOUND] Add `id` field with CURIE pattern validation to `LigandSearchCandidate`
- [ ] T012 [P] [FOUND] Add `name` field to `LigandSearchCandidate`
- [ ] T013 [P] [FOUND] Add `type` field to `LigandSearchCandidate`
- [ ] T014 [P] [FOUND] Add `approved` field with default False to `LigandSearchCandidate`
- [ ] T015 [P] [FOUND] Add `score` field with bounds 0.0-1.0 to `LigandSearchCandidate`
- [ ] T016 [P] [FOUND] Implement `validate_iuphar_curie` field validator in `LigandSearchCandidate`
- [ ] T017 [P] [FOUND] Implement `Ligand` base class structure in `src/lifesciences_mcp/models/pharmacology.py`
- [ ] T018 [P] [FOUND] Add `id` field with CURIE pattern validation to `Ligand`
- [ ] T019 [P] [FOUND] Add `ligand_id` integer field to `Ligand`
- [ ] T020 [P] [FOUND] Add `name` field to `Ligand`
- [ ] T021 [P] [FOUND] Add `approved_name` optional field to `Ligand`
- [ ] T022 [P] [FOUND] Add `type` field to `Ligand`
- [ ] T023 [P] [FOUND] Add `abbreviation` optional field to `Ligand`
- [ ] T024 [P] [FOUND] Add `approved` boolean field to `Ligand`
- [ ] T025 [P] [FOUND] Add `approval_source` optional field to `Ligand`
- [ ] T026 [P] [FOUND] Add `who_essential` optional field to `Ligand`
- [ ] T027 [P] [FOUND] Add `withdrawn` optional field to `Ligand`
- [ ] T028 [P] [FOUND] Add `synonyms` list field to `Ligand`
- [ ] T029 [P] [FOUND] Add `cross_references` field using CrossReferences to `Ligand`
- [ ] T030 [P] [FOUND] Implement `validate_iuphar_curie` field validator in `Ligand`
- [ ] T031 [P] [FOUND] Implement `model_dump` with exclude_none in `Ligand`
- [ ] T032 [P] [FOUND] Implement `to_search_candidate` method in `Ligand`
- [ ] T033 [P] [FOUND] Implement `TargetSearchCandidate` base class structure in `src/lifesciences_mcp/models/pharmacology.py`
- [ ] T034 [P] [FOUND] Add `id` field with CURIE pattern validation to `TargetSearchCandidate`
- [ ] T035 [P] [FOUND] Add `name` field to `TargetSearchCandidate`
- [ ] T036 [P] [FOUND] Add `family` field to `TargetSearchCandidate`
- [ ] T037 [P] [FOUND] Add `type` field to `TargetSearchCandidate`
- [ ] T038 [P] [FOUND] Add `score` field with bounds 0.0-1.0 to `TargetSearchCandidate`
- [ ] T039 [P] [FOUND] Implement `validate_iuphar_curie` field validator in `TargetSearchCandidate`
- [ ] T040 [P] [FOUND] Implement `Target` base class structure in `src/lifesciences_mcp/models/pharmacology.py`
- [ ] T041 [P] [FOUND] Add `id` field with CURIE pattern validation to `Target`
- [ ] T042 [P] [FOUND] Add `target_id` integer field to `Target`
- [ ] T043 [P] [FOUND] Add `name` field to `Target`
- [ ] T044 [P] [FOUND] Add `target_family` field to `Target`
- [ ] T045 [P] [FOUND] Add `family_ids` optional list field to `Target`
- [ ] T046 [P] [FOUND] Add `species` field with default "Homo sapiens" to `Target`
- [ ] T047 [P] [FOUND] Add `gene_symbol` optional field to `Target`
- [ ] T048 [P] [FOUND] Add `cross_references` field using CrossReferences to `Target`
- [ ] T049 [P] [FOUND] Implement `validate_iuphar_curie` field validator in `Target`
- [ ] T050 [P] [FOUND] Implement `strip_html_from_name` field validator in `Target`
- [ ] T051 [P] [FOUND] Implement `model_dump` with exclude_none in `Target`
- [ ] T052 [P] [FOUND] Implement `to_search_candidate` method in `Target`

### Foundational Client (depends on models)

- [ ] T053 [FOUND] Implement `IUPHARClient` class extending `LifeSciencesClient` in `src/lifesciences_mcp/clients/iuphar.py`
- [ ] T054 [FOUND] Add `BASE_URL` constant for GtoPdb API to `IUPHARClient`
- [ ] T055 [FOUND] Add `RATE_LIMIT_DELAY` constant (1.0 seconds) to `IUPHARClient`
- [ ] T056 [FOUND] Add `MAX_RETRIES` constant (3) to `IUPHARClient`
- [ ] T057 [FOUND] Implement `__init__` with httpx.AsyncClient initialization in `IUPHARClient`
- [ ] T058 [FOUND] Add `_lock` asyncio.Lock for rate limiting in `IUPHARClient`
- [ ] T059 [FOUND] Add `_last_request_time` float for rate limiting in `IUPHARClient`
- [ ] T060 [FOUND] Implement `_rate_limited_get` method with 1 req/s enforcement in `IUPHARClient`
- [ ] T061 [FOUND] Implement exponential backoff (2^attempt) in `_rate_limited_get`
- [ ] T062 [FOUND] Implement thundering herd prevention (re-check time after lock acquire) in `_rate_limited_get`
- [ ] T063 [FOUND] Implement `_get` wrapper method for error handling in `IUPHARClient`
- [ ] T064 [FOUND] Implement cursor encoding helper in `IUPHARClient` (base64 + JSON with offset)
- [ ] T065 [FOUND] Implement cursor decoding helper in `IUPHARClient` (base64 + JSON parse)
- [ ] T066 [FOUND] Implement `_calculate_score` helper for position-based relevance in `IUPHARClient`
- [ ] T067 [FOUND] Implement `_map_ligand_cross_references` for ChEMBL/DrugBank/PubChem in `IUPHARClient`
- [ ] T068 [FOUND] Implement `_map_target_cross_references` for UniProt/Ensembl/HGNC in `IUPHARClient`
- [ ] T069 [FOUND] Implement `_strip_html_tags` helper for target name cleanup in `IUPHARClient`
- [ ] T070 [FOUND] Implement `_handle_http_error` method returning ErrorEnvelope in `IUPHARClient`

### Foundational Tests (can run in parallel after models complete)

- [ ] T071 [P] [FOUND] Unit test: IUPHAR CURIE pattern validation (valid formats) in `tests/unit/test_pharmacology_models.py`
- [ ] T072 [P] [FOUND] Unit test: IUPHAR CURIE pattern validation (invalid formats) in `tests/unit/test_pharmacology_models.py`
- [ ] T073 [P] [FOUND] Unit test: LigandSearchCandidate field validation in `tests/unit/test_pharmacology_models.py`
- [ ] T074 [P] [FOUND] Unit test: LigandSearchCandidate score bounds (0.0-1.0) in `tests/unit/test_pharmacology_models.py`
- [ ] T075 [P] [FOUND] Unit test: Ligand field validation in `tests/unit/test_pharmacology_models.py`
- [ ] T076 [P] [FOUND] Unit test: Ligand cross_references mapping in `tests/unit/test_pharmacology_models.py`
- [ ] T077 [P] [FOUND] Unit test: Ligand to_search_candidate conversion in `tests/unit/test_pharmacology_models.py`
- [ ] T078 [P] [FOUND] Unit test: Ligand model_dump exclude_none behavior in `tests/unit/test_pharmacology_models.py`
- [ ] T079 [P] [FOUND] Unit test: TargetSearchCandidate field validation in `tests/unit/test_pharmacology_models.py`
- [ ] T080 [P] [FOUND] Unit test: TargetSearchCandidate score bounds (0.0-1.0) in `tests/unit/test_pharmacology_models.py`
- [ ] T081 [P] [FOUND] Unit test: Target field validation in `tests/unit/test_pharmacology_models.py`
- [ ] T082 [P] [FOUND] Unit test: Target HTML stripping in name field in `tests/unit/test_pharmacology_models.py`
- [ ] T083 [P] [FOUND] Unit test: Target cross_references mapping in `tests/unit/test_pharmacology_models.py`
- [ ] T084 [P] [FOUND] Unit test: Target to_search_candidate conversion in `tests/unit/test_pharmacology_models.py`
- [ ] T085 [P] [FOUND] Unit test: Target model_dump exclude_none behavior in `tests/unit/test_pharmacology_models.py`

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Fuzzy Ligand Search (Priority: P1) 🎯 MVP

**Goal**: Enable agents to search for pharmacological ligands (drugs, chemicals) using natural language

**Independent Test**: Search for "ibuprofen" and verify ranked candidates with valid IUPHAR IDs

### Implementation for User Story 1

- [ ] T086 [US1] Implement `_fetch_ligands` method in `IUPHARClient` to call `/ligands` endpoint
- [ ] T087 [US1] Add query parameter mapping (name, type, approved) in `_fetch_ligands`
- [ ] T088 [US1] Implement response parsing to list[dict] in `_fetch_ligands`
- [ ] T089 [US1] Implement `_fetch_ligand_synonyms` method in `IUPHARClient` to call `/ligands/{id}/synonyms`
- [ ] T090 [US1] Implement client-side pagination with offset/limit in `search_ligands`
- [ ] T091 [US1] Implement cursor generation (base64 JSON) in `search_ligands`
- [ ] T092 [US1] Implement position-based relevance scoring (1.0 - i*0.05, min 0.1) in `search_ligands`
- [ ] T093 [US1] Implement type_filter parameter handling in `search_ligands`
- [ ] T094 [US1] Implement approved_only parameter handling in `search_ligands`
- [ ] T095 [US1] Implement LigandSearchCandidate construction from raw responses in `search_ligands`
- [ ] T096 [US1] Implement PaginationEnvelope wrapping in `search_ligands`
- [ ] T097 [US1] Implement empty result handling (204 -> empty items array) in `search_ligands`
- [ ] T098 [US1] Add `@mcp.tool()` decorator to `search_ligands` in `src/lifesciences_mcp/servers/iuphar.py`
- [ ] T099 [US1] Add query parameter with minLength=2 validation in `search_ligands` tool
- [ ] T100 [US1] Add type_filter optional parameter in `search_ligands` tool
- [ ] T101 [US1] Add approved_only optional parameter in `search_ligands` tool
- [ ] T102 [US1] Add cursor optional parameter in `search_ligands` tool
- [ ] T103 [US1] Add page_size parameter (default 50, max 100) in `search_ligands` tool
- [ ] T104 [US1] Implement tool docstring with examples in `search_ligands`

### Tests for User Story 1

- [ ] T105 [P] [US1] Integration test: search_ligands("ibuprofen") returns results in `tests/integration/test_iuphar_api.py`
- [ ] T106 [P] [US1] Integration test: search_ligands top result has valid IUPHAR CURIE in `tests/integration/test_iuphar_api.py`
- [ ] T107 [P] [US1] Integration test: search_ligands("NSAID") returns anti-inflammatory compounds in `tests/integration/test_iuphar_api.py`
- [ ] T108 [P] [US1] Integration test: search_ligands("COX inhibitor") returns enzyme inhibitors in `tests/integration/test_iuphar_api.py`
- [ ] T109 [P] [US1] Integration test: search_ligands pagination (cursor handling) in `tests/integration/test_iuphar_api.py`
- [ ] T110 [P] [US1] Integration test: search_ligands with type_filter parameter in `tests/integration/test_iuphar_api.py`
- [ ] T111 [P] [US1] Integration test: search_ligands with approved_only=True in `tests/integration/test_iuphar_api.py`
- [ ] T112 [P] [US1] Integration test: search_ligands empty results (obscure query) in `tests/integration/test_iuphar_api.py`
- [ ] T113 [P] [US1] Integration test: search_ligands score ordering (descending) in `tests/integration/test_iuphar_api.py`
- [ ] T114 [P] [US1] Integration test: search_ligands response time < 2s (SC-001) in `tests/integration/test_iuphar_api.py`
- [ ] T115 [P] [US1] Unit test: cursor encoding/decoding roundtrip in `tests/unit/test_iuphar_client.py`
- [ ] T116 [P] [US1] Unit test: position-based score calculation in `tests/unit/test_iuphar_client.py`
- [ ] T117 [P] [US1] Unit test: query parameter mapping in `tests/unit/test_iuphar_client.py`

**Checkpoint**: User Story 1 should be fully functional and testable independently

---

## Phase 4: User Story 2 - Strict Ligand Lookup (Priority: P2)

**Goal**: Enable agents to retrieve complete ligand records using resolved IUPHAR IDs

**Independent Test**: Call `get_ligand("IUPHAR:2713")` and verify all required fields are populated

### Implementation for User Story 2

- [ ] T118 [US2] Implement `_fetch_ligand_detail` method in `IUPHARClient` to call `/ligands/{id}`
- [ ] T119 [US2] Implement `_fetch_ligand_db_links` method in `IUPHARClient` to call `/ligands/{id}/databaseLinks`
- [ ] T120 [US2] Implement CURIE to numeric ID extraction in `get_ligand`
- [ ] T121 [US2] Implement CURIE format validation (reject raw strings) in `get_ligand`
- [ ] T122 [US2] Implement ChEMBL cross-reference mapping (strip "CHEMBL" prefix if present) in `_map_ligand_cross_references`
- [ ] T123 [US2] Implement DrugBank cross-reference mapping (DB\d{5}) in `_map_ligand_cross_references`
- [ ] T124 [US2] Implement PubChem cross-reference mapping (numeric CID) in `_map_ligand_cross_references`
- [ ] T125 [US2] Implement synonym fetching and merging in `get_ligand`
- [ ] T126 [US2] Implement Ligand entity construction from combined responses in `get_ligand`
- [ ] T127 [US2] Implement 404 error handling (ENTITY_NOT_FOUND with recovery hint) in `get_ligand`
- [ ] T128 [US2] Implement invalid CURIE error handling (UNRESOLVED_ENTITY) in `get_ligand`
- [ ] T129 [US2] Add `@mcp.tool()` decorator to `get_ligand` in `src/lifesciences_mcp/servers/iuphar.py`
- [ ] T130 [US2] Add iuphar_id parameter with CURIE pattern validation in `get_ligand` tool
- [ ] T131 [US2] Implement tool docstring with examples in `get_ligand`

### Tests for User Story 2

- [ ] T132 [P] [US2] Integration test: get_ligand("IUPHAR:2713") returns complete ibuprofen record in `tests/integration/test_iuphar_api.py`
- [ ] T133 [P] [US2] Integration test: get_ligand cross_references populated (ChEMBL, DrugBank, PubChem) in `tests/integration/test_iuphar_api.py`
- [ ] T134 [P] [US2] Integration test: get_ligand synonyms list populated in `tests/integration/test_iuphar_api.py`
- [ ] T135 [P] [US2] Integration test: get_ligand with invalid format returns UNRESOLVED_ENTITY in `tests/integration/test_iuphar_api.py`
- [ ] T136 [P] [US2] Integration test: get_ligand("ibuprofen") rejects raw string in `tests/integration/test_iuphar_api.py`
- [ ] T137 [P] [US2] Integration test: get_ligand with non-existent ID returns ENTITY_NOT_FOUND in `tests/integration/test_iuphar_api.py`
- [ ] T138 [P] [US2] Integration test: Fuzzy-to-Fact workflow (search -> get) in `tests/integration/test_iuphar_api.py`
- [ ] T139 [P] [US2] Integration test: get_ligand omit-if-null pattern (no null cross_references) in `tests/integration/test_iuphar_api.py`
- [ ] T140 [P] [US2] Unit test: CURIE extraction (IUPHAR:2713 -> 2713) in `tests/unit/test_iuphar_client.py`
- [ ] T141 [P] [US2] Unit test: ChEMBL cross-reference normalization in `tests/unit/test_iuphar_client.py`
- [ ] T142 [P] [US2] Unit test: DrugBank ID format validation in `tests/unit/test_iuphar_client.py`

**Checkpoint**: User Stories 1 AND 2 should both work independently

---

## Phase 5: User Story 3 - Fuzzy Target Search (Priority: P3)

**Goal**: Enable agents to search for pharmacological targets (receptors, enzymes, ion channels)

**Independent Test**: Search for "dopamine receptor" and verify ranked candidates

### Implementation for User Story 3

- [ ] T143 [US3] Implement `_fetch_targets` method in `IUPHARClient` to call `/targets` endpoint
- [ ] T144 [US3] Add query parameter mapping (name, type, geneSymbol) in `_fetch_targets`
- [ ] T145 [US3] Implement response parsing to list[dict] in `_fetch_targets`
- [ ] T146 [US3] Implement HTML tag stripping in target name parsing in `_fetch_targets`
- [ ] T147 [US3] Implement client-side pagination with offset/limit in `search_targets`
- [ ] T148 [US3] Implement cursor generation (base64 JSON) in `search_targets`
- [ ] T149 [US3] Implement position-based relevance scoring in `search_targets`
- [ ] T150 [US3] Implement type_filter parameter handling in `search_targets`
- [ ] T151 [US3] Implement TargetSearchCandidate construction from raw responses in `search_targets`
- [ ] T152 [US3] Implement PaginationEnvelope wrapping in `search_targets`
- [ ] T153 [US3] Implement empty result handling (204 -> empty items array) in `search_targets`
- [ ] T154 [US3] Add `@mcp.tool()` decorator to `search_targets` in `src/lifesciences_mcp/servers/iuphar.py`
- [ ] T155 [US3] Add query parameter with minLength=2 validation in `search_targets` tool
- [ ] T156 [US3] Add type_filter optional parameter in `search_targets` tool
- [ ] T157 [US3] Add cursor optional parameter in `search_targets` tool
- [ ] T158 [US3] Add page_size parameter (default 50, max 100) in `search_targets` tool
- [ ] T159 [US3] Implement tool docstring with examples in `search_targets`

### Tests for User Story 3

- [ ] T160 [P] [US3] Integration test: search_targets("dopamine receptor") returns results in `tests/integration/test_iuphar_api.py`
- [ ] T161 [P] [US3] Integration test: search_targets top result has valid IUPHAR CURIE in `tests/integration/test_iuphar_api.py`
- [ ] T162 [P] [US3] Integration test: search_targets("GPCR") returns G protein-coupled receptors in `tests/integration/test_iuphar_api.py`
- [ ] T163 [P] [US3] Integration test: search_targets("DRD2") returns D2 receptor in `tests/integration/test_iuphar_api.py`
- [ ] T164 [P] [US3] Integration test: search_targets pagination (cursor handling) in `tests/integration/test_iuphar_api.py`
- [ ] T165 [P] [US3] Integration test: search_targets with type_filter parameter in `tests/integration/test_iuphar_api.py`
- [ ] T166 [P] [US3] Integration test: search_targets empty results (obscure query) in `tests/integration/test_iuphar_api.py`
- [ ] T167 [P] [US3] Integration test: search_targets HTML stripping (D<sub>2</sub> -> D2) in `tests/integration/test_iuphar_api.py`
- [ ] T168 [P] [US3] Integration test: search_targets score ordering (descending) in `tests/integration/test_iuphar_api.py`
- [ ] T169 [P] [US3] Integration test: search_targets response time < 2s (SC-001) in `tests/integration/test_iuphar_api.py`
- [ ] T170 [P] [US3] Unit test: HTML tag stripping regex in `tests/unit/test_iuphar_client.py`

**Checkpoint**: User Stories 1, 2, AND 3 should all work independently

---

## Phase 6: User Story 4 - Strict Target Lookup (Priority: P4)

**Goal**: Enable agents to retrieve complete target records using resolved IUPHAR IDs

**Independent Test**: Call `get_target("IUPHAR:215")` and verify all required fields

### Implementation for User Story 4

- [ ] T171 [US4] Implement `_fetch_target_detail` method in `IUPHARClient` to call `/targets/{id}`
- [ ] T172 [US4] Implement `_fetch_target_db_links` method in `IUPHARClient` to call `/targets/{id}/databaseLinks`
- [ ] T173 [US4] Implement CURIE to numeric ID extraction in `get_target`
- [ ] T174 [US4] Implement CURIE format validation (reject raw strings) in `get_target`
- [ ] T175 [US4] Implement species filtering (Human only) in `_map_target_cross_references`
- [ ] T176 [US4] Implement UniProt cross-reference mapping (list format per ADR-001) in `_map_target_cross_references`
- [ ] T177 [US4] Implement Ensembl gene cross-reference mapping in `_map_target_cross_references`
- [ ] T178 [US4] Implement Entrez gene cross-reference mapping in `_map_target_cross_references`
- [ ] T179 [US4] Implement RefSeq cross-reference mapping (list format) in `_map_target_cross_references`
- [ ] T180 [US4] Implement ChEMBL target cross-reference mapping in `_map_target_cross_references`
- [ ] T181 [US4] Implement OMIM cross-reference mapping in `_map_target_cross_references`
- [ ] T182 [US4] Implement Orphanet cross-reference mapping in `_map_target_cross_references`
- [ ] T183 [US4] Implement HTML tag stripping in target name field in `get_target`
- [ ] T184 [US4] Implement Target entity construction from combined responses in `get_target`
- [ ] T185 [US4] Implement 404 error handling (ENTITY_NOT_FOUND with recovery hint) in `get_target`
- [ ] T186 [US4] Implement invalid CURIE error handling (UNRESOLVED_ENTITY) in `get_target`
- [ ] T187 [US4] Add `@mcp.tool()` decorator to `get_target` in `src/lifesciences_mcp/servers/iuphar.py`
- [ ] T188 [US4] Add iuphar_id parameter with CURIE pattern validation in `get_target` tool
- [ ] T189 [US4] Implement tool docstring with examples in `get_target`

### Tests for User Story 4

- [ ] T190 [P] [US4] Integration test: get_target("IUPHAR:215") returns complete D2 receptor record in `tests/integration/test_iuphar_api.py`
- [ ] T191 [P] [US4] Integration test: get_target cross_references populated (UniProt, Ensembl, HGNC, etc.) in `tests/integration/test_iuphar_api.py`
- [ ] T192 [P] [US4] Integration test: get_target gene_symbol populated ("DRD2") in `tests/integration/test_iuphar_api.py`
- [ ] T193 [P] [US4] Integration test: get_target name has HTML stripped in `tests/integration/test_iuphar_api.py`
- [ ] T194 [P] [US4] Integration test: get_target with invalid format returns UNRESOLVED_ENTITY in `tests/integration/test_iuphar_api.py`
- [ ] T195 [P] [US4] Integration test: get_target("DRD2") rejects gene symbol in `tests/integration/test_iuphar_api.py`
- [ ] T196 [P] [US4] Integration test: get_target with non-existent ID returns ENTITY_NOT_FOUND in `tests/integration/test_iuphar_api.py`
- [ ] T197 [P] [US4] Integration test: Fuzzy-to-Fact workflow for targets (search -> get) in `tests/integration/test_iuphar_api.py`
- [ ] T198 [P] [US4] Integration test: get_target omit-if-null pattern (no null cross_references) in `tests/integration/test_iuphar_api.py`
- [ ] T199 [P] [US4] Integration test: get_target species filter (only human cross_references) in `tests/integration/test_iuphar_api.py`
- [ ] T200 [P] [US4] Unit test: species filtering logic in `tests/unit/test_iuphar_client.py`
- [ ] T201 [P] [US4] Unit test: UniProt list format in `tests/unit/test_iuphar_client.py`

**Checkpoint**: All four primary user stories (fuzzy + strict for both ligands and targets) should work independently

---

## Phase 7: User Story 5 - Error Recovery (Priority: P5)

**Goal**: Enable agents to self-correct from errors using actionable recovery hints

**Independent Test**: Trigger each error type and verify recovery hints are actionable

### Implementation for User Story 5

- [ ] T202 [US5] Implement UNRESOLVED_ENTITY error for ligand search in `search_ligands`
- [ ] T203 [US5] Implement UNRESOLVED_ENTITY error for target search in `search_targets`
- [ ] T204 [US5] Implement UNRESOLVED_ENTITY recovery hint (point to search_ligands) in `get_ligand`
- [ ] T205 [US5] Implement UNRESOLVED_ENTITY recovery hint (point to search_targets) in `get_target`
- [ ] T206 [US5] Implement ENTITY_NOT_FOUND recovery hint for ligands (suggest get_target) in `get_ligand`
- [ ] T207 [US5] Implement ENTITY_NOT_FOUND recovery hint for targets (suggest get_ligand) in `get_target`
- [ ] T208 [US5] Implement RATE_LIMITED error detection (429, 503 status codes) in `_handle_http_error`
- [ ] T209 [US5] Implement RATE_LIMITED recovery hint (retry with backoff) in `_handle_http_error`
- [ ] T210 [US5] Implement UPSTREAM_ERROR error for network failures in `_handle_http_error`
- [ ] T211 [US5] Implement UPSTREAM_ERROR recovery hint (GtoPdb availability) in `_handle_http_error`
- [ ] T212 [US5] Implement AMBIGUOUS_QUERY error for overly broad searches in `search_ligands`
- [ ] T213 [US5] Implement AMBIGUOUS_QUERY recovery hint (refine query) in `search_ligands`

### Tests for User Story 5

- [ ] T214 [P] [US5] Integration test: UNRESOLVED_ENTITY error for get_ligand("ibuprofen") in `tests/integration/test_iuphar_api.py`
- [ ] T215 [P] [US5] Integration test: UNRESOLVED_ENTITY recovery hint leads to successful search in `tests/integration/test_iuphar_api.py`
- [ ] T216 [P] [US5] Integration test: UNRESOLVED_ENTITY error for get_target("DRD2") in `tests/integration/test_iuphar_api.py`
- [ ] T217 [P] [US5] Integration test: UNRESOLVED_ENTITY recovery hint leads to successful search for targets in `tests/integration/test_iuphar_api.py`
- [ ] T218 [P] [US5] Integration test: ENTITY_NOT_FOUND error for get_ligand("IUPHAR:999999") in `tests/integration/test_iuphar_api.py`
- [ ] T219 [P] [US5] Integration test: ENTITY_NOT_FOUND recovery hint suggests trying get_target in `tests/integration/test_iuphar_api.py`
- [ ] T220 [P] [US5] Integration test: Complete error->hint->recovery->success cycle for ligands in `tests/integration/test_iuphar_api.py`
- [ ] T221 [P] [US5] Integration test: Complete error->hint->recovery->success cycle for targets in `tests/integration/test_iuphar_api.py`
- [ ] T222 [P] [US5] Unit test: ErrorEnvelope structure validation in `tests/unit/test_iuphar_client.py`
- [ ] T223 [P] [US5] Unit test: All error codes have recovery hints in `tests/unit/test_iuphar_client.py`
- [ ] T224 [P] [US5] Unit test: Error code enum values match spec in `tests/unit/test_iuphar_client.py`

**Checkpoint**: All error scenarios should provide actionable recovery guidance

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

### Documentation

- [ ] T225 [P] Update `specs/011-iuphar-mcp-server/quickstart.md` with actual tool signatures
- [ ] T226 [P] Add cross-database integration examples to quickstart.md
- [ ] T227 [P] Add error recovery examples to quickstart.md
- [ ] T228 [P] Update main README.md with IUPHAR server entry
- [ ] T229 [P] Add IUPHAR server to Claude.md project table

### Performance & Reliability

- [ ] T230 [P] Performance test: search_ligands response time < 2s for 95% of queries in `tests/integration/test_iuphar_api.py`
- [ ] T231 [P] Performance test: get_ligand response time < 2s for 95% of queries in `tests/integration/test_iuphar_api.py`
- [ ] T232 [P] Performance test: search_targets response time < 2s for 95% of queries in `tests/integration/test_iuphar_api.py`
- [ ] T233 [P] Performance test: get_target response time < 2s for 95% of queries in `tests/integration/test_iuphar_api.py`
- [ ] T234 [P] Rate limiting test: 1 req/s enforcement in `tests/integration/test_iuphar_api.py`
- [ ] T235 [P] Rate limiting test: concurrent request handling (no race conditions) in `tests/integration/test_iuphar_api.py`
- [ ] T236 [P] Rate limiting test: exponential backoff on 429 errors in `tests/integration/test_iuphar_api.py`
- [ ] T237 [P] Rate limiting test: thundering herd prevention in `tests/integration/test_iuphar_api.py`

### Cross-Reference Coverage

- [ ] T238 [P] Integration test: ligand cross_references populated for approved drugs in `tests/integration/test_iuphar_api.py`
- [ ] T239 [P] Integration test: target cross_references populated for GPCRs in `tests/integration/test_iuphar_api.py`
- [ ] T240 [P] Integration test: ChEMBL cross-reference format validation in `tests/integration/test_iuphar_api.py`
- [ ] T241 [P] Integration test: DrugBank cross-reference format validation in `tests/integration/test_iuphar_api.py`
- [ ] T242 [P] Integration test: UniProt cross-reference format validation in `tests/integration/test_iuphar_api.py`
- [ ] T243 [P] Integration test: Ensembl cross-reference format validation in `tests/integration/test_iuphar_api.py`

### Edge Cases

- [ ] T244 [P] Integration test: ligand with no targets returns empty associations in `tests/integration/test_iuphar_api.py`
- [ ] T245 [P] Integration test: target with no ligands returns empty associations in `tests/integration/test_iuphar_api.py`
- [ ] T246 [P] Integration test: ligand with no cross_references has empty object in `tests/integration/test_iuphar_api.py`
- [ ] T247 [P] Integration test: target with no cross_references has empty object in `tests/integration/test_iuphar_api.py`
- [ ] T248 [P] Integration test: ligand with no synonyms has empty list in `tests/integration/test_iuphar_api.py`
- [ ] T249 [P] Integration test: multi-word query handling in search_ligands in `tests/integration/test_iuphar_api.py`
- [ ] T250 [P] Integration test: multi-word query handling in search_targets in `tests/integration/test_iuphar_api.py`
- [ ] T251 [P] Integration test: special characters in query (parentheses, hyphens) in `tests/integration/test_iuphar_api.py`

### Validation & Schema Compliance

- [ ] T252 [P] Validation test: all ligand types match GtoPdb enum in `tests/integration/test_iuphar_api.py`
- [ ] T253 [P] Validation test: all target types match GtoPdb enum in `tests/integration/test_iuphar_api.py`
- [ ] T254 [P] Validation test: PaginationEnvelope schema compliance in `tests/integration/test_iuphar_api.py`
- [ ] T255 [P] Validation test: ErrorEnvelope schema compliance in `tests/integration/test_iuphar_api.py`
- [ ] T256 [P] Validation test: Ligand schema matches data-model.md in `tests/integration/test_iuphar_api.py`
- [ ] T257 [P] Validation test: Target schema matches data-model.md in `tests/integration/test_iuphar_api.py`

### Code Quality

- [ ] T258 [P] Run `uv run ruff check --fix .` and verify no issues
- [ ] T259 [P] Run `uv run ruff format .` and verify formatting
- [ ] T260 [P] Run `uv run pyright` and verify no type errors
- [ ] T261 [P] Verify all docstrings follow project conventions
- [ ] T262 [P] Verify all error messages are actionable
- [ ] T263 [P] Verify all recovery hints specify correct tool names

### Quickstart Validation

- [ ] T264 Run all workflows from `quickstart.md` and verify they work as documented
- [ ] T265 Verify all example CURIEs in quickstart.md are valid
- [ ] T266 Verify all example responses in quickstart.md match actual output
- [ ] T267 Verify all error examples in quickstart.md are accurate

### Final Integration

- [ ] T268 End-to-end test: ligand discovery workflow (search -> get -> cross-ref to ChEMBL) in `tests/integration/test_iuphar_api.py`
- [ ] T269 End-to-end test: target discovery workflow (search -> get -> cross-ref to UniProt) in `tests/integration/test_iuphar_api.py`
- [ ] T270 End-to-end test: multi-database workflow (IUPHAR ligand -> ChEMBL compound -> targets) in `tests/integration/test_iuphar_api.py`
- [ ] T271 Verify all 5 user stories pass independent tests
- [ ] T272 Verify all success criteria (SC-001 through SC-008) are met
- [ ] T273 Run full test suite and verify 100% pass rate

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3-7)**: All depend on Foundational phase completion
  - User stories can then proceed in parallel (if staffed)
  - Or sequentially in priority order (P1 → P2 → P3 → P4 → P5)
- **Polish (Phase 8)**: Depends on all user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P2)**: Can start after Foundational (Phase 2) - Integrates with US1 for Fuzzy-to-Fact
- **User Story 3 (P3)**: Can start after Foundational (Phase 2) - Independent from ligand stories
- **User Story 4 (P4)**: Can start after Foundational (Phase 2) - Integrates with US3 for Fuzzy-to-Fact
- **User Story 5 (P5)**: Can start after Foundational (Phase 2) - Affects all stories but testable independently

### Within Each User Story

- Tests SHOULD be written and FAIL before implementation (TDD)
- Models before services
- Services before endpoints
- Core implementation before integration
- Story complete before moving to next priority

### Parallel Opportunities

- All Setup tasks (T001-T009) can run in parallel
- All Foundational model tasks (T010-T052) can run in parallel
- All Foundational test tasks (T071-T085) can run in parallel after models complete
- Once Foundational phase completes, all user stories can start in parallel (if team capacity allows)
- All tests for a user story can run in parallel
- All documentation tasks in Phase 8 can run in parallel

---

## Implementation Strategy

### MVP First (User Story 1 + 2 Only)

1. Complete Phase 1: Setup (T001-T009)
2. Complete Phase 2: Foundational (T010-T085) - CRITICAL
3. Complete Phase 3: User Story 1 - Fuzzy Ligand Search (T086-T117)
4. Complete Phase 4: User Story 2 - Strict Ligand Lookup (T118-T142)
5. **STOP and VALIDATE**: Test ligand workflows independently
6. Deploy/demo if ready (basic drug discovery capability)

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 + 2 → Test independently → Deploy/Demo (MVP - ligand discovery!)
3. Add User Story 3 + 4 → Test independently → Deploy/Demo (full dual-entity support!)
4. Add User Story 5 → Test independently → Deploy/Demo (autonomous error recovery!)
5. Polish phase → Final validation → Production release

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together (T001-T085)
2. Once Foundational is done:
   - Developer A: User Story 1 + 2 (ligands)
   - Developer B: User Story 3 + 4 (targets)
   - Developer C: User Story 5 (error recovery)
3. Stories complete and integrate independently
4. Team collaborates on Phase 8 polish

---

## Task Statistics

**Total Tasks**: 273

### Tasks by Phase

| Phase | Tasks | Percentage |
|-------|-------|------------|
| Setup | 9 | 3% |
| Foundational | 76 | 28% |
| User Story 1 | 32 | 12% |
| User Story 2 | 25 | 9% |
| User Story 3 | 28 | 10% |
| User Story 4 | 31 | 11% |
| User Story 5 | 23 | 8% |
| Polish | 49 | 18% |

### Tasks by Category

| Category | Tasks |
|----------|-------|
| Implementation | 128 |
| Integration Tests | 89 |
| Unit Tests | 37 |
| Documentation | 5 |
| Validation | 14 |

### Parallel Opportunities

**Phase 1 (Setup)**: 9 tasks can run in parallel
**Phase 2 (Foundational Models)**: 43 tasks can run in parallel (T010-T052)
**Phase 2 (Foundational Tests)**: 15 tasks can run in parallel (T071-T085)
**Phase 8 (Polish)**: Most tasks can run in parallel (estimated 35+ parallel tasks)

**Total Parallel Tasks**: ~102 tasks (37% can run in parallel with proper staffing)

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- All error codes must have recovery hints (FR-020, FR-021)
- Rate limiting at 1 req/s (FR-022) stricter than HGNC/UniProt (10 req/s)
- CURIE pattern `^IUPHAR:\d+$` applies to both ligands and targets
- HTML stripping required for target names only
- Species filtering (human only) required for target cross-references
- Omit-if-null pattern for all optional fields (Constitution Principle III)
- Position-based scoring (1.0 - i*0.05, min 0.1) for both search tools
