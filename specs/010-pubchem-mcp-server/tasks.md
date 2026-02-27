# Tasks: PubChem MCP Server

**Input**: Design documents from `/specs/010-pubchem-mcp-server/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/
**Branch**: `010-pubchem-mcp-server`
**Date**: 2026-01-01

## Artifact Reference Table

| Document | Purpose | Key Sections |
|----------|---------|--------------|
| [spec.md](./spec.md) | User stories P1-P4, FR-001 to FR-020 | Fuzzy search, strict lookup, cross-refs, error recovery |
| [plan.md](./plan.md) | Technical context, project structure | R1-R7 research summary, Constitution check |
| [research.md](./research.md) | API research, rate limiting, cross-refs | R1-R7 decisions |
| [data-model.md](./data-model.md) | Pydantic models, field mappings | PubChemSearchCandidate, PubChemCompound |
| [contracts/search_compounds.json](./contracts/search_compounds.json) | Tool contract | Parameters, returns, errors |
| [contracts/get_compound.json](./contracts/get_compound.json) | Tool contract | Parameters, returns, cross-ref keys |
| [quickstart.md](./quickstart.md) | Usage examples, workflows | 4 workflows, error recovery |

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3, US4)
- Include exact file paths in descriptions

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [X] T001 Verify branch `010-pubchem-mcp-server` is checked out and clean
- [X] T002 [P] Create `src/lifesciences_mcp/clients/pubchem.py` with PubChemClient class stub extending LifeSciencesClient
- [X] T003 [P] Create `src/lifesciences_mcp/models/pubchem_compound.py` with model stubs (PubChemSearchCandidate, PubChemCompound)
- [X] T004 [P] Create `src/lifesciences_mcp/servers/pubchem.py` with FastMCP server stub and module-level singleton pattern (ADR-004)
- [X] T005 [P] Create `tests/unit/test_pubchem_models.py` with test stubs
- [X] T006 [P] Create `tests/unit/test_pubchem_client.py` with test stubs
- [X] T007 [P] Create `tests/integration/test_pubchem_api.py` with test stubs and integration marker

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

### Rate Limiting Infrastructure (Constitution v1.1.0 MANDATORY)

- [X] T008 Implement PubChemClient base class extending LifeSciencesClient in `src/lifesciences_mcp/clients/pubchem.py`
- [X] T009 Configure PUBCHEM_BASE_URL constant: `https://pubchem.ncbi.nlm.nih.gov/rest/pug`
- [X] T010 Configure PROPERTY_LIST constant: `MolecularFormula,MolecularWeight,CanonicalSMILES,IsomericSMILES,InChI,InChIKey,IUPACName,Title`
- [X] T011 Implement `_rate_lock` asyncio.Lock for thread-safe rate limiting
- [X] T012 Implement per-second rate limiting (5 req/s) with 200ms minimum interval (FR-017, R2)
- [X] T013 Implement per-minute rate limiting (400 req/min) with rolling window deque (FR-018, R2)
- [X] T014 Implement `_rate_limited_get()` method combining both rate limits
- [X] T015 Implement exponential backoff on HTTP 503 responses (FR-019, R2): base 1s, max 60s
- [X] T016 Implement thundering herd prevention: lock + re-check timing after acquire (FR-020, R2)
- [X] T017 Add jitter to backoff delay for distributed systems safety

### CURIE Validation Infrastructure (R4)

- [X] T018 Define PUBCHEM_CURIE_PATTERN regex constant: `^PubChem:CID\d+$` in `models/pubchem_compound.py`
- [X] T019 Implement `_validate_pubchem_curie()` method returning int CID or ErrorEnvelope
- [X] T020 Implement `_format_pubchem_curie()` helper: `f"PubChem:CID{cid}"`

### Error Envelope Infrastructure (R6)

- [X] T021 Import ErrorEnvelope, ErrorDetail, ErrorCode from `models/envelopes.py`
- [X] T022 Implement `_map_http_error()` method mapping HTTP status to canonical error codes
- [X] T023 Map HTTP 400 → UNRESOLVED_ENTITY with CURIE format hint
- [X] T024 Map HTTP 404 → ENTITY_NOT_FOUND with search_compounds hint
- [X] T025 Map HTTP 503 → RATE_LIMITED with wait time hint
- [X] T026 Map HTTP 500/502/timeout → UPSTREAM_ERROR with retry hint

