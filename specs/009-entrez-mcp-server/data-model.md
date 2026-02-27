# Data Model: NCBI Entrez MCP Server

**Phase**: 1 (Design & Contracts)
**Date**: 2026-01-01
**Source**: Derived from [spec.md](./spec.md) and [research.md](./research.md)

## Overview

This document defines the data models for NCBI Entrez gene entities and PubMed literature links, following the Agentic Biolink schema (ADR-001 Section 4) and Constitution Principle III (Schema Determinism).

## Core Entities

### GeneSearchCandidate

**Purpose**: Lightweight search result for fuzzy gene discovery (Principle II: Fuzzy-to-Fact)

**Usage**: Returned by `search_genes` tool to enable agent triage before strict lookup

**Schema**:
```python
from pydantic import BaseModel, Field

class GeneSearchCandidate(BaseModel):
    """Lightweight gene search result for fuzzy discovery.

    Token Budget: ~30-40 tokens in slim mode, ~60-80 tokens in full mode
    """

    id: str = Field(
        ...,
        description="NCBIGene CURIE in format NCBIGene:<numeric_id>",
        pattern=r"^NCBIGene:\d+$",
        examples=["NCBIGene:7157", "NCBIGene:672"]
    )

    symbol: str = Field(
        ...,
        description="Official gene symbol from NCBI Gene",
        examples=["TP53", "BRCA1", "EGFR"]
    )

    name: str = Field(
        ...,
        description="Full gene name/description",
        examples=["tumor protein p53", "BRCA1 DNA repair associated"]
    )

    description: str | None = Field(
        None,
        description="Additional gene description or designations",
        examples=["cellular tumor antigen p53; phosphoprotein p53; antigen NY-CO-13"]
    )

    organism: str = Field(
        ...,
        description="Scientific name of the organism",
        examples=["Homo sapiens", "Mus musculus"]
    )

    score: float = Field(
        ...,
        ge=0.0,
        le=1.0,
        description="Relevance score (1.0 = first result, decreasing by position)"
    )

    model_config = {
        "json_schema_extra": {
            "examples": [{
                "id": "NCBIGene:7157",
                "symbol": "TP53",
                "name": "tumor protein p53",
                "description": "cellular tumor antigen p53; phosphoprotein p53",
                "organism": "Homo sapiens",
                "score": 1.0
            }]
        }
    }
```

**Validation Rules**:
- `id` MUST match regex `^NCBIGene:\d+$`
- `score` MUST be in range [0.0, 1.0]
- `symbol`, `name`, `organism` are required
- `description` MAY be None if not available in esummary response

**Token Budget** (Constitution Principle IV):
- **Slim mode**: `id`, `symbol`, `name`, `score` only -> ~30 tokens
- **Full mode**: All fields including description and organism -> ~60-80 tokens

---

### EntrezGene

**Purpose**: Complete gene record with cross-references for strict factual lookups

**Usage**: Returned by `get_gene` for Phase 2 of Fuzzy-to-Fact protocol

