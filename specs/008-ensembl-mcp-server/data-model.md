# Phase 1 Data Model: Ensembl MCP Server

**Feature**: 008-ensembl-mcp-server
**Date**: 2026-01-01
**Status**: Draft

## Entity Overview

```
+---------------------------------------------------------------------+
|                         Tool Outputs                                 |
+---------------------------------------------------------------------+
|                                                                     |
|  search_genes()                    get_gene()   get_transcript()    |
|       |                                 |             |             |
|       v                                 v             v             |
|  PaginationEnvelope<GeneSearchCandidate>   EnsemblGene   EnsemblTranscript
|       |                                 |             |             |
|       +----------------+----------------+             |             |
|                        |                              |             |
|                        v                              |             |
|                CrossReferences <----------------------+             |
|                                                                     |
+---------------------------------------------------------------------+
|                         Error Handling                              |
|                                                                     |
|  ErrorEnvelope (all error responses)                                |
|                                                                     |
+---------------------------------------------------------------------+
```

## Entities

### EnsemblGene

The complete gene record from Ensembl with Agentic Biolink cross-references.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | str | Yes | Ensembl Gene ID (e.g., "ENSG00000141510") |
| symbol | str | Yes | Official gene symbol (e.g., "TP53") |
| name | str | Yes | Gene description/name |
| biotype | str | Yes | Gene biotype (e.g., "protein_coding", "lncRNA") |
| species | str | Yes | Species name (e.g., "homo_sapiens") |
| assembly_name | str | Yes | Genome assembly (e.g., "GRCh38") |
| chromosome | str | Yes | Chromosome/seq_region_name (e.g., "17") |
| start | int | Yes | Genomic start position |
| end | int | Yes | Genomic end position |
| strand | int | Yes | Strand (+1 or -1) |
| transcripts | list[str] | No | List of transcript IDs for this gene |
| cross_references | CrossReferences | Yes | External database identifiers |

**Validation Rules**:
- `id` must match regex `^ENSG\d{11}$`
- `strand` must be +1 or -1
- `start` must be < `end`
- `cross_references` must omit keys with no value (never null)

**Biotype Values** (common):
- `protein_coding` - Protein-coding gene
- `lncRNA` - Long non-coding RNA
- `pseudogene` - Pseudogene
- `miRNA` - MicroRNA
- `snRNA` - Small nuclear RNA
- `snoRNA` - Small nucleolar RNA

### EnsemblTranscript

Transcript-level information from Ensembl.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | str | Yes | Ensembl Transcript ID (e.g., "ENST00000269305") |
| display_name | str | Yes | Transcript display name (e.g., "TP53-201") |
| biotype | str | Yes | Transcript biotype (e.g., "protein_coding") |
| parent_gene | str | Yes | Parent Ensembl Gene ID |
| is_canonical | bool | Yes | Whether this is the canonical transcript |
| species | str | Yes | Species name |
| assembly_name | str | Yes | Genome assembly |
| chromosome | str | Yes | Chromosome |
| start | int | Yes | Genomic start position |
| end | int | Yes | Genomic end position |
| strand | int | Yes | Strand (+1 or -1) |
| cross_references | CrossReferences | Yes | External database identifiers |

**Validation Rules**:
- `id` must match regex `^ENST\d{11}$`
- `parent_gene` must match regex `^ENSG\d{11}$`
- `strand` must be +1 or -1
- `is_canonical` defaults to False

### GeneSearchCandidate

Lightweight gene representation for fuzzy search results.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | str | Yes | Ensembl Gene ID |
| symbol | str | Yes | Official gene symbol |
| name | str | Yes | Gene description |
| biotype | str | Yes | Gene biotype |
| species | str | Yes | Species name |
| score | float | Yes | Relevance score (0.0-1.0) |

**Slim Mode**: When `slim=True`, only these 6 fields are returned (~25 tokens).
Full EnsemblGene record is ~150-350 tokens depending on cross-references.

