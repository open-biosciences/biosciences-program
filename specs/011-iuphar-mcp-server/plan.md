# Implementation Plan: IUPHAR/GtoPdb MCP Server

**Branch**: `011-iuphar-mcp-server` | **Date**: 2026-01-01 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/011-iuphar-mcp-server/spec.md`

## Summary

Build a FastMCP server for IUPHAR/GtoPdb (Guide to PHARMACOLOGY) pharmacological data following the Fuzzy-to-Fact protocol. This is a **dual-entity server** supporting both ligands (drugs, chemicals) and targets (receptors, enzymes, ion channels, transporters). Agents can search for pharmacological entities using natural language queries, then retrieve complete records with cross-references to ChEMBL, DrugBank, PubChem (ligands) and UniProt, HGNC, Ensembl (targets). The server implements the same architectural patterns as HGNC/UniProt with a conservative 1 req/s rate limit appropriate for this community resource.

## Technical Context

**Language/Version**: Python 3.11+ (per ADR-001 Section 2 and project pyproject.toml)
**Primary Dependencies**: FastMCP >=2.0, httpx >=0.27, pydantic >=2.0
**Storage**: N/A (stateless - all queries are live to GtoPdb REST API)
**Testing**: pytest >=8.0, pytest-asyncio >=0.23
**Target Platform**: Linux server (same as HGNC/UniProt servers)
**Project Type**: Single (extends existing src/lifesciences_mcp structure)
**Performance Goals**: <2s response for 95% of common pharmacology queries (SC-001)
**Constraints**: 1 req/s rate limit (conservative for community resource), <500ms p95 latency for cached responses
**Scale/Scope**: Support 12,000+ ligands and 3,000+ targets from GtoPdb

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence |
|-----------|--------|----------|
| **I. Async-First Architecture** | PASS | `IUPHARClient` extends `LifeSciencesClient` using `httpx.AsyncClient` with connection pooling. GtoPdb REST API is native async-compatible. |
| **II. Fuzzy-to-Fact Resolution** | PASS | Four-tool workflow: `search_ligands`/`search_targets` (fuzzy) -> `get_ligand`/`get_target` (strict CURIE only). FR-005/FR-012 explicitly reject raw strings in strict tools. |
| **III. Schema Determinism** | PASS | Uses canonical `PaginationEnvelope` (FR-004/FR-011) and `ErrorEnvelope` (FR-019). Cross-references use 22-key registry with omit-if-null pattern (FR-017/FR-018). |
| **IV. Token Budgeting** | PASS | Implements `slim=True` mode for search results. LigandSearchCandidate/TargetSearchCandidate provide ~20 token/entity output. |
| **V. Specification-Before-Code** | PASS | Currently in `/speckit.plan` phase with approved spec.md |
| **VI. Platform Skill Delegation** | PASS | Will use project patterns for server scaffolding consistent with HGNC/UniProt |

**Summary**: No Constitution violations. All 6 core principles satisfied per specification requirements.

## Project Structure

### Documentation (this feature)

```text
specs/011-iuphar-mcp-server/
├── plan.md              # This file (/speckit.plan output)
├── research.md          # Phase 0 output (GtoPdb API research)
├── data-model.md        # Phase 1 output (Ligand/Target entity models)
├── quickstart.md        # Phase 1 output (developer guide)
├── contracts/           # Phase 1 output
│   └── iuphar.openapi.yaml
└── tasks.md             # Phase 2 output (/speckit.tasks - NOT yet created)
```

### Source Code (repository root)

```text
src/lifesciences_mcp/
├── __init__.py          # Updated: export IUPHARClient
├── clients/
│   ├── base.py          # Reuse: LifeSciencesClient base class
│   ├── hgnc.py          # Existing
│   ├── uniprot.py       # Existing
│   └── iuphar.py        # NEW: IUPHARClient with 1 req/s rate limiting
├── models/
│   ├── __init__.py      # Updated: export Ligand, Target models
│   ├── envelopes.py     # Reuse: PaginationEnvelope, ErrorEnvelope
│   ├── gene.py          # Reuse: CrossReferences registry
│   ├── protein.py       # Existing
│   └── pharmacology.py  # NEW: Ligand, Target, LigandSearchCandidate, TargetSearchCandidate
└── servers/
    ├── hgnc.py          # Existing
    ├── uniprot.py       # Existing
    └── iuphar.py        # NEW: search_ligands, get_ligand, search_targets, get_target tools

