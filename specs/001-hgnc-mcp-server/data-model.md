# Phase 1 Data Model: HGNC MCP Server

**Feature**: 001-hgnc-mcp-server
**Date**: 2025-12-21
**Status**: Complete

## Entity Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Tool Outputs                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  search_genes()                    get_gene()                   │
│       │                                 │                       │
│       ▼                                 ▼                       │
│  PaginationEnvelope<SearchCandidate>   Gene                    │
│       │                                 │                       │
│       └──────────────┬──────────────────┘                       │
│                      │                                          │
│                      ▼                                          │
│              CrossReferences                                    │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                         Error Handling                          │
│                                                                 │
│  ErrorEnvelope (all error responses)                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Entities

### Gene

The complete gene record from HGNC with Agentic Biolink cross-references.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | str | Yes | HGNC CURIE (e.g., "HGNC:1100") |
| symbol | str | Yes | Official gene symbol (e.g., "BRCA1") |
| name | str | Yes | Full gene name |
| status | str | Yes | Approval status: "Approved", "Withdrawn" |
| locus_type | str | No | Gene type (e.g., "gene with protein product") |
| locus_group | str | No | Gene group (e.g., "protein-coding gene") |
| location | str | No | Chromosomal location (e.g., "17q21.31") |
| alias_symbols | list[str] | No | Alternative symbols |
| alias_names | list[str] | No | Alternative names |
| prev_symbols | list[str] | No | Previous symbols |
| prev_names | list[str] | No | Previous names |
| cross_references | CrossReferences | Yes | External database identifiers |

**Validation Rules**:
- `id` must match regex `^HGNC:\d+$`
- `status` must be one of: "Approved", "Withdrawn", "Entry Withdrawn"
- `cross_references` must omit keys with no value (never null)

### CrossReferences

Flattened object containing external database identifiers per ADR-001 Appendix A.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| ensembl_gene | str | No | Ensembl gene ID (e.g., "ENSG00000012048") |
| ensembl_transcript | list[str] | No | Ensembl transcript IDs |
| uniprot | list[str] | No | UniProt accessions |
| entrez | str | No | NCBI Entrez gene ID |
| refseq | list[str] | No | RefSeq accessions |
| omim | str | No | OMIM ID |
| chembl | str | No | ChEMBL target ID |
| pubchem_compound | str | No | PubChem compound ID |

**Validation Rules**:
- Each field must match the regex defined in ADR-001 Appendix A
- Omit keys entirely if no value (do not set to null or empty string)
- List fields may contain multiple values (isoforms, transcripts)

**Regex Patterns** (from ADR-001):

| Key | Regex |
|-----|-------|
| ensembl_gene | `^ENSG\d{11}$` |
| ensembl_transcript | `^ENST\d{11}$` |
| uniprot | `^[A-Z0-9]{6,10}$` |
| entrez | `^\d+$` |
| refseq | `^[NX][MR]_\d+$` |
| omim | `^\d{6}$` |
| chembl | `^CHEMBL\d+$` |

### SearchCandidate

Lightweight gene representation for fuzzy search results.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| id | str | Yes | HGNC CURIE |
| symbol | str | Yes | Official gene symbol |
| name | str | Yes | Full gene name |
| score | float | Yes | Relevance score (0.0-1.0) |

**Slim Mode**: When `slim=True`, only these 4 fields are returned (~20 tokens).
Full Gene record is ~115-300 tokens depending on cross-references.

### PaginationEnvelope

Generic wrapper for paginated list responses per ADR-001 §8.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| items | list[T] | Yes | Data payload (SearchCandidate or Gene) |
| pagination | Pagination | Yes | Pagination metadata |

**Pagination subobject**:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| cursor | str | null | No | Opaque cursor for next page; null = end |
| total_count | int | null | No | Total items if known |
| page_size | int | Yes | Items per page (default 50) |

### ErrorEnvelope

Structured error response for agent self-recovery per ADR-001 §8.

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
| UNRESOLVED_ENTITY | Raw string passed to strict tool |
| ENTITY_NOT_FOUND | Valid CURIE but ID not in database |
| AMBIGUOUS_QUERY | Search returned >100 results |
| RATE_LIMITED | Upstream API 429 |
| UPSTREAM_ERROR | Upstream API 500/502 |

## State Transitions

This feature is stateless. All data is fetched live from HGNC API.

## Relationships

```
SearchCandidate ──(expands to)──> Gene
                                    │
                                    └──(contains)──> CrossReferences
```

- `search_genes` returns `SearchCandidate` list
- `get_gene` returns full `Gene` with embedded `CrossReferences`
- Agent workflow: search → select candidate → lookup full record
