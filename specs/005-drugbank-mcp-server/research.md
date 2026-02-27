# Research: DrugBank MCP Server

**Phase**: 0 (Research & Technical Decisions)
**Date**: 2025-12-22
**Source**: DrugBank Documentation + API exploration

## R1: DrugBank API Structure

**Decision**: Use DrugBank public website endpoints for basic drug data

**Rationale**:
- DrugBank has tiered API access:
  - **Public website**: Basic drug pages accessible at `https://go.drugbank.com/drugs/DBXXXXX`
  - **Commercial API**: Full REST API requiring subscription and API key
- For initial implementation, use public endpoints with HTML/JSON parsing
- Optional enhancement: Support `DRUGBANK_API_KEY` environment variable for commercial access

**API Endpoints**:

**Public (no auth)**:
```
GET https://go.drugbank.com/drugs/DB00945              # Drug page (HTML)
GET https://go.drugbank.com/unearth/q?query=aspirin    # Search (HTML with JSON data)
```

**Commercial (with API key)**:
```
GET https://api.drugbank.com/v1/drugs?q=aspirin        # Search endpoint
GET https://api.drugbank.com/v1/drugs/DB00945          # Drug details
Authorization: Bearer {DRUGBANK_API_KEY}
```

**Alternatives Considered**:
- Commercial-only API - rejected, limits accessibility for basic usage
- Web scraping - rejected, fragile and may violate ToS
- Public endpoints with JSON extraction - selected for initial implementation

## R2: Rate Limiting & Performance

**Decision**: Assume 10 requests/second rate limit with exponential backoff (same as HGNC/UniProt/Open Targets pattern)

**Rationale**:
- DrugBank documentation doesn't explicitly state rate limits for public access
- Conservative 10 req/s default prevents overwhelming the service
- Exponential backoff with jitter prevents thundering herd on 429 responses
- Commercial API may have different limits (to be documented when API key support added)

**Rate Limiter Configuration**:
```python
RATE_LIMIT_DELAY = 0.1  # 10 req/s = 100ms between requests
MAX_RETRIES = 3
BACKOFF_FACTOR = 2.0  # Exponential backoff: 1s, 2s, 4s
```

**Performance Validation**:
- Will monitor actual response times in integration tests
- SC-001: 95% of queries <2s
- SC-002: 100% of lookups <1s

**Alternatives Considered**:
- Higher rate limits (25 req/s) - rejected due to lack of documentation
- No rate limiting - rejected, could cause API throttling

## R3: DrugBank ID Validation

**Decision**: Validate DrugBank IDs with regex `^DrugBank:DB\d{5}$`

**Rationale**:
- DrugBank IDs follow pattern `DBXXXXX` where X is a digit (0-9)
- IDs are always 5 digits with leading zeros (e.g., DB00001, DB00945, DB01050)
- Some newer IDs may use 6+ digits (e.g., DB100001) - future enhancement
- CURIE format: `DrugBank:DB00945`

**Validation Logic**:
```python
DRUGBANK_ID_PATTERN = r"^DrugBank:DB\d{5}$"

def validate_drugbank_id(drugbank_id: str) -> bool:
    """Validate DrugBank CURIE format."""
    return bool(re.match(DRUGBANK_ID_PATTERN, drugbank_id))
```

**Error Handling**:
- Invalid format → UNRESOLVED_ENTITY with hint: "Use format DrugBank:DBXXXXX (e.g., DrugBank:DB00945)"
- Valid format but not found → ENTITY_NOT_FOUND with hint: "Drug not found in DrugBank database"

**Alternatives Considered**:
- Support variable digit counts (DB\d{5,7}) - future enhancement if needed
- No validation - rejected, poor UX (cryptic API errors)

## R4: Cross-Reference Mapping

**Decision**: Map DrugBank external identifiers to our 22-key CrossReferences schema

**Mapping Table**:

| DrugBank Field | Our 22-Key Registry | Example Value | Notes |
|----------------|---------------------|---------------|-------|
| ChEMBL ID | `chembl` | `["CHEMBL:25"]` | CURIE format |
| UniProt Targets | `uniprot` | `["UniProtKB:P00734"]` | CURIE format |
| KEGG Drug | `kegg` | `["D00109"]` | KEGG drug ID |
| PubChem Compound | `pubchem_compound` | `["2244"]` | CID only |
| PubChem Substance | `pubchem_substance` | `["46505899"]` | SID only |
| PDB | `pdb` | `["1BNA"]` | Structure ID |
| PharmGKB | N/A | - | Not in 22-key registry |
| Wikipedia | N/A | - | Not in 22-key registry |

**Transformation Logic**:
```python
def map_cross_references(drugbank_refs: dict) -> CrossReferences:
    """Map DrugBank external identifiers to 22-key schema."""
    mapping = {}
    if drugbank_refs.get("chembl_id"):
        mapping["chembl"] = f"CHEMBL:{drugbank_refs['chembl_id']}"
    if drugbank_refs.get("uniprot_ids"):
        mapping["uniprot"] = [f"UniProtKB:{uid}" for uid in drugbank_refs["uniprot_ids"]]
    if drugbank_refs.get("kegg_drug_id"):
        mapping["kegg"] = drugbank_refs["kegg_drug_id"]
    if drugbank_refs.get("pubchem_cid"):
        mapping["pubchem_compound"] = str(drugbank_refs["pubchem_cid"])
    return CrossReferences(**mapping)
```