tests/
├── unit/
│   ├── test_models.py              # Existing gene tests
│   └── test_pharmacology_models.py # NEW: Ligand/Target model validation tests
├── integration/
│   ├── test_hgnc_api.py            # Existing
│   ├── test_uniprot_api.py         # Existing
│   └── test_iuphar_api.py          # NEW: GtoPdb API integration tests
└── conftest.py                     # Updated: add IUPHARClient fixtures
```

**Structure Decision**: Extends existing single-project structure. Reuses canonical envelopes and cross-reference models from HGNC implementation. New pharmacology-specific models added to `models/pharmacology.py` to handle the dual-entity design (both ligands and targets).

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

No violations to justify. All requirements align with Constitution principles.

---

## Phase 0: Research & Discovery

**Objective**: Resolve all technical questions and establish implementation foundation

### Research Tasks

**R1: GtoPdb REST API Base Endpoints**
- **Question**: What are the exact API endpoints for ligand and target operations?
- **Research**: Tested against https://www.guidetopharmacology.org/services/
- **Findings**:
  - **Base URL**: `https://www.guidetopharmacology.org/services/`
  - **Ligand List**: `GET /ligands` with optional query params
  - **Ligand Detail**: `GET /ligands/{ligandId}`
  - **Ligand DB Links**: `GET /ligands/{ligandId}/databaseLinks`
  - **Ligand Synonyms**: `GET /ligands/{ligandId}/synonyms`
  - **Target List**: `GET /targets` with optional query params
  - **Target Detail**: `GET /targets/{targetId}`
  - **Target DB Links**: `GET /targets/{targetId}/databaseLinks`
  - **Authentication**: None required (open access)
  - **Data Format**: JSON only

**R2: Search API Query Parameters**
- **Question**: How does GtoPdb search work? What parameters are available?
- **Research**: Tested query parameters on `/ligands` and `/targets`
- **Findings**:
  - **Ligand Search Parameters**:
    - `name` - Search by ligand name (fuzzy substring match)
    - `type` - Filter by type (Synthetic organic, Metabolite, Peptide, Antibody, etc.)
    - `accession` + `database` - Search by external database ID (default: PubChemCID)
    - `approved` - Boolean filter for approved drugs
    - `inchikey` - Structure search by InChIKey
  - **Target Search Parameters**:
    - `name` - Search by target name (fuzzy substring match)
    - `type` - Filter by class (GPCR, NHR, LGIC, VGIC, OtherIC, Enzyme, CatalyticReceptor, Transporter, OtherProtein)
    - `geneSymbol` - Search by human/mouse/rat gene symbol
    - `accession` + `database` - Search by external ID (default: UniProt)
  - **Result Ranking**: Results are returned in database order (no explicit ranking algorithm)

**R3: Pagination Patterns**
- **Question**: Does GtoPdb support pagination?
- **Research**: Tested large result sets
- **Findings**:
  - **No Server-Side Pagination**: GtoPdb API returns all matching results in a single response
  - **Implementation Strategy**: Use client-side pagination with offset/limit slicing (same as HGNC)
  - **Cursor Format**: Base64-encoded JSON with offset field
  - **Maximum Results**: No explicit limit; well-formed queries return manageable result sets

