# Data Model: PubChem MCP Server

**Phase**: 1 (Design)
**Date**: 2026-01-01
**Purpose**: Define Pydantic models for PubChem compound data

## Overview

This document defines the data models for the PubChem MCP Server, following ADR-001 v1.2 (Agentic Biolink Schema) and Constitution Principle III (Schema Determinism).

## Models

### PubChemSearchCandidate

Lightweight model for fuzzy search results. Used by `search_compounds` tool.

**Token Budget**: ~20-30 tokens per entity (slim mode)

```python
import re
from typing import Any
from pydantic import BaseModel, Field, field_validator

# CURIE validation pattern
PUBCHEM_CURIE_PATTERN = re.compile(r"^PubChem:CID\d+$")


class PubChemSearchCandidate(BaseModel):
    """Lightweight compound search result for fuzzy discovery.

    Token Budget: ~20-30 tokens in slim mode, ~40-50 tokens in full mode

    Usage: Returned by search_compounds tool to enable agent triage
    before strict lookup with get_compound.
    """

    id: str = Field(
        ...,
        description="PubChem CURIE in format 'PubChem:CID{number}' (e.g., 'PubChem:CID2244')",
    )

    name: str | None = Field(
        None,
        description="Common compound name (Title from PubChem)",
    )

    molecular_formula: str | None = Field(
        None,
        description="Molecular formula (e.g., 'C9H8O4' for aspirin)",
    )

    score: float = Field(
        ...,
        ge=0.0,
        le=1.0,
        description="Relevance score (1.0 = best match, decreasing by position)",
    )

    @field_validator("id")
    @classmethod
    def validate_pubchem_curie(cls, v: str) -> str:
        """Validate PubChem CURIE format."""
        if not PUBCHEM_CURIE_PATTERN.match(v):
            raise ValueError(
                f"Invalid PubChem CURIE format: '{v}'. "
                "Must match 'PubChem:CID{number}' (e.g., 'PubChem:CID2244')"
            )
        return v

    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "id": "PubChem:CID2244",
                    "name": "Aspirin",
                    "molecular_formula": "C9H8O4",
                    "score": 1.0,
                },
                {
                    "id": "PubChem:CID3672",
                    "name": "Ibuprofen",
                    "molecular_formula": "C13H18O2",
                    "score": 0.95,
                },
            ]
        }
    }
```

### PubChemCompound

Complete compound record with cross-references. Used by `get_compound` tool.

**Token Budget**: ~115-300 tokens full mode, ~20 tokens slim mode

