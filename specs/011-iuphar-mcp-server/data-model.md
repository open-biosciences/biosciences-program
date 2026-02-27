# Data Model: IUPHAR/GtoPdb MCP Server

**Feature**: 011-iuphar-mcp-server
**Date**: 2026-01-01
**Status**: Complete

## Overview

This document defines the Pydantic models for the IUPHAR/GtoPdb MCP Server. The server is a **dual-entity design** supporting both pharmacological ligands (drugs, chemicals) and targets (receptors, enzymes, ion channels).

All models follow the Agentic Biolink schema defined in ADR-001, with cross-references using the 22-key registry.

## Model Hierarchy

```
models/pharmacology.py
├── LigandSearchCandidate     # Slim search result (~20 tokens)
├── Ligand                    # Full entity (~100-200 tokens)
├── TargetSearchCandidate     # Slim search result (~20 tokens)
└── Target                    # Full entity (~100-200 tokens)

models/gene.py (reused)
└── CrossReferences           # 22-key registry
```

---

## Ligand Models

### LigandSearchCandidate

Lightweight representation for fuzzy search results. Optimized for token efficiency (~20 tokens/entity).

```python
import re
from typing import Annotated
from pydantic import BaseModel, Field, field_validator

IUPHAR_CURIE_PATTERN = re.compile(r"^IUPHAR:\d+$")

class LigandSearchCandidate(BaseModel):
    """Lightweight ligand match for fuzzy search results.

    Used in slim mode to reduce token usage (~20 tokens per entity).
    Contains only essential fields for agent decision-making.
    """

    id: Annotated[str, Field(
        pattern=r"^IUPHAR:\d+$",
        description="IUPHAR ligand CURIE (e.g., 'IUPHAR:2713')"
    )]
    name: str = Field(
        description="Ligand display name (e.g., 'ibuprofen')"
    )
    type: str = Field(
        description="Ligand classification (e.g., 'Synthetic organic', 'Peptide')"
    )
    approved: bool = Field(
        default=False,
        description="Whether ligand is an approved drug"
    )
    score: float = Field(
        ge=0.0,
        le=1.0,
        description="Relevance score (0.0-1.0)"
    )

    @field_validator("id")
    @classmethod
    def validate_iuphar_curie(cls, v: str) -> str:
        """Validate IUPHAR CURIE format."""
        if not IUPHAR_CURIE_PATTERN.match(v):
            msg = f"Invalid IUPHAR CURIE format: {v}"
            raise ValueError(msg)
        return v
```

**Fields**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | str | Yes | IUPHAR CURIE: `IUPHAR:\d+` |
| `name` | str | Yes | Ligand display name |
| `type` | str | Yes | Classification (Synthetic organic, Peptide, etc.) |
| `approved` | bool | No (default: false) | Approved drug status |
| `score` | float | Yes | Relevance score 0.0-1.0 |

**Token Estimate**: ~20 tokens

---

### Ligand

Complete ligand record with full details and cross-references. Returned by `get_ligand` tool.

```python
from pydantic import BaseModel, Field
from lifesciences_mcp.models.gene import CrossReferences

class Ligand(BaseModel):
    """Complete ligand record from GtoPdb with Agentic Biolink cross-references.

    Represents pharmacological compounds including drugs, metabolites, peptides,
    and antibodies from the IUPHAR/BPS Guide to PHARMACOLOGY.

    Full record returned by get_ligand (~100-200 tokens depending on cross-refs).
    """

    # Core identifiers
    id: Annotated[str, Field(
        pattern=r"^IUPHAR:\d+$",
        description="IUPHAR ligand CURIE"
    )]
    ligand_id: int = Field(
        description="Raw GtoPdb numeric identifier"
    )

    # Names and classification
    name: str = Field(
        description="Display name (e.g., 'ibuprofen')"
    )
    approved_name: str | None = Field(
        default=None,
        description="International Nonproprietary Name (INN) if available"
    )
    type: str = Field(
        description="Ligand classification"
    )
    abbreviation: str | None = Field(
        default=None,
        description="Short name/abbreviation if available"
    )

    # Regulatory status
    approved: bool = Field(
        default=False,
        description="Regulatory approval status"
    )
    approval_source: str | None = Field(
        default=None,
        description="Approval source (e.g., 'FDA (1974), EMA (2004)')"
    )
    who_essential: bool | None = Field(
        default=None,
        description="WHO essential medicine list status"
    )
    withdrawn: bool | None = Field(
        default=None,
        description="Drug withdrawal status"
    )

    # Synonyms
    synonyms: list[str] | None = Field(
        default=None,
        description="Brand names and alternative names"
    )

    # Cross-references (uses 22-key registry)
    cross_references: CrossReferences = Field(
        default_factory=CrossReferences,
        description="External database identifiers (ChEMBL, DrugBank, PubChem)"
    )

    @field_validator("id")
    @classmethod
    def validate_iuphar_curie(cls, v: str) -> str:
        """Validate IUPHAR CURIE format."""
        if not IUPHAR_CURIE_PATTERN.match(v):
            msg = f"Invalid IUPHAR CURIE format: {v}"
            raise ValueError(msg)
        return v

    def to_search_candidate(self, score: float = 1.0) -> LigandSearchCandidate:
        """Convert to LigandSearchCandidate for search results."""
        return LigandSearchCandidate(
            id=self.id,
            name=self.name,
            type=self.type,
            approved=self.approved,
            score=score,
        )
```

