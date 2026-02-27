# Implementation Plan: NCBI Entrez MCP Server

**Branch**: `009-entrez-mcp-server` | **Date**: 2026-01-01 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/009-entrez-mcp-server/spec.md`

## Summary

NCBI Entrez MCP Server provides gene discovery and literature link retrieval using NCBI E-utilities API. The server implements the Fuzzy-to-Fact protocol: fuzzy search (`esearch` + `esummary`) returns ranked gene candidates with NCBIGene CURIEs, strict lookup (`efetch` with XML parsing) retrieves complete gene records with cross-references to HGNC, Ensembl, UniProt, and RefSeq, and a literature tool (`elink`) returns PubMed IDs associated with genes for evidence gathering.

**Technical Approach**:
- Two-step API pattern: `esearch` (returns IDs) -> `efetch`/`esummary` (returns data)
- XML parsing using `defusedxml` for security (prevents XXE attacks)
- Rate limiting: 3 req/s default, 10 req/s with NCBI_API_KEY
- Cross-reference extraction from `Entrezgene_xref` XML elements

## Technical Context

**Language/Version**: Python 3.11+
**Primary Dependencies**: fastmcp, httpx, pydantic, defusedxml
**Storage**: Stateless (live queries to NCBI E-utilities API)
**Testing**: pytest-asyncio
**Target Platform**: Linux server (MCP protocol over stdio/SSE/HTTP)
**Project Type**: Single (MCP server package)
**Performance Goals**: SC-001 mandates 95% of queries < 2 seconds
**Constraints**:
- 3 req/s without API key, 10 req/s with NCBI_API_KEY
- XML response format (no JSON for efetch)
- Two-step esearch + efetch pattern required
**Scale/Scope**: Tier 4 genomics API, ~2.5M gene records in NCBI Gene database

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### Principle I: Async-First Architecture
- **Compliance**: PASS - Uses native `httpx` async client with connection pooling
- **Evidence**: All E-utilities calls use `async with httpx.AsyncClient()` pattern
- **Exception**: None required (XML parsing is CPU-bound but fast)

### Principle II: Fuzzy-to-Fact Resolution Protocol
- **Compliance**: PASS - Implements bi-modal workflow
- **Evidence**:
  - FR-001: `search_genes(query)` returns ranked GeneSearchCandidate with NCBIGene CURIEs
  - FR-007: `get_gene(entrez_id)` accepts ONLY `NCBIGene:\d+` CURIEs
  - FR-009: UNRESOLVED_ENTITY error when raw strings passed to strict tools

### Principle III: Schema Determinism
- **Compliance**: PASS - Uses Canonical Envelopes
- **Evidence**:
  - FR-005: PaginationEnvelope for search_genes
  - FR-015: ErrorEnvelope with code/message/recovery_hint/invalid_input
  - FR-012: cross_references using 22-key registry

### Principle IV: Token Budgeting
- **Compliance**: PASS - Page size and limit controls
- **Evidence**:
  - FR-004: page_size default 50 with cursor pagination
  - FR-008: `get_pubmed_links(limit=N)` parameter supported

### Principle V: Specification-Before-Code
- **Compliance**: PASS - Following SpecKit workflow
- **Evidence**: This plan generated from spec.md via `/speckit.plan`

### Principle VI: Platform Skill Delegation
- **Compliance**: PASS - Would use `/scaffold-fastmcp` for new server
- **Evidence**: Implementation follows established HGNC/UniProt patterns

### Rate Limiting Pattern (Constitution v1.1.0)
- **Compliance**: PASS - Client-side rate limiting required
- **Evidence**:
  - FR-019: 3 req/s default rate limit
  - FR-020: 10 req/s with NCBI_API_KEY
  - FR-021/FR-022: Exponential backoff and thundering herd prevention

**GATE STATUS**: PASS - All Constitution principles satisfied

## Project Structure

### Documentation (this feature)

```text
specs/009-entrez-mcp-server/
├── spec.md              # Feature specification (existing)
├── plan.md              # This file
├── research.md          # Phase 0: E-utilities API research, XML parsing
├── data-model.md        # Phase 1: Gene, GeneSearchCandidate, PubMedLink models
├── quickstart.md        # Phase 1: MCP server usage examples
├── contracts/           # Phase 1: Tool contracts (OpenAPI-style)
│   ├── search_genes.yaml
│   ├── get_gene.yaml
│   └── get_pubmed_links.yaml
└── checklists/
    └── requirements.md  # Constitution compliance checklist
