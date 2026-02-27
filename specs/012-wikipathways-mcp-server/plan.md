# Implementation Plan: WikiPathways MCP Server

**Branch**: `012-wikipathways-mcp-server` | **Date**: 2026-01-03 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/012-wikipathways-mcp-server/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

Build a WikiPathways MCP server implementing the Fuzzy-to-Fact protocol for biological pathway discovery and analysis. The server provides four tools: (1) search_pathways for topic-based pathway discovery with organism filtering, (2) get_pathway for strict pathway lookup with complete metadata and cross-references, (3) get_pathways_for_gene for reverse gene→pathway lookup, and (4) get_pathway_components for extracting pathway constituents (genes, proteins, metabolites, interactions). All outputs follow the Agentic Biolink schema with canonical pagination and error envelopes. Rate limiting: conservative 1 req/sec with exponential backoff.

## Technical Context

**Language/Version**: Python 3.11+
**Primary Dependencies**: FastMCP >=2.0, httpx >=0.27, pydantic >=2.0, defusedxml (NEEDS CLARIFICATION: confirm if GPML XML parsing required)
**Storage**: N/A (stateless, live queries to WikiPathways REST API)
**Testing**: pytest-asyncio with unit tests (mocked) and integration tests (live API)
**Target Platform**: Linux server, macOS, WSL2 (cross-platform Python)
**Project Type**: Single (MCP server following ADR-001 structure)
**Performance Goals**: <2s response time for 95th percentile queries (SC-001), 1 req/sec rate limit
**Constraints**: Conservative rate limiting (1 req/sec), 10-second timeout per request, token budgeting (slim mode support)
**Scale/Scope**: 4 MCP tools, 5 Pydantic models (Pathway, PathwaySearchCandidate, PathwayComponents, DataNode, Interaction), ~500-800 lines client code, ~400-600 lines server code, 40-60 tests

**Research Needed**:
- NEEDS CLARIFICATION: WikiPathways API endpoint structure (search, getPathwayInfo, findPathwaysByXref, getPathwayAs format)
- NEEDS CLARIFICATION: Response format (JSON vs GPML XML) for pathway components and cross-references
- NEEDS CLARIFICATION: Pagination mechanism (cursor-based, offset-based, or no native pagination requiring client-side implementation)
- NEEDS CLARIFICATION: Rate limiting headers and documented limits (currently assuming conservative 1 req/sec)
- NEEDS CLARIFICATION: Cross-reference database mapping (Reactome, KEGG, GO identifiers in pathway metadata vs GPML annotations)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Principle I: Async-First Architecture
- ✅ **PASS**: WikiPathways REST API will use native httpx async client
- ✅ **PASS**: No synchronous SDK dependencies identified
- ✅ **PASS**: All network I/O will be async (no blocking calls)

### Principle II: Fuzzy-to-Fact Resolution Protocol
- ✅ **PASS**: search_pathways (fuzzy) returns PathwaySearchCandidate with WP:WPNNNNN IDs
- ✅ **PASS**: get_pathway (strict) accepts ONLY WP:WPNNNNN format, validates with regex ^WP:WP\d+$
- ✅ **PASS**: get_pathways_for_gene (fuzzy) returns pathway candidates
- ✅ **PASS**: get_pathway_components (strict) validates pathway ID format before API call
- ✅ **PASS**: UNRESOLVED_ENTITY error with recovery_hint on invalid formats

### Principle III: Schema Determinism
- ✅ **PASS**: All list tools (search_pathways, get_pathways_for_gene) use Canonical PaginationEnvelope
- ✅ **PASS**: All errors use Canonical ErrorEnvelope with recovery_hint
- ✅ **PASS**: cross_references object uses 22-key registry (reactome, kegg_pathway, gene_ontology, hgnc, uniprot, etc.)
- ✅ **PASS**: Omit-if-null pattern (never use null or empty string for cross-references)

### Principle IV: Token Budgeting
- ✅ **PASS**: search_pathways supports slim=True mode (id, title, organism, score ~20 tokens)
- ✅ **PASS**: Default page_size: 50, max: 100
- ✅ **PASS**: Slim mode mandatory for list operations

### Principle V: Specification-Before-Code
- ✅ **PASS**: Following /speckit.plan → /speckit.tasks → implementation workflow
- ✅ **PASS**: Human gate required before implementation begins

### Principle VI: Platform Skill Delegation
- ✅ **PASS**: Using /scaffold-fastmcp skill for initial server structure