### Unit Tests for Foundation

- [X] T027 [P] Write unit test `test_rate_limit_per_second` in `tests/unit/test_pubchem_client.py`
- [X] T028 [P] Write unit test `test_rate_limit_per_minute` with mock time
- [X] T029 [P] Write unit test `test_curie_validation_valid` for valid CURIE formats
- [X] T030 [P] Write unit test `test_curie_validation_invalid` for invalid formats (CID2244, 2244, PubChem:2244)
- [X] T031 [P] Write unit test `test_exponential_backoff` with mock responses
- [X] T032 [P] Write unit test `test_error_mapping_400` for HTTP 400 handling
- [X] T033 [P] Write unit test `test_error_mapping_404` for HTTP 404 handling
- [X] T034 [P] Write unit test `test_error_mapping_503` for HTTP 503 handling

**Checkpoint**: ✅ Foundation ready - rate limiting, CURIE validation, error handling complete

---

## Phase 3: User Story 1 - Fuzzy Compound Search (Priority: P1) 🎯 MVP

**Goal**: An LLM agent needs to find chemical compounds in PubChem using natural language queries (compound names, synonyms, or chemical identifiers). The agent receives ranked candidates with PubChem CIDs that can be used for strict lookups.

**Independent Test**: Search for "aspirin" and verify ranked candidates are returned with valid PubChem CIDs.

**References**: FR-001 to FR-005, R1, R7, contracts/search_compounds.json

### Models for User Story 1

- [X] T035 [P] [US1] Create PubChemSearchCandidate model with fields: id, name, molecular_formula, score in `models/pubchem_compound.py`
- [X] T036 [P] [US1] Add @field_validator for id field enforcing PUBCHEM_CURIE_PATTERN
- [X] T037 [P] [US1] Add Field constraints: score ge=0.0, le=1.0 (data-model.md)
- [X] T038 [P] [US1] Add model_config with json_schema_extra examples from data-model.md
- [X] T039 [US1] Import PaginationEnvelope from `models/envelopes.py`

### Client Methods for User Story 1

- [X] T040 [US1] Implement `_search_by_name()` private method calling `/compound/name/{name}/cids/JSON` (R1)
- [X] T041 [US1] Handle URL encoding for compound names with special characters using urllib.parse.quote
- [X] T042 [US1] Parse IdentifierList.CID array from PubChem response (R1)
- [X] T043 [US1] Handle HTTP 404 response as empty result set (zero matches)
- [X] T044 [US1] Implement `_get_compound_properties()` method to fetch Title and MolecularFormula for CID list
- [X] T045 [US1] Implement `_get_compound_properties()` method to fetch Title and MolecularFormula for CID list
- [X] T046 [US1] Implement cursor-based pagination with base64-encoded offset (FR-003)
- [X] T047 [US1] Implement `_encode_cursor()` and `_decode_cursor()` helper methods
- [X] T048 [US1] Implement linear decay scoring algorithm: score = max(0.1, 1.0 - (position * 0.05)) (R7)
- [X] T049 [US1] Implement minimum query length validation (2 chars) returning AMBIGUOUS_QUERY error
- [X] T050 [US1] Implement `search_compounds()` public method in PubChemClient

### Server Tool for User Story 1

- [X] T051 [US1] Implement `get_client()` singleton function in `servers/pubchem.py` (ADR-004)
- [X] T052 [US1] Implement `@mcp.tool() search_compounds` tool with parameters: query, slim, cursor, page_size
- [X] T053 [US1] Add comprehensive docstring with usage examples from contracts/search_compounds.json
- [X] T054 [US1] Return PaginationEnvelope with items, pagination.cursor, pagination.total_count, pagination.page_size

### Unit Tests for User Story 1

- [X] T055 [P] [US1] Write unit test `test_search_candidate_model_valid` in `tests/unit/test_pubchem_models.py`
- [X] T056 [P] [US1] Write unit test `test_search_candidate_curie_validation` for invalid CURIEs
- [X] T057 [P] [US1] Write unit test `test_search_candidate_score_bounds` for score validation
- [X] T058 [P] [US1] Write unit test `test_search_by_name_aspirin` with mocked response
- [X] T059 [P] [US1] Write unit test `test_search_pagination_cursor` for cursor encoding/decoding
- [X] T060 [P] [US1] Write unit test `test_search_scoring_algorithm` verifying linear decay