```

### Source Code (repository root)

```text
src/lifesciences_mcp/
├── models/
│   ├── entrez.py            # NEW: EntrezGene, GeneSearchCandidate, PubMedLink models
│   ├── envelopes.py         # Existing: PaginationEnvelope, ErrorEnvelope
│   └── __init__.py          # Add exports
├── clients/
│   ├── base.py              # Existing: LifeSciencesClient
│   ├── entrez.py            # NEW: EntrezClient with XML parsing, rate limiting
│   └── __init__.py          # Add exports
└── servers/
    ├── entrez.py            # NEW: FastMCP server with 3 tools
    └── __init__.py          # Add exports

tests/
├── integration/
│   ├── test_entrez_api.py         # All 4 user stories integration tests
│   ├── test_entrez_performance.py # SC-001 performance validation
│   └── fixtures/
│       └── tier4_entrez_data.py   # Test fixtures
└── unit/
    ├── test_entrez_models.py      # Model validation tests
    └── test_entrez_client.py      # Client mocking tests
```

**Structure Decision**: Single project structure. Entrez is a standalone MCP server following the established pattern from HGNC/UniProt/ChEMBL/STRING implementations.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| Rate limit varies by API key | NCBI officially documents 3 req/s (no key) vs 10 req/s (with key) | Fixed rate would waste capacity or violate limits |

**Note**: This is not a Constitution violation but a documented exception for adaptive rate limiting based on environment configuration. The `NCBI_API_KEY` environment variable enables the higher rate.

## Phase 0: Research

### Research Tasks

1. **NCBI E-utilities API Endpoints**
   - `esearch.fcgi`: Search database, returns list of IDs (WebEnv/query_key for pagination)
   - `esummary.fcgi`: Get document summaries (lightweight, useful for search candidates)
   - `efetch.fcgi`: Get full records (XML format for Gene database)
   - `elink.fcgi`: Find related records across databases (gene -> pubmed)

2. **Two-Step esearch + efetch Pattern**
   - Step 1: `esearch` with query -> returns ID list + WebEnv/query_key
   - Step 2: `esummary` or `efetch` with IDs -> returns full data
   - Pagination via `retstart` (offset) and `retmax` (page size)

3. **XML Parsing Approach**
   - Use `defusedxml.ElementTree` for secure XML parsing (prevents XXE)
   - Parse `Entrezgene-Set` -> `Entrezgene` elements
   - Extract gene symbol, name, description, summary from nested elements

4. **Rate Limit Policy**
   - No API key: 3 requests/second
   - With NCBI_API_KEY: 10 requests/second
   - Key obtained free from: https://www.ncbi.nlm.nih.gov/account/settings/

5. **Cross-Reference Extraction**
   - Location: `Entrezgene_xref` -> `Dbtag` elements
   - Parse `Dbtag_db` (database name) and `Object-id_id` or `Object-id_str` (identifier)
   - Map to 22-key registry: HGNC, Ensembl, UniProt, RefSeq, OMIM, etc.

**Output**: [research.md](research.md)

## Phase 1: Design & Contracts

### Data Models

**Core Entities** (from spec FR-011, FR-013, FR-014):

1. **GeneSearchCandidate**
   - Fields: id (NCBIGene CURIE), symbol, name, description, organism, score
   - Purpose: Fuzzy search result for Phase 1 of Fuzzy-to-Fact

2. **EntrezGene**
   - Fields: id, symbol, name, description, summary, map_location, chromosome, aliases, organism, cross_references
   - Purpose: Complete gene record for Phase 2 of Fuzzy-to-Fact

3. **CrossReferences**
   - Fields: hgnc, ensembl_gene, uniprot, refseq, omim (all optional)
   - Purpose: Cross-database links following 22-key registry

4. **PubMedLink**
   - Fields: pmid (PubMed ID as string)
   - Purpose: Gene-to-literature association from elink

### API Contracts

**Tool 1: search_genes** (Phase 1 - Fuzzy)
- Input: query (str, min 2 chars), organism (str, optional, default all), page_size (int, 1-100), cursor (str, optional)
- Output: PaginationEnvelope[GeneSearchCandidate]
- Errors: AMBIGUOUS_QUERY (query too short)

**Tool 2: get_gene** (Phase 2 - Strict)
- Input: entrez_id (NCBIGene CURIE, pattern `^NCBIGene:\d+$`)
- Output: EntrezGene with cross_references
- Errors: UNRESOLVED_ENTITY (invalid CURIE), ENTITY_NOT_FOUND (no gene)

**Tool 3: get_pubmed_links** (Strict - Literature)
- Input: entrez_id (NCBIGene CURIE), limit (int, optional, default 10)
- Output: list[str] (PubMed IDs)
- Errors: UNRESOLVED_ENTITY, ENTITY_NOT_FOUND

**Output**:
- [data-model.md](data-model.md)
- [contracts/search_genes.yaml](contracts/search_genes.yaml)
- [contracts/get_gene.yaml](contracts/get_gene.yaml)
- [contracts/get_pubmed_links.yaml](contracts/get_pubmed_links.yaml)
- [quickstart.md](quickstart.md)

## Phase 2: Task Generation

**Note**: Task generation happens via `/speckit.tasks` command (separate from `/speckit.plan`).

Expected task categories:
1. Models implementation (entrez.py with 4 Pydantic models)
2. Client implementation (entrez.py with XML parsing, rate limiting)
3. Server implementation (3 FastMCP tools)
4. Integration tests (covering all 4 user stories)
5. Unit tests (model validation, client mocking)

## ADR Compliance Matrix

| ADR Section | Requirement | Implementation | Status |
|-------------|-------------|----------------|--------|
| ADR-001 §2 | Async httpx client | EntrezClient with httpx.AsyncClient | Planned |
| ADR-001 §3 | Fuzzy-to-Fact protocol | search_genes -> get_gene | Planned |
| ADR-001 §4 | Agentic Biolink schema | cross_references with 22-key registry | Planned |
| ADR-001 §8 | Canonical Envelopes | PaginationEnvelope, ErrorEnvelope | Planned |
| Constitution v1.1 | Rate limiting (10 req/s + backoff) | 3 or 10 req/s based on API key | Planned |

## CURIE Format

**Pattern**: `NCBIGene:\d+`
**Examples**:
- `NCBIGene:7157` (TP53)
- `NCBIGene:672` (BRCA1)
- `NCBIGene:1017` (CDK2)

**Validation Regex**: `^NCBIGene:\d+$`

## Implementation Notes

### XML Parsing Security

NCBI E-utilities return XML format for efetch on the Gene database. Use `defusedxml` to prevent XML External Entity (XXE) attacks:

```python
import defusedxml.ElementTree as ET

def parse_gene_xml(xml_content: str) -> list[dict]:
    root = ET.fromstring(xml_content)
    genes = []
    for gene_elem in root.findall(".//Entrezgene"):
        # Extract gene data safely
        ...
    return genes
```

### Rate Limiting Implementation

```python
import os

class EntrezClient(LifeSciencesClient):
    def __init__(self):
        self.api_key = os.getenv("NCBI_API_KEY")
        # 3 req/s without key, 10 req/s with key
        self.rate_limit_delay = 0.1 if self.api_key else 0.333
        # ... rest of initialization
```

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `NCBI_API_KEY` | None | Optional API key for 10 req/s rate limit |

**To obtain API key**: https://www.ncbi.nlm.nih.gov/account/settings/ (free registration)

## Next Steps

1. Complete Phase 0: Create research.md with full API documentation
2. Complete Phase 1: Create data-model.md, contracts/, and quickstart.md
3. Run `/speckit.tasks` to generate task list
4. Await human approval before Phase 2 implementation
