# Data Model: STRING MCP Server

**Phase**: 1 (Design & Contracts)
**Status**: Complete
**Date**: 2025-12-24

## Model Overview

STRING MCP Server uses 5 Pydantic models following the Agentic Biolink schema:

| Model | Purpose | Used In |
|-------|---------|---------|
| `InteractionSearchCandidate` | Fuzzy search result | search_proteins (Phase 1) |
| `EvidenceScores` | 7-channel evidence breakdown | Interaction |
| `Interaction` | Single protein-protein edge | InteractionNetwork.interactions |
| `InteractionCrossReferences` | Cross-database links | InteractionNetwork |
| `InteractionNetwork` | Complete network response | get_interactions (Phase 2) |

## Model Definitions

### 1. InteractionSearchCandidate

**Purpose**: Represents a protein candidate from fuzzy search (Fuzzy-to-Fact Phase 1).

```python
class InteractionSearchCandidate(BaseModel):
    """Fuzzy search result for protein identification."""

    id: str
    # STRING CURIE (e.g., "STRING:9606.ENSP00000269305")
    # Pattern: ^STRING:\d+\.(ENSP\d+|[a-z]\d+)$

    preferred_name: str
    # Gene symbol (e.g., "TP53")

    annotation: str | None = None
    # Protein description (e.g., "Cellular tumor antigen p53")

    taxon_id: int
    # NCBI Taxonomy ID (e.g., 9606 for Homo sapiens)

    score: float
    # Relevance score 0.0-1.0 (higher = better match)
```

**Validation Rules**:
- `id` must match STRING CURIE pattern
- `score` must be in range [0.0, 1.0]
- `taxon_id` must be positive integer

**Example**:
```json
{
  "id": "STRING:9606.ENSP00000269305",
  "preferred_name": "TP53",
  "annotation": "Cellular tumor antigen p53",
  "taxon_id": 9606,
  "score": 0.95
}
```

### 2. EvidenceScores

**Purpose**: Breakdown of 7 evidence channels for a protein-protein interaction.

```python
class EvidenceScores(BaseModel):
    """Evidence breakdown for protein-protein interaction."""

    nscore: float
    # Neighborhood: Gene proximity (conserved gene order)
    # Range: 0.0-1.0

    fscore: float
    # Gene Fusion: Fusion events across species
    # Range: 0.0-1.0

    pscore: float
    # Phyletic Profiles: Co-occurrence across species
    # Range: 0.0-1.0

    ascore: float
    # Co-expression: mRNA expression correlation
    # Range: 0.0-1.0

    escore: float
    # Experimental: Physical binding assays
    # Range: 0.0-1.0

    dscore: float
    # Database: Curated pathway/complex databases
    # Range: 0.0-1.0

    tscore: float
    # Textmining: Literature co-mentions
    # Range: 0.0-1.0
```

**Validation Rules**:
- All scores must be in range [0.0, 1.0]

**Example**:
```json
{
  "nscore": 0.0,
  "fscore": 0.0,
  "pscore": 0.12,
  "ascore": 0.45,
  "escore": 0.92,
  "dscore": 0.90,
  "tscore": 0.99
}
```

**Interpretation**:
- High `escore` + `dscore`: Experimentally validated + curated (gold standard)
- High `tscore` only: Literature-supported but needs experimental validation
- High `nscore` + `fscore`: Evolutionary evidence (functional linkage)

### 3. Interaction

**Purpose**: Represents a single protein-protein interaction edge.

```python
class Interaction(BaseModel):
    """Single protein-protein interaction with evidence."""

    string_id_a: str
    # STRING protein ID (without namespace prefix)
    # Example: "9606.ENSP00000269305"

    string_id_b: str
    # STRING protein ID (partner protein)

    preferred_name_a: str
    # Gene symbol for protein A (e.g., "TP53")

    preferred_name_b: str
    # Gene symbol for protein B (e.g., "MDM2")

    taxon_id: int
    # NCBI Taxonomy ID (should match for both proteins)

    score: float
    # Combined interaction score 0.0-1.0 (Bayesian integration)

    evidence: EvidenceScores
    # Evidence breakdown by channel
```

**Validation Rules**:
- `score` must be in range [0.0, 1.0]
- `taxon_id` must be positive integer
- `evidence` must pass EvidenceScores validation

**Example**:
```json
{
  "string_id_a": "9606.ENSP00000269305",
  "string_id_b": "9606.ENSP00000258149",
  "preferred_name_a": "TP53",
  "preferred_name_b": "MDM2",
  "taxon_id": 9606,
  "score": 0.999,
  "evidence": {
    "nscore": 0.0,
    "fscore": 0.0,
    "pscore": 0.12,
    "ascore": 0.45,
    "escore": 0.92,
    "dscore": 0.90,
    "tscore": 0.99
  }
}
```

### 4. InteractionCrossReferences

**Purpose**: Cross-database links following the 22-key registry (ADR-001 Appendix A).