**R4: Rate Limiting Policy**
- **Question**: What is GtoPdb's rate limit?
- **Research**: No official documentation found; conservative approach recommended
- **Findings**:
  - **Official Limit**: Not documented
  - **Recommended Limit**: 1 req/s (conservative for community resource)
  - **Error Codes**: 204 No Content, 303 Redirect, 404 Not Found
  - **No Retry-After Header**: Implement exponential backoff without header guidance
  - **Usage Tracking**: "Basic anonymous tracking mechanism" mentioned (respect resource limits)

**R5: Ligand Cross-Reference Mappings**
- **Question**: What database links are available for ligands?
- **Research**: Fetched `/ligands/2713/databaseLinks` (ibuprofen)
- **Findings**:
  | GtoPdb Database Name | Our Registry Key | Example ID |
  |---------------------|------------------|------------|
  | ChEMBL Ligand | chembl | CHEMBL521 |
  | DrugBank Ligand | drugbank | DB01050 |
  | PubChem CID | pubchem_compound | 3672 |
  | ChEBI | chebi | CHEBI:5855 |
  | BindingDB Ligand | - (not in registry) | 50009859 |
  | CAS Registry No. | - (not in registry) | 15687-27-1 |
  | DrugCentral Ligand | - (not in registry) | 1407 |
  | Wikipedia | - (not in registry) | Ibuprofen |

**R6: Target Cross-Reference Mappings**
- **Question**: What database links are available for targets?
- **Research**: Fetched `/targets/215/databaseLinks` (D2 receptor)
- **Findings**:
  | GtoPdb Database Name | Our Registry Key | Example ID (Human) |
  |---------------------|------------------|-------------------|
  | UniProtKB | uniprot | P14416 |
  | Ensembl Gene ID | ensembl_gene | ENSG00000149295 |
  | Entrez Gene | entrez | 1813 |
  | RefSeq Nucleotide | refseq | NM_000795 |
  | ChEMBL Target | chembl | CHEMBL217 |
  | OMIM | omim | 126450 |
  | Orphanet | orphanet | ORPHA121177 |
  | DrugBank | drugbank | P14416 (target ID) |

  **Note**: Target database links include multi-species data (human, mouse, rat). We prioritize human data.

**R7: CURIE Format Validation**
- **Question**: What format do IUPHAR IDs use?
- **Research**: Analyzed ligandId and targetId fields
- **Findings**:
  - **Ligand CURIE**: `IUPHAR:NNNNN` where NNNNN is integer (e.g., `IUPHAR:2713` for ibuprofen)
  - **Target CURIE**: `IUPHAR:NNNNN` where NNNNN is integer (e.g., `IUPHAR:215` for D2 receptor)
  - **Validation Regex**: `^IUPHAR:\d+$`
  - **Note**: Both ligands and targets use the same CURIE prefix. Tools must disambiguate by context.

**R8: Response Structure Analysis**
- **Question**: What fields are in ligand and target responses?
- **Research**: Analyzed sample responses
- **Findings**:
  - **Ligand Fields** (from `/ligands/{id}`):
    - `ligandId` (int): Unique identifier
    - `name` (string): Display name
    - `type` (string): Classification (Synthetic organic, Metabolite, Peptide, etc.)
    - `abbreviation` (string): Short name
    - `inn` (string): International Nonproprietary Name
    - `approvalSource` (string): Regulatory approval info
    - `approved` (boolean): Approval status
    - `whoEssential` (boolean): WHO essential medicine
    - `withdrawn` (boolean): Withdrawal status
    - `antibacterial`, `immuno`, `malaria`, `labelled`, `radioactive` (boolean): Category flags
  - **Target Fields** (from `/targets/{id}`):
    - `targetId` (int): Unique identifier
    - `name` (string): Display name (may contain HTML like `<sub>`)
    - `type` (string): Classification (GPCR, Enzyme, LGIC, etc.)
    - `familyIds` (array[int]): Parent family IDs
    - `subunitIds` (array[int]): Component subunits
    - `complexIds` (array[int]): Associated complexes
  - **Synonym Fields** (from `/ligands/{id}/synonyms`):
    - `name` (string): Synonym text
    - `refs` (array): Reference citations

