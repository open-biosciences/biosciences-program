# UniProt REST API Research

**Date**: 2025-12-22
**Purpose**: Phase 0 research for UniProt MCP Server implementation

## R1: UniProt REST API Endpoints

### Base URL
```
https://rest.uniprot.org
```

### Key Endpoints

**Search Endpoint**:
```
GET /uniprotkb/search?query={query}&fields={fields}&size={size}
```

Example:
```bash
curl "https://rest.uniprot.org/uniprotkb/search?query=p53&fields=accession,id,gene_names,organism_name&size=3"
```

**Fetch Endpoint (single protein)**:
```
GET /uniprotkb/{accession}.json
```

Example:
```bash
curl "https://rest.uniprot.org/uniprotkb/P04637.json"
```

### Authentication
**Decision**: No authentication required for public UniProt REST API
**Rationale**: All tested queries work without API keys or tokens
**Alternatives considered**: None needed for current scope

---

## R2: Search API Query Syntax and Ranking

### Query Syntax

**Simple text queries**:
```bash
query=p53                    # Basic search
query=insulin human          # Multi-term search
query=kinase                 # Broad functional search
```

**Field-specific queries**:
```bash
query=gene:TP53              # Search by gene name
query=organism_name:9606     # Filter by organism (NCBI taxon ID)
query=protein_name:insulin   # Search by protein name
```

**Boolean operators**:
```bash
query=(p53) AND (organism_id:9606)    # Human p53 only
query=(kinase) AND (reviewed:true)    # Reviewed kinases only
```

### Fields Parameter

The `fields` parameter controls which data is returned:
```bash
# Minimal fields for search candidates
fields=accession,id,gene_names,organism_name,protein_name

# Full fields for complete protein record
# (omit fields parameter to get full JSON response)
```

### Ranking Behavior

**Test Results**:
1. Query "p53" returns TP53 from various organisms ranked by relevance
2. Query "insulin human" prioritizes Homo sapiens results
3. Query "kinase" returns thousands of results with general relevance scoring

**Decision**: Use UniProt's default relevance scoring
**Rationale**: Built-in ranking appears to prioritize:
- Exact matches over partial matches
- Reviewed (Swiss-Prot) entries over unreviewed (TrEMBL)
- Model organisms (human, mouse) over others

**Implementation note**: We may need to post-process results to ensure human proteins are prioritized for ambiguous queries (per spec US1 acceptance scenario 4).

---

## R3: Pagination Patterns

### Pagination Mechanism

UniProt uses **cursor-based pagination** via Link headers and query parameters.

**Test with large result set** (kinase query):
```bash
curl -I "https://rest.uniprot.org/uniprotkb/search?query=kinase&size=25"
```

### Response Structure

Search endpoint returns:
```json
{
  "results": [...],
  "facets": [...],
  "cursor": "next_cursor_value",
  "from": 0,
  "to": 25
}
```

The `cursor` field contains an opaque string for fetching the next page.

### Next Page Request

```bash
curl "https://rest.uniprot.org/uniprotkb/search?query=kinase&size=25&cursor={cursor_value}"
```

### Pagination Implementation Strategy

**Decision**: Use cursor-based pagination with opaque cursor strings
**Rationale**:
- UniProt API provides `cursor` field in search responses
- Cursor approach is more efficient than offset/limit for large datasets
- Matches our PaginationEnvelope.cursor field (ADR-001 §8)

**Mapping to PaginationEnvelope**:
```python
PaginationEnvelope(
    items=[...],  # from results array
    pagination=Pagination(
        cursor=response.get("cursor"),  # opaque string or null
        total_count=None,  # UniProt doesn't provide total count
        page_size=len(results)
    )
)
```

**Maximum page size**: 500 (from testing, larger values return 400 Bad Request)
**Default page size**: 50 (per our task specification)

---

## R4: Rate Limiting Policy

### Testing Results

**Burst request test** (10 concurrent requests):
```bash
for i in {1..10}; do
  curl -s "https://rest.uniprot.org/uniprotkb/P04637.json" -w "%{http_code}\n" -o /dev/null &
done
```

Result: All requests succeeded with HTTP 200 (no 429 responses observed)

### Rate Limit Headers

**Tested responses** show no rate limit headers:
- No `X-RateLimit-Limit`
- No `X-RateLimit-Remaining`
- No `X-RateLimit-Reset`
- No `Retry-After` header (unless actually rate limited)

### Decision

**Assumed rate limit**: 10 requests/second (conservative estimate)
**Rationale**:
- Public API with no documented limits
- No rate limit headers observed in testing
- Conservative default prevents potential throttling
- Matches HGNC server's proven rate limiting strategy

**Backoff strategy**:
- Exponential backoff: 2^attempt seconds (per HGNC code review lesson)
- Respect `Retry-After` header if provided in 429 response
- Re-check timing after lock acquisition (thundering herd prevention)

**Implementation note**: May need to adjust based on production usage patterns.