**Omit-if-null Pattern**: Keys are omitted entirely when no cross-reference exists (Constitution Principle III).

**Alternatives Considered**:
- Include all sources (even unmapped ones) - rejected, violates 22-key schema
- Use null for missing keys - rejected, violates Constitution

## R5: Search Mechanism

**Decision**: Use DrugBank search endpoint with query string parameter

**Search Approach**:
- Public: Use `https://go.drugbank.com/unearth/q?query={query}` and parse results
- Commercial: Use `https://api.drugbank.com/v1/drugs?q={query}` (JSON response)

**Search Parameters**:
```python
{
    "query": "aspirin",       # Search term (min 2 chars)
    "page": 1,                # Page number (1-indexed)
    "per_page": 50            # Results per page (max 100)
}
```

**Response Parsing**:
- Extract drug ID, name, type, categories from search results
- Calculate relevance score based on match quality (exact > partial > fuzzy)
- Sort by relevance score descending

**Alternatives Considered**:
- Full-text search with Elasticsearch - not available in public API
- Client-side filtering - rejected, inefficient for large datasets

## R6: Error Response Mapping

**Decision**: Map DrugBank HTTP errors to our 5 error codes

**Error Code Mapping**:

| HTTP Status | DrugBank Meaning | Our Error Code | Recovery Hint Example |
|-------------|------------------|----------------|----------------------|
| 400 | Bad request | AMBIGUOUS_QUERY | "Minimum query length is 2 characters" |
| 401/403 | Auth required | UPSTREAM_ERROR | "Commercial API access required for this feature" |
| 404 | Drug not found | ENTITY_NOT_FOUND | "Drug DrugBank:DB99999 not found in database" |
| 429 | Rate limited | RATE_LIMITED | "Wait {backoff_seconds}s before retrying" |
| 500/502/503 | Server error | UPSTREAM_ERROR | "DrugBank API temporarily unavailable - retry in 60 seconds" |
| N/A | Invalid CURIE format | UNRESOLVED_ENTITY | "Use format DrugBank:DBXXXXX" |

**Error Detection Logic**:
```python
async def handle_drugbank_error(response: httpx.Response, drugbank_id: str) -> ErrorEnvelope:
    if response.status_code == 429:
        return ErrorEnvelope.rate_limited()
    if response.status_code == 404:
        return ErrorEnvelope.entity_not_found(drugbank_id)
    if response.status_code in (401, 403):
        return ErrorEnvelope.upstream_error(
            response.status_code,
            "Commercial API access may be required for this feature"
        )
    if response.status_code >= 500:
        return ErrorEnvelope.upstream_error(response.status_code)
    return None  # Success case
```

**Alternatives Considered**:
- Generic error code for all failures - rejected, poor UX
- Detailed error parsing from response body - future enhancement

## R7: Pagination Mechanism

**Decision**: Use offset-based pagination encoded as opaque cursor

**Pagination Strategy**:
- DrugBank uses page/per_page pagination
- Encode as base64 cursor to maintain opacity with our PaginationEnvelope
- Cursor format: `base64({"page": N, "per_page": 50})`

**Cursor Implementation**:
```python
import base64
import json

def encode_cursor(page: int, per_page: int) -> str:
    """Encode pagination state as opaque cursor."""
    cursor_data = {"page": page, "per_page": per_page}
    return base64.b64encode(json.dumps(cursor_data).encode()).decode()

def decode_cursor(cursor: str | None) -> tuple[int, int]:
    """Decode cursor to pagination state."""
    if cursor is None:
        return (1, 50)  # Default first page
    cursor_data = json.loads(base64.b64decode(cursor).decode())
    return (cursor_data["page"], cursor_data["per_page"])
```

**Pagination Envelope**:
```python
{
  "items": [...],
  "pagination": {
    "cursor": "eyJwYWdlIjoyLCJwZXJfcGFnZSI6NTB9" if has_next else null,
    "total_count": 1540,
    "page_size": 50
  }
}
```

**Alternatives Considered**:
- Expose raw page numbers - rejected, not opaque
- Cursor-based (keyset) pagination - not supported by DrugBank API

---

## Research Summary

All 7 research items (R1-R7) resolved. Key findings:

1. **API Structure** (R1): Tiered access - public endpoints for initial implementation, optional commercial API with key
2. **Rate Limiting** (R2): Conservative 10 req/s with exponential backoff (same as HGNC/UniProt)
3. **ID Validation** (R3): Strict `^DrugBank:DB\d{5}$` regex for 5-digit IDs
4. **Cross-References** (R4): 6 mappable sources to 22-key registry (chembl, uniprot, kegg, pubchem_compound, pdb)
5. **Search Mechanism** (R5): Query-based search with relevance ranking
6. **Error Handling** (R6): 5 error codes map cleanly to HTTP status codes
7. **Pagination** (R7): Base64-encoded cursor wrapping page/per_page

**No blockers identified**. Ready for Phase 1 design (data models, contracts, quickstart).

---

## Next Phase

**Phase 1**: Generate `data-model.md`, `contracts/`, and `quickstart.md` using research findings.
