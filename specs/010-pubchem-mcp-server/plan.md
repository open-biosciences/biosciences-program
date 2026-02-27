# Implementation Plan: PubChem MCP Server

**Branch**: `010-pubchem-mcp-server` | **Date**: 2026-01-01 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/010-pubchem-mcp-server/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

The PubChem MCP Server provides AI agents with access to chemical compound data through PubChem's PUG REST API. The server implements the Fuzzy-to-Fact protocol, allowing agents to search for compounds by name, synonym, or identifier, then retrieve complete compound records including SMILES, InChI, InChIKey, molecular properties, and cross-references to ChEMBL, DrugBank, and other databases.

**Technical Approach**: Native async `httpx` client with dual rate limiting (5 req/s + 400 req/min), cursor-based pagination, and cross-reference extraction from PubChem's synonyms and xrefs endpoints.

## Technical Context

**Language/Version**: Python 3.11+
**Primary Dependencies**: FastMCP >=2.0, httpx >=0.27, pydantic >=2.0
**Storage**: N/A (stateless - all queries are live to PubChem PUG REST API)
**Testing**: pytest with integration tests against live PubChem API
**Target Platform**: Linux server, macOS development
**Project Type**: Single package (lifesciences-research)
**Performance Goals**: <2s for 95% of queries (SC-001)
**Constraints**: 5 req/s, 400 req/min rate limits, 30s timeout per request
**Scale/Scope**: Public chemical database (~118M compounds)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence/Notes |
|-----------|--------|----------------|
| I. Async-First Architecture | PASS | Native `httpx` async client with connection pooling |
| II. Fuzzy-to-Fact Protocol | PASS | `search_compounds` (fuzzy) -> `get_compound` (strict with CURIE) |
| III. Schema Determinism | PASS | PaginationEnvelope, ErrorEnvelope, 22-key cross-references |
| IV. Token Budgeting | PASS | `slim=True` parameter on all tools, page_size default 50 |
| V. Specification-Before-Code | PASS | Full SpecKit workflow (spec -> plan -> tasks -> implement) |
| VI. Platform Skill Delegation | PASS | Using existing client patterns from LifeSciencesClient base |
| Rate Limiting (Constitution v1.1.0) | PASS | Dual rate limiting: 5 req/s + 400 req/min with backoff |
| Unbounded Concurrency Prevention | PASS | `_rate_limited_get()` pattern from base client |

## Project Structure

### Documentation (this feature)

```text
specs/010-pubchem-mcp-server/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
│   ├── search_compounds.json
│   └── get_compound.json
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)

```text
src/lifesciences_mcp/
├── __init__.py          # Package exports (add PubChemClient)
├── clients/
│   ├── base.py          # LifeSciencesClient base class (reuse)
│   └── pubchem.py       # NEW: PubChemClient (async httpx)
├── models/
│   ├── __init__.py      # Model exports (add PubChem models)
│   ├── envelopes.py     # Reuse PaginationEnvelope, ErrorEnvelope
│   └── pubchem_compound.py # NEW: PubChemCompound, PubChemSearchCandidate
└── servers/
    └── pubchem.py       # NEW: PubChem MCP server (search_compounds, get_compound)

tests/
├── integration/
│   └── test_pubchem_api.py  # NEW: Integration tests against live API
└── unit/
    ├── test_pubchem_client.py  # NEW: Unit tests with mocked responses
    └── test_pubchem_models.py  # NEW: Model validation tests
```

**Structure Decision**: Single package structure following existing pattern from ChEMBL/UniProt servers. New files created in `clients/pubchem.py`, `models/pubchem_compound.py`, and `servers/pubchem.py`.

## Phase 0: Research Summary

*See [research.md](./research.md) for complete research findings.*

### R1: PUG REST API Structure

**Decision**: Use PubChem PUG REST API (`https://pubchem.ncbi.nlm.nih.gov/rest/pug/`)

**URL Structure**:
```
{prolog}/{domain}/{namespace}/{identifiers}/{operation}/{output}
```

**Key Endpoints**:
| Endpoint | Purpose | Example |
|----------|---------|---------|
| `/compound/name/{name}/cids/JSON` | Search by name -> get CIDs | `/compound/name/aspirin/cids/JSON` |
| `/compound/cid/{cid}/property/{props}/JSON` | Get properties by CID | `/compound/cid/2244/property/MolecularFormula,CanonicalSMILES/JSON` |
| `/compound/cid/{cid}/synonyms/JSON` | Get all synonyms | `/compound/cid/2244/synonyms/JSON` |
| `/compound/cid/{cid}/xrefs/RegistryID/JSON` | Get cross-references | `/compound/cid/2244/xrefs/RegistryID/JSON` |

### R2: Rate Limiting Strategy

**Decision**: Dual rate limiting with exponential backoff

