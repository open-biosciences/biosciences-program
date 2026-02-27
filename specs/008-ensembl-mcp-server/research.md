# Ensembl REST API Research

**Date**: 2026-01-01
**Purpose**: Phase 0 research for Ensembl MCP Server implementation

## R1: Ensembl REST API Endpoints

### Base URL

```
https://rest.ensembl.org
```

**API Version**: 15.10 (as of December 2025)

### Key Endpoints for Gene Discovery

**1. Lookup by Ensembl ID** (Primary - for `get_gene`):
```
GET /lookup/id/{id}
```

Parameters:
- `id` (required): Ensembl stable ID (e.g., ENSG00000157764)
- `expand` (optional): Include connected features (transcripts, exons)
- `species` (optional): Species name/alias (homo_sapiens, human)
- `format` (optional): Output format (full, condensed)
- `phenotypes` (optional): Include phenotypes (genes only)

Example:
```bash
curl "https://rest.ensembl.org/lookup/id/ENSG00000141510?content-type=application/json&expand=1"
```

Response (simplified):
```json
{
  "id": "ENSG00000141510",
  "display_name": "TP53",
  "description": "tumor protein p53 [Source:HGNC Symbol;Acc:HGNC:11998]",
  "biotype": "protein_coding",
  "object_type": "Gene",
  "species": "homo_sapiens",
  "assembly_name": "GRCh38",
  "seq_region_name": "17",
  "start": 7661779,
  "end": 7687538,
  "strand": -1,
  "Transcript": [
    {
      "id": "ENST00000269305",
      "display_name": "TP53-201",
      "biotype": "protein_coding",
      "is_canonical": 1
    }
  ]
}
```

**2. Lookup by Symbol** (Primary - for `search_genes`):
```
GET /xrefs/symbol/{species}/{symbol}
```

Parameters:
- `species` (required): Species name (e.g., homo_sapiens)
- `symbol` (required): Gene symbol (e.g., BRCA1)
- `external_db` (optional): Filter by external database (e.g., HGNC)
- `object_type` (optional): Filter by feature type (gene, transcript)

Example:
```bash
curl "https://rest.ensembl.org/xrefs/symbol/homo_sapiens/BRCA1?content-type=application/json"
```

Response:
```json
[
  {
    "type": "gene",
    "id": "ENSG00000012048"
  }
]
```

**3. Cross-References by ID** (For populating cross_references):
```
GET /xrefs/id/{id}
```

Parameters:
- `id` (required): Ensembl stable ID
- `all_levels` (optional): Retrieve from linked features
- `external_db` (optional): Filter by database (HGNC, RefSeq, etc.)

Example:
```bash
curl "https://rest.ensembl.org/xrefs/id/ENSG00000141510?content-type=application/json"
```

Response (simplified):
```json
[
  {
    "dbname": "HGNC",
    "primary_id": "11998",
    "display_id": "TP53",
    "description": "tumor protein p53"
  },
  {
    "dbname": "UniProtKB/Swiss-Prot",
    "primary_id": "P04637",
    "display_id": "P53_HUMAN"
  },
  {
    "dbname": "EntrezGene",
    "primary_id": "7157",
    "display_id": "TP53"
  },
  {
    "dbname": "RefSeq_mRNA",
    "primary_id": "NM_000546",
    "display_id": "NM_000546.6"
  }
]
```

**4. Lookup by Name** (Alternative search mechanism):
```
GET /xrefs/name/{species}/{name}
```

Parameters:
- `species` (required): Species name
- `name` (required): Primary accession or display label

Useful for fuzzy matching when symbol lookup fails.

### Authentication

**Decision**: No authentication required for public Ensembl REST API
**Rationale**: All endpoints work without API keys or tokens

---

## R2: Rate Limiting Policy

### Official Ensembl Rate Limits