**Validation Rules**:
- `id` must match regex `^ENSG\d{11}$`
- `score` must be between 0.0 and 1.0

### CrossReferences

Flattened object containing external database identifiers per ADR-001 Appendix A.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| hgnc | str | No | HGNC ID with prefix (e.g., "HGNC:11998") |
| ensembl_gene | str | No | Ensembl Gene ID (self-reference for gene entities) |
| ensembl_transcript | list[str] | No | Ensembl transcript IDs |
| uniprot | list[str] | No | UniProt accessions |
| entrez | str | No | NCBI Entrez gene ID |
| refseq | list[str] | No | RefSeq accessions |
| omim | str | No | OMIM ID |
| pdb | list[str] | No | PDB structure IDs |
| kegg | str | No | KEGG gene ID |
| chembl | str | No | ChEMBL target ID |
| string | str | No | STRING protein ID |
| biogrid | str | No | BioGRID protein ID |

**Validation Rules**:
- Each field must match the regex defined in ADR-001 Appendix A
- Omit keys entirely if no value (do not set to null or empty string)
- List fields may contain multiple values (isoforms, transcripts, structures)

**Regex Patterns** (from ADR-001):

| Key | Regex |
|-----|-------|
| hgnc | `^HGNC:\d+$` |
| ensembl_gene | `^ENSG\d{11}$` |
| ensembl_transcript | `^ENST\d{11}$` |
| uniprot | `^[A-Z0-9]{6,10}$` |
| entrez | `^\d+$` |
| refseq | `^[NX][MR]_\d+$` |
| omim | `^\d{6}$` |
| pdb | `^[0-9][A-Z0-9]{3}$` |
| kegg | `^[a-z]{3,4}:\d+$` |
| chembl | `^CHEMBL\d+$` |
| string | `^\d+\.[A-Za-z0-9]+$` |
| biogrid | `^\d+$` |

### PaginationEnvelope

Generic wrapper for paginated list responses per ADR-001 Section 8.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| items | list[T] | Yes | Data payload (GeneSearchCandidate) |
| pagination | Pagination | Yes | Pagination metadata |

**Pagination subobject**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| cursor | str | null | No | Opaque cursor for next page; null = end |
| total_count | int | null | No | Total items if known |
| page_size | int | Yes | Items per page (default 50) |

### ErrorEnvelope

Structured error response for agent self-recovery per ADR-001 Section 8.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| success | bool | Yes | Always `false` for errors |
| error | ErrorDetail | Yes | Error details |

**ErrorDetail subobject**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| code | str | Yes | Error code from registry |
| message | str | Yes | Human-readable message |
| recovery_hint | str | Yes | Agent-actionable guidance |
| invalid_input | str | No | The input that caused the error |

**Error Codes** (from ADR-001 Appendix B):

| Code | Meaning |
|------|---------|
| UNRESOLVED_ENTITY | Invalid ID format passed to strict tool |
| ENTITY_NOT_FOUND | Valid ID format but not in database |
| AMBIGUOUS_QUERY | Search query too broad (>100 results) or too short |
| RATE_LIMITED | Ensembl API 429 response |
| UPSTREAM_ERROR | Ensembl API 500/502/503 |

## State Transitions

This feature is stateless. All data is fetched live from Ensembl REST API.

## Relationships

```
GeneSearchCandidate --(expands to)--> EnsemblGene
                                          |
                                          +--(contains)--> CrossReferences
                                          |
                                          +--(has many)--> EnsemblTranscript
                                                               |
                                                               +--(contains)--> CrossReferences
```

- `search_genes` returns `GeneSearchCandidate` list
- `get_gene` returns full `EnsemblGene` with embedded `CrossReferences` and transcript list
- `get_transcript` returns full `EnsemblTranscript` with embedded `CrossReferences`
- Agent workflow: search --> select candidate --> lookup full record