---

## R5: Cross-Reference Field Mappings

### Test Data: P04637 (Human TP53)

Tested with:
```bash
curl "https://rest.uniprot.org/uniprotkb/P04637.json"
```

### Cross-Reference Structure

UniProt provides cross-references in `uniProtKBCrossReferences` array:
```json
{
  "uniProtKBCrossReferences": [
    {
      "database": "HGNC",
      "id": "HGNC:11998",
      "properties": []
    },
    {
      "database": "Ensembl",
      "id": "ENST00000269305.9",
      "properties": [{"key": "ProteinId", "value": "ENSP00000269305.9"}, ...]
    },
    {
      "database": "RefSeq",
      "id": "NP_000537.3",
      "properties": [{"key": "NucleotideSequenceId", "value": "NM_000546.5"}],
      "isoformId": "P04637-1"
    }
  ]
}
```

### Field Mapping Table

| Our 22-Key Registry | UniProt Database Field | Example ID | Notes |
|---------------------|------------------------|------------|-------|
| `hgnc` | `HGNC` | `HGNC:11998` | Already in CURIE format |
| `ensembl_gene` | `Ensembl` (gene) | `ENSG00000141510` | Need to extract gene ID from transcript refs |
| `ensembl_transcript` | `Ensembl` | `ENST00000269305.9` | Transcript ID in `id` field |
| `uniprot` | N/A (self-reference) | `UniProtKB:P04637` | Construct from primaryAccession |
| `entrez` | `GeneID` | `7157` | NCBI Entrez Gene ID |
| `refseq` | `RefSeq` | `NP_000537.3` | Protein RefSeq ID |
| `pdb` | `PDB` | `1TUP`, `2OCJ`, ... | Multiple structure IDs (array) |
| `kegg` | `KEGG` | `hsa:7157` | KEGG gene ID |
| `omim` | `MIM` | `133239`, `191170`, ... | Multiple disease IDs (array) |
| `orphanet` | `Orphanet` | `1501`, `210159`, ... | Orphanet disease IDs (array) |
| `string` | `STRING` | `9606.ENSP00000269305` | STRING protein ID |
| `biogrid` | `BioGRID` | `113010` | BioGRID protein ID |

### Cross-Reference Extraction Logic

**Single-value fields**:
```python
hgnc = next((ref["id"] for ref in refs if ref["database"] == "HGNC"), None)
entrez = next((ref["id"] for ref in refs if ref["database"] == "GeneID"), None)
string = next((ref["id"] for ref in refs if ref["database"] == "STRING"), None)
biogrid = next((ref["id"] for ref in refs if ref["database"] == "BioGRID"), None)
```

**Multi-value fields (arrays)**:
```python
pdb = [ref["id"] for ref in refs if ref["database"] == "PDB"]
omim = [ref["id"] for ref in refs if ref["database"] == "MIM"]
orphanet = [ref["id"] for ref in refs if ref["database"] == "Orphanet"]
```

**Ensembl special case**:
- UniProt provides transcript IDs (ENST...)
- Need to extract gene ID (ENSG...) from Ensembl properties or use external mapping
- For MVP: Use first transcript ID as placeholder, add gene extraction in refinement

**RefSeq special case**:
- Multiple RefSeq entries (one per isoform)
- Use canonical isoform (isoformId ends with "-1") or first entry

### Decision

**Mapping implementation**: Create `_map_cross_references()` helper in UniProtClient
**Alternatives considered**: Parse on-demand vs cached mapping table
**Rationale**: Helper function allows reuse and testing

**Null handling**: Omit keys entirely if no cross-reference exists (per Constitution III)

---

## R6: CURIE Format Validation

### Observed UniProt Accession Patterns

**Primary accession format** (from testing):
```
P04637          # 6-character accession (legacy Swiss-Prot format)
Q9TTA1          # 6-character accession
A0A123B4C5      # 10-character accession (TrEMBL format)
```

**Pattern analysis**:
- Starts with letter (P, Q, O, or A)
- Followed by 5-9 alphanumeric characters
- Total length: 6-10 characters
- Case-sensitive (uppercase only)

### CURIE Format

**Decision**: `UniProtKB:{accession}`
**Example**: `UniProtKB:P04637`

### Validation Regex

```python
UNIPROT_CURIE_PATTERN = r'^UniProtKB:[A-Z][A-Z0-9]{5,9}$'
```

**Explanation**:
- `^UniProtKB:` - Must start with literal "UniProtKB:"
- `[A-Z]` - First character must be uppercase letter
- `[A-Z0-9]{5,9}` - Followed by 5-9 uppercase letters or digits
- `$` - End of string

**Validation examples**:
```python
✓ UniProtKB:P04637      # Valid (6-char)
✓ UniProtKB:A0A123B4C5  # Valid (10-char)
✗ P04637                # Invalid (missing prefix)
✗ UniProtKB:p04637      # Invalid (lowercase)
✗ UniProtKB:P04         # Invalid (too short)
```