| Metric | Value | Notes |
|--------|-------|-------|
| **Requests per Hour** | 55,000 | Hard limit per IP |
| **Requests per Second** | ~15 | Calculated from hourly limit |
| **Period** | 3,600 seconds | 1 hour rolling window |

### Rate Limit Headers

Ensembl provides rate limit status via HTTP headers:

| Header | Description | Example |
|--------|-------------|---------|
| `X-RateLimit-Limit` | Total allowed requests | 55000 |
| `X-RateLimit-Period` | Time window (seconds) | 3600 |
| `X-RateLimit-Remaining` | Requests left | 54985 |
| `X-RateLimit-Reset` | Seconds until quota reset | 3500 |
| `Retry-After` | Wait time when rate-limited | 60.5 |

### Rate Limit Response

When rate limited, Ensembl returns:
- **HTTP Status**: 429 Too Many Requests
- **Retry-After Header**: Floating-point seconds to wait
- **X-RateLimit-Remaining**: 0

### Constitution Compliance

**Constitution v1.1 requires**: 10 req/s baseline with exponential backoff
**Ensembl allows**: 15 req/s (55,000/3600)

**Decision**: Use 15 req/s for Ensembl-specific limit (API allows it), but implement full exponential backoff and thundering herd prevention per Constitution.

**Implementation**:
```python
RATE_LIMIT_DELAY = 1.0 / 15  # 66.67ms between requests (15 req/s)
MAX_RETRIES = 3
```

**Backoff Strategy**:
1. Read `Retry-After` header if present
2. Otherwise use exponential backoff: 2^attempt seconds
3. Re-check timing after lock acquisition (thundering herd prevention)

---

## R3: ID Format Validation

### Ensembl Stable ID Formats

| ID Type | Pattern | Regex | Example |
|---------|---------|-------|---------|
| Gene | ENSG + 11 digits | `^ENSG\d{11}$` | ENSG00000141510 |
| Transcript | ENST + 11 digits | `^ENST\d{11}$` | ENST00000269305 |
| Protein | ENSP + 11 digits | `^ENSP\d{11}$` | ENSP00000269305 |
| Exon | ENSE + 11 digits | `^ENSE\d{11}$` | ENSE00000936617 |

**Note**: Ensembl IDs can have version suffixes (e.g., ENSG00000141510.12), but we normalize to base ID.

### CURIE Format (for this implementation)

**Decision**: Use bare Ensembl IDs as identifiers (not CURIE format)
**Rationale**:
- Ensembl IDs are globally unique
- ADR-001 cross-reference registry uses `ensembl_gene` and `ensembl_transcript` without prefix
- Matches existing project patterns (e.g., gene.py CrossReferences uses bare IDs)

**Validation Regex**:
```python
ENSEMBL_GENE_PATTERN = re.compile(r"^ENSG\d{11}$")
ENSEMBL_TRANSCRIPT_PATTERN = re.compile(r"^ENST\d{11}$")
```

### Version Handling

Ensembl IDs may include versions (ENSG00000141510.12). Strategy:
- Accept versioned IDs but strip version before lookup
- Return unversioned IDs in responses
- Version stripping: `ensembl_id.split('.')[0]`

---

## R4: Cross-Reference Field Mappings

### Test Data: ENSG00000141510 (Human TP53)

Tested with:
```bash
curl "https://rest.ensembl.org/xrefs/id/ENSG00000141510?content-type=application/json"
```

### Cross-Reference Structure

Ensembl returns cross-references as array of objects:
```json
[
  {
    "dbname": "HGNC",
    "primary_id": "11998",
    "display_id": "TP53",
    "synonyms": [],
    "description": "tumor protein p53",
    "info_type": "DIRECT",
    "info_text": ""
  }
]
```

### Field Mapping Table

