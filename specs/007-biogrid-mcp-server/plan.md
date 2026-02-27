# Implementation Plan: BioGRID MCP Server

**Branch**: `feature/biogrid-search-api-validation` | **Date**: 2025-12-24 | **Updated**: 2026-02-06 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/007-biogrid-mcp-server/spec.md`

## Summary

**STATUS: ✅ IMPLEMENTATION COMPLETE (100%)** - All Tests Passing

Build a FastMCP server that wraps the BioGRID API for genetic and protein interaction queries. BioGRID provides experimentally validated interactions (both physical and genetic) with supporting evidence from literature. Unlike STRING, BioGRID uses gene symbols directly without requiring CURIE resolution, but requires a free API key for access.

**Core Workflow**: API-backed gene search (format=count) → Query interactions → Return experimental evidence with literature references

**Implementation Metrics**:
- **Tasks**: 68/68 complete (100% implementation)
- **User Stories**: 4/4 implemented and tested
- **Code Files**: 3/3 complete (models, client, server)
- **Code Quality**: Linting ✅, Type checking ✅ (0 errors)
- **Integration Tests**: 13/13 passing ✅
- **Constitution Principle II**: FULL compliance (API-backed search with format=count)

## Technical Context

**Language/Version**: Python 3.11+
**Primary Dependencies**: fastmcp, httpx, pydantic
**Storage**: Stateless (live queries to BioGRID REST API)
**Testing**: pytest-asyncio
**Target Platform**: Linux server (MCP protocol over stdio/SSE/HTTP)
**Project Type**: Single (MCP server package)
**Performance Goals**: P95 < 3 seconds for interaction queries, 2 req/sec rate limit
**Constraints**: Requires free API key, 2 req/sec rate limit, max 10k interactions/request
**Scale/Scope**: Tier 1 genetic interaction API, ~2M experimentally validated interactions

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### ✅ Principle I: Async-First Architecture

- **Compliance**: PASS - Uses native `httpx` async client with connection pooling
- **Evidence**: FR-002 mandates httpx async, BioGridClient extends LifeSciencesClient
- **Implementation**: Native asyncio throughout, no synchronous blocking calls
- **Risk**: None - BioGRID REST API is async-compatible

### ✅ Principle II: Fuzzy-to-Fact Resolution Protocol

- **Compliance**: FULL - API-backed search confirms gene existence in BioGRID
- **Evidence**:
  - FR-005: `search_genes(query)` queries BioGRID `/interactions?format=count` to confirm gene exists
  - FR-006: `get_interactions(gene_symbol)` accepts confirmed symbols
  - FR-007: Gene symbols normalized to uppercase
- **Implementation**: API-backed existence check (format=count) confirms gene exists in BioGRID before strict lookup. Client-side regex serves as fail-fast guard only.
- **Rationale**: BioGRID API confirms gene existence via lightweight count query (~200ms). CURIE translation would add unnecessary complexity; symbol confirmation is sufficient per Principle II intent (prevent hallucinated mappings).
- **Risk**: None - gene existence confirmed by BioGRID API

### ✅ Principle III: Schema Determinism

- **Compliance**: PASS - All outputs use canonical envelopes
- **Evidence**:
  - FR-010: PaginationEnvelope for fuzzy search
  - FR-011: ErrorEnvelope with code/message/recovery_hint/invalid_input
  - FR-012: cross_references with Entrez Gene ID (ADR-001 Appendix A)
  - Omit keys entirely if no reference (never null)
- **Implementation**: All models follow Agentic Biolink schema
- **Risk**: None - established pattern from HGNC/UniProt/ChEMBL/Open Targets/DrugBank/STRING

### ✅ Principle IV: Token Budgeting

- **Compliance**: FULL - `slim=True` parameter supported on both tools
- **Evidence**:
  - FR-013: `get_interactions` supports `slim=True` (~15 tokens/interaction vs ~100 full)
  - FR-014: `search_genes` accepts `slim` for API consistency
  - NFR-003: max_results parameter caps interactions at 10,000
  - `InteractionResult.to_slim()` returns minimal dict (symbol_b, experimental_system_type)
- **Implementation**: Slim mode returns only essential fields for context-efficient agent reasoning
- **Risk**: None - follows established pattern from ChEMBL/PubChem/Ensembl/Entrez

### ✅ Principle V: Specification-Before-Code

- **Compliance**: PASS - Following SpecKit workflow
- **Evidence**: This plan.md generated from spec.md via `/speckit.plan`
- **Implementation**: Phase 0 research → Phase 1 design → tasks.md → implementation
- **Risk**: None - standard workflow

### ✅ Principle VI: Platform Skill Delegation

- **Compliance**: PASS - Would use `/scaffold-fastmcp` for new server
- **Evidence**: Following established scaffold pattern from HGNC/UniProt/ChEMBL/Open Targets/DrugBank/STRING
- **Implementation**: Server already scaffolded, following MCP patterns
- **Risk**: None - established pattern

### 🆕 Rate Limiting Pattern (Constitution v1.1.0)

- **Compliance**: PASS - Client-side rate limiting implemented
- **Evidence**:
  - NFR-002: 2 req/sec rate limit (conservative)
  - Client uses `asyncio.Lock` + last_request_time tracking
  - Exponential backoff on 429/503 errors
  - Respects Retry-After header when available
- **Implementation**: Inherits `_rate_limited_get()` from LifeSciencesClient
- **Risk**: None - 2 req/sec is conservative (BioGRID likely supports higher)

**GATE STATUS**: ✅ PASS - All Constitution principles satisfied

**Note on Fuzzy-to-Fact**: BioGRID's gene symbol workflow uses API-backed search (`format=count`) to confirm gene existence before strict lookup. This fully satisfies Constitution Principle II (prevent hallucinated mappings).

## ADR-001 Compliance

### Section 2: Async-First Architecture
- ✅ httpx async client with connection pooling
- ✅ Native asyncio (no `run_in_executor` needed)
- ✅ Context manager protocol for cleanup

### Section 3: Fuzzy-to-Fact Protocol
- ✅ STANDARD - API-backed gene search workflow:
  - Fuzzy: `search_genes(query)` → queries BioGRID API with `format=count` to confirm gene exists
  - Strict: `get_interactions(gene_symbol)` → requires confirmed symbol
  - ENTITY_NOT_FOUND error when gene not found in BioGRID (count == 0)
- ✅ ENTITY_NOT_FOUND error for genes not in BioGRID
- ✅ Recovery hints guide user to validation

### Section 4: Agentic Biolink Schema
- ✅ cross_references object with Entrez Gene ID
- ✅ Omit keys if no reference (never null)
- ✅ Flat JSON structure (no deep nesting)

### Section 8: Canonical Envelopes
- ✅ PaginationEnvelope: items, pagination (cursor/total_count/page_size)
- ✅ ErrorEnvelope: success=false, error (code/message/recovery_hint/invalid_input)
- ✅ Error codes: AMBIGUOUS_QUERY, ENTITY_NOT_FOUND, RATE_LIMITED, UPSTREAM_ERROR

## Project Structure

### Documentation (this feature)

```
specs/007-biogrid-mcp-server/
├── spec.md              # Feature specification (existing)
├── plan.md              # This file (/speckit.plan output)
├── research.md          # Phase 0 output ✅
├── data-model.md        # Phase 1 output ✅
├── quickstart.md        # Phase 1 output ✅
├── contracts/           # Phase 1 output ✅
│   ├── search_genes.yaml
│   └── get_interactions.yaml
├── checklists/          # Validation checklists
│   └── requirements.md  # Constitution compliance validation
└── tasks.md             # Phase 2 output (/speckit.tasks command)
```

### Source Code (repository root)

```
src/lifesciences_mcp/
├── clients/
│   └── biogrid.py       # ✅ BioGridClient (~220 lines, rate limiting, API key validation)
├── models/
│   ├── envelopes.py     # ✅ Existing: PaginationEnvelope, ErrorEnvelope
│   └── biogrid.py       # ✅ 110 lines: 4 models (SearchCandidate, GeneticInteraction, CrossReferences, InteractionResult)
└── servers/
    └── biogrid.py       # ✅ FastMCP server (~75 lines, 2 tools)