```python
class PubChemCompound(BaseModel):
    """Complete PubChem compound record with Agentic Biolink cross-references.

    Token Budget:
    - Full mode: ~115-300 tokens (depends on cross-references and synonyms)
    - Slim mode: ~20 tokens (id, name, molecular_formula only)

    Usage: Returned by get_compound for strict factual lookups.
    Cross-references enable multi-hop reasoning to ChEMBL, DrugBank, etc.
    """

    id: str = Field(
        ...,
        description="PubChem CURIE in format 'PubChem:CID{number}'",
    )

    name: str | None = Field(
        None,
        description="Common compound name (Title from PubChem)",
    )

    iupac_name: str | None = Field(
        None,
        description="IUPAC systematic name",
    )

    molecular_formula: str | None = Field(
        None,
        description="Molecular formula (e.g., 'C9H8O4')",
    )

    molecular_weight: float | None = Field(
        None,
        description="Molecular weight in Daltons (g/mol)",
    )

    canonical_smiles: str | None = Field(
        None,
        description="Canonical SMILES string (unique molecular representation)",
    )

    isomeric_smiles: str | None = Field(
        None,
        description="Isomeric SMILES string (includes stereochemistry)",
    )

    inchi: str | None = Field(
        None,
        description="IUPAC International Chemical Identifier (InChI)",
    )

    inchikey: str | None = Field(
        None,
        description="Hashed InChI key (27 characters, for searching/comparison)",
    )

    synonyms: list[str] = Field(
        default_factory=list,
        description="Alternative names, trade names, and identifiers",
    )

    cross_references: dict[str, list[str]] = Field(
        default_factory=dict,
        description="Cross-references to other databases (22-key registry)",
    )

    @field_validator("id")
    @classmethod
    def validate_pubchem_curie(cls, v: str) -> str:
        """Validate PubChem CURIE format."""
        if not PUBCHEM_CURIE_PATTERN.match(v):
            raise ValueError(
                f"Invalid PubChem CURIE format: '{v}'. "
                "Must match 'PubChem:CID{number}' (e.g., 'PubChem:CID2244')"
            )
        return v

    def to_slim(self) -> dict[str, Any]:
        """Return slim representation with minimal fields (~20 tokens).

        Used for token budgeting in batch operations.
        """
        return {
            "id": self.id,
            "name": self.name,
            "molecular_formula": self.molecular_formula,
        }

    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "id": "PubChem:CID2244",
                    "name": "Aspirin",
                    "iupac_name": "2-acetoxybenzoic acid",
                    "molecular_formula": "C9H8O4",
                    "molecular_weight": 180.16,
                    "canonical_smiles": "CC(=O)OC1=CC=CC=C1C(=O)O",
                    "isomeric_smiles": "CC(=O)OC1=CC=CC=C1C(=O)O",
                    "inchi": "InChI=1S/C9H8O4/c1-6(10)13-8-5-3-2-4-7(8)9(11)12/h2-5H,1H3,(H,11,12)",
                    "inchikey": "BSYNRYMUTXBXSQ-UHFFFAOYSA-N",
                    "synonyms": [
                        "acetylsalicylic acid",
                        "2-acetoxybenzoic acid",
                        "ASA",
                        "Ecotrin",
                        "CHEMBL25",
                        "DB00945",
                    ],
                    "cross_references": {
                        "pubchem_compound": ["2244"],
                        "chembl": ["CHEMBL:25"],
                        "drugbank": ["DB00945"],
                    },
                }
            ]
        }
    }
```

## Cross-Reference Key Registry

Per ADR-001 v1.2 Appendix A, PubChem compounds may include these cross-reference keys:

| Key | ID Type | Format Regex | Cardinality | Notes |
|-----|---------|--------------|-------------|-------|
| `pubchem_compound` | PubChem CID | `^\d+$` | String | Self-reference (CID as string) |
| `chembl` | ChEMBL ID | `^CHEMBL:\d+$` | List[String] | Extracted from synonyms |
| `drugbank` | DrugBank ID | `^DB\d{5}$` | List[String] | Extracted from synonyms |
| `uniprot` | UniProt Accession | `^UniProtKB:[A-Z0-9]+$` | List[String] | From xrefs (targets) |
| `pdb` | PDB Structure ID | `^[0-9][A-Z0-9]{3}$` | List[String] | From xrefs |
| `chebi` | ChEBI ID | `^CHEBI:\d+$` | List[String] | From xrefs |

**Null Handling Policy** (Constitution Principle III):
- Omit keys entirely if no cross-reference exists
- Never use `null` or empty string
- Minimizes token usage for sparse cross-references

## API Response Structures

