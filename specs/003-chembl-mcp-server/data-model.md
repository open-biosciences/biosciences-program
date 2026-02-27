# Data Model: ChEMBL MCP Server

**Phase**: 1 (Design & Contracts)
**Date**: 2025-12-22
**Source**: Derived from [spec.md](./spec.md) and [research.md](./research.md)

## Overview

This document defines the data models for ChEMBL compound entities, following the Agentic Biolink schema (ADR-001 §4) and Constitution Principle III (Schema Determinism).

## Core Entities

### CompoundSearchCandidate

**Purpose**: Lightweight search result for fuzzy compound discovery (Principle II: Fuzzy-to-Fact)

**Usage**: Returned by `search_compounds` tool to enable agent triage before strict lookup

**Schema**:
```python
from pydantic import BaseModel, Field

class CompoundSearchCandidate(BaseModel):
    """Lightweight compound search result for fuzzy discovery.

    Token Budget: ~20-30 tokens in slim mode, ~40-50 tokens in full mode
    """

    id: str = Field(
        ...,
        description="ChEMBL CURIE in format 'CHEMBL:NNNNN' (e.g., 'CHEMBL:25', 'CHEMBL:1201583')",
        pattern=r"^CHEMBL:[0-9]+$"
    )

    name: str | None = Field(
        None,
        description="Preferred compound name (IUPAC, trade name, or ChEMBL ID if no name available)",
        examples=["Aspirin", "Acetylsalicylic acid", "CHEMBL25"]
    )

    molecular_formula: str | None = Field(
        None,
        description="Molecular formula (e.g., 'C9H8O4' for aspirin)",
        examples=["C9H8O4", "C29H40N7O17P3S"]
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
                "id": "CHEMBL:25",
                "name": "Aspirin",
                "molecular_formula": "C9H8O4",
                "score": 1.0
            }]
        }
    }
```

**Validation Rules**:
- `id` MUST match regex `^CHEMBL:[0-9]+$`
- `score` MUST be in range [0.0, 1.0]
- `name` MAY be None if ChEMBL has no preferred name
- `molecular_formula` MAY be None if not available

**Token Budget** (FR-026):
- **Slim mode**: `id`, `name`, `molecular_formula`, `score` → ~20-30 tokens
- **Full mode**: Same fields (no additional fields for search candidates) → ~40-50 tokens

---

### Compound

**Purpose**: Complete compound record with molecular structure, properties, and cross-references

**Usage**: Returned by `get_compound` and `get_compounds_batch` for strict factual lookups

**Schema**:
```python
from pydantic import BaseModel, Field
from lifesciences_mcp.models.envelopes import CrossReferences

class Compound(BaseModel):
    """Complete ChEMBL compound record with Agentic Biolink cross-references.

    Token Budget: ~115-300 tokens in full mode, ~20 tokens in slim mode
    """

    id: str = Field(
        ...,
        description="ChEMBL CURIE in format 'CHEMBL:NNNNN'",
        pattern=r"^CHEMBL:[0-9]+$"
    )

    name: str | None = Field(
        None,
        description="Preferred compound name (IUPAC, trade name, or synonym)",
        examples=["Aspirin", "Imatinib"]
    )

    molecular_formula: str | None = Field(
        None,
        description="Molecular formula",
        examples=["C9H8O4"]
    )

    molecular_weight: float | None = Field(
        None,
        description="Molecular weight in g/mol",
        examples=[180.16]
    )

    smiles: str | None = Field(
        None,
        description="Simplified Molecular-Input Line-Entry System (SMILES) notation",
        examples=["CC(=O)Oc1ccccc1C(=O)O"]
    )

    inchi: str | None = Field(
        None,
        description="International Chemical Identifier (InChI)",
        examples=["InChI=1S/C9H8O4/c1-6(10)13-8-5-3-2-4-7(8)9(11)12/h2-5H,1H3,(H,11,12)"]
    )

    canonical_name: str | None = Field(
        None,
        description="Canonical IUPAC name",
        examples=["2-acetoxybenzoic acid"]
    )

    synonyms: list[str] = Field(
        default_factory=list,
        description="Alternative names and trade names",
        examples=[["Acetylsalicylic acid", "ASA", "Ecotrin", "Bufferin"]]
    )

    cross_references: CrossReferences = Field(
        default_factory=CrossReferences,
        description="Cross-references to other biological databases (22-key registry)"
    )

    model_config = {
        "json_schema_extra": {
            "examples": [{
                "id": "CHEMBL:25",
                "name": "Aspirin",
                "molecular_formula": "C9H8O4",
                "molecular_weight": 180.16,
                "smiles": "CC(=O)Oc1ccccc1C(=O)O",
                "inchi": "InChI=1S/C9H8O4/c1-6(10)13-8-5-3-2-4-7(8)9(11)12/h2-5H,1H3,(H,11,12)",
                "canonical_name": "2-acetoxybenzoic acid",
                "synonyms": ["Acetylsalicylic acid", "ASA", "Ecotrin"],
                "cross_references": {
                    "uniprot": ["UniProtKB:P23219"],
                    "pdb": ["1PTY", "3LN1"],
                    "pubchem_compound": ["2244"],
                    "drugbank": ["DB:00945"],
                    "chembl": ["CHEMBL:25"]
                }
            }]
        }
    }
```