```python
class InteractionCrossReferences(BaseModel):
    """Cross-references to other biological databases."""

    ensembl_gene: str | None = None
    # Ensembl Gene ID (e.g., "ENSG00000141510")
    # Omit key if not available (never use null in JSON)

    uniprot: str | None = None
    # UniProt accession (e.g., "P04637")

    hgnc: str | None = None
    # HGNC CURIE (e.g., "HGNC:11998")
```

**Validation Rules**:
- Omit keys entirely if no cross-reference exists (Constitution Principle III)
- Never emit `null` or empty string values

**Example**:
```json
{
  "ensembl_gene": "ENSG00000141510",
  "uniprot": "P04637",
  "hgnc": "HGNC:11998"
}
```

**Note**: Initial implementation may omit cross-references (requires additional API calls).

### 5. InteractionNetwork

**Purpose**: Complete response for `get_interactions()` (Fuzzy-to-Fact Phase 2).

```python
class InteractionNetwork(BaseModel):
    """Complete protein interaction network response."""

    id: str
    # Query protein STRING CURIE (e.g., "STRING:9606.ENSP00000269305")

    preferred_name: str
    # Gene symbol for query protein (e.g., "TP53")

    annotation: str | None = None
    # Protein description

    taxon_id: int
    # NCBI Taxonomy ID

    interaction_count: int
    # Number of interactions returned (may be less than total due to limit)

    interactions: list[Interaction]
    # List of interaction edges

    network_image_url: str
    # URL for network visualization (STRING web interface)

    cross_references: InteractionCrossReferences | None = None
    # Cross-database links for query protein
```

**Validation Rules**:
- `id` must match STRING CURIE pattern
- `interaction_count` must equal `len(interactions)`
- `network_image_url` must be valid URL

**Example**:
```json
{
  "id": "STRING:9606.ENSP00000269305",
  "preferred_name": "TP53",
  "annotation": "Cellular tumor antigen p53",
  "taxon_id": 9606,
  "interaction_count": 3,
  "interactions": [
    {
      "string_id_a": "9606.ENSP00000269305",
      "string_id_b": "9606.ENSP00000258149",
      "preferred_name_a": "TP53",
      "preferred_name_b": "MDM2",
      "taxon_id": 9606,
      "score": 0.999,
      "evidence": { ... }
    },
    ...
  ],
  "network_image_url": "https://string-db.org/api/highres_image/network?identifiers=9606.ENSP00000269305&species=9606&add_nodes=10",
  "cross_references": {
    "ensembl_gene": "ENSG00000141510",
    "uniprot": "P04637",
    "hgnc": "HGNC:11998"
  }
}
```

## Envelope Models

STRING tools use Canonical Envelopes (Constitution Principle III):

### PaginationEnvelope[InteractionSearchCandidate]

Used by `search_proteins()`:

```python
{
  "items": [ ... ],  # List of InteractionSearchCandidate
  "pagination": {
    "cursor": "eyJvZmZzZXQiOiAxMH0=",  # Base64-encoded offset
    "total_count": 45,
    "page_size": 10
  }
}
```

**Note**: STRING API doesn't support native pagination. Cursor implements client-side pagination over results list.

### ErrorEnvelope

Used by all tools for errors:

```python
{
  "success": false,
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "The input '9606.ENSP00000269305' is not a valid STRING CURIE.",
    "recovery_hint": "Call search_proteins to resolve the identifier first. Expected format: STRING:<taxid>.ENSP<id>",
    "invalid_input": "9606.ENSP00000269305"
  }
}
```

**Error Codes**:
- `AMBIGUOUS_QUERY`: Query too short or too many results
- `UNRESOLVED_ENTITY`: Invalid STRING CURIE format
- `ENTITY_NOT_FOUND`: No interactions found for valid protein
- `RATE_LIMITED`: Exceeded 1 req/sec limit
- `UPSTREAM_ERROR`: STRING API error (5xx)

## Model Relationships

```
search_proteins() returns:
  PaginationEnvelope[
    InteractionSearchCandidate  ← Phase 1: Fuzzy search
  ]

get_interactions(string_id) returns:
  InteractionNetwork  ← Phase 2: Strict lookup
    ├── interactions: list[Interaction]
    │   └── evidence: EvidenceScores
    └── cross_references: InteractionCrossReferences
```

## Validation Patterns

### CURIE Pattern

```python
import re

STRING_CURIE_PATTERN = re.compile(r"^STRING:\d+\.(ENSP\d+|[a-z]\d+)$")

def validate_string_curie(curie: str) -> bool:
    return bool(STRING_CURIE_PATTERN.match(curie))
```

### Score Range Validation

```python
from pydantic import Field, field_validator

class EvidenceScores(BaseModel):
    nscore: float = Field(ge=0.0, le=1.0)
    fscore: float = Field(ge=0.0, le=1.0)
    # ... (repeat for all scores)
```

## Implementation File

All models implemented in: `src/lifesciences_mcp/models/interaction.py`

Exports:
- `InteractionSearchCandidate`
- `EvidenceScores`
- `Interaction`
- `InteractionCrossReferences`
- `InteractionNetwork`
- `STRING_CURIE_PATTERN` (compiled regex)