### Integration Tests for User Story 1

- [X] T061 [P] [US1] Write integration test `test_search_compounds_aspirin` in `tests/integration/test_pubchem_api.py`
- [X] T062 [P] [US1] Write integration test `test_search_compounds_acetylsalicylic_acid` (synonym search)
- [X] T063 [P] [US1] Write integration test `test_search_compounds_ibuprofen` with related compounds
- [X] T064 [P] [US1] Write integration test `test_search_compounds_empty_results` for nonexistent compound
- [X] T065 [P] [US1] Write integration test `test_search_compounds_pagination` with cursor
- [X] T066 [P] [US1] Write integration test `test_search_compounds_query_too_short` for AMBIGUOUS_QUERY

**Checkpoint**: User Story 1 complete - agents can discover compound CIDs via fuzzy search

---

## Phase 4: User Story 2 - Strict Compound Lookup (Priority: P2)

**Goal**: An LLM agent needs to retrieve complete compound information using a resolved PubChem CID. The agent receives full compound details including SMILES, InChI, molecular formula, weight, IUPAC name, and cross-references to other databases.

**Independent Test**: Call `get_compound("PubChem:CID2244")` (aspirin) and verify all required fields are populated.

**References**: FR-006 to FR-013, R4, R5, contracts/get_compound.json

### Models for User Story 2

- [ ] T067 [P] [US2] Create PubChemCompound model with all fields from data-model.md in `models/pubchem_compound.py`
- [ ] T068 [P] [US2] Add id field with @field_validator enforcing PUBCHEM_CURIE_PATTERN
- [ ] T069 [P] [US2] Add name, iupac_name, molecular_formula fields (str | None)
- [ ] T070 [P] [US2] Add molecular_weight field (float | None) with description "in Daltons (g/mol)" (FR-013)
- [ ] T071 [P] [US2] Add canonical_smiles, isomeric_smiles fields (str | None) (FR-010)
- [ ] T072 [P] [US2] Add inchi, inchikey fields (str | None) (FR-010)
- [ ] T073 [P] [US2] Add synonyms field: list[str] with default_factory=list
- [ ] T074 [P] [US2] Add cross_references field: dict[str, list[str]] with default_factory=dict (FR-011)
- [ ] T075 [US2] Implement `to_slim()` method returning dict with id, name, molecular_formula only
- [ ] T076 [US2] Add model_config with json_schema_extra examples from data-model.md

### Client Methods for User Story 2

- [ ] T077 [US2] Implement `_get_compound_properties()` method calling `/compound/cid/{cid}/property/{props}/JSON` (R5)
- [ ] T078 [US2] Parse PropertyTable.Properties[0] from PubChem response (R5)
- [ ] T079 [US2] Map PubChem field `CID` → our field `id` with CURIE prefix (data-model.md)
- [ ] T080 [US2] Map PubChem field `Title` → our field `name` (data-model.md)
- [ ] T081 [US2] Map PubChem field `IUPACName` → our field `iupac_name` (data-model.md)
- [ ] T082 [US2] Map PubChem field `MolecularFormula` → our field `molecular_formula` (data-model.md)
- [ ] T083 [US2] Map PubChem field `MolecularWeight` → our field `molecular_weight` as float (data-model.md)
- [ ] T084 [US2] Map PubChem field `CanonicalSMILES` → our field `canonical_smiles` (data-model.md)
- [ ] T085 [US2] Map PubChem field `IsomericSMILES` → our field `isomeric_smiles` (data-model.md)
- [ ] T086 [US2] Map PubChem field `InChI` → our field `inchi` (data-model.md)
- [ ] T087 [US2] Map PubChem field `InChIKey` → our field `inchikey` (data-model.md)
- [ ] T088 [US2] Implement `_get_compound_synonyms()` method calling `/compound/cid/{cid}/synonyms/JSON` (R1)
- [ ] T089 [US2] Parse InformationList.Information[0].Synonym array (R1)
- [ ] T090 [US2] Limit synonyms to first 20 entries for token efficiency
- [ ] T091 [US2] Implement `get_compound()` public method in PubChemClient
- [ ] T092 [US2] Add slim parameter support: return to_slim() when slim=True

