# Implementation Plan: UniProt MCP Server

**Branch**: `002-uniprot-mcp-server` | **Date**: 2025-12-22 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/002-uniprot-mcp-server/spec.md`

## Summary

Build a FastMCP server for UniProt protein data following the Fuzzy-to-Fact protocol. Agents can search for proteins using natural language queries, then retrieve complete protein records with cross-references to gene databases (HGNC, Ensembl), sequence databases (RefSeq), structural databases (PDB), and pathway databases (KEGG). The server implements the same architectural patterns as HGNC with improvements from code review findings.

## Technical Context

**Language/Version**: Python 3.11+ (per ADR-001 §2 and project pyproject.toml)
**Primary Dependencies**: FastMCP >=2.0, httpx >=0.27, pydantic >=2.0
**Storage**: N/A (stateless - all queries are live to UniProt REST API)
**Testing**: pytest >=8.0, pytest-asyncio >=0.23
**Target Platform**: Linux server (same as HGNC server)
**Project Type**: Single (extends existing src/lifesciences_mcp structure)
**Performance Goals**: <2s response for 95% of common protein queries (SC-001)
**Constraints**: 10 req/s rate limit (assumed, to be verified in research), <200ms p95 latency, 100 concurrent requests without degradation (SC-003)
**Scale/Scope**: Support top 1000 human proteins, 90% accuracy on unambiguous queries (SC-002), 80% cross-reference coverage for well-annotated proteins (SC-007)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence |
|-----------|--------|----------|
| **I. Async-First Architecture** | ✅ PASS | `UniProtClient` extends `LifeSciencesClient` using `httpx.AsyncClient` with connection pooling (per spec FR-014) |
| **II. Fuzzy-to-Fact Resolution** | ✅ PASS | Two-phase workflow: `search_proteins` (fuzzy) → `get_protein` (strict CURIE only). FR-004 explicitly rejects fuzzy queries in strict tools. |
| **III. Schema Determinism** | ✅ PASS | Uses canonical `PaginationEnvelope` (FR-007) and `ErrorEnvelope` (FR-009). Cross-references use 22-key registry with omit-if-null pattern (FR-006). |
| **IV. Token Budgeting** | ✅ PASS | Implements `slim=True` mode reducing token usage by 80% (FR-008, SC-005) |
| **V. Specification-Before-Code** | ✅ PASS | Currently in `/speckit.plan` phase with approved spec.md |
| **VI. Platform Skill Delegation** | ✅ PASS | Used `/scaffold-fastmcp uniprot` to generate server/client/test stubs |

**Summary**: No Constitution violations. All 6 core principles satisfied per specification requirements.

## Project Structure

### Documentation (this feature)

```text
specs/002-uniprot-mcp-server/
├── plan.md              # This file (/speckit.plan output)
├── research.md          # Phase 0 output (UniProt API research)
├── data-model.md        # Phase 1 output (Protein entity model)
├── quickstart.md        # Phase 1 output (developer guide)
├── contracts/           # Phase 1 output (OpenAPI spec)
│   └── uniprot.openapi.yaml
├── checklists/
│   └── requirements.md  # Spec quality validation (COMPLETE)
└── tasks.md             # Phase 2 output (/speckit.tasks - NOT yet created)
```

### Source Code (repository root)

```text
src/lifesciences_mcp/
├── __init__.py          # Updated: export UniProtClient
├── client.py            # Updated: add UniProtClient class
├── models/
│   ├── __init__.py
│   ├── envelopes.py     # Reuse: PaginationEnvelope, ErrorEnvelope
│   ├── gene.py          # Reuse: CrossReferences registry
│   └── protein.py       # NEW: Protein, ProteinSearchCandidate models
└── servers/
    ├── hgnc.py          # Existing
    └── uniprot.py       # NEW: search_proteins, get_protein tools