| Limit | Value | Implementation |
|-------|-------|----------------|
| Per-second | 5 req/s | AsyncLimiter with 200ms minimum interval |
| Per-minute | 400 req/min | Rolling window counter |
| Backoff | Exponential | Base 1s, max 60s, triggered on HTTP 503 |
| Thundering herd | Prevented | Lock + re-check timing after acquire |

**HTTP Headers**: PubChem returns throttling info in headers:
- `X-Throttling-Control`: Current system load state
- Automatic 503 when limits exceeded

### R3: Cross-Reference Mapping

**Decision**: Extract cross-references from synonyms and xrefs endpoints

| PubChem Source | Our 22-Key Registry | Extraction Method |
|----------------|---------------------|-------------------|
| Synonyms containing "CHEMBL" | `chembl` | Pattern: `CHEMBL\d+` |
| Synonyms containing "DB" | `drugbank` | Pattern: `DB\d{5}` |
| xrefs/RegistryID with "UniProt" | `uniprot` | Direct mapping |
| xrefs/RegistryID with "PDB" | `pdb` | Direct mapping |
| xrefs/RegistryID with "ChEBI" | `chebi` | Direct mapping |

**Note**: PubChem cross-references are limited. Primary mapping is via synonyms containing ChEMBL/DrugBank IDs.

### R4: CURIE Format

**Decision**: Use `PubChem:CID{number}` format

**Regex Pattern**: `^PubChem:CID\d+$`

**Validation Examples**:
| Input | Valid? | Reason |
|-------|--------|--------|
| `PubChem:CID2244` | PASS | Correct format |
| `PubChem:CID123456789` | PASS | Large CIDs valid |
| `CID2244` | FAIL | Missing prefix |
| `2244` | FAIL | Missing CURIE |
| `PubChem:2244` | FAIL | Missing CID |

### R5: Property Retrieval

**Decision**: Single API call for all properties

**Properties to retrieve**:
```
MolecularFormula,MolecularWeight,CanonicalSMILES,IsomericSMILES,InChI,InChIKey,IUPACName,Title
```

**Response Structure**:
```json
{
  "PropertyTable": {
    "Properties": [
      {
        "CID": 2244,
        "MolecularFormula": "C9H8O4",
        "MolecularWeight": 180.16,
        "CanonicalSMILES": "CC(=O)OC1=CC=CC=C1C(=O)O",
        "IsomericSMILES": "CC(=O)OC1=CC=CC=C1C(=O)O",
        "InChI": "InChI=1S/C9H8O4/...",
        "InChIKey": "BSYNRYMUTXBXSQ-UHFFFAOYSA-N",
        "IUPACName": "2-acetoxybenzoic acid",
        "Title": "Aspirin"
      }
    ]
  }
}
```

### R6: Error Handling

**Decision**: Map HTTP status to canonical error codes

| HTTP Status | PubChem Meaning | Our Error Code | Recovery Hint |
|-------------|-----------------|----------------|---------------|
| 400 | Bad request / standardization failed | `UNRESOLVED_ENTITY` | Check CURIE format |
| 404 | CID not found | `ENTITY_NOT_FOUND` | Verify CID exists |
| 503 | Rate limit / server busy | `RATE_LIMITED` | Retry with backoff |
| 500/502 | Server error | `UPSTREAM_ERROR` | Retry in 60s |
| Timeout | Request took >30s | `UPSTREAM_ERROR` | Simplify query |

### R7: Synonyms for Fuzzy Search

**Decision**: Use synonyms endpoint for name resolution

PubChem's `/compound/name/{name}/cids/JSON` returns CIDs matching the name.
For fuzzy ranking, we combine:
1. Title match (from property API)
2. Synonym matches (from synonyms endpoint)
3. Linear score decay by position

## Phase 1: Design Artifacts

### Data Model

*See [data-model.md](./data-model.md) for complete model definitions.*

**Key Models**:

1. **PubChemSearchCandidate** (~20 tokens in slim mode)
   - `id`: str (CURIE: `PubChem:CID2244`)
   - `name`: str | None (Title from PubChem)
   - `molecular_formula`: str | None
   - `score`: float (0.0-1.0, relevance ranking)

2. **PubChemCompound** (~115-300 tokens full, ~20 tokens slim)
   - `id`: str (CURIE)
   - `name`: str | None (Title)
   - `iupac_name`: str | None
   - `molecular_formula`: str | None
   - `molecular_weight`: float | None (Daltons)
   - `canonical_smiles`: str | None
   - `isomeric_smiles`: str | None
   - `inchi`: str | None
   - `inchikey`: str | None
   - `synonyms`: list[str]
   - `cross_references`: dict[str, list[str]] (22-key registry)

### Tool Contracts

*See [contracts/](./contracts/) for JSON contract files.*

