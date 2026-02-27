# Data Model: DrugBank MCP Server

**Phase**: 1 (Design & Contracts)
**Date**: 2025-12-22
**Source**: Derived from [spec.md](./spec.md) and [research.md](./research.md)

## Overview

This document defines the data models for DrugBank drug entities, following the Agentic Biolink schema (ADR-001 §4) and Constitution Principle III (Schema Determinism).

## Core Entities

### DrugSearchCandidate

**Purpose**: Lightweight search result for fuzzy drug discovery (Principle II: Fuzzy-to-Fact)

**Usage**: Returned by `search_drugs` tool to enable agent triage before strict lookup

**Schema**:
```python
from pydantic import BaseModel, Field

class DrugSearchCandidate(BaseModel):
    """Lightweight drug search result for fuzzy discovery.

    Token Budget: ~20-30 tokens in slim mode, ~40-50 tokens in full mode
    """

    id: str = Field(
        ...,
        description="DrugBank CURIE in format DrugBank:DBXXXXX (e.g., 'DrugBank:DB00945')",
        pattern=r"^DrugBank:DB\d{5}$"
    )

    name: str = Field(
        ...,
        description="Primary drug name (e.g., 'Aspirin', 'Metformin')",
        examples=["Aspirin", "Metformin", "Atorvastatin"]
    )

    drug_type: str | None = Field(
        None,
        description="Drug type classification",
        examples=["Small molecule", "Biotech"]
    )

    categories: list[str] | None = Field(
        None,
        description="Therapeutic categories",
        examples=[["Analgesics", "Antipyretics", "NSAIDs"]]
    )

    score: float = Field(
        ...,
        ge=0.0,
        le=1.0,
        description="Relevance score (1.0 = exact match, decreasing for partial matches)"
    )

    model_config = {
        "json_schema_extra": {
            "examples": [{
                "id": "DrugBank:DB00945",
                "name": "Aspirin",
                "drug_type": "Small molecule",
                "categories": ["Analgesics", "Antipyretics", "Anti-Inflammatory Agents, Non-Steroidal"],
                "score": 1.0
            }]
        }
    }
```

**Validation Rules**:
- `id` MUST match regex `^DrugBank:DB\d{5}$`
- `score` MUST be in range [0.0, 1.0]
- `name` is REQUIRED (never null)
- `drug_type` and `categories` MAY be None if not available

**Token Budget** (FR-023, FR-024):
- **Slim mode**: `id`, `name`, `drug_type`, `categories`, `score` → ~20-30 tokens
- **Full mode**: Same fields (no additional fields for search candidates) → ~40-50 tokens

---

### Drug

**Purpose**: Complete drug record with pharmacology and cross-references

**Usage**: Returned by `get_drug` for strict factual lookups