tests/
├── unit/
│   ├── test_models.py   # Existing gene tests
│   └── test_protein_models.py  # NEW: Protein model validation tests
├── integration/
│   ├── test_hgnc_api.py    # Existing
│   ├── test_uniprot_api.py # NEW: UniProt API integration tests
│   └── test_concurrency.py # Updated: add UniProt concurrency tests
└── conftest.py          # Updated: add UniProtClient fixtures
```

**Structure Decision**: Extends existing single-project structure. Reuses canonical envelopes and cross-reference models from HGNC implementation. New protein-specific models added to `models/protein.py` to avoid coupling with gene models.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

No violations to justify. All requirements align with Constitution principles.

---

## Phase 0: Research & Discovery

**Objective**: Resolve all NEEDS CLARIFICATION items and establish technical foundation

### Research Tasks

**R1: UniProt REST API Endpoints**
- **Question**: What are the exact API endpoints for search and lookup?
- **Research**: Explore https://rest.uniprot.org documentation
- **Deliverable**: Document base URL, search endpoint, fetch endpoint, authentication requirements

**R2: Search API Query Syntax**
- **Question**: How does UniProt search API work? What query syntax? How are results ranked?
- **Research**: Test queries for "p53", "insulin human", "kinase" to understand:
  - Query parameter format
  - Organism filtering options
  - Result ranking algorithm (if any)
  - Maximum results per query
- **Deliverable**: Query pattern examples and ranking behavior

**R3: Pagination Patterns**
- **Question**: Does UniProt support cursor-based or offset-based pagination?
- **Research**: Test large result sets to identify:
  - Pagination mechanism (Link headers, query params)
  - Cursor format if applicable
  - Maximum page size limits
- **Deliverable**: Pagination implementation strategy

**R4: Rate Limiting Policy**
- **Question**: What is UniProt's actual rate limit? Are there Retry-After headers?
- **Research**: Review UniProt documentation and test with burst requests
- **Deliverable**: Rate limit value (req/s), backoff strategy, header format

**R5: Cross-Reference Field Mappings**
- **Question**: What field names does UniProt use for cross-references (HGNC, Ensembl, RefSeq, PDB, etc)?
- **Research**: Fetch well-annotated proteins (e.g., P04637/TP53, P38398/BRCA1) and map response fields to our 22-key registry
- **Deliverable**: Field mapping table from UniProt JSON to CrossReferences model

**R6: CURIE Format Validation**
- **Question**: What is the exact format of UniProt accessions? How do we construct the CURIE?
- **Research**: Review UniProt ID format rules (e.g., P12345, A0A123B4C5)
- **Deliverable**: Regex pattern for UniProt CURIE validation

**R7: Error Response Formats**
- **Question**: What error codes and formats does UniProt API return?
- **Research**: Test invalid queries, rate limits, not-found cases
- **Deliverable**: Mapping from UniProt errors to our ErrorEnvelope codes

### Research Output Location

`specs/002-uniprot-mcp-server/research.md` containing:
- Decision: [chosen approach]
- Rationale: [why chosen]
- Alternatives considered: [what else evaluated]
- Code examples: [query patterns, response parsing]

---

## Phase 1: Design & Contracts

**Prerequisites**: research.md complete

### D1: Data Model Design

**Input**: FR-001 through FR-015, research findings on UniProt response format

**Output**: `data-model.md` containing:

#### Protein Entity

```python
class Protein(BaseModel):
    """Complete protein record from UniProt with Agentic Biolink cross-references."""

    id: str  # UniProt CURIE: "UniProtKB:P04637"
    accession: str  # Raw UniProt ID: "P04637"
    name: str  # Protein name
    full_name: str | None  # Recommended full name
    gene_names: list[str] | None  # Associated gene symbols
    organism: str  # Scientific name (e.g., "Homo sapiens")
    organism_id: int | None  # NCBI Taxonomy ID
    function: str | None  # Functional description
    sequence_length: int | None  # Amino acid count
    cross_references: CrossReferences  # Reuse from models/gene.py
```

#### ProteinSearchCandidate Entity

```python
class ProteinSearchCandidate(BaseModel):
    """Lightweight protein match for fuzzy search results."""

    id: str  # UniProt CURIE: "UniProtKB:P04637"
    name: str  # Protein name
    organism: str  # Scientific name
    gene_names: list[str] | None  # Associated genes
    score: float  # Relevance score (0.0-1.0)
```

**Validation Rules**:
- CURIE format: `^UniProtKB:[A-Z0-9]{6,10}$` (from research)
- Score bounds: 0.0 <= score <= 1.0
- Organism: non-empty string
- Cross-references: Reuse existing CrossReferences validators from gene.py

**State Transitions**: N/A (stateless queries)

### D2: API Contracts

**Input**: FR-001 through FR-015, User Stories 1-4

**Output**: `contracts/uniprot.openapi.yaml`

#### Endpoint 1: search_proteins

```yaml
/search_proteins:
  post:
    summary: Fuzzy search for proteins (Phase 1 of Fuzzy-to-Fact)
    parameters:
      - name: query
        in: body
        required: true
        schema:
          type: string
          minLength: 2
      - name: slim
        in: body
        schema:
          type: boolean
          default: false
      - name: cursor
        in: body
        schema:
          type: string
      - name: page_size
        in: body
        schema:
          type: integer
          minimum: 1
          maximum: 100
          default: 50
    responses:
      200:
        schema:
          oneOf:
            - $ref: '#/components/schemas/PaginationEnvelope[ProteinSearchCandidate]'
            - $ref: '#/components/schemas/ErrorEnvelope'
