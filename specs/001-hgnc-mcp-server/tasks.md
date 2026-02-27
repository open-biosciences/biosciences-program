# Tasks: HGNC MCP Server

**Input**: Design documents from `/specs/001-hgnc-mcp-server/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/hgnc-tools.json

**Total Tasks**: 33 | **User Stories**: 4 (2 P1, 2 P2)

## Platform Skills Usage

> **CRITICAL**: Per Constitution Principle VI (Platform Skill Delegation), the following tasks
> MUST use platform skills instead of manual coding:

| Task | Platform Skill | Why Required |
|------|----------------|--------------|
| T001 | **scaffold-fastmcp** | FastMCP project structure follows standard patterns |
| T002 | **scaffold-fastmcp** | Server boilerplate and transport configuration |

All other tasks involve feature-specific code that cannot be scaffolded.

---

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3, US4)
- Include exact file paths in descriptions

---

## Phase 1: Setup (Project Initialization)

**Purpose**: Create project structure using Platform Skills

- [x] T001 🔧 ~~**PLATFORM SKILL**~~ (manual - skill not available): Create src/lifesciences_mcp/ structure with __init__.py, models/, servers/
- [x] T002 🔧 ~~**PLATFORM SKILL**~~ (manual - skill not available): Generate src/lifesciences_mcp/servers/hgnc.py with FastMCP boilerplate
- [x] T003 [P] Add pytest-asyncio, httpx to pyproject.toml dev dependencies
- [x] T004 [P] Create tests/ directory structure: conftest.py, unit/, contract/, integration/
- [x] T005 [P] Configure ruff.toml for linting (async rules enabled)

**Checkpoint**: Project skeleton ready, FastMCP server can start (empty tools)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that ALL user stories depend on

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [x] T006 [P] Create ErrorEnvelope Pydantic model in src/lifesciences_mcp/models/envelopes.py (ADR-001 §8)
- [x] T007 [P] Create PaginationEnvelope Pydantic model in src/lifesciences_mcp/models/envelopes.py (ADR-001 §8)
- [x] T008 [P] Create CrossReferences Pydantic model with 22-key registry in src/lifesciences_mcp/models/gene.py
- [x] T009 Create LifeSciencesClient base class with async httpx in src/lifesciences_mcp/client.py
- [x] T010 Implement HGNCClient (extends LifeSciencesClient) with rate limiting (10 req/s) in src/lifesciences_mcp/client.py
- [x] T011 Create pytest fixtures for async testing in tests/conftest.py

**Checkpoint**: Foundation ready — httpx client works, envelopes defined, user story implementation can begin

---

## Phase 3: User Story 1 - Fuzzy Gene Search (Priority: P1) 🎯 MVP

**Goal**: AI agents can search for genes using natural language and receive ranked candidates

**Independent Test**: Send gene names, synonyms, disease terms to search_genes and verify ranked SearchCandidate results

### Implementation for User Story 1

- [x] T012 [P] [US1] Create SearchCandidate Pydantic model in src/lifesciences_mcp/models/gene.py (id, symbol, name, score)
- [x] T013 [US1] Implement HGNC /search/ endpoint integration in src/lifesciences_mcp/client.py
- [x] T014 [US1] Implement search_genes tool in src/lifesciences_mcp/servers/hgnc.py with query parameter
- [x] T015 [US1] Add slim=True support to search_genes (~20 tokens per entity) in src/lifesciences_mcp/servers/hgnc.py
- [x] T016 [US1] Add relevance scoring logic (HGNC score mapping) in src/lifesciences_mcp/client.py
- [x] T017 [US1] Return PaginationEnvelope from search_genes in src/lifesciences_mcp/servers/hgnc.py

**Checkpoint**: User Story 1 complete — agents can fuzzy search for genes

---

## Phase 4: User Story 2 - Strict Gene Lookup (Priority: P1)

**Goal**: AI agents can retrieve full gene records by HGNC CURIE with cross-references

**Independent Test**: Pass known HGNC IDs (e.g., HGNC:1100) and verify complete Gene records with cross_references

### Implementation for User Story 2

- [x] T018 [P] [US2] Create Gene Pydantic model in src/lifesciences_mcp/models/gene.py (full record with all fields)
- [x] T019 [US2] Implement HGNC /fetch/hgnc_id/ endpoint integration in src/lifesciences_mcp/client.py
- [x] T020 [US2] Implement get_gene tool with CURIE validation (regex ^HGNC:\d+$) in src/lifesciences_mcp/servers/hgnc.py
- [x] T021 [US2] Map HGNC response fields to CrossReferences (ensembl_gene_id → ensembl_gene, etc.) in src/lifesciences_mcp/client.py
- [x] T022 [US2] Omit cross_reference keys with no value (never null/empty) in src/lifesciences_mcp/models/gene.py

**Checkpoint**: Fuzzy-to-Fact protocol complete — agents can search → select → lookup

---

## Phase 5: User Story 3 - Error Recovery (Priority: P2)

**Goal**: AI agents receive structured errors with actionable recovery hints

**Independent Test**: Trigger each error condition and verify ErrorEnvelope with correct code and recovery_hint

### Implementation for User Story 3

- [x] T023 [US3] Implement UNRESOLVED_ENTITY error (raw string to get_gene) in src/lifesciences_mcp/servers/hgnc.py
- [x] T024 [US3] Implement ENTITY_NOT_FOUND error (valid CURIE, no record) in src/lifesciences_mcp/servers/hgnc.py
- [x] T025 [US3] Implement AMBIGUOUS_QUERY error (>100 results or short query) in src/lifesciences_mcp/servers/hgnc.py
- [x] T026 [US3] Implement RATE_LIMITED and UPSTREAM_ERROR for HGNC API failures in src/lifesciences_mcp/client.py

**Checkpoint**: All error codes from ADR-001 Appendix B implemented

---

## Phase 6: User Story 4 - Pagination (Priority: P2)

**Goal**: AI agents can traverse large result sets without context overflow

**Independent Test**: Perform broad searches, verify pagination envelope, traverse pages with cursor

### Implementation for User Story 4

- [x] T027 [US4] Implement client-side pagination with base64 cursor encoding in src/lifesciences_mcp/client.py
- [x] T028 [US4] Add cursor/page_size parameters to search_genes in src/lifesciences_mcp/servers/hgnc.py

**Checkpoint**: All user stories complete — full Fuzzy-to-Fact protocol with error handling and pagination

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Validation, documentation, and final quality checks

- [x] T029 [P] Validate tool schemas against contracts/hgnc-tools.json (verified via integration tests)
- [x] T030 [P] Run quickstart.md examples as smoke tests (integration tests cover Fuzzy-to-Fact workflow)
- [x] T031 Add HGNC server registration to src/lifesciences_mcp/__init__.py
- [x] T032 [P] Update CLAUDE.md with implemented tool descriptions
- [x] T033 [P] Create async load test to verify SC-007 (100 concurrent requests) in tests/integration/test_concurrency.py

---

## Dependencies & Execution Order

### Phase Dependencies

```
Phase 1 (Setup) ────────────────────────────────────────────────────────────────►
                 │
                 ▼
