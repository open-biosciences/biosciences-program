# Research: STRING MCP Server

**Phase**: 0 (Outline & Research)
**Status**: Complete
**Date**: 2025-12-24

## Research Questions

From Technical Context NEEDS CLARIFICATION markers and unknowns:

1. STRING API endpoints and response formats
2. Evidence score semantics and interpretation
3. CURIE format validation requirements
4. Rate limiting strategy and backoff behavior

## Findings

### 1. STRING API Endpoints

**Base URL**: `https://string-db.org/api`
**Protocol**: REST with JSON output
**Authentication**: None (public API)

| Endpoint | Purpose | Key Parameters |
|----------|---------|----------------|
| `/json/resolve` | Map gene symbols to STRING protein IDs | identifiers, species, format=json |
| `/json/network` | Get interaction network with evidence | identifiers, species, required_score, limit |
| `/json/get_link` | Generate network visualization URL | identifiers, species, network_flavor |

**Decision**: Use `/json/resolve` for fuzzy search, `/json/network` for strict lookup, `/json/get_link` for utility.

**Rationale**: These endpoints align with Fuzzy-to-Fact protocol:
- `/resolve` returns multiple candidates with ranking
- `/network` requires exact STRING ID and returns authoritative interaction data
- `/get_link` is a simple URL builder (no API call needed)

**Alternatives Considered**:
- TSV format: Rejected (harder to parse, no advantage over JSON)
- `/image` endpoint: Rejected (requires pre-rendering, `/get_link` is sufficient)

### 2. Evidence Score Semantics

STRING provides 7 evidence channels, each scored 0.0-1.0 (float):

| Channel | Code | Meaning | Example Use Case |
|---------|------|---------|------------------|
| Neighborhood | `nscore` | Gene proximity on chromosome (conserved gene order) | Identify bacterial operons |
| Gene Fusion | `fscore` | Fusion events (genes fused into single ORF in some species) | Functional linkage across evolution |
| Phyletic Profiles | `pscore` | Co-occurrence across species (presence/absence correlation) | Identify pathway components |
| Co-expression | `ascore` | mRNA expression correlation (Gene Atlas, RNA-seq) | Find co-regulated genes |
| Experimental | `escore` | Physical protein binding (Y2H, affinity capture, etc.) | Validate direct interactions |
| Database | `dscore` | Curated pathway/complex databases (KEGG, Reactome) | High-confidence annotations |
| Textmining | `tscore` | Literature co-mentions (PubMed abstracts) | Discover hypothesized links |

**Combined Score**: Bayesian integration of all channels (not a simple sum). STRING provides this pre-calculated as `score` field (0.0-1.0).

**Decision**: Expose all 7 channels plus combined score in `EvidenceScores` model.

**Rationale**:
- Agents can filter by evidence type (e.g., "only experimental" = escore > 0.7)
- Combined score is useful for general high-confidence filtering
- All scores normalized 0-1 for consistent thresholds

**Alternatives Considered**:
- Only combined score: Rejected (loses granularity for evidence-type filtering)
- Score bands (high/medium/low): Rejected (agents need numeric values for reasoning)

### 3. CURIE Format

**Pattern**: `STRING:<taxid>.<protein_id>`
**Components**:
- `STRING:` - Namespace prefix (required)
- `<taxid>` - NCBI Taxonomy ID (e.g., 9606 for human)
- `<protein_id>` - Ensembl Protein ID (e.g., ENSP00000269305)

**Examples**:
- Human TP53: `STRING:9606.ENSP00000269305`
- Mouse Tp53: `STRING:10090.ENSMUSP00000005371`
- E. coli RecA: `STRING:511145.b2699`

**Validation Regex**: `^STRING:\d+\.(ENSP\d+|[a-z]\d+)$`

**Decision**: Validate CURIE format in `get_interactions()` before API call.

**Rationale**:
- Prevents API errors from malformed identifiers
- Provides clear UNRESOLVED_ENTITY error with recovery hint
- Follows Fuzzy-to-Fact principle (strict tools must validate input)