### Server Tool for User Story 2

- [ ] T093 [US2] Implement `@mcp.tool() get_compound` tool with parameters: pubchem_id, slim
- [ ] T094 [US2] Add comprehensive docstring with usage examples from contracts/get_compound.json
- [ ] T095 [US2] Validate pubchem_id CURIE format before API call (FR-006)
- [ ] T096 [US2] Return UNRESOLVED_ENTITY error for invalid CURIE format (FR-007)
- [ ] T097 [US2] Return ENTITY_NOT_FOUND error for valid CURIE with no results (FR-008)

### Unit Tests for User Story 2

- [ ] T098 [P] [US2] Write unit test `test_compound_model_valid` in `tests/unit/test_pubchem_models.py`
- [ ] T099 [P] [US2] Write unit test `test_compound_curie_validation` for CURIE format
- [ ] T100 [P] [US2] Write unit test `test_compound_to_slim` for slim mode output
- [ ] T101 [P] [US2] Write unit test `test_get_properties_aspirin` with mocked response
- [ ] T102 [P] [US2] Write unit test `test_get_synonyms_parsing` with mocked response
- [ ] T103 [P] [US2] Write unit test `test_field_mapping_pubchem_to_model` for all field mappings

### Integration Tests for User Story 2

- [ ] T104 [P] [US2] Write integration test `test_get_compound_aspirin` verifying all fields populated
- [ ] T105 [P] [US2] Write integration test `test_get_compound_structure_smiles` verifying SMILES fields
- [ ] T106 [P] [US2] Write integration test `test_get_compound_structure_inchi` verifying InChI/InChIKey
- [ ] T107 [P] [US2] Write integration test `test_get_compound_molecular_weight` verifying weight in Daltons
- [ ] T108 [P] [US2] Write integration test `test_get_compound_slim_mode` verifying minimal fields
- [ ] T109 [P] [US2] Write integration test `test_get_compound_invalid_curie` for UNRESOLVED_ENTITY
- [ ] T110 [P] [US2] Write integration test `test_get_compound_not_found` for ENTITY_NOT_FOUND

**Checkpoint**: User Stories 1 AND 2 complete - full Fuzzy-to-Fact workflow operational

---

## Phase 5: User Story 3 - Cross-Database Integration (Priority: P3)

**Goal**: An LLM agent needs to link PubChem compounds to other databases (ChEMBL, DrugBank, UniProt targets). The agent can use cross-references to triangulate information across sources.

**Independent Test**: Call `get_compound("PubChem:CID2244")` and verify cross-references to ChEMBL and DrugBank are populated.

**References**: FR-011, FR-012, R3, data-model.md Cross-Reference Key Registry

### Cross-Reference Extraction (R3)

- [ ] T111 [US3] Implement `_extract_cross_references()` method in PubChemClient (R3)
- [ ] T112 [US3] Add self-reference: cross_references["pubchem_compound"] = [str(cid)]
- [ ] T113 [US3] Implement ChEMBL extraction from synonyms with pattern `^CHEMBL(\d+)$` (R3)
- [ ] T114 [US3] Format ChEMBL as `CHEMBL:{number}` in cross_references["chembl"] list
- [ ] T115 [US3] Implement DrugBank extraction from synonyms with pattern `^DB\d{5}$` (R3)
- [ ] T116 [US3] Store DrugBank IDs as-is in cross_references["drugbank"] list
- [ ] T117 [US3] Implement `_get_compound_xrefs()` method calling `/compound/cid/{cid}/xrefs/RegistryID/JSON` (R3)
- [ ] T118 [US3] Extract UniProt references from xrefs SourceName containing "UniProt" (R3)
- [ ] T119 [US3] Format UniProt as `UniProtKB:{registry_id}` in cross_references["uniprot"]
- [ ] T120 [US3] Extract PDB references from xrefs (R3)
- [ ] T121 [US3] Apply omit-if-null pattern: only include keys with non-empty values (FR-012, Constitution III)
- [ ] T122 [US3] Integrate cross_references into get_compound response