### Required Patterns
- ✅ **PASS**: Canonical Pagination Envelope on all list tools
- ✅ **PASS**: Canonical Error Envelope on all errors
- ✅ **PASS**: Cross-reference regex validation (WP:WP\d+ for pathway IDs)
- ✅ **PASS**: Async httpx client for WikiPathways API
- ✅ **PASS**: slim=True support on batch tools
- ✅ **PASS**: Client-side rate limiting (1 req/sec, exponential backoff, thundering herd prevention)

### Forbidden Patterns
- ✅ **PASS**: No synchronous blocking in async context
- ✅ **PASS**: No hardcoded credentials (CC0 public API, no auth required)
- ✅ **PASS**: No raw strings to strict tools (WP ID validation enforced)
- ✅ **PASS**: No null cross-references (omit-if-null pattern)
- ✅ **PASS**: No deep JSON nesting (flattened Agentic Biolink schema)
- ✅ **PASS**: No unbounded concurrency (rate limiting enforced)

**GATE STATUS**: ✅ ALL CHECKS PASS - Proceed to Phase 0 Research

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
src/lifesciences_mcp/
├── __init__.py
├── clients/
│   ├── __init__.py
│   ├── base.py                    # LifeSciencesClient base class
│   └── wikipathways.py            # WikiPathwaysClient (new)
├── models/
│   ├── __init__.py
│   ├── envelopes.py               # PaginationEnvelope, ErrorEnvelope (existing)
│   ├── pathway.py                 # Pathway, PathwaySearchCandidate (new)
│   ├── pathway_components.py      # PathwayComponents, DataNode, Interaction (new)
│   └── [other existing models]
└── servers/
    ├── __init__.py
    └── wikipathways.py            # WikiPathways MCP server (new)

tests/
├── integration/
│   └── test_wikipathways_api.py   # Live API integration tests (new)
└── unit/
    ├── test_wikipathways_client.py  # Client unit tests (new)
    ├── test_wikipathways_models.py  # Model unit tests (new)
    └── [other existing tests]
```

**Structure Decision**: Following Option 1 (Single project - DEFAULT) per ADR-001 architecture. WikiPathways implementation follows the established pattern: client in `clients/`, models in `models/`, server in `servers/`, tests split by unit/integration. Follows HGNC, UniProt, ChEMBL, Open Targets, Ensembl, Entrez precedent.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

No violations. All Constitutional principles are satisfied.

---

## Phase 0: Research (COMPLETED)

**Output**: `research.md`

### Research Findings Summary

1. **API Endpoint Structure** ✅ RESOLVED
   - Base URL: `http://webservice.wikipathways.org`
   - All endpoints support JSON via `format=json`
   - Endpoints: `/findPathwaysByText`, `/getPathwayInfo`, `/findPathwaysByXref`, `/getXrefList`

2. **Response Format** ✅ RESOLVED
   - JSON exclusively (no GPML parsing needed for core functionality)
   - Cross-references via bulk JSON file (`findPathwaysByXref.json`)
   - Contains: `ncbigene`, `ensembl`, `hgnc`, `uniprot`, `chebi`

3. **Pagination** ✅ RESOLVED
   - No native API pagination support
   - Solution: Client-side cursor-based pagination using base64-encoded offsets
   - Typical result sets: 50-200 pathways (manageable in memory)

4. **Rate Limiting** ✅ RESOLVED
   - No documented rate limits (tested, no 429/headers)
   - Conservative implementation: 1 req/s with `asyncio.Lock`
   - Exponential backoff for 503/429 responses

5. **Cross-Reference Extraction** ✅ RESOLVED
   - Hybrid approach: REST API + JSON bulk file
   - BridgeDb system codes for `getXrefList`: L, S, En, H, Ce
   - No GPML parsing required

**All unknowns from Technical Context have been resolved.**

---

## Phase 1: Design & Contracts (COMPLETED)

**Outputs**: `data-model.md`, `contracts/`, `quickstart.md`, `CLAUDE.md` updated

### Data Model Summary

| Entity | Purpose | Token Budget |
|--------|---------|--------------|
| `Pathway` | Complete pathway record | ~300 tokens |
| `PathwaySearchCandidate` | Fuzzy search results | ~20 tokens |
| `PathwayComponents` | Pathway composition | ~500-1000 tokens |
| `DataNode` | Pathway component | ~50 tokens |
| `Interaction` | Component relationship | ~15 tokens |