**Schema**:
```python
from pydantic import BaseModel, Field
from lifesciences_mcp.models.envelopes import CrossReferences

class Drug(BaseModel):
    """Complete DrugBank drug record with Agentic Biolink cross-references.

    Token Budget: ~115-300 tokens in full mode, ~20 tokens in slim mode
    """

    id: str = Field(
        ...,
        description="DrugBank CURIE in format DrugBank:DBXXXXX",
        pattern=r"^DrugBank:DB\d{5}$"
    )

    name: str = Field(
        ...,
        description="Primary drug name",
        examples=["Aspirin", "Metformin"]
    )

    drug_type: str | None = Field(
        None,
        description="Drug type (Small molecule, Biotech)",
        examples=["Small molecule", "Biotech"]
    )

    categories: list[str] | None = Field(
        None,
        description="Therapeutic categories"
    )

    description: str | None = Field(
        None,
        description="Therapeutic description and background"
    )

    indication: str | None = Field(
        None,
        description="Approved therapeutic uses and indications"
    )

    mechanism_of_action: str | None = Field(
        None,
        description="How the drug works at molecular/cellular level"
    )

    pharmacodynamics: str | None = Field(
        None,
        description="Effects on the body"
    )

    absorption: str | None = Field(
        None,
        description="Absorption characteristics"
    )

    metabolism: str | None = Field(
        None,
        description="Metabolic pathway information"
    )

    half_life: str | None = Field(
        None,
        description="Elimination half-life"
    )

    cross_references: CrossReferences = Field(
        default_factory=CrossReferences,
        description="Cross-references to other biological databases (22-key registry)"
    )

    model_config = {
        "json_schema_extra": {
            "examples": [{
                "id": "DrugBank:DB00945",
                "name": "Aspirin",
                "drug_type": "Small molecule",
                "categories": ["Analgesics", "Antipyretics", "Anti-Inflammatory Agents, Non-Steroidal"],
                "description": "The prototypical analgesic used in the treatment of mild to moderate pain.",
                "indication": "For the treatment of mild to moderate pain, inflammation, and fever.",
                "mechanism_of_action": "Irreversibly inhibits cyclooxygenase (COX) enzymes.",
                "half_life": "15-20 minutes (aspirin); 2-3 hours (salicylate)",
                "cross_references": {
                    "chembl": "CHEMBL:25",
                    "uniprot": ["UniProtKB:P23219", "UniProtKB:P35354"],
                    "kegg": "D00109",
                    "pubchem_compound": "2244"
                }
            }]
        }
    }
```

**Validation Rules** (from research.md R3):
- `id` MUST match regex `^DrugBank:DB\d{5}$`
- `name` is REQUIRED
- All other fields MAY be None
- `cross_references` defaults to empty CrossReferences (omit-if-null pattern)

**Token Budget** (FR-024, FR-025):
- **Slim mode**: `id`, `name`, `drug_type`, `categories` only → ~20 tokens
- **Full mode**: All fields including cross_references → ~115-300 tokens

---

## Cross-Reference Mappings

**Source**: Research findings (R4: Cross-Reference Mapping)

DrugBank cross-references map to our 22-key registry as follows:

### Supported Keys

| Registry Key | DrugBank Source | Example Value | Notes |
|--------------|-----------------|---------------|-------|
| `chembl` | ChEMBL ID | `"CHEMBL:25"` | CURIE format |
| `uniprot` | UniProt Targets | `["UniProtKB:P23219"]` | CURIE format, array |
| `kegg` | KEGG Drug | `"D00109"` | KEGG drug ID |
| `pubchem_compound` | PubChem CID | `"2244"` | CID only |
| `pubchem_substance` | PubChem SID | `"46505899"` | SID only |
| `pdb` | PDB Structures | `"1BNA"` | Structure ID |

### Omitted Keys (Never Populated by DrugBank)

The following keys from the 22-key registry will **never** be populated by DrugBank:
- `hgnc`, `ensembl_gene`, `ensembl_transcript` (gene-specific, not drug)
- `entrez`, `refseq` (genomic, not drug)
- `string`, `biogrid`, `stitch`, `iuphar` (interaction databases)
- `kegg_pathway`, `omim`, `orphanet`, `mondo`, `efo` (disease/pathway-specific)

**Constitution Compliance**: Keys are omitted entirely when no cross-reference exists (Constitution Principle III: never use `null` or empty string).

---

## CURIE Normalization Rules

**Source**: Research findings (R4: Cross-Reference Mapping)

When transforming DrugBank data to our schema, apply these normalization rules:

| Input Format | Output Format | Example |
|--------------|---------------|---------|
| DrugBank ID | `DrugBank:DBXXXXX` | `DB00945` → `DrugBank:DB00945` |
| ChEMBL ID | `CHEMBL:NNNNN` | `CHEMBL25` → `CHEMBL:25` |
| UniProt accession | `UniProtKB:ACCESSION` | `P23219` → `UniProtKB:P23219` |
| KEGG Drug | `D\d{5}` (unchanged) | `D00109` → `D00109` |
| PubChem CID | Numeric string | `2244` → `"2244"` |

