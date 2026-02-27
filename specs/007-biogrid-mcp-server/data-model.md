# Data Model: BioGRID MCP Server

**Phase**: 1 (Design & Contracts)
**Date**: 2025-12-24
**Purpose**: Document Pydantic models for BioGRID interaction data

## Model Hierarchy

```
BioGridSearchCandidate     # Fuzzy search result (gene symbol validation)
    └── Used in: search_genes tool → PaginationEnvelope

GeneticInteraction         # Single protein-protein or genetic interaction
    ├── experimental_system
    ├── experimental_system_type
    ├── pubmed_id
    └── throughput

BioGridCrossReferences     # Cross-database identifiers
    └── entrez              # Entrez Gene ID

InteractionResult          # Complete interaction response
    ├── query_gene
    ├── interactions: List[GeneticInteraction]
    ├── cross_references: BioGridCrossReferences
    ├── physical_count
    └── genetic_count
```

## Model Definitions

### BioGridSearchCandidate

**Purpose**: Gene search result confirmed to exist in BioGRID (Fuzzy Phase 1)

**Usage**: `search_genes` tool returns confirmed candidates with interaction count

**Fields**:
```python
class BioGridSearchCandidate(BaseModel):
    """Gene search result confirmed to exist in BioGRID."""

    symbol: str
    # Gene symbol (uppercase normalized)
    # Example: "TP53"
    # Validation: ^[A-Z0-9][A-Z0-9\-_@.]{0,29}$

    organism: str
    # Organism name
    # Example: "Homo sapiens"

    taxon_id: int
    # NCBI Taxonomy ID
    # Example: 9606

    interaction_count: int
    # Total interactions in BioGRID for this gene
    # Example: 14523
    # Confirmed via API (format=count)
```

**Validation Rules**:
- `symbol`: Must match gene symbol pattern (alphanumeric + hyphens/underscores)
- `taxon_id`: Must be positive integer
- `interaction_count`: Must be >= 0 (presence in results implies count > 0)

**Example**:
```json
{
  "symbol": "TP53",
  "organism": "Homo sapiens",
  "taxon_id": 9606,
  "interaction_count": 14523
}
```

### GeneticInteraction

**Purpose**: Single experimentally validated interaction

**Usage**: Returned in `InteractionResult.interactions` list

**Fields**:
```python
class GeneticInteraction(BaseModel):
    """Single genetic or protein-protein interaction from BioGRID."""

    biogrid_interaction_id: int
    # Unique BioGRID interaction identifier
    # Example: 1234567

    symbol_a: str
    # Query gene symbol
    # Example: "TP53"

    symbol_b: str
    # Interacting partner symbol
    # Example: "MDM2"

    experimental_system: str
    # Experimental method used
    # Example: "Affinity Capture-Western"
    # Values: See research.md §3 for full list

    experimental_system_type: Literal["physical", "genetic"]
    # Interaction category
    # Example: "physical"

    pubmed_id: int | None = None
    # PubMed ID for supporting publication
    # Example: 12345678
    # Optional: Some high-throughput studies lack PubMed IDs

    throughput: Literal["High Throughput", "Low Throughput"] | None = None
    # Study type classification
    # Example: "Low Throughput"
    # Optional: Not always provided by BioGRID

    organism_a_id: int
    # NCBI Taxonomy ID for gene A
    # Example: 9606

    organism_b_id: int
    # NCBI Taxonomy ID for gene B
    # Example: 9606

    entrez_gene_a: int | None = None
    # Entrez Gene ID for gene A
    # Example: 7157
    # Optional: Available when BioGRID provides ENTREZ_GENE field

    entrez_gene_b: int | None = None
    # Entrez Gene ID for gene B
    # Example: 4193
    # Optional: Available when BioGRID provides ENTREZ_GENE field
```

**Validation Rules**:
- `biogrid_interaction_id`: Must be positive integer
- `symbol_a`, `symbol_b`: Must match gene symbol pattern
- `experimental_system_type`: Must be exactly "physical" or "genetic"
- `throughput`: If present, must be exactly "High Throughput" or "Low Throughput"
- `pubmed_id`: If present, must be positive integer
- `organism_a_id`, `organism_b_id`: Must be positive integers

**Example**:
```json
{
  "biogrid_interaction_id": 1234567,
  "symbol_a": "TP53",
  "symbol_b": "MDM2",
  "experimental_system": "Affinity Capture-Western",
  "experimental_system_type": "physical",
  "pubmed_id": 12345678,
  "throughput": "Low Throughput",
  "organism_a_id": 9606,
  "organism_b_id": 9606,
  "entrez_gene_a": 7157,
  "entrez_gene_b": 4193
}
```