**Fields**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | str | Yes | IUPHAR CURIE |
| `ligand_id` | int | Yes | Raw numeric ID |
| `name` | str | Yes | Display name |
| `approved_name` | str | No | INN name |
| `type` | str | Yes | Classification |
| `abbreviation` | str | No | Short name |
| `approved` | bool | No | Approval status |
| `approval_source` | str | No | Regulatory source |
| `who_essential` | bool | No | WHO essential list |
| `withdrawn` | bool | No | Withdrawal status |
| `synonyms` | list[str] | No | Brand names |
| `cross_references` | CrossReferences | Yes | External DB links |

**Cross-Reference Keys Used**:
- `chembl` - ChEMBL compound ID (e.g., "CHEMBL521")
- `drugbank` - DrugBank ID (e.g., "DB01050")
- `pubchem_compound` - PubChem CID (e.g., "3672")

**Token Estimate**: ~100-200 tokens

---

## Target Models

### TargetSearchCandidate

Lightweight representation for fuzzy search results. Optimized for token efficiency (~20 tokens/entity).

```python
class TargetSearchCandidate(BaseModel):
    """Lightweight target match for fuzzy search results.

    Used in slim mode to reduce token usage (~20 tokens per entity).
    Contains only essential fields for agent decision-making.
    """

    id: Annotated[str, Field(
        pattern=r"^IUPHAR:\d+$",
        description="IUPHAR target CURIE (e.g., 'IUPHAR:215')"
    )]
    name: str = Field(
        description="Target display name (HTML stripped, e.g., 'D2 receptor')"
    )
    family: str = Field(
        description="Target family/class (e.g., 'GPCR', 'Enzyme')"
    )
    type: str = Field(
        description="Full classification type"
    )
    score: float = Field(
        ge=0.0,
        le=1.0,
        description="Relevance score (0.0-1.0)"
    )

    @field_validator("id")
    @classmethod
    def validate_iuphar_curie(cls, v: str) -> str:
        """Validate IUPHAR CURIE format."""
        if not IUPHAR_CURIE_PATTERN.match(v):
            msg = f"Invalid IUPHAR CURIE format: {v}"
            raise ValueError(msg)
        return v
```

**Fields**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | str | Yes | IUPHAR CURIE: `IUPHAR:\d+` |
| `name` | str | Yes | Target display name (HTML stripped) |
| `family` | str | Yes | Target family (GPCR, Enzyme, etc.) |
| `type` | str | Yes | Full classification |
| `score` | float | Yes | Relevance score 0.0-1.0 |

**Token Estimate**: ~20 tokens

---

### Target

Complete target record with full details and cross-references. Returned by `get_target` tool.

```python
class Target(BaseModel):
    """Complete target record from GtoPdb with Agentic Biolink cross-references.

    Represents pharmacological targets including GPCRs, ion channels, enzymes,
    transporters, and other proteins from the IUPHAR/BPS Guide to PHARMACOLOGY.

    Full record returned by get_target (~100-200 tokens depending on cross-refs).
    """

    # Core identifiers
    id: Annotated[str, Field(
        pattern=r"^IUPHAR:\d+$",
        description="IUPHAR target CURIE"
    )]
    target_id: int = Field(
        description="Raw GtoPdb numeric identifier"
    )

    # Names and classification
    name: str = Field(
        description="Target display name (HTML stripped)"
    )
    target_family: str = Field(
        description="Target family classification (GPCR, Enzyme, etc.)"
    )
    family_ids: list[int] | None = Field(
        default=None,
        description="Parent family IDs for hierarchy navigation"
    )

    # Species and gene info
    species: str = Field(
        default="Homo sapiens",
        description="Primary species (defaults to human)"
    )
    gene_symbol: str | None = Field(
        default=None,
        description="Human gene symbol if available"
    )

    # Cross-references (uses 22-key registry)
    cross_references: CrossReferences = Field(
        default_factory=CrossReferences,
        description="External database identifiers (UniProt, Ensembl, HGNC)"
    )

    @field_validator("id")
    @classmethod
    def validate_iuphar_curie(cls, v: str) -> str:
        """Validate IUPHAR CURIE format."""
        if not IUPHAR_CURIE_PATTERN.match(v):
            msg = f"Invalid IUPHAR CURIE format: {v}"
            raise ValueError(msg)
        return v

    @field_validator("name", mode="before")
    @classmethod
    def strip_html_from_name(cls, v: str) -> str:
        """Strip HTML tags from target name."""
        import re
        return re.sub(r"<[^>]+>", "", v)

    def to_search_candidate(self, score: float = 1.0) -> TargetSearchCandidate:
        """Convert to TargetSearchCandidate for search results."""
        return TargetSearchCandidate(
            id=self.id,
            name=self.name,
            family=self.target_family,
            type=self.target_family,  # Same as family for consistency
            score=score,
        )
```