**Schema**:
```python
from pydantic import BaseModel, Field
from lifesciences_mcp.models.envelopes import CrossReferences

class EntrezGene(BaseModel):
    """Complete NCBI Gene record with Agentic Biolink cross-references.

    Token Budget: ~115-300 tokens in full mode, ~25 tokens in slim mode
    """

    id: str = Field(
        ...,
        description="NCBIGene CURIE in format NCBIGene:<numeric_id>",
        pattern=r"^NCBIGene:\d+$"
    )

    symbol: str = Field(
        ...,
        description="Official gene symbol",
        examples=["TP53", "BRCA1"]
    )

    name: str = Field(
        ...,
        description="Full gene name/description",
        examples=["tumor protein p53"]
    )

    description: str | None = Field(
        None,
        description="Extended gene description from otherdesignations"
    )

    summary: str | None = Field(
        None,
        description="Functional summary of the gene (from Entrezgene_summary)"
    )

    map_location: str | None = Field(
        None,
        description="Cytogenetic map location",
        examples=["17p13.1", "17q21.31"]
    )

    chromosome: str | None = Field(
        None,
        description="Chromosome number or identifier",
        examples=["17", "X", "MT"]
    )

    aliases: list[str] | None = Field(
        None,
        description="Alternative gene symbols and names",
        examples=[["P53", "TRP53", "LFS1", "BCC7"]]
    )

    organism: str = Field(
        ...,
        description="Scientific name of the organism",
        examples=["Homo sapiens"]
    )

    taxon_id: int | None = Field(
        None,
        description="NCBI Taxonomy ID",
        examples=[9606, 10090]
    )

    cross_references: CrossReferences = Field(
        default_factory=CrossReferences,
        description="Cross-references to other biological databases (22-key registry)"
    )

    model_config = {
        "json_schema_extra": {
            "examples": [{
                "id": "NCBIGene:7157",
                "symbol": "TP53",
                "name": "tumor protein p53",
                "description": "cellular tumor antigen p53; phosphoprotein p53",
                "summary": "This gene encodes a tumor suppressor protein containing transcriptional activation, DNA binding, and oligomerization domains...",
                "map_location": "17p13.1",
                "chromosome": "17",
                "aliases": ["P53", "TRP53", "LFS1", "BCC7"],
                "organism": "Homo sapiens",
                "taxon_id": 9606,
                "cross_references": {
                    "hgnc": "HGNC:11998",
                    "ensembl_gene": "ENSG00000141510",
                    "uniprot": "UniProtKB:P04637",
                    "refseq": "NM_000546.6",
                    "omim": "191170"
                }
            }]
        }
    }
```

**Validation Rules** (from research.md R6):
- `id` MUST match regex `^NCBIGene:\d+$`
- `symbol` and `name` are required
- All optional fields MAY be None
- `cross_references` defaults to empty CrossReferences (omit-if-null pattern)
- `aliases` defaults to None (not empty list) if no aliases exist

**Token Budget** (FR-027, FR-028 equivalent):
- **Slim mode**: `id`, `symbol`, `name`, `organism` only -> ~25 tokens
- **Full mode**: All fields including summary and cross_references -> ~115-300 tokens

---

### CrossReferences (Extended for Entrez)

**Purpose**: Cross-database links following the 22-key Agentic Biolink registry

**Usage**: Nested in EntrezGene to provide links to external databases

**Schema** (extension of existing CrossReferences):
```python
from pydantic import BaseModel, Field, model_validator
from typing import Any

class CrossReferences(BaseModel):
    """Cross-references to biological databases following 22-key registry.

    Constitution Principle III: Omit keys entirely if no reference exists.
    Never use null or empty string.
    """

    hgnc: str | None = Field(
        None,
        description="HGNC gene nomenclature ID",
        pattern=r"^HGNC:\d+$",
        examples=["HGNC:11998"]
    )

    ensembl_gene: str | None = Field(
        None,
        description="Ensembl gene ID",
        pattern=r"^ENSG\d{11}$",
        examples=["ENSG00000141510"]
    )

    ensembl_transcript: str | None = Field(
        None,
        description="Ensembl transcript ID",
        pattern=r"^ENST\d{11}",
        examples=["ENST00000269305"]
    )

    uniprot: str | list[str] | None = Field(
        None,
        description="UniProt accession(s)",
        examples=["UniProtKB:P04637", ["UniProtKB:P04637", "UniProtKB:P04637-2"]]
    )

    entrez: str | None = Field(
        None,
        description="NCBI Entrez Gene ID (for cross-reference from other entities)",
        examples=["7157"]
    )

    refseq: str | list[str] | None = Field(
        None,
        description="RefSeq accession(s)",
        examples=["NM_000546.6", ["NM_000546.6", "NP_000537.3"]]
    )

    omim: str | None = Field(
        None,
        description="OMIM gene/phenotype ID",
        examples=["191170"]
    )

    kegg: str | None = Field(
        None,
        description="KEGG gene ID",
        examples=["hsa:7157"]
    )

    string: str | None = Field(
        None,
        description="STRING protein ID",
        examples=["9606.ENSP00000269305"]
    )

    biogrid: str | None = Field(
        None,
        description="BioGRID gene/protein ID",
        examples=["113418"]
    )

    @model_validator(mode="before")
    @classmethod
    def omit_null_values(cls, data: Any) -> Any:
        """Remove None, empty string, and empty list values.

        Constitution Principle III: Never include null cross-references.
        """
        if isinstance(data, dict):
            return {
                k: v for k, v in data.items()
                if v is not None and v != "" and v != []
            }
        return data

    model_config = {
        "extra": "allow"  # Allow additional keys from 22-key registry
    }
```