```

#### Endpoint 2: get_protein

```yaml
/get_protein:
  post:
    summary: Strict lookup by UniProt CURIE (Phase 2 of Fuzzy-to-Fact)
    parameters:
      - name: uniprot_id
        in: body
        required: true
        schema:
          type: string
          pattern: '^UniProtKB:[A-Z0-9]{6,10}$'
    responses:
      200:
        schema:
          oneOf:
            - $ref: '#/components/schemas/Protein'
            - $ref: '#/components/schemas/ErrorEnvelope'
```

### D3: Quickstart Guide

**Output**: `quickstart.md` containing:

```markdown
# UniProt MCP Server Quickstart

## Prerequisites
- Python 3.11+
- uv package manager

## Running the Server

\`\`\`bash
uv run fastmcp run src/lifesciences_mcp/servers/uniprot.py
\`\`\`

## Example Usage

### 1. Fuzzy Search (find protein candidates)

\`\`\`python
# Search for p53
result = await search_proteins(query="p53 tumor suppressor", page_size=10)

# Result:
# {
#   "items": [
#     {"id": "UniProtKB:P04637", "name": "Cellular tumor antigen p53", "organism": "Homo sapiens", "score": 0.95},
#     {"id": "UniProtKB:P02340", "name": "Cellular tumor antigen p53", "organism": "Mus musculus", "score": 0.85}
#   ],
#   "pagination": {"cursor": "...", "total_count": 156, "page_size": 10}
# }
\`\`\`

### 2. Strict Lookup (get complete protein record)

\`\`\`python
# Get human p53 details
protein = await get_protein(uniprot_id="UniProtKB:P04637")

# Result includes:
# - Full protein name and function
# - Gene names
# - Sequence length
# - cross_references.hgnc: "HGNC:11998" (TP53 gene)
# - cross_references.ensembl_gene: "ENSG00000141510"
# - cross_references.pdb: ["1TUP", "2OCJ", ...]
\`\`\`

### 3. Slim Mode (batch operations)

\`\`\`python
# Get 50 kinase candidates with minimal data
result = await search_proteins(query="kinase", slim=True, page_size=50)

# Each item is ~20 tokens instead of ~115 tokens
\`\`\`
```

---

## Phase 2: Tasks (Not Created by /speckit.plan)

**Note**: This phase is executed by `/speckit.tasks`, which generates a granular task list from this plan.

**Expected Task Categories**:
1. **Setup**: Create Protein models, update __init__.py exports
2. **Foundational**: Implement UniProtClient with rate limiting
3. **User Story 1**: Implement search_proteins with pagination
4. **User Story 2**: Implement get_protein with cross-references
5. **User Story 3**: Validate cross-reference mappings (HGNC, Ensembl, PDB, etc)
6. **User Story 4**: Implement error recovery hints
7. **Polish**: Write tests (unit + integration + concurrency), update documentation

---

## Implementation Notes

### Reuse from HGNC Implementation

**Reuse without changes**:
- `models/envelopes.py` - PaginationEnvelope, ErrorEnvelope
- `models/gene.py` - CrossReferences registry (22 keys)
- `client.py` - LifeSciencesClient base class

**Extend/customize**:
- `client.py` - Add UniProtClient with UniProt-specific rate limiting
- `__init__.py` - Export UniProtClient and Protein models
- `tests/conftest.py` - Add UniProtClient fixtures

### Lessons Applied from HGNC Code Review

1. **Concurrency Safety** (FR-14): Implement rate limiter with `_get` inside lock from the start (no race condition)
2. **Resource Cleanup** (FR-15): Use module-level singleton pattern (see ADR-004) - FastMCP handles lifecycle internally, `@mcp.on_event("shutdown")` does not exist
3. **Exponential Backoff** (FR-12): Use 2^attempt backoff, not linear
4. **Thundering Herd Prevention**: Re-check time boundary after acquiring lock in retry loop

### Known Challenges

**Challenge 1: Organism Disambiguation**
- Spec requires "human proteins prioritized" (US1, acceptance scenario 4)
- Solution: Add organism filter to search queries or post-rank results

**Challenge 2: Isoform Handling**
- Edge case: "What happens when a protein has multiple isoforms?" (spec edge cases)
- Solution: Return canonical isoform, document isoform support as future enhancement

**Challenge 3: Cross-Reference Availability**
- Not all proteins have all 22 cross-reference types
- Solution: Omit keys entirely (never null) per Constitution III

---

## Validation Criteria

Before marking this plan complete, verify:

- [x] All Constitution principles addressed
- [x] Research tasks defined for all NEEDS CLARIFICATION items
- [x] Data models specified with validation rules
- [x] API contracts defined for all user stories
- [x] Quickstart guide created
- [x] Reuse strategy documented
- [x] Code review lessons incorporated
- [x] Known challenges identified with solutions

**Status**: ✅ Plan ready for Phase 0 research execution