### Unit Tests for User Story 3

- [ ] T123 [P] [US3] Write unit test `test_extract_chembl_from_synonyms` with mock data
- [ ] T124 [P] [US3] Write unit test `test_extract_drugbank_from_synonyms` with mock data
- [ ] T125 [P] [US3] Write unit test `test_extract_uniprot_from_xrefs` with mock data
- [ ] T126 [P] [US3] Write unit test `test_omit_if_null_pattern` verifying empty keys excluded
- [ ] T127 [P] [US3] Write unit test `test_cross_reference_format_chembl` for CHEMBL: prefix
- [ ] T128 [P] [US3] Write unit test `test_cross_reference_format_uniprot` for UniProtKB: prefix

### Integration Tests for User Story 3

- [ ] T129 [P] [US3] Write integration test `test_cross_references_aspirin_chembl` verifying CHEMBL:25
- [ ] T130 [P] [US3] Write integration test `test_cross_references_aspirin_drugbank` verifying DB00945
- [ ] T131 [P] [US3] Write integration test `test_cross_references_self_reference` verifying pubchem_compound key
- [ ] T132 [P] [US3] Write integration test `test_cross_references_omit_empty` verifying no null values

**Checkpoint**: User Story 3 complete - cross-database linking enables multi-hop reasoning

---

## Phase 6: User Story 4 - Error Recovery (Priority: P4)

**Goal**: An LLM agent encounters errors and needs actionable guidance to self-correct. All errors include recovery hints that guide the agent to the correct tool or action.

**Independent Test**: Trigger each error type and verify recovery hints are actionable.

