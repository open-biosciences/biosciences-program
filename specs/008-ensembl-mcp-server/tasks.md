# Tasks: Ensembl MCP Server

**Input**: Design documents from `/specs/008-ensembl-mcp-server/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: Tests ARE requested per spec.md ("All integration tests pass against live Ensembl REST API")

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3, US4)
- Include exact file paths in descriptions

## Path Conventions

Based on plan.md project structure:
- Source: `src/lifesciences_mcp/`
- Tests: `tests/`
- Specs: `specs/008-ensembl-mcp-server/`

## Artifact References

| Artifact | Location | Key Elements |
|----------|----------|--------------|
| Data Model | `specs/008-ensembl-mcp-server/data-model.md` | EnsemblGene, EnsemblTranscript, GeneSearchCandidate, CrossReferences |
| search_genes Contract | `specs/008-ensembl-mcp-server/contracts/search_genes.yaml` | Fuzzy search tool specification |
| get_gene Contract | `specs/008-ensembl-mcp-server/contracts/get_gene.yaml` | Strict gene lookup specification |
| get_transcript Contract | `specs/008-ensembl-mcp-server/contracts/get_transcript.yaml` | Transcript lookup specification |
| Research | `specs/008-ensembl-mcp-server/research.md` | R1-R7 API research findings |
| Quickstart | `specs/008-ensembl-mcp-server/quickstart.md` | Usage workflows and examples |

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization, file creation, and shared constants

- [X] T001 Verify project structure matches plan.md (`src/lifesciences_mcp/{models,clients,servers}/`, `tests/{unit,integration}/`)
- [X] T002 [P] Create Ensembl model file `src/lifesciences_mcp/models/ensembl.py` with module docstring referencing data-model.md
- [X] T003 [P] Create Ensembl client file `src/lifesciences_mcp/clients/ensembl.py` with module docstring
- [X] T004 [P] Create Ensembl server file `src/lifesciences_mcp/servers/ensembl.py` with FastMCP app initialization
- [X] T005 [P] Add Ensembl ID validation patterns as constants: `ENSEMBL_GENE_PATTERN = re.compile(r"^ENSG\d{11}$")`, `ENSEMBL_TRANSCRIPT_PATTERN = re.compile(r"^ENST\d{11}$")` per research.md R3 in `src/lifesciences_mcp/models/ensembl.py`
- [X] T006 [P] Verify PaginationEnvelope and ErrorEnvelope exist in `src/lifesciences_mcp/models/envelopes.py` (reuse from existing)
- [X] T007 [P] Add species aliases mapping per research.md R7 in `src/lifesciences_mcp/clients/ensembl.py` (human->homo_sapiens, mouse->mus_musculus, etc.)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**CRITICAL**: No user story work can begin until this phase is complete

### Models (from data-model.md)

- [X] T008 Implement GeneSearchCandidate model per data-model.md in `src/lifesciences_mcp/models/ensembl.py`:
  - Fields: id (str, validated ENSG pattern), symbol (str), name (str), biotype (str), species (str), score (float 0.0-1.0)
  - Add field_validator for id and score range validation
- [X] T009 [P] Implement EnsemblGene model per data-model.md in `src/lifesciences_mcp/models/ensembl.py`:
  - Fields: id, symbol, name, biotype, species, assembly_name, chromosome, start (int), end (int), strand (int +1/-1), transcripts (list[str] | None), cross_references
  - Add field_validators for id, strand (+1/-1 only), start < end
- [X] T010 [P] Implement EnsemblTranscript model per data-model.md in `src/lifesciences_mcp/models/ensembl.py`:
  - Fields: id, display_name, biotype, parent_gene, is_canonical (bool), species, assembly_name, chromosome, start, end, strand, cross_references
  - Add field_validators for id (ENST pattern), parent_gene (ENSG pattern)
- [X] T011 [P] Extend CrossReferences model in `src/lifesciences_mcp/models/gene.py` (or create new) to include all 12 Ensembl-relevant keys per data-model.md:
  - hgnc, ensembl_gene, ensembl_transcript (list), uniprot (list), entrez, refseq (list), omim, pdb (list), kegg, chembl, string, biogrid
  - Apply omit-if-null pattern (exclude_none=True in model_config)
- [X] T012 Update `src/lifesciences_mcp/models/__init__.py` to export GeneSearchCandidate, EnsemblGene, EnsemblTranscript

### Client Infrastructure (from research.md)

- [X] T013 Implement EnsemblClient class skeleton in `src/lifesciences_mcp/clients/ensembl.py`:
  - Extend LifeSciencesClient base class (if exists) or create standalone
  - BASE_URL = "https://rest.ensembl.org"
  - Default timeout, default headers (content-type: application/json)
- [X] T014 [P] Implement rate limiting infrastructure in EnsemblClient per research.md R2:
  - RATE_LIMIT_DELAY = 1.0 / 15 (66.67ms between requests for 15 req/s)
  - asyncio.Lock for request serialization
  - Last request timestamp tracking
- [X] T015 [P] Implement exponential backoff for 429/503 errors in EnsemblClient per research.md R2:
  - MAX_RETRIES = 3
  - Read Retry-After header if present, otherwise 2^attempt seconds
  - Thundering herd prevention: re-check timing after lock acquisition
- [X] T016 [P] Implement _rate_limited_get helper method in EnsemblClient:
  - Acquire lock, calculate delay, make request, handle retries
  - Return parsed JSON or raise appropriate exception
- [X] T017 Implement error mapping from Ensembl API to ErrorEnvelope per research.md R5:
  - HTTP 400 "ID not found" + format check -> UNRESOLVED_ENTITY or ENTITY_NOT_FOUND
  - HTTP 429 -> RATE_LIMITED with recovery_hint
  - HTTP 500/502/503 -> UPSTREAM_ERROR
- [X] T018 Update `src/lifesciences_mcp/clients/__init__.py` to export EnsemblClient

**Checkpoint**: Foundation ready - user story implementation can now begin

---

## Phase 3: User Story 1 - Fuzzy Gene Search (Priority: P1) MVP

**Goal**: LLM agents can search for genes by symbol/name and get ranked candidates with Ensembl Gene IDs

**Independent Test**: Query "BRCA1" and verify ranked candidates include ENSG00000012048 with correct species

**Contract Reference**: `specs/008-ensembl-mcp-server/contracts/search_genes.yaml`

### Tests for User Story 1

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [X] T019 [P] [US1] Integration test for BRCA1 search in `tests/integration/test_ensembl_api.py::test_search_genes_brca1`:
  - Call search_genes("BRCA1", species="human")
  - Verify response contains PaginationEnvelope with items
  - Verify top result has id matching ENSG pattern, symbol="BRCA1"
- [X] T020 [P] [US1] Integration test for partial name search in `tests/integration/test_ensembl_api.py::test_search_genes_tumor_protein`:
  - Call search_genes("tumor protein")
  - Verify multiple candidates returned with relevance scores
- [X] T021 [P] [US1] Integration test for species filter in `tests/integration/test_ensembl_api.py::test_search_genes_species_filter`:
  - Call search_genes("TP53", species="human")
  - Verify all results have species="homo_sapiens"
- [X] T022 [P] [US1] Integration test for short query error in `tests/integration/test_ensembl_api.py::test_search_genes_short_query`:
  - Call search_genes("p")
  - Verify AMBIGUOUS_QUERY error returned with recovery_hint

### Implementation for User Story 1

- [X] T023 [US1] Implement _normalize_species helper in EnsemblClient per research.md R7:
  - Map aliases to Ensembl format (human->homo_sapiens, mouse->mus_musculus)
  - Handle lowercase/whitespace normalization
- [X] T024 [US1] Implement _search_genes_raw method in EnsemblClient using /xrefs/symbol/{species}/{symbol} endpoint per research.md R1:
  - Query Ensembl xrefs/symbol endpoint
  - Parse response to extract gene IDs and metadata
  - Handle empty results gracefully
- [X] T025 [US1] Implement relevance scoring logic in _search_genes_raw:
  - Exact symbol match = 1.0
  - Partial match with position-based decay
  - Normalize scores to 0.0-1.0 range
- [X] T026 [US1] Implement cursor-based pagination per research.md R6 in EnsemblClient:
  - Ensembl returns all results (no server-side paging)
  - Client-side: base64-encoded offset cursor
  - Apply page_size slicing to results
- [X] T027 [US1] Create search_genes MCP tool in `src/lifesciences_mcp/servers/ensembl.py` per contracts/search_genes.yaml:
  - @mcp.tool decorator with description from contract
  - Parameters: query (str, required), species (str, default="homo_sapiens"), page_size (int, 1-100, default=50), cursor (str, optional), slim (bool, default=False)
  - Return PaginationEnvelope[GeneSearchCandidate]
- [X] T028 [US1] Add query validation to search_genes tool:
  - Minimum 2 characters per contract error spec
  - Return AMBIGUOUS_QUERY ErrorEnvelope for invalid queries

**Checkpoint**: User Story 1 complete - agents can discover genes via fuzzy search

---

## Phase 4: User Story 2 - Strict Gene Lookup (Priority: P2)

**Goal**: LLM agents can retrieve complete gene records with cross-references using resolved Ensembl Gene IDs

**Independent Test**: Call get_gene("ENSG00000141510") (TP53) and verify all required fields populated

**Contract Reference**: `specs/008-ensembl-mcp-server/contracts/get_gene.yaml`

### Tests for User Story 2

- [X] T029 [P] [US2] Integration test for TP53 gene lookup in `tests/integration/test_ensembl_api.py::test_get_gene_tp53`:
  - Call get_gene("ENSG00000141510")
  - Verify EnsemblGene returned with all required fields (symbol="TP53", biotype="protein_coding", etc.)
  - Verify cross_references contains hgnc, uniprot, entrez keys
- [X] T030 [P] [US2] Integration test for invalid ID format in `tests/integration/test_ensembl_api.py::test_get_gene_invalid_format`:
  - Call get_gene("BRCA1") (raw symbol, not Ensembl ID)
  - Verify UNRESOLVED_ENTITY error with recovery_hint pointing to search_genes
- [X] T031 [P] [US2] Integration test for non-existent ID in `tests/integration/test_ensembl_api.py::test_get_gene_not_found`:
  - Call get_gene("ENSG99999999999")
  - Verify ENTITY_NOT_FOUND error
- [X] T032 [P] [US2] Integration test for cross-reference extraction in `tests/integration/test_ensembl_api.py::test_get_gene_cross_references`:
  - Call get_gene for known gene with many xrefs
  - Verify HGNC, UniProt, Entrez, RefSeq mappings correct
  - Verify omit-if-null pattern (no null values in cross_references)
- [X] T033 [P] [US2] Integration test for slim mode in `tests/integration/test_ensembl_api.py::test_get_gene_slim`:
  - Call get_gene("ENSG00000012048", slim=True)
  - Verify cross_references omitted in response

### Implementation for User Story 2

- [X] T034 [US2] Implement _get_gene_raw method in EnsemblClient using /lookup/id/{id} endpoint per research.md R1:
  - Query Ensembl lookup/id endpoint with expand=1 for transcripts
  - Parse response to EnsemblGene model fields
  - Extract transcript IDs from Transcript array
- [X] T035 [US2] Implement _get_xrefs method in EnsemblClient using /xrefs/id/{id} endpoint per research.md R4:
  - Query Ensembl xrefs/id endpoint
  - Return raw xref array for processing
- [X] T036 [US2] Implement _map_cross_references helper per research.md R4 and data-model.md:
  - Map Ensembl dbname to 22-key registry (HGNC->hgnc, Uniprot/SWISSPROT->uniprot, EntrezGene->entrez, etc.)
  - Handle list fields (uniprot, refseq, pdb) by collecting multiple values
  - Format HGNC as "HGNC:{primary_id}"
  - Apply omit-if-null: only include keys with values
- [X] T037 [US2] Add Ensembl ID format validation to get_gene:
  - Validate input matches ENSEMBL_GENE_PATTERN
  - Return UNRESOLVED_ENTITY ErrorEnvelope with recovery_hint if invalid
- [X] T038 [US2] Add ENTITY_NOT_FOUND handling:
  - If Ensembl returns 400 "not found" for valid format ID, return ENTITY_NOT_FOUND
  - Include recovery_hint: "Verify the ID or use search_genes"
- [X] T039 [US2] Create get_gene MCP tool in `src/lifesciences_mcp/servers/ensembl.py` per contracts/get_gene.yaml:
  - @mcp.tool decorator with description from contract
  - Parameters: ensembl_id (str, required, ENSG pattern), slim (bool, default=False)
  - Return EnsemblGene or ErrorEnvelope
- [X] T040 [US2] Implement slim mode in get_gene:
  - When slim=True, omit cross_references from response
  - Reduces token usage from ~150-350 to ~50 tokens

**Checkpoint**: User Stories 1 AND 2 complete - full Fuzzy-to-Fact workflow for genes

---

## Phase 5: User Story 3 - Transcript Lookup (Priority: P3)

**Goal**: LLM agents can retrieve transcript details by Ensembl Transcript ID

**Independent Test**: Call get_transcript("ENST00000269305") and verify transcript details returned

**Contract Reference**: `specs/008-ensembl-mcp-server/contracts/get_transcript.yaml`

### Tests for User Story 3

- [X] T041 [P] [US3] Integration test for TP53 transcript lookup in `tests/integration/test_ensembl_api.py::test_get_transcript_tp53`:
  - Call get_transcript("ENST00000269305")
  - Verify EnsemblTranscript returned with display_name, parent_gene, is_canonical
- [X] T042 [P] [US3] Integration test for gene ID passed to transcript in `tests/integration/test_ensembl_api.py::test_get_transcript_wrong_id_type`:
  - Call get_transcript("ENSG00000141510") (gene ID, not transcript)
  - Verify UNRESOLVED_ENTITY error with hint to use get_gene
- [X] T043 [P] [US3] Integration test for transcript not found in `tests/integration/test_ensembl_api.py::test_get_transcript_not_found`:
  - Call get_transcript("ENST99999999999")
  - Verify ENTITY_NOT_FOUND error

### Implementation for User Story 3

- [X] T044 [US3] Implement _get_transcript_raw method in EnsemblClient using /lookup/id/{id} endpoint:
  - Query Ensembl lookup/id endpoint with ENST ID
  - Parse response to EnsemblTranscript model fields
  - Extract parent gene reference from Parent field
- [X] T045 [US3] Add Ensembl Transcript ID validation:
  - Validate input matches ENSEMBL_TRANSCRIPT_PATTERN
  - Distinguish from ENSG pattern for better error messages
  - Return UNRESOLVED_ENTITY with hint "Use ENST* ID for transcripts"
- [X] T046 [US3] Create get_transcript MCP tool in `src/lifesciences_mcp/servers/ensembl.py` per contracts/get_transcript.yaml:
  - @mcp.tool decorator with description from contract
  - Parameters: transcript_id (str, required, ENST pattern), slim (bool, default=False)
  - Return EnsemblTranscript or ErrorEnvelope
- [X] T047 [US3] Implement slim mode in get_transcript:
  - When slim=True, omit cross_references from response

**Checkpoint**: User Stories 1, 2, AND 3 complete - gene and transcript queries supported

---

## Phase 6: User Story 4 - Error Recovery (Priority: P4)

**Goal**: LLM agents can self-correct from errors using actionable recovery hints

**Independent Test**: Trigger each error type and verify recovery hints are actionable

### Tests for User Story 4

- [X] T048 [P] [US4] Integration test for UNRESOLVED_ENTITY recovery in `tests/integration/test_ensembl_api.py::test_error_recovery_unresolved`:
  - Call get_gene("BRCA1") -> get UNRESOLVED_ENTITY
  - Follow recovery_hint: call search_genes("BRCA1")
  - Get valid Ensembl ID
  - Call get_gene with valid ID -> success
- [X] T049 [P] [US4] Integration test for error envelope format compliance in `tests/integration/test_ensembl_api.py::test_error_envelope_format`:
  - Trigger various errors
  - Verify all contain: code, message, recovery_hint, invalid_input (when applicable)
- [X] T050 [P] [US4] Integration test for transcript/gene ID confusion in `tests/integration/test_ensembl_api.py::test_error_recovery_id_type`:
  - Call get_transcript with ENSG ID
  - Verify hint explains the difference
  - Call get_gene with same ID -> success

### Implementation for User Story 4

- [X] T051 [US4] Create _create_error_envelope helper in EnsemblClient:
  - Standard factory for ErrorEnvelope creation
  - Ensure consistent format across all error types
- [X] T052 [US4] Add recovery_hint for UNRESOLVED_ENTITY errors per contracts:
  - For get_gene: "Use search_genes to find valid Ensembl Gene IDs. Format: ENSG + 11 digits"
  - For get_transcript: "Use get_gene to retrieve transcript IDs from a gene record. Format: ENST + 11 digits"
- [X] T053 [US4] Add recovery_hint for RATE_LIMITED errors:
  - "Wait for the time specified in Retry-After header before retrying"
  - Include actual wait time if known
- [X] T054 [US4] Add recovery_hint for UPSTREAM_ERROR:
  - "Ensembl service may be temporarily unavailable. Retry after a few seconds."
- [X] T055 [US4] Add recovery_hint for AMBIGUOUS_QUERY:
  - "Provide at least 2 characters or refine your search term"
- [X] T056 [US4] Verify all error envelopes include invalid_input field:
  - Echo back the problematic input for agent context

**Checkpoint**: All 4 user stories complete - robust error handling with recovery guidance

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Quality improvements, documentation, and validation

### Unit Tests

- [X] T057 [P] Create unit tests for GeneSearchCandidate model in `tests/unit/test_ensembl_models.py`:
  - Valid ID pattern acceptance
  - Invalid ID rejection
  - Score range validation (0.0-1.0)
- [X] T058 [P] Create unit tests for EnsemblGene model in `tests/unit/test_ensembl_models.py`:
  - Field validation
  - Strand validation (+1/-1 only)
  - Start < end validation
- [X] T059 [P] Create unit tests for EnsemblTranscript model in `tests/unit/test_ensembl_models.py`:
  - ID pattern validation
  - Parent gene pattern validation
- [X] T060 [P] Create unit tests for CrossReferences in `tests/unit/test_ensembl_models.py`:
  - Omit-if-null pattern verification
  - List field handling (uniprot, refseq, pdb)
- [X] T061 [P] Create unit tests for EnsemblClient with mocked responses in `tests/unit/test_ensembl_client.py`:
  - Rate limiting logic
  - Exponential backoff
  - Error mapping

### Documentation

- [X] T062 [P] Add module-level docstrings to `src/lifesciences_mcp/models/ensembl.py`
- [X] T063 [P] Add module-level docstrings to `src/lifesciences_mcp/clients/ensembl.py`
- [X] T064 [P] Add module-level docstrings to `src/lifesciences_mcp/servers/ensembl.py`
- [X] T065 [P] Update `tests/conftest.py` with EnsemblClient fixtures

### Validation

- [X] T066 Run linting: `uv run ruff check --fix . && uv run ruff format .`
- [X] T067 Run type checking: `uv run pyright src/lifesciences_mcp/{models,clients,servers}/ensembl.py`
- [X] T068 Verify all integration tests pass: `uv run pytest tests/integration/test_ensembl_api.py -v -m integration`
- [ ] T069 [P] Performance test validating SC-001 (<2s for 95% of queries) in `tests/integration/test_ensembl_performance.py`

### Final Validation

- [X] T070 Verify quickstart.md workflows execute successfully (Workflows 1-4)
- [X] T071 Validate success criteria from spec.md:
  - SC-001: 95% of search queries < 2 seconds
  - SC-002: Rate limiting prevents 429 errors
  - SC-003: Cross-references populated for genes with xrefs
  - SC-004: Fuzzy-to-Fact workflow works without human intervention
  - SC-005: Error recovery hints enable 90% self-correction
  - SC-006: All integration tests pass
- [X] T072 Update CLAUDE.md with Ensembl server documentation

**Checkpoint**: Ensembl MCP Server implementation complete

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3-6)**: All depend on Foundational phase completion
  - User stories can proceed in parallel (if staffed) after Phase 2
  - Or sequentially in priority order (P1 -> P2 -> P3 -> P4)
- **Polish (Phase 7)**: Depends on all user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - MVP milestone
- **User Story 2 (P2)**: Can start after Phase 2 - Completes Fuzzy-to-Fact workflow
- **User Story 3 (P3)**: Can start after Phase 2 - Adds transcript capability
- **User Story 4 (P4)**: Can start after Phase 2 - Cross-cutting error handling (touches all tools)

### Within Each User Story

- Tests MUST be written and FAIL before implementation
- Models before client methods
- Client methods before server tools
- Core implementation before integration
- Story complete before moving to next priority

### Parallel Opportunities

**Phase 1 (Setup)**: T002-T007 can run in parallel after T001

**Phase 2 (Foundational)**:
- T009-T011 (models) can run in parallel after T008
- T014-T016 (client infra) can run in parallel after T013

**User Story 1**:
- Tests T019-T022 can run in parallel (write first)
- After tests: T023-T028 mostly sequential

**User Story 2**:
- Tests T029-T033 can run in parallel
- After tests: T034-T040 mostly sequential

**User Story 3**:
- Tests T041-T043 can run in parallel
- After tests: T044-T047 sequential

**User Story 4**:
- Tests T048-T050 can run in parallel
- Implementation T051-T056 can mostly run in parallel

**Phase 7 (Polish)**:
- All unit test tasks (T057-T061) can run in parallel
- All documentation tasks (T062-T065) can run in parallel
- Validation tasks run sequentially for clear results

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001-T007)
2. Complete Phase 2: Foundational (T008-T018) - CRITICAL, blocks all stories
3. Complete Phase 3: User Story 1 (T019-T028)
4. **STOP and VALIDATE**: Run tests for User Story 1 independently
5. Deploy/demo if ready - agents can now discover Ensembl genes

**MVP Checkpoint**: Fuzzy search tool functional. Agents can discover gene IDs but cannot yet get full records.

### Full Fuzzy-to-Fact (Add User Story 2)

1. Complete MVP (above)
2. Add User Story 2 (T029-T040) - Strict gene lookup
3. **VALIDATE**: Complete Fuzzy-to-Fact workflow works
4. Agents can now: search -> select -> get complete gene data with xrefs

### Incremental Delivery

1. Setup + Foundational (T001-T018) -> Foundation ready
2. Add User Story 1 (T019-T028) -> Test -> Deploy (**MVP: Fuzzy gene search**)
3. Add User Story 2 (T029-T040) -> Test -> Deploy (**Core: Fuzzy-to-Fact workflow**)
4. Add User Story 3 (T041-T047) -> Test -> Deploy (Transcript lookup)
5. Add User Story 4 (T048-T056) -> Test -> Deploy (Error recovery)
6. Add Polish (T057-T072) -> Final validation -> Production-ready

Each story adds value without breaking previous stories.

---

## Task Summary

| Phase | Tasks | Complete | Parallel | Story |
|-------|-------|----------|----------|-------|
| Phase 1: Setup | 7 | 7 (100%) | 6 | N/A |
| Phase 2: Foundational | 11 | 11 (100%) | 6 | N/A |
| Phase 3: User Story 1 (P1) | 10 | 10 (100%) | 4 | US1 |
| Phase 4: User Story 2 (P2) | 12 | 12 (100%) | 5 | US2 |
| Phase 5: User Story 3 (P3) | 7 | 7 (100%) | 3 | US3 |
| Phase 6: User Story 4 (P4) | 9 | 9 (100%) | 3 | US4 |
| Phase 7: Polish | 16 | 15 (94%) | 10 | N/A |
| **TOTAL** | **72** | **71 (99%)** | **37** | **4 stories** |

**Parallel efficiency**: 51% of tasks can run in parallel (37/72)

**Remaining Tasks** (1):
- T069: Performance test (optional - can be added later)

**Independent test criteria** from spec.md:
- US1: Search "BRCA1" -> Get ranked candidates with ENSG IDs
- US2: Call get_gene("ENSG00000141510") -> Get full gene with xrefs
- US3: Call get_transcript("ENST00000269305") -> Get transcript details
- US4: Trigger errors -> Get actionable recovery hints

---

## Notes

- **[P] tasks** = different files, no dependencies, can run in parallel
- **[Story] label** maps task to specific user story for traceability
- Each user story independently completable and testable
- Tests written FIRST and should FAIL before implementation (TDD)
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Rate limit: 15 req/s (Ensembl policy) per research.md R2
- Cross-reference keys: 12 Ensembl-relevant from 22-key registry per data-model.md
- Canonical envelopes: PaginationEnvelope, ErrorEnvelope per ADR-001 Section 8