| Our 22-Key Registry | Ensembl dbname | Transform | Notes |
|---------------------|----------------|-----------|-------|
| `hgnc` | `HGNC` | `HGNC:` + primary_id | e.g., "HGNC:11998" |
| `uniprot` | `Uniprot/SWISSPROT`, `Uniprot/SPTREMBL` | primary_id | e.g., "P04637" |
| `entrez` | `EntrezGene` | primary_id | e.g., "7157" |
| `refseq` | `RefSeq_mRNA`, `RefSeq_peptide` | primary_id | e.g., "NM_000546" |
| `omim` | `MIM_GENE`, `MIM_MORBID` | primary_id | e.g., "191170" |
| `pdb` | `PDB` | primary_id | e.g., "1TUP" |
| `kegg` | `KEGG_Enzyme` | primary_id | e.g., "hsa:7157" |
| `chembl` | `ChEMBL` | primary_id | e.g., "CHEMBL3927" |
| `string` | N/A | Derived from species + ENSP | e.g., "9606.ENSP00000269305" |

### Cross-Reference Extraction Logic

```python
def _map_cross_references(xrefs: list[dict]) -> CrossReferences:
    """Map Ensembl xrefs to 22-key registry."""
    refs = {}

    for xref in xrefs:
        dbname = xref.get("dbname", "")
        primary_id = xref.get("primary_id", "")

        if dbname == "HGNC":
            refs["hgnc"] = f"HGNC:{primary_id}"
        elif dbname in ("Uniprot/SWISSPROT", "Uniprot/SPTREMBL"):
            refs.setdefault("uniprot", []).append(primary_id)
        elif dbname == "EntrezGene":
            refs["entrez"] = primary_id
        elif dbname.startswith("RefSeq"):
            refs.setdefault("refseq", []).append(primary_id)
        elif dbname in ("MIM_GENE", "MIM_MORBID"):
            refs["omim"] = primary_id
        elif dbname == "PDB":
            refs.setdefault("pdb", []).append(primary_id)
        # ... etc

    return CrossReferences(**refs)
```

### Null Handling

Per Constitution III and ADR-001: Omit keys entirely if no cross-reference exists (never use null or empty string).

---

## R5: Error Response Formats

### Ensembl Error Responses

**Invalid ID format**:
```bash
curl "https://rest.ensembl.org/lookup/id/INVALID?content-type=application/json"
```
Response: HTTP 400 Bad Request
```json
{
  "error": "ID 'INVALID' not found"
}
```

**Not found (valid format, no record)**:
```bash
curl "https://rest.ensembl.org/lookup/id/ENSG99999999999?content-type=application/json"
```
Response: HTTP 400 Bad Request
```json
{
  "error": "ID 'ENSG99999999999' not found"
}
```

**Rate limited**:
Response: HTTP 429 Too Many Requests
Headers: `Retry-After: 60.0`
```json
{
  "error": "You have been rate-limited; wait and retry"
}
```

**Server error**:
Response: HTTP 500/502/503

### Error Mapping Table

| HTTP Status | Ensembl Error | Our ErrorEnvelope Code | Recovery Hint |
|-------------|---------------|------------------------|---------------|
| 400 (not found) | "ID 'X' not found" | `ENTITY_NOT_FOUND` | "The Ensembl ID '{id}' does not exist. Verify the ID format or use search_genes to find valid IDs." |
| 400 (invalid format) | "ID 'X' not found" + format check | `UNRESOLVED_ENTITY` | "Invalid Ensembl ID format. Use search_genes to find valid Ensembl Gene IDs (format: ENSG + 11 digits)." |
| 429 | "rate-limited" | `RATE_LIMITED` | "Rate limit exceeded. Wait {retry_after} seconds before retrying." |
| 500/502/503 | Server errors | `UPSTREAM_ERROR` | "Ensembl API is temporarily unavailable. Retry after a few seconds." |

---

## R6: Pagination Strategy

### Ensembl API Pagination

**Key Finding**: Ensembl's lookup and xrefs endpoints do NOT support server-side pagination. They return all matching results in a single response.