**References**: FR-014 to FR-016, R6, contracts/*.json errors section

### Error Recovery Implementation

- [ ] T133 [US4] Verify UNRESOLVED_ENTITY error includes recovery_hint: "Use format 'PubChem:CID{number}'..." (FR-016)
- [ ] T134 [US4] Verify ENTITY_NOT_FOUND error includes recovery_hint: "CID not found. Verify from search_compounds results."
- [ ] T135 [US4] Verify AMBIGUOUS_QUERY error includes recovery_hint: "Minimum query length is 2 characters"
- [ ] T136 [US4] Verify RATE_LIMITED error includes recovery_hint with wait time
- [ ] T137 [US4] Verify UPSTREAM_ERROR error includes recovery_hint: "Retry in 60 seconds"
- [ ] T138 [US4] Verify all errors include invalid_input field with original input value
- [ ] T139 [US4] Verify all errors use Canonical ErrorEnvelope format (FR-014)
- [ ] T140 [US4] Document error→recovery→success workflow in code comments

### Unit Tests for User Story 4

- [ ] T141 [P] [US4] Write unit test `test_error_envelope_format` verifying success=false, error object
- [ ] T142 [P] [US4] Write unit test `test_recovery_hint_unresolved_entity` verifying actionable hint
- [ ] T143 [P] [US4] Write unit test `test_recovery_hint_entity_not_found` verifying actionable hint
- [ ] T144 [P] [US4] Write unit test `test_recovery_hint_ambiguous_query` verifying actionable hint
- [ ] T145 [P] [US4] Write unit test `test_recovery_hint_rate_limited` verifying actionable hint
- [ ] T146 [P] [US4] Write unit test `test_invalid_input_preserved` verifying original input in error

### Integration Tests for User Story 4

- [ ] T147 [P] [US4] Write integration test `test_error_recovery_invalid_curie` testing UNRESOLVED_ENTITY
- [ ] T148 [P] [US4] Write integration test `test_error_recovery_not_found` testing ENTITY_NOT_FOUND
- [ ] T149 [P] [US4] Write integration test `test_error_recovery_query_too_short` testing AMBIGUOUS_QUERY
- [ ] T150 [P] [US4] Write integration test `test_error_recovery_workflow` testing error→hint→success cycle

**Checkpoint**: User Story 4 complete - agents can self-correct from all error scenarios

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

### Documentation

- [X] T151 [P] Add comprehensive docstrings to PubChemClient class and all public methods
- [X] T152 [P] Add comprehensive docstrings to PubChemSearchCandidate and PubChemCompound models
- [X] T153 [P] Add comprehensive docstrings to search_compounds and get_compound tools
- [X] T154 [P] Update `src/lifesciences_mcp/__init__.py` to export PubChemClient
- [X] T155 [P] Update `src/lifesciences_mcp/models/__init__.py` to export PubChem models

### Code Quality

- [X] T156 [P] Run ruff check and fix any linting issues
- [X] T157 [P] Run ruff format on all new files
- [X] T158 [P] Run pyright type checking and fix any type errors
- [X] T159 [P] Verify all imports are correctly organized

### Performance Validation

- [ ] T160 Write performance test `test_search_performance_sc001` verifying <2s for 95% of queries (SC-001) **OPTIONAL**
- [ ] T161 Write performance test `test_rate_limit_prevents_429` verifying SC-002 **OPTIONAL**
- [ ] T162 Benchmark token usage: verify slim mode ~20 tokens vs full mode ~115-150 tokens **OPTIONAL**

### Integration Validation

- [X] T163 Run full unit test suite: `uv run pytest tests/unit/test_pubchem_*.py -v`
- [X] T164 Run full integration test suite: `uv run pytest tests/integration/test_pubchem_api.py -v -m integration`
- [X] T165 Verify all tests pass (target: 100% pass rate)

### Documentation Updates

- [X] T166 [P] Update CLAUDE.md with PubChem server entry in "Implemented Tools" section
- [X] T167 [P] Add PubChem usage examples to CLAUDE.md
- [ ] T168 Validate quickstart.md examples work with implemented server **OPTIONAL**
- [X] T169 Final code review and cleanup

---

## Dependencies & Execution Order

### Phase Dependencies

```
Phase 1: Setup ─────────────────────────────────────────┐
                                                        │
Phase 2: Foundational (BLOCKING) ◄──────────────────────┘
    │
    ├──► Phase 3: User Story 1 (P1) ─ MVP
    │         │
    │         ▼
    ├──► Phase 4: User Story 2 (P2) ─ Fuzzy-to-Fact Complete
    │         │
    │         ▼
    ├──► Phase 5: User Story 3 (P3) ─ Cross-Database Links
    │         │
    │         ▼
    └──► Phase 6: User Story 4 (P4) ─ Error Recovery
              │
              ▼
         Phase 7: Polish
```

### User Story Dependencies

| Story | Depends On | Can Parallelize After |
|-------|------------|----------------------|
| US1 (Fuzzy Search) | Phase 2 Foundational | Phase 2 complete |
| US2 (Strict Lookup) | Phase 2 Foundational | Phase 2 complete |
| US3 (Cross-Refs) | US2 (uses get_compound) | US2 complete |
| US4 (Error Recovery) | Phase 2 Foundational | Phase 2 complete |

### Within Each User Story

1. Models FIRST (data structures)
2. Client methods SECOND (API interaction)
3. Server tools THIRD (MCP exposure)
4. Unit tests in PARALLEL with implementation
5. Integration tests AFTER implementation complete

---

## Parallel Execution Examples

### Phase 1: Setup (All parallel after T001)

```bash
# After T001 completes, launch T002-T007 in parallel:
Task: T002 "Create clients/pubchem.py stub"
Task: T003 "Create models/pubchem_compound.py stub"
Task: T004 "Create servers/pubchem.py stub"
Task: T005 "Create tests/unit/test_pubchem_models.py stub"
Task: T006 "Create tests/unit/test_pubchem_client.py stub"
Task: T007 "Create tests/integration/test_pubchem_api.py stub"
```

### Phase 2: Unit Tests (Parallel)

```bash
# All foundation unit tests can run in parallel:
Task: T027 "test_rate_limit_per_second"
Task: T028 "test_rate_limit_per_minute"
Task: T029 "test_curie_validation_valid"
Task: T030 "test_curie_validation_invalid"
Task: T031 "test_exponential_backoff"
Task: T032 "test_error_mapping_400"
Task: T033 "test_error_mapping_404"
Task: T034 "test_error_mapping_503"
```

### Phase 3: US1 Models (Parallel)

```bash
# Model tasks can run in parallel:
Task: T035 "Create PubChemSearchCandidate model"
Task: T036 "Add @field_validator for id"
Task: T037 "Add Field constraints for score"
Task: T038 "Add model_config examples"
```

### Phase 4: US2 Field Mappings (Parallel)

```bash
# Field mapping tasks can run in parallel:
Task: T079 "Map CID → id"
Task: T080 "Map Title → name"
Task: T081 "Map IUPACName → iupac_name"
Task: T082 "Map MolecularFormula → molecular_formula"
Task: T083 "Map MolecularWeight → molecular_weight"
Task: T084 "Map CanonicalSMILES → canonical_smiles"
Task: T085 "Map IsomericSMILES → isomeric_smiles"
Task: T086 "Map InChI → inchi"
Task: T087 "Map InChIKey → inchikey"
```

### Phase 5: US3 Cross-Ref Tests (Parallel)

```bash
# Cross-reference unit tests can run in parallel:
Task: T123 "test_extract_chembl_from_synonyms"
Task: T124 "test_extract_drugbank_from_synonyms"
Task: T125 "test_extract_uniprot_from_xrefs"
Task: T126 "test_omit_if_null_pattern"
```

---

## Task Summary

| Phase | Tasks | Parallel | Story Coverage |
|-------|-------|----------|----------------|
| Phase 1: Setup | T001-T007 (7) | 6 | Shared infrastructure |
| Phase 2: Foundational | T008-T034 (27) | 8 | Rate limiting, CURIE, errors |
| Phase 3: User Story 1 | T035-T066 (32) | 18 | Fuzzy Compound Search |
| Phase 4: User Story 2 | T067-T110 (44) | 24 | Strict Compound Lookup |
| Phase 5: User Story 3 | T111-T132 (22) | 10 | Cross-Database Integration |
| Phase 6: User Story 4 | T133-T150 (18) | 12 | Error Recovery |
| Phase 7: Polish | T151-T169 (19) | 10 | Cross-cutting concerns |

**Total**: 169 tasks, 88 parallelizable (52% parallel efficiency)

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (7 tasks)
2. Complete Phase 2: Foundational (27 tasks) - **CRITICAL**
3. Complete Phase 3: User Story 1 (32 tasks)
4. **STOP and VALIDATE**: Test Fuzzy Compound Search independently
5. Deploy/demo if ready

### Incremental Delivery

1. Setup + Foundational → Foundation ready (34 tasks)
2. Add User Story 1 → **MVP!** (66 tasks total)
3. Add User Story 2 → Fuzzy-to-Fact complete (110 tasks total)
4. Add User Story 3 → Cross-database links (132 tasks total)
5. Add User Story 4 → Full error recovery (150 tasks total)
6. Polish → Production ready (169 tasks total)

### Parallel Team Strategy

With multiple developers after Phase 2:
- Developer A: User Story 1 (fuzzy search)
- Developer B: User Story 2 (strict lookup)
- Developer C: User Story 4 (error recovery)
- Developer D: User Story 3 (after US2 complete)

---

## Test Coverage Summary

| Category | Count | Coverage |
|----------|-------|----------|
| Unit Tests (Foundation) | 8 | Rate limiting, CURIE, errors |
| Unit Tests (US1) | 6 | Models, search, pagination, scoring |
| Unit Tests (US2) | 6 | Models, properties, synonyms, mapping |
| Unit Tests (US3) | 6 | Cross-ref extraction, formatting |
| Unit Tests (US4) | 6 | Error envelopes, recovery hints |
| Integration Tests (US1) | 6 | Search scenarios, pagination |
| Integration Tests (US2) | 7 | Compound retrieval, structure data |
| Integration Tests (US3) | 4 | Cross-reference validation |
| Integration Tests (US4) | 4 | Error recovery workflows |
| Performance Tests | 3 | SC-001, SC-002, token budgets |

**Total Test Tasks**: 56 (33% of all tasks)

---

## Notes

- PubChem uses JSON responses (simpler than Entrez XML parsing)
- Dual rate limits (per-second AND per-minute) require separate tracking mechanisms
- Structure data (SMILES, InChI, InChIKey) is core differentiator from gene/protein servers
- Cross-references primarily from synonyms (ChEMBL, DrugBank) not xrefs endpoint
- Token budgeting: slim mode reduces ~115-150 tokens to ~20 tokens (80% reduction)
- Commit after each task or logical group
- Run `uv run ruff check --fix . && uv run ruff format .` before each commit