**NCBI to 22-Key Registry Mapping** (from research.md R3):

| NCBI Dbtag_db | Registry Key | Transformation |
|---------------|--------------|----------------|
| `HGNC` | `hgnc` | Add `HGNC:` prefix if missing |
| `Ensembl` | `ensembl_gene` | Use as-is (ENSG format) |
| `MIM` | `omim` | Use numeric value |
| `UniProtKB/Swiss-Prot` | `uniprot` | Add `UniProtKB:` prefix |
| `RefSeq` | `refseq` | Use as-is (NM_/NP_ format) |
| `STRING` | `string` | Use as-is (taxid.ENSP format) |
| `BioGRID` | `biogrid` | Use numeric value |

---

### PubMedLink (Simplified)

**Purpose**: Gene-to-literature association from NCBI elink

**Usage**: Returned by `get_pubmed_links` tool as a list of PubMed IDs

**Decision**: Return simple `list[str]` instead of complex model

**Rationale**:
- PubMed links are just IDs (integers as strings)
- No additional metadata from elink response
- Simpler output reduces token overhead
- Agents can call PubMed MCP server for full article details

**Response Format**:
```python
# get_pubmed_links returns:
list[str]  # e.g., ["39804234", "39804163", "39803982"]
```

If a formal model is needed later:
```python
class PubMedLink(BaseModel):
    """PubMed citation link for a gene."""

    pmid: str = Field(
        ...,
        description="PubMed ID",
        pattern=r"^\d+$",
        examples=["39804234"]
    )
```

---

## Cross-Reference Extraction Logic

**Source**: research.md R3 - XML Parsing

```python
def _build_cross_references(self, xrefs: dict[str, str]) -> CrossReferences:
    """Map NCBI Dbtag entries to CrossReferences model.

    Per Constitution Principle III: omit keys with no value.
    """
    result = {}

    # HGNC - add prefix if needed
    if "HGNC" in xrefs:
        hgnc_id = xrefs["HGNC"]
        result["hgnc"] = f"HGNC:{hgnc_id}" if not hgnc_id.startswith("HGNC:") else hgnc_id

    # Ensembl - use as-is
    if "Ensembl" in xrefs:
        result["ensembl_gene"] = xrefs["Ensembl"]

    # OMIM - use numeric value
    if "MIM" in xrefs:
        result["omim"] = xrefs["MIM"]

    # UniProt - add prefix if needed
    if "UniProtKB/Swiss-Prot" in xrefs:
        uniprot_id = xrefs["UniProtKB/Swiss-Prot"]
        result["uniprot"] = f"UniProtKB:{uniprot_id}" if not uniprot_id.startswith("UniProtKB:") else uniprot_id

    # RefSeq - may have multiple entries
    refseq_keys = [k for k in xrefs if k.startswith("RefSeq")]
    if refseq_keys:
        refseq_ids = [xrefs[k] for k in refseq_keys]
        result["refseq"] = refseq_ids[0] if len(refseq_ids) == 1 else refseq_ids

    return CrossReferences(**result)
```