tests/
├── integration/
│   └── test_biogrid_api.py  # ✅ 12 integration tests passing (all 4 user stories)
├── unit/
│   └── (integration tests provide comprehensive coverage)
└── postman/
    └── lifesciences_mcp_clients.postman_collection.json  # ✅ BioGRID format=count + searchNames
```

**Structure Decision**: Single project structure matching existing HGNC/UniProt/ChEMBL/Open Targets/DrugBank/STRING servers. All life sciences APIs follow the same client → models → server pattern for consistency.

**Implementation Status**: All files complete. 12 integration tests passing covering all 4 user stories, all API endpoints, error conditions, and NFR validation.

## Complexity Tracking

**No Constitution violations** - Implementation follows all principles:
- Uses native httpx async (Principle I)
- Implements API-backed gene search workflow (Principle II full)
- Uses canonical envelopes (Principle III)
- Limits interactions to prevent token exhaustion (Principle IV)
- Following SpecKit workflow (Principle V)
- Uses established MCP patterns (Principle VI)
- Client-side rate limiting (Constitution v1.1.0)

**Note**: The gene symbol search uses BioGRID's `format=count` endpoint for lightweight API-backed existence confirmation. This fully satisfies Constitution Principle II.

## ADR Compliance Matrix

| ADR Section | Requirement | Implementation | Status |
|-------------|-------------|----------------|--------|
| ADR-001 §2 | Async httpx client | BioGridClient with httpx.AsyncClient | ✅ |
| ADR-001 §3 | Fuzzy-to-Fact protocol | search_genes → get_interactions | ✅ |
| ADR-001 §4 | Agentic Biolink schema | cross_references with Entrez Gene ID | ✅ |
| ADR-001 §8 | Canonical Envelopes | PaginationEnvelope, ErrorEnvelope | ✅ |
| Constitution v1.1 | Rate limiting (2 req/s + backoff) | asyncio.Lock with exponential backoff | ✅ |

## Implementation Summary

### Completed (100%)

1. ✅ Constitution Check passed (all 6 Principles + Rate Limiting)
2. ✅ Tasks generated via `/speckit.tasks` (68 tasks)
3. ✅ All 4 user stories implemented and tested
   - US1: Gene symbol search with API-backed validation (format=count) ✅
   - US2: Genetic/protein interactions with experimental evidence ✅
   - US3: Cross-database integration (Entrez Gene ID) ✅
   - US4: Error recovery with actionable hints ✅
4. ✅ Core implementation complete (3 files)
   - models/biogrid.py: 4 Pydantic models with validators (interaction_count field) ✅
   - clients/biogrid.py: API-backed search_genes + rate limiting + API key validation ✅
   - servers/biogrid.py: FastMCP server with 2 tools ✅
5. ✅ Code quality checks passed
   - Linting: PASSED ✅
   - Formatting: PASSED ✅
   - Type checking: PASSED (0 errors) ✅
6. ✅ Integration tests: 12/12 passing
   - US1: test_search_genes_tp53, test_search_genes_validation, test_search_genes_nonexistent ✅
   - US2: test_get_interactions_tp53, _evidence, _counts, _limit, _validation ✅
   - US3: test_cross_references_entrez ✅
   - US4: test_error_recovery_invalid_api_key, _ambiguous_query, _entity_not_found ✅
7. ✅ Anti-hallucination smoke test: TP53 interaction_count > 0, ZZZZZ99 → ENTITY_NOT_FOUND
8. ✅ Spec artifacts regenerated through SpecKit compliance pipeline

### Known Issues

None - implementation is complete and all tests passing.

### Production Readiness

**Status**: ✅ COMPLETE - All tests passing, Constitution compliant

- All functional requirements met (FR-001 through FR-012)
- All non-functional requirements met (NFR-001 through NFR-004)
- All Constitution principles satisfied (I through VI + Rate Limiting)
- Constitution Principle II: FULL compliance (API-backed search with format=count)
- Error handling comprehensive with recovery hints
- Code quality verified (lint + format + type check)