**R9: Error Response Formats**
- **Question**: How does GtoPdb handle errors?
- **Research**: Tested invalid queries
- **Findings**:
  - **204 No Content**: Empty result set (not an error)
  - **303 See Other**: Redirect (follow Location header)
  - **404 Not Found**: Invalid endpoint or ID
  - **No structured error body**: Errors return empty or HTML content
  - **Mapping to ErrorEnvelope**:
    - 404 with valid CURIE format -> ENTITY_NOT_FOUND
    - Invalid CURIE format -> UNRESOLVED_ENTITY (client-side validation)
    - Network timeout -> UPSTREAM_ERROR
    - Empty results -> Return empty PaginationEnvelope (not an error)

### Research Output Location

Research findings documented inline above. Key decisions:
- **Rate Limit**: 1 req/s (conservative)
- **CURIE Format**: `IUPHAR:\d+` for both ligands and targets
- **Pagination**: Client-side (GtoPdb returns all results)
- **Cross-References**: Ligands -> ChEMBL, DrugBank, PubChem; Targets -> UniProt, Ensembl, HGNC

---

## Phase 1: Design & Contracts

**Prerequisites**: research.md complete (see Phase 0 above)

### D1: Data Model Design

**Input**: FR-001 through FR-018, research findings on GtoPdb response format

**Output**: `data-model.md` containing:

#### Ligand Entity

```python
class Ligand(BaseModel):
    """Complete ligand record from GtoPdb with Agentic Biolink cross-references.

    Represents pharmacological compounds including drugs, metabolites, peptides,
    and antibodies from the IUPHAR/BPS Guide to PHARMACOLOGY.
    """

    id: str  # IUPHAR CURIE: "IUPHAR:2713"
    ligand_id: int  # Raw GtoPdb ID: 2713
    name: str  # Display name: "ibuprofen"
    approved_name: str | None  # INN if available
    type: str  # Classification: "Synthetic organic", "Peptide", "Antibody", etc.
    abbreviation: str | None  # Short name if available
    approved: bool  # Regulatory approval status
    approval_source: str | None  # e.g., "FDA (1974), EMA (2004)"
    who_essential: bool | None  # WHO essential medicine list
    withdrawn: bool | None  # Withdrawal status
    synonyms: list[str] | None  # Brand names and alternative names
    cross_references: CrossReferences  # ChEMBL, DrugBank, PubChem mappings
```

#### LigandSearchCandidate Entity

```python
class LigandSearchCandidate(BaseModel):
    """Lightweight ligand match for fuzzy search results (~20 tokens/entity)."""

    id: str  # IUPHAR CURIE: "IUPHAR:2713"
    name: str  # Display name
    type: str  # Classification
    approved: bool  # Quick filter for approved drugs
    score: float  # Relevance score (0.0-1.0)
```

#### Target Entity

```python
class Target(BaseModel):
    """Complete target record from GtoPdb with Agentic Biolink cross-references.

    Represents pharmacological targets including GPCRs, ion channels, enzymes,
    transporters, and other proteins from the IUPHAR/BPS Guide to PHARMACOLOGY.
    """

    id: str  # IUPHAR CURIE: "IUPHAR:215"
    target_id: int  # Raw GtoPdb ID: 215
    name: str  # Display name (HTML stripped): "D2 receptor"
    target_family: str  # Classification: "GPCR", "Enzyme", "LGIC", etc.
    family_ids: list[int] | None  # Parent family IDs for hierarchy
    species: str  # Primary species (default: "Homo sapiens")
    gene_symbol: str | None  # Human gene symbol if available
    cross_references: CrossReferences  # UniProt, Ensembl, HGNC mappings
```

#### TargetSearchCandidate Entity

```python
class TargetSearchCandidate(BaseModel):
    """Lightweight target match for fuzzy search results (~20 tokens/entity)."""

    id: str  # IUPHAR CURIE: "IUPHAR:215"
    name: str  # Display name
    family: str  # Target family/class
    type: str  # Full classification
    score: float  # Relevance score (0.0-1.0)
```