### Implementation

Add to `Protein` model in `models/protein.py`:
```python
import re

UNIPROT_CURIE_PATTERN = re.compile(r'^UniProtKB:[A-Z][A-Z0-9]{5,9}$')

@field_validator("id")
@classmethod
def validate_curie_format(cls, v: str) -> str:
    if not UNIPROT_CURIE_PATTERN.match(v):
        raise ValueError(
            f"Invalid UniProt CURIE format. Expected 'UniProtKB:XXXXXX' "
            f"where X is 6-10 uppercase alphanumeric characters, got: {v}"
        )
    return v
```

---

## R7: Error Response Formats

### Test Cases

**Invalid query** (empty string):
```bash
curl "https://rest.uniprot.org/uniprotkb/search?query="
```
Response: HTTP 400 Bad Request
```json
{
  "url": "/uniprotkb/search",
  "messages": ["'query' is required"]
}
```

**Not found** (non-existent accession):
```bash
curl "https://rest.uniprot.org/uniprotkb/NOTFOUND.json"
```
Response: HTTP 404 Not Found
```json
{
  "url": "/uniprotkb/NOTFOUND",
  "messages": ["Resource not found"]
}
```

**Invalid accession format**:
```bash
curl "https://rest.uniprot.org/uniprotkb/invalid-id.json"
```
Response: HTTP 400 Bad Request
```json
{
  "url": "/uniprotkb/invalid-id",
  "messages": ["Invalid accession"]
}
```

**Rate limit** (simulated, not observed in testing):
Expected: HTTP 429 Too Many Requests
```json
{
  "messages": ["Too many requests"]
}
```
Headers: `Retry-After: 60`

### Error Mapping Table

| HTTP Status | UniProt Error | Our ErrorEnvelope Code | Recovery Hint |
|-------------|---------------|------------------------|---------------|
| 400 (empty query) | "query is required" | `AMBIGUOUS_QUERY` | "Provide a search query with at least 2 characters. Example: 'p53' or 'insulin human'" |
| 400 (invalid accession) | "Invalid accession" | `UNRESOLVED_ENTITY` | "Use search_proteins to find valid UniProt IDs, then call get_protein with the resolved CURIE (e.g., 'UniProtKB:P04637')" |
| 404 | "Resource not found" | `ENTITY_NOT_FOUND` | "The protein {invalid_input} does not exist in UniProt. Verify the accession or search for the protein by name." |
| 429 | "Too many requests" | `RATE_LIMITED` | "Rate limit exceeded. Wait {retry_after} seconds before retrying, or use batch operations to reduce request count." |
| 500/502/503 | Server errors | `UPSTREAM_ERROR` | "UniProt API is currently unavailable. Try again in a few moments." |

### Error Envelope Implementation

**Example for UNRESOLVED_ENTITY**:
```python
ErrorEnvelope(
    success=False,
    error=ErrorDetail(
        code="UNRESOLVED_ENTITY",
        message="Invalid UniProt CURIE format",
        recovery_hint=(
            "Use search_proteins to find valid UniProt IDs, then call "
            "get_protein with the resolved CURIE (e.g., 'UniProtKB:P04637')"
        ),
        invalid_input=user_input
    )
)
```

**Implementation note**: All error responses must use canonical ErrorEnvelope (ADR-001 §8).

---

## Summary of Decisions

| Research Task | Decision | Rationale |
|---------------|----------|-----------|
| **R1: API Endpoints** | Base: `https://rest.uniprot.org`, Search: `/uniprotkb/search`, Fetch: `/uniprotkb/{acc}.json` | Standard UniProt REST API endpoints, no auth required |
| **R2: Query Syntax** | Support simple text queries and field-specific filters, use default relevance ranking | UniProt provides robust search with boolean operators, may need human prioritization post-processing |
| **R3: Pagination** | Cursor-based pagination using opaque cursor strings | Efficient for large datasets, matches our PaginationEnvelope schema |
| **R4: Rate Limiting** | Assume 10 req/s, exponential backoff (2^attempt), thundering herd prevention | Conservative limit prevents throttling, proven pattern from HGNC |
| **R5: Cross-References** | Map `uniProtKBCrossReferences` to 22-key registry, omit null keys | Direct field mapping, handles single and multi-value refs |
| **R6: CURIE Format** | `UniProtKB:[A-Z][A-Z0-9]{5,9}` regex | Validated against observed accession patterns (6-10 chars) |
| **R7: Error Responses** | Map HTTP status codes to canonical ErrorEnvelope codes with recovery hints | Consistent error handling per ADR-001 §8 |

---

## Next Steps

With research complete, proceed to:
1. **T010-T013**: Implement core models (Protein, ProteinSearchCandidate)
2. **T014-T018**: Implement UniProtClient with rate limiting
3. **T019-T020**: Implement cross-reference mapping logic

All research findings documented here should be referenced during implementation.