## Token Budgeting

| Entity | Slim Mode | Full Mode |
|--------|-----------|-----------|
| GeneSearchCandidate | N/A | ~25 tokens |
| EnsemblGene | ~50 tokens (no xrefs) | ~150-350 tokens |
| EnsemblTranscript | ~50 tokens (no xrefs) | ~100-200 tokens |
| CrossReferences | Omitted | ~50-200 tokens |

**Recommendations**:
- Use `slim=True` for search results when discovering candidates
- Use full mode only when detailed cross-references are needed
- Page size should default to 50 (max 100 for searches)

## Pydantic Model Sketch

```python
import re
from pydantic import BaseModel, Field, field_validator

ENSEMBL_GENE_PATTERN = re.compile(r"^ENSG\d{11}$")
ENSEMBL_TRANSCRIPT_PATTERN = re.compile(r"^ENST\d{11}$")

class GeneSearchCandidate(BaseModel):
    """Lightweight gene representation for fuzzy search results."""

    id: str = Field(description="Ensembl Gene ID")
    symbol: str = Field(description="Official gene symbol")
    name: str = Field(description="Gene description")
    biotype: str = Field(description="Gene biotype")
    species: str = Field(description="Species name")
    score: float = Field(ge=0.0, le=1.0, description="Relevance score")

    @field_validator("id")
    @classmethod
    def validate_ensembl_gene_id(cls, v: str) -> str:
        if not ENSEMBL_GENE_PATTERN.match(v):
            raise ValueError(f"Invalid Ensembl Gene ID format: {v}")
        return v


class EnsemblGene(BaseModel):
    """Complete gene record from Ensembl."""

    id: str = Field(description="Ensembl Gene ID")
    symbol: str = Field(description="Official gene symbol")
    name: str = Field(description="Gene description")
    biotype: str = Field(description="Gene biotype")
    species: str = Field(description="Species name")
    assembly_name: str = Field(description="Genome assembly")
    chromosome: str = Field(description="Chromosome")
    start: int = Field(description="Genomic start position")
    end: int = Field(description="Genomic end position")
    strand: int = Field(description="Strand (+1 or -1)")
    transcripts: list[str] | None = Field(default=None, description="Transcript IDs")
    cross_references: CrossReferences = Field(
        default_factory=CrossReferences,
        description="External database identifiers"
    )

    @field_validator("id")
    @classmethod
    def validate_ensembl_gene_id(cls, v: str) -> str:
        if not ENSEMBL_GENE_PATTERN.match(v):
            raise ValueError(f"Invalid Ensembl Gene ID format: {v}")
        return v

    @field_validator("strand")
    @classmethod
    def validate_strand(cls, v: int) -> int:
        if v not in (1, -1):
            raise ValueError(f"Strand must be 1 or -1, got: {v}")
        return v


class EnsemblTranscript(BaseModel):
    """Transcript record from Ensembl."""

    id: str = Field(description="Ensembl Transcript ID")
    display_name: str = Field(description="Transcript display name")
    biotype: str = Field(description="Transcript biotype")
    parent_gene: str = Field(description="Parent Ensembl Gene ID")
    is_canonical: bool = Field(default=False, description="Is canonical transcript")
    species: str = Field(description="Species name")
    assembly_name: str = Field(description="Genome assembly")
    chromosome: str = Field(description="Chromosome")
    start: int = Field(description="Genomic start position")
    end: int = Field(description="Genomic end position")
    strand: int = Field(description="Strand (+1 or -1)")
    cross_references: CrossReferences = Field(
        default_factory=CrossReferences,
        description="External database identifiers"
    )

    @field_validator("id")
    @classmethod
    def validate_ensembl_transcript_id(cls, v: str) -> str:
        if not ENSEMBL_TRANSCRIPT_PATTERN.match(v):
            raise ValueError(f"Invalid Ensembl Transcript ID format: {v}")
        return v
```