**Regex Patterns**:
```python
DRUGBANK_ID_PATTERN = r"^DrugBank:DB\d{5}$"
CHEMBL_CURIE_PATTERN = r"^CHEMBL:\d+$"
UNIPROT_CURIE_PATTERN = r"^UniProtKB:[A-Z][A-Z0-9]{5,9}$"
```

---

## Field Optionality Matrix

| Field | DrugSearchCandidate | Drug (Full) | Drug (Slim) |
|-------|---------------------|-------------|-------------|
| `id` | ✅ Required | ✅ Required | ✅ Required |
| `name` | ✅ Required | ✅ Required | ✅ Required |
| `drug_type` | ⚠️ Optional | ⚠️ Optional | ✅ Included |
| `categories` | ⚠️ Optional | ⚠️ Optional | ✅ Included |
| `score` | ✅ Required | ❌ N/A | ❌ N/A |
| `description` | ❌ N/A | ⚠️ Optional | ❌ Excluded |
| `indication` | ❌ N/A | ⚠️ Optional | ❌ Excluded |
| `mechanism_of_action` | ❌ N/A | ⚠️ Optional | ❌ Excluded |
| `pharmacodynamics` | ❌ N/A | ⚠️ Optional | ❌ Excluded |
| `absorption` | ❌ N/A | ⚠️ Optional | ❌ Excluded |
| `metabolism` | ❌ N/A | ⚠️ Optional | ❌ Excluded |
| `half_life` | ❌ N/A | ⚠️ Optional | ❌ Excluded |
| `cross_references` | ❌ N/A | ✅ Default: {} | ❌ Excluded |

**Legend**:
- ✅ Required: Field always present
- ⚠️ Optional: Field may be None if DrugBank lacks data
- ❌ N/A: Field not applicable to this entity type
- Default: Field has default value if missing

---

## Validation Examples

### Valid DrugSearchCandidate

```json
{
  "id": "DrugBank:DB00945",
  "name": "Aspirin",
  "drug_type": "Small molecule",
  "categories": ["Analgesics", "Antipyretics"],
  "score": 1.0
}
```

### Valid Drug (Full Mode)

```json
{
  "id": "DrugBank:DB00945",
  "name": "Aspirin",
  "drug_type": "Small molecule",
  "categories": ["Analgesics", "Antipyretics", "Anti-Inflammatory Agents, Non-Steroidal"],
  "description": "The prototypical analgesic used in the treatment of mild to moderate pain.",
  "indication": "For the treatment of mild to moderate pain, inflammation, and fever.",
  "mechanism_of_action": "Irreversibly inhibits cyclooxygenase (COX) enzymes.",
  "half_life": "15-20 minutes (aspirin); 2-3 hours (salicylate)",
  "cross_references": {
    "chembl": "CHEMBL:25",
    "uniprot": ["UniProtKB:P23219", "UniProtKB:P35354"],
    "kegg": "D00109",
    "pubchem_compound": "2244"
  }
}
```

### Valid Drug (Slim Mode)

```json
{
  "id": "DrugBank:DB00945",
  "name": "Aspirin",
  "drug_type": "Small molecule",
  "categories": ["Analgesics", "Antipyretics"]
}
```

---

## Implementation Notes

### Pydantic Models Location

```
src/lifesciences_mcp/models/drug.py  # New file
```

### Import Structure

```python
# In src/lifesciences_mcp/models/__init__.py
from lifesciences_mcp.models.drug import (
    Drug,
    DrugSearchCandidate,
)

__all__ = [
    "Drug",
    "DrugSearchCandidate",
    # ... existing exports
]
```

### Runtime Validation

All models use Pydantic v2 validators:
- DrugBank ID format validation via `pattern` field constraint
- Optional fields default to None (not omitted)
- Cross-references use omit-if-null pattern (Constitution Principle III)

**Next Phase**: Generate API contracts (Phase 1 continuation)