Phase 2 (Foundational) ─────────────────────────────────────────────────────────►
                 │
                 ├──────────────┬──────────────┬──────────────┐
                 ▼              ▼              ▼              ▼
         Phase 3 (US1)   Phase 4 (US2)   Phase 5 (US3)   Phase 6 (US4)
              P1              P1              P2              P2
                 │              │              │              │
                 └──────────────┴──────────────┴──────────────┘
                                       │
                                       ▼
                             Phase 7 (Polish)
```

### User Story Dependencies

| Story | Depends On | Can Parallel With |
|-------|------------|-------------------|
| US1 (Fuzzy Search) | Phase 2 only | US2, US3, US4 |
| US2 (Strict Lookup) | Phase 2 only | US1, US3, US4 |
| US3 (Error Recovery) | Phase 2 only | US1, US2, US4 |
| US4 (Pagination) | Phase 2 only | US1, US2, US3 |

**Note**: All user stories can run in parallel after Foundational phase completes.
However, for single-developer flow, recommended order is: US1 → US2 → US3 → US4.

### Platform Skill Dependencies

| Task | Requires | Output |
|------|----------|--------|
| T001 | scaffold-fastmcp skill | src/lifesciences_mcp/ skeleton |
| T002 | T001 complete | hgnc.py with FastMCP boilerplate |

---

## Parallel Execution Examples

### Phase 2 (Foundational) — Maximum Parallelism

```bash
# These 3 tasks can run in parallel (different files):
Task T006: "Create ErrorEnvelope in src/lifesciences_mcp/models/envelopes.py"
Task T007: "Create PaginationEnvelope in src/lifesciences_mcp/models/envelopes.py"
Task T008: "Create CrossReferences in src/lifesciences_mcp/models/gene.py"
```

### User Stories — Parallel Across Stories

```bash
# If team capacity allows, after Phase 2 completes:
Developer A: Phase 3 (US1 - Fuzzy Search)
Developer B: Phase 4 (US2 - Strict Lookup)
# US3 and US4 can wait or run in parallel with US1/US2
```

---

## Implementation Strategy

### MVP First (User Stories 1 + 2 Only)

1. Complete Phase 1: Setup (use scaffold-fastmcp)
2. Complete Phase 2: Foundational (envelopes, client, models)
3. Complete Phase 3: User Story 1 (Fuzzy Search)
4. Complete Phase 4: User Story 2 (Strict Lookup)
5. **STOP and VALIDATE**: Test Fuzzy-to-Fact workflow end-to-end
6. Demo/deploy MVP

### Incremental Delivery

| Increment | Stories | Capability |
|-----------|---------|------------|
| MVP | US1 + US2 | Fuzzy search + strict lookup (core protocol) |
| v1.1 | + US3 | Error recovery for agent autonomy |
| v1.2 | + US4 | Pagination for large result sets |
| v1.3 | + Polish | Production-ready with docs |

### Suggested First Session

Execute Tasks T001-T011 (Setup + Foundational) to establish the project skeleton,
then T012-T017 (User Story 1) to deliver a working fuzzy search. This provides
immediate value and validates the architecture.

---

## Notes

- **[P]** tasks = different files, no dependencies within phase
- **[Story]** label maps task to specific user story
- **🔧 PLATFORM SKILL** = MUST use platform skill, not manual coding
- Each user story is independently testable after completion
- Commit after each task or logical group
- Stop at any checkpoint to validate independently