**Validation Rules**:
- CURIE format: `^IUPHAR:\d+$`
- Score bounds: 0.0 <= score <= 1.0
- Type values: Validated against known GtoPdb classifications
- Cross-references: Reuse existing CrossReferences validators from gene.py

**State Transitions**: N/A (stateless queries)

### D2: API Contracts

**Input**: FR-001 through FR-021, User Stories 1-5

**Output**: `contracts/iuphar.openapi.yaml`

#### Endpoint 1: search_ligands

```yaml
/search_ligands:
  post:
    summary: Fuzzy search for pharmacological ligands (Phase 1 of Fuzzy-to-Fact)
    description: |
      Search the IUPHAR/BPS Guide to PHARMACOLOGY database for ligands
      (drugs, chemicals, metabolites, peptides, antibodies) using natural
      language queries. Returns ranked candidates for resolution.
    parameters:
      - name: query
        in: body
        required: true
        description: Search term (ligand name, synonym, or drug class)
        schema:
          type: string
          minLength: 2
      - name: type_filter
        in: body
        required: false
        description: Filter by ligand type
        schema:
          type: string
          enum: [Synthetic organic, Metabolite, Natural product, Endogenous peptide, Peptide, Antibody, Inorganic]
      - name: approved_only
        in: body
        required: false
        description: Return only approved drugs
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
            - $ref: '#/components/schemas/PaginationEnvelope[LigandSearchCandidate]'
            - $ref: '#/components/schemas/ErrorEnvelope'
```

#### Endpoint 2: get_ligand

```yaml
/get_ligand:
  post:
    summary: Strict lookup by IUPHAR ligand CURIE (Phase 2 of Fuzzy-to-Fact)
    description: |
      Retrieve complete ligand record using a resolved IUPHAR identifier.
      Returns full Agentic Biolink entity with cross-references to ChEMBL,
      DrugBank, and PubChem.
    parameters:
      - name: iuphar_id
        in: body
        required: true
        description: IUPHAR ligand CURIE (e.g., "IUPHAR:2713")
        schema:
          type: string
          pattern: '^IUPHAR:\d+$'
    responses:
      200:
        schema:
          oneOf:
            - $ref: '#/components/schemas/Ligand'
            - $ref: '#/components/schemas/ErrorEnvelope'
```

#### Endpoint 3: search_targets

```yaml
/search_targets:
  post:
    summary: Fuzzy search for pharmacological targets (Phase 1 of Fuzzy-to-Fact)
    description: |
      Search the IUPHAR/BPS Guide to PHARMACOLOGY database for targets
      (receptors, ion channels, enzymes, transporters) using natural
      language queries. Returns ranked candidates for resolution.
    parameters:
      - name: query
        in: body
        required: true
        description: Search term (target name, gene symbol, or family)
        schema:
          type: string
          minLength: 2
      - name: type_filter
        in: body
        required: false
        description: Filter by target type
        schema:
          type: string
          enum: [GPCR, NHR, LGIC, VGIC, OtherIC, Enzyme, CatalyticReceptor, Transporter, OtherProtein]
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
            - $ref: '#/components/schemas/PaginationEnvelope[TargetSearchCandidate]'
            - $ref: '#/components/schemas/ErrorEnvelope'
```

#### Endpoint 4: get_target

```yaml
/get_target:
  post:
    summary: Strict lookup by IUPHAR target CURIE (Phase 2 of Fuzzy-to-Fact)
    description: |
      Retrieve complete target record using a resolved IUPHAR identifier.
      Returns full Agentic Biolink entity with cross-references to UniProt,
      Ensembl, and HGNC.
    parameters:
      - name: iuphar_id
        in: body
        required: true
        description: IUPHAR target CURIE (e.g., "IUPHAR:215")
        schema:
          type: string
          pattern: '^IUPHAR:\d+$'
    responses:
      200:
        schema:
          oneOf:
            - $ref: '#/components/schemas/Target'
            - $ref: '#/components/schemas/ErrorEnvelope'
```