**Search via xrefs/symbol**: Returns all Ensembl objects linked to the symbol (typically 1-10 results)

**Implication**: Client-side pagination is required, similar to HGNC implementation.

### Pagination Implementation

```python
# Client-side pagination (same pattern as HGNC)
def _paginate_results(
    items: list,
    cursor: str | None,
    page_size: int
) -> PaginationEnvelope:
    """Apply client-side pagination to results."""
    offset = 0
    if cursor:
        cursor_data = json.loads(base64.b64decode(cursor))
        offset = cursor_data.get("offset", 0)

    page_start = offset
    page_end = offset + page_size
    page_items = items[page_start:page_end]

    next_cursor = None
    if page_end < len(items):
        next_cursor = base64.b64encode(
            json.dumps({"offset": page_end}).encode()
        ).decode()

    return PaginationEnvelope.create(
        items=page_items,
        cursor=next_cursor,
        total_count=len(items),
        page_size=page_size,
    )
```

---

## R7: Species Handling

### Default Species

**Decision**: Default to human (Homo sapiens) for species parameter
**Rationale**:
- Most common use case for drug discovery/repurposing
- Reduces ambiguity in search results
- Matches existing patterns in HGNC (human-only) and UniProt (human-prioritized)

### Species Identifiers

Ensembl accepts multiple formats:
- Scientific name: `homo_sapiens`
- Common name: `human`
- NCBI Taxonomy ID: Not directly supported (unlike STRING)

**Implementation**: Use `homo_sapiens` as default, accept common aliases.

### Species Validation

```python
SPECIES_ALIASES = {
    "human": "homo_sapiens",
    "mouse": "mus_musculus",
    "rat": "rattus_norvegicus",
    "zebrafish": "danio_rerio",
    "drosophila": "drosophila_melanogaster",
    "c. elegans": "caenorhabditis_elegans",
}

def _normalize_species(species: str) -> str:
    """Normalize species name to Ensembl format."""
    lower = species.lower().strip()
    return SPECIES_ALIASES.get(lower, lower)
```

---

## Summary of Decisions

| Research Task | Decision | Rationale |
|---------------|----------|-----------|
| **R1: API Endpoints** | Use /lookup/id for get_gene, /xrefs/symbol for search_genes, /xrefs/id for cross-refs | Standard Ensembl REST endpoints, no auth |
| **R2: Rate Limiting** | 15 req/s with exponential backoff and Retry-After header handling | Ensembl allows 15 req/s; Constitution requires backoff |
| **R3: ID Formats** | ENSG/ENST + 11 digits; strip versions; validate with regex | Ensembl stable ID format, version-agnostic |
| **R4: Cross-References** | Map dbname to 22-key registry; omit null keys | Direct field mapping, Constitution III compliance |
| **R5: Error Responses** | Map HTTP 400/429/5xx to ErrorEnvelope codes | Consistent error handling per ADR-001 |
| **R6: Pagination** | Client-side pagination (API returns all results) | Same pattern as HGNC; Ensembl doesn't support server-side paging |
| **R7: Species** | Default to homo_sapiens; accept common aliases | Human-centric for drug discovery |

---

## Next Steps

With research complete, proceed to:
1. **Phase 1 Design**: Create data-model.md, contracts, quickstart.md
2. **Implementation**: EnsemblClient, Ensembl models, Ensembl MCP server

All research findings documented here should be referenced during implementation.

---

## References

- [Ensembl REST API Documentation](https://rest.ensembl.org)
- [Ensembl REST API Rate Limits](https://github.com/Ensembl/ensembl-rest/wiki/Rate-Limits)
- [Ensembl REST API HTTP Headers](https://github.com/Ensembl/ensembl-rest/wiki/HTTP-Headers)
- [ADR-001 v1.2](../../docs/adr/accepted/adr-001-v1.2.md)
- [Constitution v1.1](../../.specify/memory/constitution.md)
