# Tasks: NCBI Entrez MCP Server

**Input**: Design documents from `/specs/009-entrez-mcp-server/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: Tests ARE requested per spec.md ("All integration tests pass against live NCBI E-utilities API")

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3, US4)
- Include exact file paths in descriptions

## Path Conventions

Based on plan.md project structure:
- Source: `src/lifesciences_mcp/`
- Tests: `tests/`
- Specs: `specs/009-entrez-mcp-server/`

## Artifact References

| Artifact | Location | Key Elements |
|----------|----------|--------------|
| Data Model | `specs/009-entrez-mcp-server/data-model.md` | EntrezGene, GeneSearchCandidate, CrossReferences |
| search_genes Contract | `specs/009-entrez-mcp-server/contracts/search_genes.yaml` | Fuzzy search tool specification |
| get_gene Contract | `specs/009-entrez-mcp-server/contracts/get_gene.yaml` | Strict gene lookup specification |
| get_pubmed_links Contract | `specs/009-entrez-mcp-server/contracts/get_pubmed_links.yaml` | Literature link tool specification |
| Research | `specs/009-entrez-mcp-server/research.md` | R1-R7 API research findings |
| Quickstart | `specs/009-entrez-mcp-server/quickstart.md` | Usage workflows and examples |

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization, file creation, and shared constants

- [x] T001 Verify project structure matches plan.md (`src/lifesciences_mcp/{models,clients,servers}/`, `tests/{unit,integration}/`)
- [x] T002 [P] Create Entrez model file `src/lifesciences_mcp/models/entrez.py` with module docstring referencing data-model.md
- [x] T003 [P] Create Entrez client file `src/lifesciences_mcp/clients/entrez.py` with module docstring
- [x] T004 [P] Create Entrez server file `src/lifesciences_mcp/servers/entrez.py` with FastMCP app initialization
- [x] T005 [P] Add NCBIGene CURIE validation pattern as constant: `NCBI_GENE_CURIE_PATTERN = re.compile(r"^NCBIGene:\d+$")` per research.md R6 in `src/lifesciences_mcp/models/entrez.py`
- [x] T006 [P] Verify PaginationEnvelope and ErrorEnvelope exist in `src/lifesciences_mcp/models/envelopes.py` (reuse from existing)
- [x] T007 [P] Add defusedxml to dependencies in `pyproject.toml` per research.md R3

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**CRITICAL**: No user story work can begin until this phase is complete

### Models (from data-model.md)

- [x] T008 Implement GeneSearchCandidate model per data-model.md in `src/lifesciences_mcp/models/entrez.py`:
  - Fields: id (str, NCBIGene CURIE pattern), symbol (str), name (str), description (str | None), organism (str), score (float 0.0-1.0)
  - Add field_validator for id pattern and score range validation
- [x] T009 [P] Implement EntrezGene model per data-model.md in `src/lifesciences_mcp/models/entrez.py`:
  - Fields: id, symbol, name, description, summary, map_location, chromosome, aliases (list[str] | None), organism, taxon_id (int | None), cross_references
  - Add field_validator for id (NCBIGene pattern)
- [x] T010 [P] Extend CrossReferences model in `src/lifesciences_mcp/models/gene.py` or `src/lifesciences_mcp/models/entrez.py` to include Entrez-relevant keys per data-model.md:
  - hgnc, ensembl_gene, ensembl_transcript, uniprot (list), entrez, refseq (list), omim, kegg, string, biogrid
  - Apply omit-if-null pattern (model_validator to filter None values)
- [x] T011 Update `src/lifesciences_mcp/models/__init__.py` to export GeneSearchCandidate, EntrezGene, NCBI_GENE_CURIE_PATTERN

### Client Infrastructure (from research.md)

- [x] T012 Implement EntrezClient class skeleton in `src/lifesciences_mcp/clients/entrez.py`:
  - Extend LifeSciencesClient base class (if exists) or create standalone
  - ENTREZ_BASE_URL = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils"
  - Read NCBI_API_KEY from environment per research.md R4
  - Default timeout, default headers
- [x] T013 [P] Implement adaptive rate limiting infrastructure in EntrezClient per research.md R4:
  - RATE_LIMIT_DELAY = 0.1 if api_key else 0.333 (10 req/s vs 3 req/s)
  - asyncio.Lock for request serialization
  - Last request timestamp tracking
- [x] T014 [P] Implement exponential backoff for 429/503 errors in EntrezClient per research.md R4:
  - MAX_RETRIES = 3
  - Read Retry-After header if present, otherwise 2^attempt seconds
  - Thundering herd prevention: re-check timing after lock acquisition
- [x] T015 [P] Implement _rate_limited_get helper method in EntrezClient:
  - Acquire lock, calculate delay, make request, handle retries
  - Add api_key to params if available
  - Return parsed JSON or raise appropriate exception
- [x] T016 [P] Implement XML parsing helper using defusedxml per research.md R3 in `src/lifesciences_mcp/clients/entrez.py`:
  - parse_gene_xml(xml_content: str) -> dict | None
  - Extract gene_id, symbol, name, map_location, aliases, summary, organism, xrefs
  - Use safe defusedxml.ElementTree for XXE prevention
- [x] T017 Implement error mapping from NCBI API to ErrorEnvelope per research.md R7:
  - Empty esearch results -> return empty PaginationEnvelope (not error)
  - esummary "error" field -> ENTITY_NOT_FOUND
  - HTTP 429 -> RATE_LIMITED with recovery_hint suggesting NCBI_API_KEY
  - HTTP 500/502/503 -> UPSTREAM_ERROR
- [x] T018 Update `src/lifesciences_mcp/clients/__init__.py` to export EntrezClient

**Checkpoint**: Foundation ready - user story implementation can now begin

---

## Phase 3: User Story 1 - Fuzzy Gene Search (Priority: P1) MVP

**Goal**: LLM agents can search for genes by symbol/name/description and get ranked candidates with NCBIGene CURIEs

**Independent Test**: Query "BRCA1" and verify ranked candidates include NCBIGene:672 with correct organism

**Contract Reference**: `specs/009-entrez-mcp-server/contracts/search_genes.yaml`

### Tests for User Story 1

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [x] T019 [P] [US1] Integration test for BRCA1 search in `tests/integration/test_entrez_api.py::test_search_genes_brca1`:
  - Call search_genes("BRCA1", organism="human")
  - Verify response contains PaginationEnvelope with items
  - Verify top result has id="NCBIGene:672", symbol="BRCA1"
- [x] T020 [P] [US1] Integration test for broad search in `tests/integration/test_entrez_api.py::test_search_genes_tumor_suppressor`:
  - Call search_genes("tumor suppressor")
  - Verify multiple candidates returned with relevance scores
- [x] T021 [P] [US1] Integration test for organism filter in `tests/integration/test_entrez_api.py::test_search_genes_organism_filter`:
  - Call search_genes("TP53", organism="human")
  - Verify all results have organism="Homo sapiens"
- [x] T022 [P] [US1] Integration test for short query error in `tests/integration/test_entrez_api.py::test_search_genes_short_query`:
  - Call search_genes("A")
  - Verify AMBIGUOUS_QUERY error returned with recovery_hint

### Implementation for User Story 1

- [x] T023 [US1] Implement _esearch method in EntrezClient per research.md R1:
  - Query esearch.fcgi endpoint with db=gene, term=query, retmode=json
  - Return parsed JSON with idlist, count, retstart
- [x] T024 [US1] Implement _esummary method in EntrezClient per research.md R1:
  - Query esummary.fcgi endpoint with db=gene, id=comma_separated_ids, retmode=json
  - Return parsed JSON with gene summaries
- [x] T025 [US1] Implement cursor encoding/decoding per research.md R2 in EntrezClient:
  - encode_cursor(offset: int) -> str using base64-encoded JSON
  - decode_cursor(cursor: str | None) -> int | None
- [x] T026 [US1] Implement relevance scoring logic in EntrezClient:
  - First result = 1.0, decreasing by position (1.0 - i * 0.05)
  - Normalize scores to 0.0-1.0 range
- [x] T027 [US1] Implement search_genes_raw method in EntrezClient using esearch + esummary two-step pattern per research.md R2:
  - Step 1: esearch to get ID list and total count
  - Step 2: esummary to get gene summaries for IDs
  - Transform to GeneSearchCandidate list with relevance scores
  - Handle pagination via retstart offset
- [x] T028 [US1] Create search_genes MCP tool in `src/lifesciences_mcp/servers/entrez.py` per contracts/search_genes.yaml:
  - @mcp.tool decorator with description from contract
  - Parameters: query (str, required), organism (str, optional), page_size (int, 1-100, default=50), cursor (str, optional)
  - Return PaginationEnvelope[GeneSearchCandidate]
- [x] T029 [US1] Add query validation to search_genes tool:
  - Minimum 2 characters per contract error spec
  - Return AMBIGUOUS_QUERY ErrorEnvelope for invalid queries

**Checkpoint**: User Story 1 complete - agents can discover genes via fuzzy search

---

## Phase 4: User Story 2 - Strict Gene Lookup (Priority: P2)

**Goal**: LLM agents can retrieve complete gene records with cross-references using resolved NCBIGene CURIEs

**Independent Test**: Call get_gene("NCBIGene:7157") (TP53) and verify all required fields populated

**Contract Reference**: `specs/009-entrez-mcp-server/contracts/get_gene.yaml`

### Tests for User Story 2

- [x] T030 [P] [US2] Integration test for TP53 gene lookup in `tests/integration/test_entrez_api.py::test_get_gene_tp53`:
  - Call get_gene("NCBIGene:7157")
  - Verify EntrezGene returned with symbol="TP53", summary present
  - Verify cross_references contains hgnc, ensembl_gene, uniprot keys
- [x] T031 [P] [US2] Integration test for invalid CURIE format in `tests/integration/test_entrez_api.py::test_get_gene_invalid_format`:
  - Call get_gene("7157") (raw numeric, not CURIE)
  - Verify UNRESOLVED_ENTITY error with recovery_hint pointing to search_genes
- [x] T032 [P] [US2] Integration test for non-existent ID in `tests/integration/test_entrez_api.py::test_get_gene_not_found`:
  - Call get_gene("NCBIGene:999999999999")
  - Verify ENTITY_NOT_FOUND error
- [x] T033 [P] [US2] Integration test for cross-reference extraction in `tests/integration/test_entrez_api.py::test_get_gene_cross_references`:
  - Call get_gene for known gene with many xrefs (TP53)
  - Verify HGNC, Ensembl, UniProt, RefSeq mappings correct
  - Verify omit-if-null pattern (no null values in cross_references)
- [x] T034 [P] [US2] Integration test for slim mode in `tests/integration/test_entrez_api.py::test_get_gene_slim`:
  - Call get_gene("NCBIGene:672", slim=True)
  - Verify summary, description, cross_references omitted in response

### Implementation for User Story 2

- [x] T035 [US2] Implement _efetch method in EntrezClient per research.md R1:
  - Query efetch.fcgi endpoint with db=gene, id=numeric_id, retmode=xml
  - Return raw XML string for parsing
- [x] T036 [US2] Implement _parse_gene_xml method using defusedxml per research.md R3:
  - Parse Entrezgene-Set -> Entrezgene elements
  - Extract gene_id, symbol, name, map_location, aliases, summary, organism, xrefs
  - Return dict or None if parsing fails
- [x] T037 [US2] Implement _build_cross_references helper per research.md R3 and data-model.md:
  - Map NCBI Dbtag_db to 22-key registry (HGNC->hgnc, Ensembl->ensembl_gene, MIM->omim, etc.)
  - Handle list fields (uniprot, refseq) by collecting multiple values
  - Format HGNC as "HGNC:{id}", UniProt as "UniProtKB:{id}"
  - Apply omit-if-null: only include keys with values
- [x] T038 [US2] Add NCBIGene CURIE format validation to get_gene:
  - Validate input matches NCBI_GENE_CURIE_PATTERN
  - Return UNRESOLVED_ENTITY ErrorEnvelope with recovery_hint if invalid
- [x] T039 [US2] Add ENTITY_NOT_FOUND handling:
  - If efetch returns empty or esummary has "error" field, return ENTITY_NOT_FOUND
  - Include recovery_hint: "Verify the NCBIGene ID or search by gene symbol"
- [x] T040 [US2] Create get_gene MCP tool in `src/lifesciences_mcp/servers/entrez.py` per contracts/get_gene.yaml:
  - @mcp.tool decorator with description from contract
  - Parameters: entrez_id (str, required, NCBIGene CURIE), slim (bool, default=False)
  - Return EntrezGene or ErrorEnvelope
- [x] T041 [US2] Implement slim mode in get_gene:
  - When slim=True, omit summary, description, cross_references from response
  - Reduces token usage from ~115-300 to ~25 tokens

**Checkpoint**: User Stories 1 AND 2 complete - full Fuzzy-to-Fact workflow for genes

---

## Phase 5: User Story 3 - PubMed Literature Links (Priority: P3)

**Goal**: LLM agents can retrieve PubMed IDs associated with a gene for evidence gathering

**Independent Test**: Call get_pubmed_links("NCBIGene:7157") and verify PubMed IDs returned

**Contract Reference**: `specs/009-entrez-mcp-server/contracts/get_pubmed_links.yaml`

### Tests for User Story 3

- [x] T042 [P] [US3] Integration test for TP53 pubmed links in `tests/integration/test_entrez_api.py::test_get_pubmed_links_tp53`:
  - Call get_pubmed_links("NCBIGene:7157", limit=10)
  - Verify list of PubMed ID strings returned
  - Verify at most 10 IDs returned
- [x] T043 [P] [US3] Integration test for invalid CURIE in `tests/integration/test_entrez_api.py::test_get_pubmed_links_invalid_curie`:
  - Call get_pubmed_links("7157") (not CURIE format)
  - Verify UNRESOLVED_ENTITY error returned
- [x] T044 [P] [US3] Integration test for gene with no links in `tests/integration/test_entrez_api.py::test_get_pubmed_links_empty`:
  - Call get_pubmed_links for obscure gene with few/no publications
  - Verify empty list returned (not error)

### Implementation for User Story 3

- [x] T045 [US3] Implement _elink method in EntrezClient per research.md R5:
  - Query elink.fcgi with dbfrom=gene, db=pubmed, id=numeric_id, linkname=gene_pubmed, retmode=json
  - Return parsed JSON with linksets
- [x] T046 [US3] Implement get_pubmed_links_raw method in EntrezClient:
  - Validate NCBIGene CURIE format
  - Call _elink to get pubmed linksets
  - Extract PubMed IDs from linksetdbs[0].links
  - Apply limit parameter
  - Return list[str] of PubMed IDs
- [x] T047 [US3] Create get_pubmed_links MCP tool in `src/lifesciences_mcp/servers/entrez.py` per contracts/get_pubmed_links.yaml:
  - @mcp.tool decorator with description from contract
  - Parameters: entrez_id (str, required, NCBIGene CURIE), limit (int, 1-100, default=10)
  - Return list[str] or ErrorEnvelope
- [x] T048 [US3] Add CURIE validation and error handling:
  - Validate input matches NCBI_GENE_CURIE_PATTERN
  - Return UNRESOLVED_ENTITY for invalid CURIE
  - Return empty list for gene with no pubmed links (not error)

**Checkpoint**: User Stories 1, 2, AND 3 complete - gene search, lookup, and literature links

---

## Phase 6: User Story 4 - Error Recovery (Priority: P4)

**Goal**: LLM agents can self-correct from errors using actionable recovery hints

**Independent Test**: Trigger each error type and verify recovery hints are actionable

### Tests for User Story 4

- [x] T049 [P] [US4] Integration test for UNRESOLVED_ENTITY recovery in `tests/integration/test_entrez_api.py::test_error_recovery_unresolved`:
  - Call get_gene("TP53") -> get UNRESOLVED_ENTITY
  - Follow recovery_hint: call search_genes("TP53")
  - Get valid NCBIGene CURIE
  - Call get_gene with valid CURIE -> success
- [x] T050 [P] [US4] Integration test for error envelope format compliance in `tests/integration/test_entrez_api.py::test_error_envelope_format`:
  - Trigger various errors
  - Verify all contain: code, message, recovery_hint, invalid_input (when applicable)
- [x] T051 [P] [US4] Integration test for RATE_LIMITED hint in `tests/integration/test_entrez_api.py::test_rate_limited_hint`:
  - Verify RATE_LIMITED error recovery_hint mentions NCBI_API_KEY per FR-018

### Implementation for User Story 4

- [x] T052 [US4] Create _create_error_envelope helper in EntrezClient:
  - Standard factory for ErrorEnvelope creation
  - Ensure consistent format across all error types
- [x] T053 [US4] Add recovery_hint for UNRESOLVED_ENTITY errors per contracts:
  - "Call search_genes to resolve the identifier first. Expected format: NCBIGene:<numeric_id>"
- [x] T054 [US4] Add recovery_hint for RATE_LIMITED errors per FR-018:
  - "Wait and retry. Add NCBI_API_KEY environment variable for 10 req/s limit (free from NCBI)"
- [x] T055 [US4] Add recovery_hint for UPSTREAM_ERROR:
  - "NCBI E-utilities may be temporarily unavailable. Retry after a few seconds."
- [x] T056 [US4] Add recovery_hint for AMBIGUOUS_QUERY:
  - "Provide at least 2 characters or add organism filter to refine search"
- [x] T057 [US4] Verify all error envelopes include invalid_input field:
  - Echo back the problematic input for agent context

**Checkpoint**: All 4 user stories complete - robust error handling with recovery guidance

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Quality improvements, documentation, and validation

### Unit Tests

- [x] T058 [P] Create unit tests for GeneSearchCandidate model in `tests/unit/test_entrez_models.py`:
  - Valid NCBIGene CURIE pattern acceptance
  - Invalid CURIE rejection (missing prefix, non-numeric)
  - Score range validation (0.0-1.0)
- [x] T059 [P] Create unit tests for EntrezGene model in `tests/unit/test_entrez_models.py`:
  - CURIE pattern validation
  - Optional field handling (summary, aliases, map_location)
  - Slim mode field exclusion
- [x] T060 [P] Create unit tests for CrossReferences in `tests/unit/test_entrez_models.py`:
  - Omit-if-null pattern verification
  - HGNC/UniProt prefix formatting
  - List field handling (uniprot, refseq)
- [x] T061 [P] Create unit tests for EntrezClient with mocked responses in `tests/unit/test_entrez_client.py`:
  - Adaptive rate limiting logic (3 vs 10 req/s)
  - Exponential backoff
  - Error mapping
  - XML parsing with defusedxml

### Documentation

- [x] T062 [P] Add module-level docstrings to `src/lifesciences_mcp/models/entrez.py`
- [x] T063 [P] Add module-level docstrings to `src/lifesciences_mcp/clients/entrez.py`
- [x] T064 [P] Add module-level docstrings to `src/lifesciences_mcp/servers/entrez.py`
- [x] T065 [P] Update `tests/conftest.py` with EntrezClient fixtures

### Validation

- [x] T066 Run linting: `uv run ruff check --fix . && uv run ruff format .`
- [x] T067 Run type checking: `uv run pyright src/lifesciences_mcp/{models,clients,servers}/entrez.py`
- [x] T068 Verify all integration tests pass: `uv run pytest tests/integration/test_entrez_api.py -v -m integration`
- [x] T069 [P] Performance test validating SC-001 (<2s for 95% of queries) in `tests/integration/test_entrez_performance.py`

### Final Validation

- [x] T070 Verify quickstart.md workflows execute successfully (Workflows 1-4)
- [x] T071 Validate success criteria from spec.md:
  - SC-001: 95% of search queries < 2 seconds
  - SC-002: Rate limiting prevents 429 errors
  - SC-003: Cross-references populated for genes with xrefs
  - SC-004: Fuzzy-to-Fact workflow works without human intervention
  - SC-005: Error recovery hints enable 90% self-correction
  - SC-006: All integration tests pass
  - SC-007: XML responses correctly parsed
- [x] T072 Update CLAUDE.md with Entrez server documentation

**Checkpoint**: NCBI Entrez MCP Server implementation complete

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
- **User Story 3 (P3)**: Can start after Phase 2 - Adds literature discovery
- **User Story 4 (P4)**: Can start after Phase 2 - Cross-cutting error handling

### Within Each User Story

- Tests MUST be written and FAIL before implementation
- Models before client methods
- Client methods before server tools
- Core implementation before integration
- Story complete before moving to next priority

### Parallel Opportunities

**Phase 1 (Setup)**: T002-T007 can run in parallel after T001

**Phase 2 (Foundational)**:
- T009-T010 (models) can run in parallel after T008
- T013-T016 (client infra) can run in parallel after T012

**User Story 1**:
- Tests T019-T022 can run in parallel (write first)
- After tests: T023-T029 mostly sequential (two-step pattern)

**User Story 2**:
- Tests T030-T034 can run in parallel
- After tests: T035-T041 mostly sequential

**User Story 3**:
- Tests T042-T044 can run in parallel
- After tests: T045-T048 sequential

**User Story 4**:
- Tests T049-T051 can run in parallel
- Implementation T052-T057 can mostly run in parallel

**Phase 7 (Polish)**:
- All unit test tasks (T058-T061) can run in parallel
- All documentation tasks (T062-T065) can run in parallel
- Validation tasks run sequentially for clear results

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (T001-T007)
2. Complete Phase 2: Foundational (T008-T018) - CRITICAL, blocks all stories
3. Complete Phase 3: User Story 1 (T019-T029)
4. **STOP and VALIDATE**: Run tests for User Story 1 independently
5. Deploy/demo if ready - agents can now discover NCBI genes

**MVP Checkpoint**: Fuzzy search tool functional. Agents can discover gene IDs but cannot yet get full records.

### Full Fuzzy-to-Fact (Add User Story 2)

1. Complete MVP (above)
2. Add User Story 2 (T030-T041) - Strict gene lookup with XML parsing
3. **VALIDATE**: Complete Fuzzy-to-Fact workflow works
4. Agents can now: search -> select -> get complete gene data with xrefs

### Incremental Delivery

1. Setup + Foundational (T001-T018) -> Foundation ready
2. Add User Story 1 (T019-T029) -> Test -> Deploy (**MVP: Fuzzy gene search**)
3. Add User Story 2 (T030-T041) -> Test -> Deploy (**Core: Fuzzy-to-Fact workflow**)
4. Add User Story 3 (T042-T048) -> Test -> Deploy (Literature links)
5. Add User Story 4 (T049-T057) -> Test -> Deploy (Error recovery)
6. Add Polish (T058-T072) -> Final validation -> Production-ready

Each story adds value without breaking previous stories.

---

## Task Summary

| Phase | Tasks | Complete | Parallel | Story |
|-------|-------|----------|----------|-------|
| Phase 1: Setup | 7 | 0 (0%) | 6 | N/A |
| Phase 2: Foundational | 11 | 0 (0%) | 5 | N/A |
| Phase 3: User Story 1 (P1) | 11 | 0 (0%) | 4 | US1 |
| Phase 4: User Story 2 (P2) | 12 | 0 (0%) | 5 | US2 |
| Phase 5: User Story 3 (P3) | 7 | 0 (0%) | 3 | US3 |
| Phase 6: User Story 4 (P4) | 9 | 0 (0%) | 3 | US4 |
| Phase 7: Polish | 15 | 0 (0%) | 9 | N/A |
| **TOTAL** | **72** | **0 (0%)** | **35** | **4 stories** |

**Parallel efficiency**: 49% of tasks can run in parallel (35/72)

**Independent test criteria** from spec.md:
- US1: Search "BRCA1" -> Get ranked candidates with NCBIGene:672
- US2: Call get_gene("NCBIGene:7157") -> Get full gene with xrefs
- US3: Call get_pubmed_links("NCBIGene:7157") -> Get PubMed IDs
- US4: Trigger errors -> Get actionable recovery hints

---

## Notes

- **[P] tasks** = different files, no dependencies, can run in parallel
- **[Story] label** maps task to specific user story for traceability
- Each user story independently completable and testable
- Tests written FIRST and should FAIL before implementation (TDD)
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Rate limit: 3 req/s without API key, 10 req/s with NCBI_API_KEY per research.md R4
- XML parsing: Use defusedxml for security (XXE prevention) per research.md R3
- Two-step pattern: esearch -> esummary/efetch per research.md R2
- CURIE format: NCBIGene:\d+ per research.md R6
- Canonical envelopes: PaginationEnvelope, ErrorEnvelope per ADR-001 Section 8