### BioGridCrossReferences

**Purpose**: Cross-database identifiers per ADR-001 Appendix A

**Usage**: Included in `InteractionResult` for query gene

**Fields**:
```python
class BioGridCrossReferences(BaseModel):
    """Cross-database identifiers for BioGRID query gene.

    Per Constitution Principle III: Omit keys entirely if no reference exists.
    Never use null or empty string.
    """

    entrez: str | None = None
    # Entrez Gene ID (if available)
    # Example: "7157"
    # Format: ^[0-9]+$
    # Omit if not provided by BioGRID
```

**Validation Rules**:
- `entrez`: If present, must match `^[0-9]+$` (numeric string)
- **Critical**: Omit `entrez` key entirely if no cross-reference available (never set to `null`)

**Example (with Entrez)**:
```json
{
  "entrez": "7157"
}
```

**Example (no cross-references)**:
```json
{}
```

**Never do this**:
```json
{
  "entrez": null  // ❌ WRONG - Constitution violation
}
```

### InteractionResult

**Purpose**: Complete response from `get_interactions` tool

**Usage**: Wraps interaction list with metadata and counts

**Fields**:
```python
class InteractionResult(BaseModel):
    """Complete interaction network response from BioGRID."""

    query_gene: str
    # Gene symbol queried
    # Example: "TP53"

    interactions: List[GeneticInteraction]
    # List of interactions
    # Sorted by biogrid_interaction_id (ascending)

    cross_references: BioGridCrossReferences
    # Cross-database identifiers for query gene
    # Example: {"entrez": "7157"}

    physical_count: int
    # Number of physical interactions
    # Example: 450

    genetic_count: int
    # Number of genetic interactions
    # Example: 50

    total_count: int
    # Total interactions returned
    # Example: 500
    # Must equal len(interactions) and physical_count + genetic_count
```

**Validation Rules**:
- `query_gene`: Must match gene symbol pattern
- `physical_count`, `genetic_count`, `total_count`: Must be non-negative integers
- `total_count`: Must equal `physical_count + genetic_count`
- `total_count`: Must equal `len(interactions)`
- `interactions`: Must not exceed NFR-003 limit (10,000)

**Example**:
```json
{
  "query_gene": "TP53",
  "interactions": [
    {
      "biogrid_interaction_id": 1234567,
      "symbol_a": "TP53",
      "symbol_b": "MDM2",
      "experimental_system": "Affinity Capture-Western",
      "experimental_system_type": "physical",
      "pubmed_id": 12345678,
      "throughput": "Low Throughput",
      "organism_a_id": 9606,
      "organism_b_id": 9606,
      "entrez_gene_a": 7157,
      "entrez_gene_b": 4193
    }
  ],
  "cross_references": {
    "entrez": "7157"
  },
  "physical_count": 450,
  "genetic_count": 50,
  "total_count": 500
}
```

## Slim Mode (Constitution Principle IV)

### InteractionResult.to_slim()

**Purpose**: Token-budgeted output for context-constrained agents

**Returns**: Reduced dict with only essential fields (~15 tokens/interaction vs ~100 full)

```python
def to_slim(self) -> dict[str, Any]:
    """Return slim representation with minimal fields."""
    return {
        "query_gene": self.query_gene,
        "interactions": [
            {
                "symbol_b": i.symbol_b,
                "experimental_system_type": i.experimental_system_type,
            }
            for i in self.interactions
        ],
        "physical_count": self.physical_count,
        "genetic_count": self.genetic_count,
        "total_count": self.total_count,
    }
```

**Example (slim=True)**:
```json
{
  "query_gene": "TP53",
  "interactions": [
    {"symbol_b": "MDM2", "experimental_system_type": "physical"},
    {"symbol_b": "ATM", "experimental_system_type": "physical"},
    {"symbol_b": "BRCA1", "experimental_system_type": "genetic"}
  ],
  "physical_count": 2,
  "genetic_count": 1,
  "total_count": 3
}
```

**Token savings**: ~85% reduction for large interaction networks (e.g., TP53 with 1000+ interactions)

## Envelopes

### PaginationEnvelope (for search_genes)

**Purpose**: Wrap search results per Constitution Principle III

**Schema**:
```python
{
  "items": List[BioGridSearchCandidate],
  "pagination": {
    "cursor": None,        # BioGRID doesn't support pagination
    "total_count": int,    # Number of candidates returned
    "page_size": int       # Always 1 for gene validation
  }
}
```

