# Research: Open Targets MCP Server

**Phase**: 0 (Research & Technical Decisions)
**Date**: 2025-12-22
**Source**: Open Targets Platform Documentation + GraphQL API exploration

## R1: Open Targets GraphQL API Structure

**Decision**: Use Open Targets Platform GraphQL API v4 (https://api.platform.opentargets.org/api/v4/graphql)

**Rationale**:
- Official production API with public access (no API key required for basic usage)
- GraphQL introspection available for schema exploration
- Well-documented with examples in platform docs

**GraphQL Query Patterns**:

**Search Query** (for search_targets):
```graphql
query SearchTargets($queryString: String!, $page: PageInput!) {
  search(queryString: $queryString, page: $page) {
    hits {
      id  # Ensembl gene ID (ENSG...)
      name  # Approved symbol (e.g., "TP53")
      approvedName  # Full name
      biotype  # Gene biotype
    }
    total
  }
}
```

**Target Query** (for get_target):
```graphql
query GetTarget($ensemblId: String!) {
  target(ensemblId: $ensemblId) {
    id
    approvedSymbol
    approvedName
    biotype
    description
    crossReferences {
      source
      id
    }
  }
}
```

**Associations Query** (for get_associations):
```graphql
query GetAssociations($ensemblId: String!, $page: PageInput!) {
  target(ensemblId: $ensemblId) {
    associatedDiseases(page: $page) {
      rows {
        disease {
          id  # EFO ID
          name
        }
        score
        datatypeScores {
          id
          score
        }
      }
      count
    }
  }
}
```

**Alternatives Considered**:
- REST API (exists but less flexible for field selection)
- Batch API (not available - GraphQL handles this natively)

## R2: Rate Limiting & Performance

**Decision**: Assume 10 requests/second rate limit with exponential backoff (same as HGNC/UniProt pattern)

**Rationale**:
- Open Targets documentation doesn't explicitly state rate limits for public API
- Conservative 10 req/s default prevents overwhelming the service
- Exponential backoff with jitter prevents thundering herd on 429 responses
- GraphQL request batching reduces total request count

**Rate Limiter Configuration**:
```python
RATE_LIMIT_DELAY = 0.1  # 10 req/s = 100ms between requests
MAX_RETRIES = 3
BACKOFF_FACTOR = 2.0  # Exponential backoff: 1s, 2s, 4s
```

**Performance Validation**:
- Will monitor actual response times in integration tests
- SC-001: 95% of queries <2s (GraphQL typically faster than REST)
- SC-002: 100% of lookups <1s (direct ID lookups are fast)

**Alternatives Considered**:
- Higher rate limits (25 req/s) - rejected due to lack of documentation
- No rate limiting - rejected, could cause API throttling

## R3: Ensembl ID Validation

**Decision**: Validate Ensembl gene IDs with regex `^ENSG\d{11}$`

**Rationale**:
- Open Targets Platform focuses on human genes (ENSG prefix)
- Ensembl gene IDs have 11-digit numeric suffix (e.g., ENSG00000141510 for TP53)
- Non-human organisms use different prefixes (ENSMUSG for mouse, etc.)
- Strict validation prevents API errors and provides clear recovery hints

**Validation Logic**:
```python
ENSEMBL_GENE_ID_PATTERN = r"^ENSG\d{11}$"

def validate_ensembl_id(ensembl_id: str) -> bool:
    """Validate Ensembl gene ID format."""
    return bool(re.match(ENSEMBL_GENE_ID_PATTERN, ensembl_id))
```

**Error Handling**:
- Invalid format → UNRESOLVED_ENTITY with hint: "Use format ENSG[11 digits] (e.g., ENSG00000141510)"
- Valid format but not found → ENTITY_NOT_FOUND with hint: "Ensembl ID not in Open Targets database"

**Alternatives Considered**:
- Support all organisms (ENSMUSG, etc.) - rejected, Open Targets focuses on human
- No validation - rejected, poor UX (cryptic GraphQL errors)

## R4: Cross-Reference Mapping

**Decision**: Map Open Targets `crossReferences` array to our 22-key CrossReferences schema

**Mapping Table**:

| Open Targets Source | Our 22-Key Registry | Example Value | Notes |
|---------------------|---------------------|---------------|-------|
| `Ensembl` | `ensembl_gene` | `["ENSG00000141510"]` | Primary ID |
| `HGNC` | `hgnc` | `["HGNC:11998"]` | CURIE format |
| `Uniprot` | `uniprot` | `["UniProtKB:P04637"]` | CURIE format |
| `ChEMBL` | `chembl` | `["CHEMBL:1993"]` | CURIE format |
| `DrugBank` | `drugbank` | `["DB:00001"]` | CURIE format |
| `OMIM` | `omim` | `["191170"]` | Numeric only |
| `Mondo` | `mondo` | `["MONDO:0007254"]` | CURIE format |
| `EFO` | `efo` | `["EFO:0000616"]` | CURIE format |
| `Entrez` | `entrez` | `["7157"]` | Numeric only |
| `RefSeq` | `refseq` | `["NM_000546"]` | Transcript ID |
| `STRING` | `string` | `["9606.ENSP00000269305"]` | Organism.protein |
| `BioGRID` | `biogrid` | `["113418"]` | Numeric only |
| `KEGG` | `kegg` | `["hsa:7157"]` | Organism:gene |

**Transformation Logic**:
```python
def map_cross_references(ot_cross_refs: list[dict]) -> CrossReferences:
    """Map Open Targets crossReferences to our 22-key schema."""
    mapping = {}
    for xref in ot_cross_refs:
        source = xref["source"].lower()
        if source in OT_TO_REGISTRY_MAP:
            key = OT_TO_REGISTRY_MAP[source]
            if key not in mapping:
                mapping[key] = []
            mapping[key].append(normalize_curie(source, xref["id"]))
    return CrossReferences(**mapping)
```

**Omit-if-null Pattern**: Keys are omitted entirely when no cross-reference exists (Constitution Principle III).

**Alternatives Considered**:
- Include all sources (even unmapped ones) - rejected, violates 22-key schema
- Use null for missing keys - rejected, violates Constitution

## R5: Association Evidence Structure

**Decision**: Return target-disease associations with aggregated evidence scores

**Association Data Model**:
```python
class Association(BaseModel):
    target_id: str  # Ensembl gene ID
    disease_id: str  # EFO ID
    disease_name: str
    score: float  # Overall association score (0.0-1.0)
    evidence_count: int  # Total number of evidence sources
    evidence_sources: list[str]  # Data type IDs (genetic_association, somatic_mutation, etc.)
```

**Evidence Aggregation**:
- Open Targets provides `datatypeScores` array with individual evidence scores
- Overall `score` is pre-computed by Open Targets (weighted aggregation)
- `evidence_sources` lists the data types contributing to the association

**GraphQL Field Selection**:
- Full mode: Include all evidence details
- Slim mode: Not applicable (associations don't support slim mode - always return full records)

**Alternatives Considered**:
- Flatten evidence into separate records - rejected, loses context
- Return raw evidence objects - rejected, too verbose (token budget)

## R6: GraphQL Error Handling

**Decision**: Map GraphQL errors to our 6 error codes based on error type

**Error Code Mapping**:

| GraphQL Error Type | HTTP Status | Our Error Code | Recovery Hint Example |
|--------------------|-------------|----------------|----------------------|
| Validation error (invalid variables) | 400 | UNRESOLVED_ENTITY | "Ensembl ID must match format ENSG[11 digits]" |
| Target not found (null result) | 200 (GraphQL success) | ENTITY_NOT_FOUND | "Ensembl ID not in Open Targets database" |
| Query too short | 400 | AMBIGUOUS_QUERY | "Minimum query length is 2 characters" |
| Rate limit exceeded | 429 | RATE_LIMITED | "Wait {backoff_seconds}s before retrying" |
| GraphQL service error | 500/502/503 | UPSTREAM_ERROR | "Open Targets API temporarily unavailable - retry in 60 seconds" |
| Invalid cross-reference | 200 (data issue) | INVALID_CROSS_REFERENCE | "Cross-reference format invalid for source {source}" |

**Error Detection Logic**:
```python
async def handle_graphql_error(response: httpx.Response) -> ErrorEnvelope | dict:
    if response.status_code == 429:
        return ErrorEnvelope(code=ErrorCode.RATE_LIMITED, ...)

    data = response.json()
    if "errors" in data:
        # GraphQL validation error
        return ErrorEnvelope(code=ErrorCode.UNRESOLVED_ENTITY, ...)

    if data.get("data", {}).get("target") is None:
        # Target not found
        return ErrorEnvelope(code=ErrorCode.ENTITY_NOT_FOUND, ...)

    # Success
    return data
```

**Alternatives Considered**:
- Generic error code for all failures - rejected, poor UX
- HTTP status-only mapping - rejected, GraphQL errors are 200 OK

## R7: Pagination Mechanism

**Decision**: Use Open Targets native pagination (index/size) and encode as opaque cursor

**Pagination Strategy**:
- Open Targets GraphQL uses `PageInput` type: `{index: int, size: int}`
- index=0 is first page, index=1 is second page, etc.
- We encode this as base64 cursor to maintain opacity
- Cursor format: `base64({\"index\": N, \"size\": 50})`

**Cursor Implementation**:
```python
import base64
import json

def encode_cursor(index: int, size: int) -> str:
    """Encode pagination state as opaque cursor."""
    cursor_data = {"index": index, "size": size}
    return base64.b64encode(json.dumps(cursor_data).encode()).decode()

def decode_cursor(cursor: str | None) -> tuple[int, int]:
    """Decode cursor to pagination state."""
    if cursor is None:
        return (0, 50)  # Default first page
    cursor_data = json.loads(base64.b64decode(cursor).decode())
    return (cursor_data["index"], cursor_data["size"])
```

**Pagination Envelope**:
```python
{
  "items": [...],
  "pagination": {
    "cursor": "eyJpbmRleCI6MSwic2l6ZSI6NTB9" if has_next else null,
    "total_count": 1540,
    "page_size": 50
  }
}
```

**Alternatives Considered**:
- Offset-based pagination (no cursor) - rejected, not opaque
- GraphQL Relay cursor standard - rejected, Open Targets doesn't use Relay

---

## Research Summary

All 7 research items (R1-R7) resolved. Key findings:

1. **GraphQL API** (R1): Well-structured API with clear query patterns for search, target, and associations
2. **Rate Limiting** (R2): Conservative 10 req/s with exponential backoff (same as HGNC/UniProt)
3. **ID Validation** (R3): Strict `^ENSG\d{11}$` regex for human genes only
4. **Cross-References** (R4): 13 mappable sources to our 22-key registry (hgnc, uniprot, chembl, etc.)
5. **Associations** (R5): Pre-aggregated scores with evidence source lists
6. **Error Handling** (R6): 6 error codes map cleanly to GraphQL error types
7. **Pagination** (R7): Base64-encoded cursor wrapping Open Targets index/size pagination

**No blockers identified**. Ready for Phase 1 design (data models, contracts, quickstart).

---

## Next Phase

**Phase 1**: Generate `data-model.md`, `contracts/`, and `quickstart.md` using research findings.