**Validation Rules** (from research.md R4):
- `id` MUST match regex `^CHEMBL:[0-9]+$`
- All optional fields (`name`, `molecular_formula`, etc.) MAY be None
- `synonyms` defaults to empty list (never None)
- `cross_references` defaults to empty CrossReferences (omit-if-null pattern)

**Token Budget** (FR-027):
- **Slim mode**: `id`, `name`, `molecular_formula` only → ~20 tokens
- **Full mode**: All fields including cross_references → ~115-300 tokens

---

## Cross-Reference Mappings

**Source**: Research findings (R3: Cross-Reference Mapping)

ChEMBL cross-references map to our 22-key registry as follows:

### Supported Keys

| Registry Key | ChEMBL Source | Example Value | Notes |
|--------------|---------------|---------------|-------|
| `uniprot` | `cross_references[xref_name="UniProt"]` | `["UniProtKB:P23219"]` | Protein targets |
| `pdb` | `cross_references[xref_name="PDBe"]` | `["1PTY", "3LN1"]` | Crystal structures |
| `pubchem_compound` | `cross_references[xref_name="PubChem"]` | `["2244"]` | PubChem CID |
| `drugbank` | `cross_references[xref_name="DrugBank"]` | `["DB:00945"]` | Approved drugs |
| `chembl` | `molecule_hierarchy.parent_chembl_id` | `["CHEMBL:25"]` | Parent compound |

### Omitted Keys (Never Populated)

The following keys from the 22-key registry will **never** be populated by ChEMBL:
- `hgnc`, `ensembl_gene`, `ensembl_transcript`, `entrez`, `refseq` (gene-specific)
- `string`, `biogrid`, `stitch`, `iuphar` (interaction-specific)
- `kegg`, `kegg_pathway`, `omim`, `orphanet`, `mondo`, `efo` (disease/pathway-specific)
- `pubchem_substance` (not provided by ChEMBL)

**Constitution Compliance**: Keys are omitted entirely when no cross-reference exists (Constitution Principle III: never use `null` or empty string).

---

## CURIE Normalization Rules

**Source**: Research findings (R3: Cross-Reference Mapping)

When transforming ChEMBL SDK results to our schema, apply these normalization rules:

| Input Format | Output Format | Example |
|--------------|---------------|---------|
| ChEMBL numeric ID | `CHEMBL:NNNNN` | `25` → `CHEMBL:25` |
| UniProt accession | `UniProtKB:ACCESSION` | `P23219` → `UniProtKB:P23219` |
| DrugBank ID (short) | `DB:NNNNN` | `00945` → `DB:00945` |
| DrugBank ID (full) | `DB:NNNNN` | `DB00945` → `DB:00945` |
| PDB ID | `PDBID` (unchanged) | `1PTY` → `1PTY` |
| PubChem CID | `CID` (numeric only) | `2244` → `2244` |