**Fields**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | str | Yes | IUPHAR CURIE |
| `target_id` | int | Yes | Raw numeric ID |
| `name` | str | Yes | Display name (HTML stripped) |
| `target_family` | str | Yes | Classification (GPCR, Enzyme, etc.) |
| `family_ids` | list[int] | No | Parent family IDs |
| `species` | str | No | Species (default: Homo sapiens) |
| `gene_symbol` | str | No | Human gene symbol |
| `cross_references` | CrossReferences | Yes | External DB links |

**Cross-Reference Keys Used**:
- `uniprot` - UniProt accession (e.g., ["P14416"])
- `ensembl_gene` - Ensembl gene ID (e.g., "ENSG00000149295")
- `entrez` - NCBI Entrez gene ID (e.g., "1813")
- `refseq` - RefSeq accessions (e.g., ["NM_000795"])
- `chembl` - ChEMBL target ID (e.g., "CHEMBL217")
- `omim` - OMIM ID (e.g., "126450")
- `orphanet` - Orphanet ID (e.g., "ORPHA121177")

**Token Estimate**: ~100-200 tokens

---

## Validation Rules

### CURIE Validation

All IUPHAR CURIEs must match:
```
^IUPHAR:\d+$
```

Examples:
- Valid: `IUPHAR:2713`, `IUPHAR:215`, `IUPHAR:1`
- Invalid: `iuphar:2713`, `IUPHAR:abc`, `2713`, `HGNC:1101`

### Score Bounds

All relevance scores must be:
```
0.0 <= score <= 1.0
```

### Type Values

**Ligand Types** (from GtoPdb):
- Synthetic organic
- Metabolite
- Natural product
- Endogenous peptide
- Peptide
- Antibody
- Inorganic

**Target Types** (from GtoPdb):
- GPCR
- NHR
- LGIC
- VGIC
- OtherIC
- Enzyme
- CatalyticReceptor
- Transporter
- OtherProtein
- AccessoryProtein

### Cross-Reference Validation

Reuse existing validators from `models/gene.py`:

| Key | Regex |
|-----|-------|
| `chembl` | `^CHEMBL\d+$` |
| `drugbank` | `^DB\d{5}$` |
| `uniprot` | `^[A-Z0-9]{6,10}$` |
| `ensembl_gene` | `^ENSG\d{11}$` |
| `entrez` | `^\d+$` |
| `omim` | `^\d{6}$` |

---

## JSON Serialization

### Omit-if-Null Pattern

Per ADR-001 and Constitution Principle III, null/empty values are omitted:

```python
def model_dump(self, **kwargs) -> dict:
    """Override to exclude None values."""
    kwargs.setdefault("exclude_none", True)
    return super().model_dump(**kwargs)
```

### Example Ligand JSON

```json
{
  "id": "IUPHAR:2713",
  "ligand_id": 2713,
  "name": "ibuprofen",
  "approved_name": "ibuprofen",
  "type": "Synthetic organic",
  "approved": true,
  "approval_source": "FDA (1974), EMA (2004)",
  "who_essential": true,
  "synonyms": ["Advil", "Brufen", "Motrin", "Nurofen"],
  "cross_references": {
    "chembl": "CHEMBL521",
    "drugbank": "DB01050",
    "pubchem_compound": "3672"
  }
}
```

### Example Target JSON

```json
{
  "id": "IUPHAR:215",
  "target_id": 215,
  "name": "D2 receptor",
  "target_family": "GPCR",
  "family_ids": [20],
  "species": "Homo sapiens",
  "gene_symbol": "DRD2",
  "cross_references": {
    "uniprot": ["P14416"],
    "ensembl_gene": "ENSG00000149295",
    "entrez": "1813",
    "refseq": ["NM_000795"],
    "chembl": "CHEMBL217",
    "omim": "126450"
  }
}
```

---

## File Location

Models will be implemented in:
```
src/lifesciences_mcp/models/pharmacology.py
```

Imports:
```python
from lifesciences_mcp.models.pharmacology import (
    Ligand,
    LigandSearchCandidate,
    Target,
    TargetSearchCandidate,
    IUPHAR_CURIE_PATTERN,
)
```
