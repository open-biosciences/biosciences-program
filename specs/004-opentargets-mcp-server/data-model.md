# Data Model: Open Targets MCP Server

**Phase**: 1 (Design & Contracts)
**Date**: 2025-12-22
**Source**: Derived from [spec.md](./spec.md) and [research.md](./research.md)

## Overview

This document defines the data models for Open Targets target and association entities, following the Agentic Biolink schema (ADR-001 §4) and Constitution Principle III (Schema Determinism).

## Core Entities

### TargetSearchCandidate

**Purpose**: Lightweight search result for fuzzy target discovery (Principle II: Fuzzy-to-Fact)

**Usage**: Returned by `search_targets` tool to enable agent triage before strict lookup

**Schema**:
```python
from pydantic import BaseModel, Field

class TargetSearchCandidate(BaseModel):
    """Lightweight target search result for fuzzy discovery.

    Token Budget: ~20-30 tokens in slim mode, ~40-50 tokens in full mode
    """

    id: str = Field(
        ...,
        description="Ensembl gene ID in format ENSG[11 digits] (e.g., 'ENSG00000141510')",
        pattern=r"^ENSG\d{11}$"
    )

    approved_symbol: str | None = Field(
        None,
        description="HGNC approved gene symbol (e.g., 'TP53', 'BRCA1')",
        examples=["TP53", "BRCA1", "EGFR"]
    )

    approved_name: str | None = Field(
        None,
        description="HGNC approved gene name",
        examples=["tumor protein p53", "BRCA1 DNA repair associated"]
    )

    score: float = Field(
        ...,
        ge=0.0,
        le=1.0,
        description="Relevance score (1.0 = perfect match, decreasing linearly)"
    )

    model_config = {
        "json_schema_extra": {
            "examples": [{
                "id": "ENSG00000141510",
                "approved_symbol": "TP53",
                "approved_name": "tumor protein p53",
                "score": 1.0
            }]
        }
    }
```

**Validation Rules**:
- `id` MUST match regex `^ENSG\d{11}$`
- `score` MUST be in range [0.0, 1.0]
- `approved_symbol` and `approved_name` MAY be None if not available

**Token Budget** (FR-026):
- **Slim mode**: `id`, `approved_symbol`, `approved_name`, `score` → ~20-30 tokens
- **Full mode**: Same fields (no additional fields for search candidates) → ~40-50 tokens

---

### Target

**Purpose**: Complete target record with gene information and cross-references

**Usage**: Returned by `get_target` for strict factual lookups

**Schema**:
```python
from pydantic import BaseModel, Field
from lifesciences_mcp.models.envelopes import CrossReferences

class Target(BaseModel):
    """Complete Open Targets target record with Agentic Biolink cross-references.

    Token Budget: ~115-300 tokens in full mode, ~20 tokens in slim mode
    """

    id: str = Field(
        ...,
        description="Ensembl gene ID in format ENSG[11 digits]",
        pattern=r"^ENSG\d{11}$"
    )

    approved_symbol: str | None = Field(
        None,
        description="HGNC approved gene symbol",
        examples=["TP53", "BRCA1"]
    )

    approved_name: str | None = Field(
        None,
        description="HGNC approved gene name",
        examples=["tumor protein p53"]
    )

    biotype: str | None = Field(
        None,
        description="Gene biotype (e.g., 'protein_coding', 'lncRNA')",
        examples=["protein_coding", "lncRNA", "pseudogene"]
    )

    description: str | None = Field(
        None,
        description="Gene function description"
    )

    associated_diseases_count: int | None = Field(
        None,
        description="Total number of associated diseases in Open Targets",
        ge=0
    )

    cross_references: CrossReferences = Field(
        default_factory=CrossReferences,
        description="Cross-references to other biological databases (22-key registry)"
    )

    model_config = {
        "json_schema_extra": {
            "examples": [{
                "id": "ENSG00000141510",
                "approved_symbol": "TP53",
                "approved_name": "tumor protein p53",
                "biotype": "protein_coding",
                "description": "Tumor suppressor gene involved in cell cycle regulation",
                "associated_diseases_count": 245,
                "cross_references": {
                    "hgnc": ["HGNC:11998"],
                    "uniprot": ["UniProtKB:P04637"],
                    "ensembl_gene": ["ENSG00000141510"],
                    "entrez": ["7157"],
                    "omim": ["191170"],
                    "chembl": ["CHEMBL:1993"]
                }
            }]
        }
    }
```

**Validation Rules** (from research.md R3):
- `id` MUST match regex `^ENSG\d{11}$`
- All optional fields MAY be None
- `cross_references` defaults to empty CrossReferences (omit-if-null pattern)
- `associated_diseases_count` MUST be >= 0 if present