---

## Field Optionality Matrix

| Field | GeneSearchCandidate | EntrezGene (Full) | EntrezGene (Slim) |
|-------|---------------------|-------------------|-------------------|
| `id` | Required | Required | Required |
| `symbol` | Required | Required | Required |
| `name` | Required | Required | Required |
| `description` | Optional | Optional | Excluded |
| `summary` | N/A | Optional | Excluded |
| `map_location` | N/A | Optional | Excluded |
| `chromosome` | N/A | Optional | Excluded |
| `aliases` | N/A | Optional | Excluded |
| `organism` | Required | Required | Required |
| `taxon_id` | N/A | Optional | Excluded |
| `score` | Required | N/A | N/A |
| `cross_references` | N/A | Default: {} | Excluded |

**Legend**:
- Required: Field always present
- Optional: Field may be None if NCBI lacks data
- N/A: Field not applicable to this entity type
- Default: Field has default value if missing
- Excluded: Field omitted in slim mode

---

## Validation Examples

### Valid GeneSearchCandidate

```json
{
  "id": "NCBIGene:7157",
  "symbol": "TP53",
  "name": "tumor protein p53",
  "description": "cellular tumor antigen p53",
  "organism": "Homo sapiens",
  "score": 1.0
}
```

### Valid EntrezGene (Full Mode)

```json
{
  "id": "NCBIGene:7157",
  "symbol": "TP53",
  "name": "tumor protein p53",
  "description": "cellular tumor antigen p53; phosphoprotein p53; antigen NY-CO-13",
  "summary": "This gene encodes a tumor suppressor protein containing transcriptional activation, DNA binding, and oligomerization domains...",
  "map_location": "17p13.1",
  "chromosome": "17",
  "aliases": ["P53", "TRP53", "LFS1", "BCC7"],
  "organism": "Homo sapiens",
  "taxon_id": 9606,
  "cross_references": {
    "hgnc": "HGNC:11998",
    "ensembl_gene": "ENSG00000141510",
    "uniprot": "UniProtKB:P04637",
    "refseq": "NM_000546.6",
    "omim": "191170"
  }
}
```

### Valid EntrezGene (Slim Mode)

```json
{
  "id": "NCBIGene:7157",
  "symbol": "TP53",
  "name": "tumor protein p53",
  "organism": "Homo sapiens"
}
```

### Valid PubMed Links Response

```json
["39804234", "39804163", "39803982", "39803897", "39803721"]
```

### Invalid Examples (Validation Errors)

```python
# Invalid: Missing NCBIGene prefix
{"id": "7157", ...}  # UNRESOLVED_ENTITY

# Invalid: Symbol missing
{"id": "NCBIGene:7157", "name": "...", ...}  # ValidationError

# Invalid: Score out of range
{"id": "NCBIGene:7157", ..., "score": 1.5}  # ValidationError
```

---

## Implementation Notes

### Pydantic Models Location

```
src/lifesciences_mcp/models/entrez.py  # New file
```

### Import Structure

```python
# In src/lifesciences_mcp/models/__init__.py
from lifesciences_mcp.models.entrez import (
    EntrezGene,
    GeneSearchCandidate,
    NCBI_GENE_CURIE_PATTERN,
)

__all__ = [
    "EntrezGene",
    "GeneSearchCandidate",
    "NCBI_GENE_CURIE_PATTERN",
    # ... existing exports
]
```

### Runtime Validation

All models use Pydantic v2 validators:
- NCBIGene CURIE format validation via `pattern` field constraint
- Optional fields default to None (not omitted from schema)
- Cross-references use omit-if-null pattern (Constitution Principle III)
- Score range validation via `ge`/`le` constraints

**Next Phase**: Generate API contracts (Phase 1 continuation)