**Example**:
```json
{
  "items": [
    {
      "symbol": "TP53",
      "organism": "Homo sapiens",
      "taxon_id": 9606,
      "interaction_count": 14523
    }
  ],
  "pagination": {
    "cursor": null,
    "total_count": 1,
    "page_size": 1
  }
}
```

### ErrorEnvelope (all errors)

**Purpose**: Standardized error reporting per Constitution Principle III

**Schema**:
```python
{
  "success": false,
  "error": {
    "code": str,           # Error code (see below)
    "message": str,        # Human-readable error
    "recovery_hint": str,  # Actionable guidance
    "invalid_input": str   # Original input that caused error
  }
}
```

**Error Codes**:
| Code | Trigger | Recovery Hint |
|------|---------|---------------|
| `AMBIGUOUS_QUERY` | Query < 2 chars | "Provide at least 2 characters" |
| `ENTITY_NOT_FOUND` | BioGRID returns no interactions | "Verify gene symbol is correct and has known interactions" |
| `UPSTREAM_ERROR` | BioGRID API error (missing/invalid key, 500, timeout) | "Check BIOGRID_API_KEY is valid. Get free key at https://webservice.thebiogrid.org/" |
| `RATE_LIMITED` | HTTP 429 or client-side rate limit | "Wait 1 second before retrying. Rate limit: 2 req/sec" |

**Example**:
```json
{
  "success": false,
  "error": {
    "code": "UPSTREAM_ERROR",
    "message": "BioGRID API returned: Invalid accesskey",
    "recovery_hint": "Set BIOGRID_API_KEY environment variable. Get free key at https://webservice.thebiogrid.org/",
    "invalid_input": ""
  }
}
```

## Mapping from BioGRID API

**BioGRID JSON Response → GeneticInteraction**:
| BioGRID Field | Model Field | Transform |
|---------------|-------------|-----------|
| `BIOGRID_INTERACTION_ID` | `biogrid_interaction_id` | int() |
| `OFFICIAL_SYMBOL_A` | `symbol_a` | str.upper() |
| `OFFICIAL_SYMBOL_B` | `symbol_b` | str.upper() |
| `EXPERIMENTAL_SYSTEM` | `experimental_system` | passthrough |
| `EXPERIMENTAL_SYSTEM_TYPE` | `experimental_system_type` | "physical" or "genetic" |
| `PUBMED_ID` | `pubmed_id` | int() if present else None |
| `THROUGHPUT` | `throughput` | passthrough if present else None |
| `ORGANISM_A_ID` | `organism_a_id` | int() |
| `ORGANISM_B_ID` | `organism_b_id` | int() |
| `ENTREZ_GENE_A` | `entrez_gene_a` | int() if present else None |
| `ENTREZ_GENE_B` | `entrez_gene_b` | int() if present else None |

## Design Decisions

### 1. Why no CURIE format?

BioGRID uses gene symbols natively, not identifiers. Forcing CURIE format (e.g., `BIOGRID:TP53`) would add unnecessary complexity without improving precision.

**Decision**: Use uppercase gene symbols directly
**Rationale**: Matches BioGRID API design; simpler for agents to use

### 2. Why separate physical_count and genetic_count?

Physical and genetic interactions represent fundamentally different biological evidence:
- Physical: Direct protein binding
- Genetic: Functional relationships (synthetic lethality, epistasis)

**Decision**: Provide separate counts in `InteractionResult`
**Rationale**: Enables agents to assess evidence type distribution

### 3. Why include throughput classification?

High-throughput studies have higher false positive rates than low-throughput focused studies.

**Decision**: Include `throughput` field in `GeneticInteraction`
**Rationale**: Enables filtering for high-confidence interactions from focused studies

### 4. Why minimal cross-references?

BioGRID primarily provides gene symbols and Entrez IDs. Other cross-references (UniProt, Ensembl) would require additional API calls.

**Decision**: Include only Entrez Gene ID
**Rationale**: Entrez enables NCBI integration; other IDs can be resolved via HGNC/UniProt tools

## ADR-001 Compliance

- ✅ **§4 Agentic Biolink**: Cross-references follow 22-key registry
- ✅ **§8 Canonical Envelopes**: PaginationEnvelope and ErrorEnvelope used
- ✅ **Omit-if-null Pattern**: cross_references omits keys without values
- ✅ **Flat JSON**: No deep nesting beyond cross_references object

## Next Steps

1. Create tool contracts in `contracts/` (search_genes.yaml, get_interactions.yaml)
2. Create quickstart.md with usage examples
3. Implement models in `src/lifesciences_mcp/models/biogrid.py`