**Alternatives Considered**:
- Accept raw protein IDs: Rejected (violates Fuzzy-to-Fact, enables hallucination)
- Server-side validation only: Rejected (wastes API quota on invalid requests)

### 4. Rate Limiting Strategy

**STRING Documentation**: "Please limit requests to 1 per second"
**Observed Behavior**: No 429 errors reported in testing (public API, generous limits)

**Implementation Decision**:

```python
class STRINGClient:
    RATE_LIMIT_DELAY = 1.0  # 1 req/s = 1000ms between requests
    MAX_RETRIES = 3

    async def _rate_limited_get(self, path, params):
        async with self._lock:
            # Ensure 1 second elapsed since last request
            elapsed = time.monotonic() - self._last_request_time
            if elapsed < self.RATE_LIMIT_DELAY:
                await asyncio.sleep(self.RATE_LIMIT_DELAY - elapsed)

            response = await self._get(path, params=params)
            self._last_request_time = time.monotonic()

        # Exponential backoff on 429/503
        for attempt in range(self.MAX_RETRIES):
            if response.status_code not in (429, 503):
                break

            retry_after = response.headers.get("Retry-After")
            wait_time = int(retry_after) if retry_after else (2 ** attempt)
            await asyncio.sleep(wait_time)

            # Retry with rate limiting
            response = await self._rate_limited_get(path, params)

        return response
```

**Rationale**:
- `asyncio.Lock` prevents thundering herd (multiple agents racing)
- `time.monotonic()` immune to system clock changes
- `Retry-After` header respected if provided
- Exponential backoff (1s, 2s, 4s) prevents retry storms

**Alternatives Considered**:
- No rate limiting: Rejected (violates Constitution v1.1.0, risks IP ban)
- Token bucket: Rejected (overkill for 1 req/s, adds complexity)
- Per-endpoint limits: Rejected (all STRING endpoints share quota)

## Technology Stack

**Final Decisions**:

| Component | Technology | Justification |
|-----------|------------|---------------|
| HTTP Client | `httpx.AsyncClient` | ADR-001 §2 mandate, connection pooling, async-native |
| Models | `pydantic` v2 | Runtime validation, JSON schema generation |
| MCP Framework | `fastmcp` | FastAPI-style decorators, automatic introspection |
| Testing | `pytest-asyncio` | Async test support, fixture ecosystem |
| Rate Limiting | `asyncio.Lock` + monotonic timer | Constitution v1.1.0 compliance |

## Cross-Database Integration

**STRING Protein IDs** are based on Ensembl Protein IDs (`ENSP*`), enabling cross-references:

| Target Database | Cross-Reference Key | Mapping Strategy |
|-----------------|---------------------|------------------|
| Ensembl Gene | `ensembl_gene` | Derive from ENSP (requires additional lookup) |
| UniProt | `uniprot` | Use STRING `/proteins` endpoint for mapping |
| HGNC | `hgnc` | Derive from gene symbol via HGNC search |

**Decision**: Cross-references marked as optional in initial implementation.

**Rationale**:
- STRING `/network` endpoint doesn't return cross-references by default
- Requires additional API calls (violates rate limit budget)
- Can be added in future iteration with dedicated `/proteins` lookup

## Implementation Notes

1. **CURIE Validation**: Use compiled `re.Pattern` at module level for performance
2. **Pagination**: STRING doesn't support native pagination; implement client-side cursor over results list
3. **Error Mapping**: Map STRING HTTP errors to Canonical Error Codes:
   - 400 → AMBIGUOUS_QUERY or UNRESOLVED_ENTITY (depends on error message)
   - 404 → ENTITY_NOT_FOUND
   - 429 → RATE_LIMITED
   - 500+ → UPSTREAM_ERROR
4. **Slim Mode**: Not needed (interaction records already minimal ~80 tokens each)

## References

- STRING API Documentation: https://string-db.org/help/api/
- STRING Paper (Szklarczyk et al., 2023): https://doi.org/10.1093/nar/gkac1000
- ADR-001 v1.2: Architecture Decision Record
- Constitution v1.1.0: Rate Limiting Amendment