### D3: Quickstart Guide

**Output**: `quickstart.md`

See Phase 1 artifacts section below.

---

## Phase 2: Tasks (Not Created by /speckit.plan)

**Note**: This phase is executed by `/speckit.tasks`, which generates a granular task list from this plan.

**Expected Task Categories**:
1. **Setup**: Create Ligand/Target models, update __init__.py exports
2. **Foundational**: Implement IUPHARClient with 1 req/s rate limiting
3. **User Story 1**: Implement search_ligands with pagination
4. **User Story 2**: Implement get_ligand with cross-references
5. **User Story 3**: Implement search_targets with pagination
6. **User Story 4**: Implement get_target with cross-references
7. **User Story 5**: Implement error recovery hints for both entity types
8. **Polish**: Write tests (unit + integration), update documentation

---

## Implementation Notes

### Reuse from HGNC/UniProt Implementation

**Reuse without changes**:
- `models/envelopes.py` - PaginationEnvelope, ErrorEnvelope
- `models/gene.py` - CrossReferences registry (22 keys)
- `clients/base.py` - LifeSciencesClient base class

**Extend/customize**:
- `clients/iuphar.py` - IUPHARClient with 1 req/s rate limiting (stricter than HGNC/UniProt 10 req/s)
- `models/pharmacology.py` - New models for dual-entity design
- `__init__.py` - Export IUPHARClient and pharmacology models
- `tests/conftest.py` - Add IUPHARClient fixtures

### Lessons Applied from HGNC/UniProt Code Review

1. **Concurrency Safety** (FR-22): Implement rate limiter with `_get` inside lock from the start (no race condition)
2. **Resource Cleanup** (FR-23): Use module-level singleton pattern (see ADR-004) - FastMCP handles lifecycle internally
3. **Exponential Backoff** (FR-23): Use 2^attempt backoff, not linear
4. **Thundering Herd Prevention** (FR-24): Re-check time boundary after acquiring lock in retry loop
5. **Conservative Rate Limit**: 1 req/s (vs 10 req/s for HGNC/UniProt) for community resource respect

### Known Challenges

**Challenge 1: Dual-Entity CURIE Disambiguation**
- Both ligands and targets use `IUPHAR:NNNNN` format
- **Solution**: Tools explicitly handle entity type. `get_ligand` queries `/ligands/{id}`, `get_target` queries `/targets/{id}`. If ID not found in expected entity type, return ENTITY_NOT_FOUND with recovery_hint suggesting the other entity type.

**Challenge 2: HTML in Target Names**
- Target names contain HTML subscripts: `"D<sub>2</sub> receptor"`
- **Solution**: Strip HTML tags in response parsing. Use regex `<[^>]+>` or html.unescape + tag stripping.

**Challenge 3: Multi-Species Target Data**
- Database links include human, mouse, and rat data
- **Solution**: Prioritize human (species filter). Store only human cross-references in cross_references object.

**Challenge 4: No Server-Side Pagination**
- GtoPdb returns all results at once
- **Solution**: Implement client-side pagination with base64-encoded cursor containing offset (same as HGNC pattern).

**Challenge 5: Relevance Scoring**
- GtoPdb doesn't provide explicit relevance scores
- **Solution**: Calculate position-based scores with decay (1.0 - i * 0.05), clamped to 0.1 minimum.

---

## Validation Criteria

Before marking this plan complete, verify:

- [x] All Constitution principles addressed
- [x] Research tasks defined for all technical questions
- [x] Data models specified with validation rules
- [x] API contracts defined for all user stories (4 tools)
- [x] Quickstart guide planned
- [x] Reuse strategy documented
- [x] Code review lessons incorporated (rate limiting, backoff, thundering herd)
- [x] Known challenges identified with solutions

**Status**: Plan ready for Phase 0/Phase 1 artifact creation