### Property API Response (from PubChem)

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
        "InChI": "InChI=1S/C9H8O4/c1-6(10)13-8-5-3-2-4-7(8)9(11)12/h2-5H,1H3,(H,11,12)",
        "InChIKey": "BSYNRYMUTXBXSQ-UHFFFAOYSA-N",
        "IUPACName": "2-acetoxybenzoic acid",
        "Title": "Aspirin"
      }
    ]
  }
}
```

### Synonyms API Response (from PubChem)

```json
{
  "InformationList": {
    "Information": [
      {
        "CID": 2244,
        "Synonym": [
          "aspirin",
          "acetylsalicylic acid",
          "2-acetoxybenzoic acid",
          "CHEMBL25",
          "DB00945",
          "50-78-2",
          "ASA",
          "Ecotrin",
          "Aspirin"
        ]
      }
    ]
  }
}
```

### Canonical Envelopes (Our Output)

**PaginationEnvelope** (search_compounds):
```json
{
  "items": [
    {
      "id": "PubChem:CID2244",
      "name": "Aspirin",
      "molecular_formula": "C9H8O4",
      "score": 1.0
    },
    {
      "id": "PubChem:CID3672",
      "name": "Ibuprofen",
      "molecular_formula": "C13H18O2",
      "score": 0.95
    }
  ],
  "pagination": {
    "cursor": "eyJvZmZzZXQiOjUwfQ==",
    "total_count": 127,
    "page_size": 50
  }
}
```

**ErrorEnvelope** (all error responses):
```json
{
  "success": false,
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "Invalid PubChem CURIE format: 'CID2244'",
    "recovery_hint": "Use format 'PubChem:CID{number}' (e.g., 'PubChem:CID2244'). Call search_compounds to find valid CIDs.",
    "invalid_input": "CID2244"
  }
}
```

## Field Mapping (PubChem -> Our Models)

| PubChem API Field | Our Model Field | Transform | Notes |
|-------------------|-----------------|-----------|-------|
| `CID` | `id` | `f"PubChem:CID{cid}"` | Add CURIE prefix |
| `Title` | `name` | As-is | Common name |
| `IUPACName` | `iupac_name` | As-is | Systematic name |
| `MolecularFormula` | `molecular_formula` | As-is | |
| `MolecularWeight` | `molecular_weight` | `float()` | In Daltons |
| `CanonicalSMILES` | `canonical_smiles` | As-is | |
| `IsomericSMILES` | `isomeric_smiles` | As-is | |
| `InChI` | `inchi` | As-is | |
| `InChIKey` | `inchikey` | As-is | |
| `Synonym[]` | `synonyms` | Filter/limit | First N synonyms |
| Synonym matches | `cross_references.chembl` | Regex extract | `CHEMBL\d+` |
| Synonym matches | `cross_references.drugbank` | Regex extract | `DB\d{5}` |

## Token Budget Analysis

### Search Candidate (slim mode)
```json
{"id": "PubChem:CID2244", "name": "Aspirin", "molecular_formula": "C9H8O4", "score": 1.0}
```
**Estimated tokens**: ~20-25

### Full Compound (all fields)
```json
{
  "id": "PubChem:CID2244",
  "name": "Aspirin",
  "iupac_name": "2-acetoxybenzoic acid",
  "molecular_formula": "C9H8O4",
  "molecular_weight": 180.16,
  "canonical_smiles": "CC(=O)OC1=CC=CC=C1C(=O)O",
  "isomeric_smiles": "CC(=O)OC1=CC=CC=C1C(=O)O",
  "inchi": "InChI=1S/C9H8O4/c1-6(10)13-8-5-3-2-4-7(8)9(11)12/h2-5H,1H3,(H,11,12)",
  "inchikey": "BSYNRYMUTXBXSQ-UHFFFAOYSA-N",
  "synonyms": ["acetylsalicylic acid", "ASA", "Ecotrin"],
  "cross_references": {
    "pubchem_compound": ["2244"],
    "chembl": ["CHEMBL:25"],
    "drugbank": ["DB00945"]
  }
}
```
**Estimated tokens**: ~115-150 (varies with SMILES length and synonyms)

### Slim Compound Mode
```json
{"id": "PubChem:CID2244", "name": "Aspirin", "molecular_formula": "C9H8O4"}
```
**Estimated tokens**: ~20

## Validation Rules

1. **CURIE Format**: Must match `^PubChem:CID\d+$`
2. **Score Range**: Must be 0.0 to 1.0
3. **Molecular Weight**: Positive float in Daltons
4. **InChIKey Format**: 27 characters, pattern `^[A-Z]{14}-[A-Z]{10}-[A-Z]$`
5. **Cross-Reference Keys**: Only use keys from 22-key registry
6. **Null Handling**: Omit missing fields, never use null

## References

- [ADR-001 v1.2 Appendix A: Cross-Reference Key Registry](../../docs/adr/accepted/adr-001-v1.2.md)
- [Constitution Principle III: Schema Determinism](../../.specify/memory/constitution.md)
- [PubChem Property Table](https://pubchem.ncbi.nlm.nih.gov/docs/pug-rest#section=Property-Tables)
- [ChEMBL Compound Model](../../src/lifesciences_mcp/models/compound.py)