**Key Patterns**:
- Omit-if-null for cross-references (Constitution Principle III)
- Slim mode support for token budgeting (Constitution Principle IV)
- Fuzzy-to-Fact protocol compliance (Constitution Principle II)

### API Contracts Generated

1. **search_pathways.md**: Fuzzy pathway discovery with organism filtering
2. **get_pathway.md**: Strict lookup with cross-references and metadata
3. **get_pathways_for_gene.md**: Reverse gene→pathway lookup
4. **get_pathway_components.md**: Component extraction with BridgeDb mapping

### Quickstart Guide

- 4 core workflows documented
- 4 error recovery scenarios
- End-to-end research example
- Cross-database integration patterns

### Agent Context Updated

**CLAUDE.md changes**:
- Added: Python 3.11+ (language)
- Added: N/A stateless (database)
- Added: WikiPathways MCP Server (013) to active technologies

---

## Constitution Check (POST-DESIGN RE-EVALUATION)

*Re-evaluated after Phase 1 design completion*

### Principle I: Async-First Architecture
- ✅ **PASS**: WikiPathwaysClient uses httpx async client (verified in data-model.md)
- ✅ **PASS**: No synchronous SDK dependencies (research.md confirms REST API only)
- ✅ **PASS**: All network I/O async (contract specifications confirm)

### Principle II: Fuzzy-to-Fact Resolution Protocol
- ✅ **PASS**: search_pathways (fuzzy) → PathwaySearchCandidate with WP:WPNNNNN IDs (contract)
- ✅ **PASS**: get_pathway (strict) validates `^WP:WP\d+$` format (contract)
- ✅ **PASS**: get_pathways_for_gene (fuzzy) → PathwaySearchCandidate results (contract)
- ✅ **PASS**: get_pathway_components (strict) validates ID format (contract)
- ✅ **PASS**: UNRESOLVED_ENTITY with recovery hints (all error contracts)

### Principle III: Schema Determinism
- ✅ **PASS**: All list tools use PaginationEnvelope (data-model.md)
- ✅ **PASS**: All errors use ErrorEnvelope (contracts/*.md)
- ✅ **PASS**: cross_references use 22-key registry (data-model.md §1, §4)
- ✅ **PASS**: Omit-if-null pattern enforced (data-model.md cross-reference mapping)

### Principle IV: Token Budgeting
- ✅ **PASS**: PathwaySearchCandidate ~20 tokens (data-model.md §2)
- ✅ **PASS**: Default page_size: 50, max: 100 (search_pathways.md contract)
- ✅ **PASS**: Slim mode on search tools (search_pathways.md, get_pathways_for_gene.md)

### Principle V: Specification-Before-Code
- ✅ **PASS**: Full specification workflow completed (this plan.md)
- ✅ **PASS**: Human gate enforced (implementation blocked until approval)

### Principle VI: Platform Skill Delegation
- ✅ **PASS**: Implementation will use `/scaffold-fastmcp` skill (noted in Constitution Check)

### Required Patterns
- ✅ **PASS**: Canonical Pagination Envelope (PaginationEnvelope in contracts)
- ✅ **PASS**: Canonical Error Envelope (ErrorEnvelope in contracts)
- ✅ **PASS**: Cross-reference regex validation (pathway.py model)
- ✅ **PASS**: Async httpx client (research.md confirms httpx usage)
- ✅ **PASS**: slim=True support (search_pathways.md, get_pathways_for_gene.md)
- ✅ **PASS**: Client-side rate limiting (research.md §4: 1 req/s + exponential backoff)

### Forbidden Patterns
- ✅ **PASS**: No synchronous blocking (httpx async client only)
- ✅ **PASS**: No hardcoded credentials (CC0 public API)
- ✅ **PASS**: No raw strings to strict tools (WP ID validation in contracts)
- ✅ **PASS**: No null cross-references (omit-if-null pattern in data-model.md)
- ✅ **PASS**: No deep JSON nesting (flattened Agentic Biolink schema)
- ✅ **PASS**: No unbounded concurrency (rate limiting enforced)

**POST-DESIGN GATE STATUS**: ✅ ALL CHECKS PASS - No violations, proceed to Phase 2 (tasks.md)

---

## Phase 2: Task Generation (NEXT STEP)

**Command**: `/speckit.tasks 012-wikipathways-mcp-server`

**Expected Output**: `tasks.md` with bounded, dependency-ordered implementation tasks

**Note**: Phase 2 task generation is NOT performed by `/speckit.plan`. It requires explicit `/speckit.tasks` command execution after human approval of this plan.