**Regex Patterns**:
```python
CHEMBL_CURIE_PATTERN = r"^CHEMBL:[0-9]+$"
UNIPROT_CURIE_PATTERN = r"^UniProtKB:[A-Z][A-Z0-9]{5,9}$"
DRUGBANK_CURIE_PATTERN = r"^DB:[0-9]{5}$"
PDB_ID_PATTERN = r"^[0-9][A-Z0-9]{3}$"
```

---

## Field Optionality Matrix

| Field | CompoundSearchCandidate | Compound (Full) | Compound (Slim) |
|-------|------------------------|-----------------|-----------------|
| `id` | ✅ Required | ✅ Required | ✅ Required |
| `name` | ⚠️ Optional | ⚠️ Optional | ✅ Required (slim includes name) |
| `molecular_formula` | ⚠️ Optional | ⚠️ Optional | ✅ Required (slim includes formula) |
| `score` | ✅ Required | ❌ N/A | ❌ N/A |
| `molecular_weight` | ❌ N/A | ⚠️ Optional | ❌ Excluded |
| `smiles` | ❌ N/A | ⚠️ Optional | ❌ Excluded |
| `inchi` | ❌ N/A | ⚠️ Optional | ❌ Excluded |
| `canonical_name` | ❌ N/A | ⚠️ Optional | ❌ Excluded |
| `synonyms` | ❌ N/A | ✅ Default: [] | ❌ Excluded |
| `cross_references` | ❌ N/A | ✅ Default: {} | ❌ Excluded |

**Legend**:
- ✅ Required: Field always present
- ⚠️ Optional: Field may be None if ChEMBL lacks data
- ❌ N/A: Field not applicable to this entity type
- Default: Field has default value if missing

---

## Validation Examples

### Valid CompoundSearchCandidate

```json
{
  "id": "CHEMBL:941",
  "name": "Imatinib",
  "molecular_formula": "C29H31N7O",
  "score": 1.0
}
```

### Valid Compound (Full Mode)

```json
{
  "id": "CHEMBL:25",
  "name": "Aspirin",
  "molecular_formula": "C9H8O4",
  "molecular_weight": 180.16,
  "smiles": "CC(=O)Oc1ccccc1C(=O)O",
  "inchi": "InChI=1S/C9H8O4/c1-6(10)13-8-5-3-2-4-7(8)9(11)12/h2-5H,1H3,(H,11,12)",
  "canonical_name": "2-acetoxybenzoic acid",
  "synonyms": ["Acetylsalicylic acid", "ASA"],
  "cross_references": {
    "uniprot": ["UniProtKB:P23219"],
    "pdb": ["1PTY"],
    "pubchem_compound": ["2244"],
    "drugbank": ["DB:00945"]
  }
}
```

### Valid Compound (Slim Mode)

```json
{
  "id": "CHEMBL:25",
  "name": "Aspirin",
  "molecular_formula": "C9H8O4"
}
```

### Invalid Examples

```json
// ❌ Invalid: Missing CHEMBL: prefix
{"id": "25", "name": "Aspirin", "score": 1.0}

// ❌ Invalid: Lowercase prefix
{"id": "chembl:25", "name": "Aspirin", "score": 1.0}

// ❌ Invalid: Score out of range
{"id": "CHEMBL:25", "name": "Aspirin", "score": 1.5}

// ❌ Invalid: Null cross-reference (should omit key)
{"id": "CHEMBL:25", "cross_references": {"uniprot": null}}
```

---

## Implementation Notes

### Pydantic Models Location

```
src/lifesciences_mcp/models/compound.py  # New file
```

### Import Structure

```python
# In src/lifesciences_mcp/models/__init__.py
from lifesciences_mcp.models.compound import Compound, CompoundSearchCandidate

__all__ = [
    "Compound",
    "CompoundSearchCandidate",
    # ... existing exports
]
```

### Runtime Validation

All models use Pydantic v2 validators:
- CURIE format validation via `pattern` field constraint
- Optional fields default to None (not omitted)
- Cross-references use omit-if-null pattern (Constitution Principle III)

**Next Phase**: Generate API contracts (Phase 1 continuation)