**Tool 1: search_compounds** (Fuzzy Discovery)
- **Input**: `query` (str), `slim` (bool), `cursor` (str|null), `page_size` (int)
- **Output**: PaginationEnvelope[PubChemSearchCandidate] | ErrorEnvelope
- **Rate Limit**: 5 req/s with backoff

**Tool 2: get_compound** (Strict Lookup)
- **Input**: `pubchem_id` (str, CURIE format), `slim` (bool)
- **Output**: PubChemCompound | ErrorEnvelope
- **Requires**: Resolved CURIE from search_compounds

### Client Architecture

```python
class PubChemClient(LifeSciencesClient):
    """PubChem PUG REST API client.

    Implements:
    - Dual rate limiting (5 req/s, 400 req/min)
    - Exponential backoff on 503
    - Cross-reference extraction
    - CURIE validation
    """

    PUBCHEM_BASE_URL = "https://pubchem.ncbi.nlm.nih.gov/rest/pug"

    # Rate limiting (Constitution v1.1.0)
    RATE_LIMIT_PER_SECOND = 5
    RATE_LIMIT_PER_MINUTE = 400

    # Properties to retrieve
    PROPERTY_LIST = "MolecularFormula,MolecularWeight,CanonicalSMILES,IsomericSMILES,InChI,InChIKey,IUPACName,Title"

    async def search_compounds(self, query: str, slim: bool = False,
                               cursor: str | None = None, page_size: int = 50) -> PaginationEnvelope | ErrorEnvelope:
        """Fuzzy search for compounds by name."""
        ...

    async def get_compound(self, pubchem_id: str, slim: bool = False) -> dict | ErrorEnvelope:
        """Get compound by PubChem CURIE."""
        ...
```

### Server Architecture

```python
from fastmcp import FastMCP

mcp = FastMCP("pubchem-mcp-server")

# Module-level singleton (ADR-004)
_client: PubChemClient | None = None

def get_client() -> PubChemClient:
    global _client
    if _client is None:
        _client = PubChemClient()
    return _client

@mcp.tool()
async def search_compounds(query: str, slim: bool = False,
                          cursor: str | None = None, page_size: int = 50) -> dict:
    """Search for compounds by name, synonym, or identifier."""
    client = get_client()
    result = await client.search_compounds(query, slim, cursor, page_size)
    return result.model_dump() if hasattr(result, 'model_dump') else result

@mcp.tool()
async def get_compound(pubchem_id: str, slim: bool = False) -> dict:
    """Get complete compound record by PubChem CURIE."""
    client = get_client()
    result = await client.get_compound(pubchem_id, slim)
    return result if isinstance(result, dict) else result.model_dump()
```

## Implementation Phases

### Phase 1: Foundation (User Story 1)
- [ ] Create `PubChemSearchCandidate` model
- [ ] Create `PubChemClient` with rate limiting
- [ ] Implement `search_compounds` tool
- [ ] Add unit tests with mocked responses
- [ ] Add integration tests against live API

### Phase 2: Strict Lookup (User Story 2)
- [ ] Create `PubChemCompound` model
- [ ] Implement `get_compound` tool
- [ ] Add CURIE validation
- [ ] Add slim mode support
- [ ] Add unit and integration tests

### Phase 3: Cross-References (User Story 3)
- [ ] Implement cross-reference extraction
- [ ] Map to 22-key registry
- [ ] Add synonyms endpoint integration
- [ ] Validate omit-if-null pattern

### Phase 4: Error Recovery (User Story 4)
- [ ] Implement all error codes
- [ ] Add recovery hints
- [ ] Test error->recovery->success cycle
- [ ] Add error recovery integration tests

### Phase 5: Polish
- [ ] Performance testing
- [ ] Documentation (quickstart.md)
- [ ] Code review and cleanup
- [ ] Final integration test suite

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| *None* | All Constitution principles satisfied | N/A |

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Rate limit exceeded | Medium | Low | Dual rate limiting + exponential backoff |
| PubChem API changes | Low | Medium | Integration tests will catch breaking changes |
| Cross-reference extraction incomplete | Medium | Low | Document limitations, use synonyms fallback |
| Large CIDs (>1B) | Low | Low | Regex allows unlimited digits |

## References

- [PubChem PUG REST Documentation](https://pubchem.ncbi.nlm.nih.gov/docs/pug-rest)
- [PubChem PUG REST Tutorial](https://pubchem.ncbi.nlm.nih.gov/docs/pug-rest-tutorial)
- [ADR-001 v1.2: Agentic-First Architecture](../../docs/adr/accepted/adr-001-v1.2.md)
- [Constitution v1.1.0](../../.specify/memory/constitution.md)
- [ChEMBL Client (reference implementation)](../../src/lifesciences_mcp/clients/chembl.py)