**Token Budget** (FR-027, FR-028):
- **Slim mode**: `id`, `approved_symbol`, `approved_name`, `biotype` only → ~20 tokens
- **Full mode**: All fields including cross_references → ~115-300 tokens

---

### Association

**Purpose**: Target-disease association with evidence scores

**Usage**: Returned by `get_associations` for disease association queries

**Schema**:
```python
from pydantic import BaseModel, Field

class Association(BaseModel):
    """Target-disease association with aggregated evidence.

    Token Budget: ~60-80 tokens per association
    """

    target_id: str = Field(
        ...,
        description="Ensembl gene ID",
        pattern=r"^ENSG\d{11}$"
    )

    disease_id: str = Field(
        ...,
        description="EFO disease ID (e.g., 'EFO_0000616' for neoplasm)",
        pattern=r"^EFO_\d+$"
    )

    disease_name: str = Field(
        ...,
        description="Human-readable disease name"
    )

    score: float = Field(
        ...,
        ge=0.0,
        le=1.0,
        description="Overall association score (0.0-1.0, higher = stronger evidence)"
    )

    evidence_count: int = Field(
        ...,
        ge=0,
        description="Total number of evidence sources supporting this association"
    )

    evidence_sources: list[str] = Field(
        default_factory=list,
        description="Data type IDs (e.g., 'genetic_association', 'somatic_mutation')",
        examples=[["genetic_association", "somatic_mutation", "drugs"]]
    )

    model_config = {
        "json_schema_extra": {
            "examples": [{
                "target_id": "ENSG00000141510",
                "disease_id": "EFO_0000616",
                "disease_name": "neoplasm",
                "score": 0.85,
                "evidence_count": 142,
                "evidence_sources": ["genetic_association", "somatic_mutation", "drugs"]
            }]
        }
    }
```

**Validation Rules**:
- `target_id` MUST match `^ENSG\d{11}$`
- `disease_id` MUST match `^EFO_\d+$`
- `score` MUST be in range [0.0, 1.0]
- `evidence_count` MUST be >= 0
- `evidence_sources` defaults to empty list (never None)

**Token Budget**:
- ~60-80 tokens per association (no slim mode - associations always return full records)

---

## Cross-Reference Mappings

**Source**: Research findings (R4: Cross-Reference Mapping)

Open Targets cross-references map to our 22-key registry as follows:

### Supported Keys

| Registry Key | Open Targets Source | Example Value | Notes |
|--------------|---------------------|---------------|-------|
| `ensembl_gene` | `Ensembl` | `["ENSG00000141510"]` | Primary ID |
| `hgnc` | `HGNC` | `["HGNC:11998"]` | CURIE format |
| `uniprot` | `Uniprot` | `["UniProtKB:P04637"]` | CURIE format |
| `chembl` | `ChEMBL` | `["CHEMBL:1993"]` | CURIE format |
| `drugbank` | `DrugBank` | `["DB:00001"]` | CURIE format |
| `entrez` | `Entrez` | `["7157"]` | Numeric only |
| `refseq` | `RefSeq` | `["NM_000546"]` | Transcript ID |
| `omim` | `OMIM` | `["191170"]` | Numeric only |
| `mondo` | `Mondo` | `["MONDO:0007254"]` | CURIE format |
| `efo` | `EFO` | `["EFO:0000616"]` | CURIE format |
| `string` | `STRING` | `["9606.ENSP00000269305"]` | Organism.protein |
| `biogrid` | `BioGRID` | `["113418"]` | Numeric only |
| `kegg` | `KEGG` | `["hsa:7157"]` | Organism:gene |

### Omitted Keys (Never Populated by Open Targets)

The following keys from the 22-key registry will **never** be populated by Open Targets:
- `ensembl_transcript` (not provided in target cross-references)
- `iuphar`, `stitch` (interaction-specific, not in Open Targets core data)
- `kegg_pathway`, `orphanet` (pathway/disease-specific, requires separate queries)
- `pdb`, `pubchem_compound`, `pubchem_substance` (compound/structure-specific)

**Constitution Compliance**: Keys are omitted entirely when no cross-reference exists (Constitution Principle III: never use `null` or empty string).

---

## CURIE Normalization Rules

**Source**: Research findings (R4: Cross-Reference Mapping)

When transforming Open Targets GraphQL results to our schema, apply these normalization rules:

| Input Format | Output Format | Example |
|--------------|---------------|---------|
| Ensembl ID | `ENSG\d{11}` (unchanged) | `ENSG00000141510` → `ENSG00000141510` |
| HGNC numeric ID | `HGNC:NNNNN` | `11998` → `HGNC:11998` |
| UniProt accession | `UniProtKB:ACCESSION` | `P04637` → `UniProtKB:P04637` |
| DrugBank ID (short) | `DB:NNNNN` | `00001` → `DB:00001` |
| ChEMBL ID | `CHEMBL:NNNNN` | `1993` → `CHEMBL:1993` |
| EFO ID | `EFO:NNNNN` or `EFO_NNNNN` | `EFO_0000616` → `EFO_0000616` |

**Regex Patterns**:
```python
ENSEMBL_GENE_ID_PATTERN = r"^ENSG\d{11}$"
EFO_DISEASE_ID_PATTERN = r"^EFO_\d+$"
HGNC_CURIE_PATTERN = r"^HGNC:\d+$"
```

---

## Field Optionality Matrix

| Field | TargetSearchCandidate | Target (Full) | Target (Slim) | Association |
|-------|-----------------------|---------------|---------------|-------------|
| `id` | ✅ Required | ✅ Required | ✅ Required | ✅ Required (target_id) |
| `approved_symbol` | ⚠️ Optional | ⚠️ Optional | ✅ Required (slim includes symbol) | ❌ N/A |
| `approved_name` | ⚠️ Optional | ⚠️ Optional | ✅ Required (slim includes name) | ❌ N/A |
| `score` | ✅ Required | ❌ N/A | ❌ N/A | ✅ Required |
| `biotype` | ❌ N/A | ⚠️ Optional | ✅ Required (slim includes biotype) | ❌ N/A |
| `description` | ❌ N/A | ⚠️ Optional | ❌ Excluded | ❌ N/A |
| `associated_diseases_count` | ❌ N/A | ⚠️ Optional | ❌ Excluded | ❌ N/A |
| `cross_references` | ❌ N/A | ✅ Default: {} | ❌ Excluded | ❌ N/A |
| `disease_id` | ❌ N/A | ❌ N/A | ❌ N/A | ✅ Required |
| `disease_name` | ❌ N/A | ❌ N/A | ❌ N/A | ✅ Required |
| `evidence_count` | ❌ N/A | ❌ N/A | ❌ N/A | ✅ Required |
| `evidence_sources` | ❌ N/A | ❌ N/A | ❌ N/A | ✅ Default: [] |

**Legend**:
- ✅ Required: Field always present
- ⚠️ Optional: Field may be None if Open Targets lacks data
- ❌ N/A: Field not applicable to this entity type
- Default: Field has default value if missing

---

## Validation Examples

### Valid TargetSearchCandidate

```json
{
  "id": "ENSG00000141510",
  "approved_symbol": "TP53",
  "approved_name": "tumor protein p53",
  "score": 1.0
}
```

### Valid Target (Full Mode)

```json
{
  "id": "ENSG00000141510",
  "approved_symbol": "TP53",
  "approved_name": "tumor protein p53",
  "biotype": "protein_coding",
  "description": "Tumor suppressor gene",
  "associated_diseases_count": 245,
  "cross_references": {
    "hgnc": ["HGNC:11998"],
    "uniprot": ["UniProtKB:P04637"],
    "ensembl_gene": ["ENSG00000141510"],
    "omim": ["191170"]
  }
}
```

### Valid Target (Slim Mode)

```json
{
  "id": "ENSG00000141510",
  "approved_symbol": "TP53",
  "approved_name": "tumor protein p53",
  "biotype": "protein_coding"
}
```

### Valid Association

```json
{
  "target_id": "ENSG00000141510",
  "disease_id": "EFO_0000616",
  "disease_name": "neoplasm",
  "score": 0.85,
  "evidence_count": 142,
  "evidence_sources": ["genetic_association", "somatic_mutation"]
}
```

---

## Implementation Notes

### Pydantic Models Location

```
src/lifesciences_mcp/models/target.py  # New file
```

### Import Structure

```python
# In src/lifesciences_mcp/models/__init__.py
from lifesciences_mcp.models.target import (
    Target,
    TargetSearchCandidate,
    Association
)

__all__ = [
    "Target",
    "TargetSearchCandidate",
    "Association",
    # ... existing exports
]
```

### Runtime Validation

All models use Pydantic v2 validators:
- Ensembl ID format validation via `pattern` field constraint
- EFO ID format validation via `pattern` field constraint
- Optional fields default to None (not omitted)
- Cross-references use omit-if-null pattern (Constitution Principle III)

**Next Phase**: Generate API contracts (Phase 1 continuation)
