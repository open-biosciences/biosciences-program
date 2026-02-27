# Research: BioGRID MCP Server

**Phase**: 0 (Research & Discovery)
**Date**: 2025-12-24
**Purpose**: Resolve technical unknowns from plan.md Technical Context

## Research Questions

### 1. BioGRID API Endpoint Details

**Question**: What are the available BioGRID REST API endpoints and their parameters?

**Findings**:
- **Base URL**: `https://webservice.thebiogrid.org`
- **Primary Endpoint**: `/interactions` (GET or POST)
- **Authentication**: `accesskey` query parameter (free registration at https://webservice.thebiogrid.org/)
- **Output Format**: `format=json` (also supports tab2, count)

**Key Parameters for `/interactions`**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `accesskey` | string (required) | API key from BioGRID registration |
| `geneList` | string | Pipe-separated gene symbols (e.g., "TP53\|MDM2") |
| `searchNames` | boolean | Whether to search gene names/synonyms (default: false) |
| `includeInteractors` | boolean | Include interacting partners (default: false) |
| `includeInteractorInteractions` | boolean | Include interactions between interactors (default: false) |
| `taxId` | integer | NCBI Taxonomy ID filter (e.g., 9606 for human) |
| `evidenceList` | string | Filter by experimental system (pipe-separated) |
| `max` | integer | Maximum interactions to return (default: 10000) |
| `interSpeciesExcluded` | boolean | Exclude cross-species interactions (default: false) |

**Decision**: Use `/interactions` endpoint with `geneList` parameter for gene-specific queries

**Rationale**: Single endpoint simplifies client implementation; parameter-based filtering matches BioGRID's design

**Alternatives Considered**:
- Separate endpoints for physical vs genetic interactions → Not available; BioGRID returns both types
- SPARQL endpoint → Too complex for agent usage

### 2. Gene Symbol Search & Validation

**Question**: How should gene symbols be validated before querying BioGRID?

**Findings**:
- **BioGRID Behavior**: Accepts gene symbols case-insensitively, normalizes to uppercase
- **Symbol Format**: Typically 1-15 alphanumeric characters, may include hyphens (e.g., "TP53", "HLA-A")
- **Invalid Symbols**: BioGRID returns count of 0 (empty result set), not error
- **Synonym Handling**: `searchNames=true` enables search by gene name or synonym
- **Count Endpoint**: `format=count` on `/interactions/` returns a single integer (~200ms response)

**Decision**: API-backed search using `/interactions?format=count&searchNames=true`

**Rationale**:
- Confirms gene exists in BioGRID database; prevents hallucinated gene symbols per Constitution Principle II
- `format=count` is lightweight (~200ms) — returns single integer, not full interaction data
- `searchNames=true` matches gene synonyms for broader coverage
- Client-side regex serves as fail-fast guard only (prevents invalid API calls)
- `count == 0` → ENTITY_NOT_FOUND error (gene not in BioGRID)
- `count > 0` → return BioGridSearchCandidate with `interaction_count` metadata

**Validation Pattern**:
```python
# Client-side regex as fail-fast guard (NOT the primary validation)
GENE_SYMBOL_PATTERN = r"^[A-Z0-9][A-Z0-9\-_@.]{0,29}$"

# Primary validation: BioGRID API with format=count
params = {
    "geneList": symbol,
    "taxId": organism,
    "searchNames": "true",
    "format": "count",
}
```

**Alternatives Considered**:
- Client-side regex only (original decision) → Violates Constitution Principle II; no proof gene exists in BioGRID
- Rely solely on BioGRID server-side validation → Too permissive (accepts invalid input without error)
- Use HGNC for symbol validation → Adds unnecessary dependency; BioGRID accepts symbols from multiple organisms
- Full `/interactions?format=json` → Too heavy for existence check; `format=count` is sufficient

### 3. Experimental System Types

**Question**: What experimental evidence types does BioGRID provide?

**Findings**:
BioGRID categorizes interactions by experimental system and experimental system type:

**Physical Interactions** (protein-protein):
- Affinity Capture-Luminescence
- Affinity Capture-MS
- Affinity Capture-RNA
- Affinity Capture-Western
- Biochemical Activity
- Co-crystal Structure
- Co-fractionation
- Co-localization
- Co-purification
- Far Western
- FRET
- PCA
- Proximity Label-MS
- Protein-peptide
- Protein-RNA
- Reconstituted Complex
- Two-hybrid
- Proximity Ligation

**Genetic Interactions**:
- Dosage Growth Defect
- Dosage Lethality
- Dosage Rescue
- Negative Genetic
- Phenotypic Enhancement
- Phenotypic Suppression
- Positive Genetic
- Synthetic Growth Defect
- Synthetic Haploinsufficiency
- Synthetic Lethality
- Synthetic Rescue

**Throughput Classification**:
- "High Throughput" - Systematic screens
- "Low Throughput" - Focused studies

**Decision**: Return experimental_system, experimental_system_type, and throughput for each interaction

**Rationale**:
- experimental_system provides granular evidence type (e.g., "Two-hybrid")
- experimental_system_type enables filtering by physical vs genetic
- throughput helps assess data quality (low throughput = more focused validation)

**Alternatives Considered**:
- Map to evidence score (0.0-1.0) → BioGRID doesn't provide confidence scores
- Aggregate by evidence type → Loses granularity needed for experimental design

### 4. API Key Authentication

**Question**: How is API key authentication handled and what errors occur without it?

**Findings**:
- **Registration**: Free at https://webservice.thebiogrid.org/ (email confirmation required)
- **Usage**: Pass as `accesskey` query parameter in all requests
- **Error Response (Missing Key)**:
  ```json
  {
    "ERROR": "Invalid accesskey"
  }
  ```
- **Error Response (Invalid Key)**: Same as missing key
- **Rate Limiting**: Not explicitly documented; conservative estimate: 2 req/sec

**Decision**: Require `BIOGRID_API_KEY` environment variable; fail fast with UPSTREAM_ERROR if missing/invalid

**Rationale**:
- Early failure prevents confusing error messages
- Recovery hint directs user to registration page
- Environment variable pattern matches other life sciences APIs

**Alternatives Considered**:
- Optional API key with degraded functionality → BioGRID requires key for all queries
- Hardcoded demo key → Security risk; violates Constitution

### 5. Rate Limiting Strategy

**Question**: What rate limits does BioGRID enforce and how should the client handle them?

**Findings**:
- **Documented Limit**: Not explicitly stated in BioGRID documentation
- **Observed Behavior**: No published rate limit information
- **Conservative Estimate**: 2 requests/second (based on typical REST API patterns)
- **Error Response**: Unknown (likely HTTP 429 Too Many Requests)

**Decision**: Implement client-side rate limiting at 2 req/sec with exponential backoff

**Rationale**:
- Conservative estimate prevents accidental API abuse
- Client-side limiting is more reliable than reactive error handling
- Exponential backoff handles unknown server-side limits

**Implementation**:
```python
# In BioGridClient
self._rate_limit_delay = 0.5  # 2 req/sec = 500ms between requests
self._last_request_time: float | None = None
self._request_lock = asyncio.Lock()

async def _rate_limited_get(self, url: str, params: dict) -> dict:
    async with self._request_lock:
        if self._last_request_time:
            elapsed = time.time() - self._last_request_time
            if elapsed < self._rate_limit_delay:
                await asyncio.sleep(self._rate_limit_delay - elapsed)

        self._last_request_time = time.time()
        return await self._get(url, params)
```

**Alternatives Considered**:
- No rate limiting → Risk of API blocking
- Higher rate (10 req/sec) → Too aggressive without documentation
- Token bucket algorithm → Over-engineered for current needs

## Cross-Reference Mapping

**Question**: What cross-references does BioGRID provide and how should they be mapped?

**Findings**:
- **Entrez Gene ID**: Available as `ENTREZ_GENE` field in interaction records
- **Organism-Specific Identifiers**: Available as `ORGANISM_<TAXID>_OFFICIAL_SYMBOL`
- **Systematic Names**: Available as `SYSTEMATIC_NAME_<A|B>`

**Decision**: Map Entrez Gene ID to `cross_references.entrez` per ADR-001 Appendix A

**Rationale**:
- Entrez enables integration with NCBI resources (PubMed, Gene, ClinVar)
- Follows 22-key cross-reference registry
- Omit key if no Entrez ID available (never null)

**Alternatives Considered**:
- Include all systematic names → Too verbose; not in 22-key registry
- Map organism IDs → Not standard cross-reference format

## Performance Benchmarks

**Question**: What are realistic performance expectations for BioGRID queries?

**Findings**:
- **Network Latency**: ~100-300ms to BioGRID servers (typical REST API)
- **Query Complexity**:
  - Single gene with few interactions: ~200-500ms
  - Well-connected gene (e.g., TP53): ~1-2 seconds
  - Batch queries: Linear scaling with gene count
- **Max Interactions**: Default limit is 10,000 per request

**Decision**: Set NFR-004 performance target at P95 < 3 seconds

**Rationale**:
- Accounts for network variability
- Handles well-connected genes (TP53 has ~1000 interactions in BioGRID)
- Aligns with STRING performance target

**Alternatives Considered**:
- <1 second P95 → Unrealistic for well-connected genes
- <5 seconds P95 → Too permissive; agents expect faster responses

## Summary

All NEEDS CLARIFICATION items from plan.md Technical Context have been resolved:

1. ✅ BioGRID API endpoint: `/interactions` with gene symbol parameter
2. ✅ Gene symbol validation: API-backed search (`format=count`) + client-side regex fail-fast guard
3. ✅ Experimental evidence: Return system, type, and throughput
4. ✅ API key handling: Require BIOGRID_API_KEY with helpful error
5. ✅ Rate limiting: 2 req/sec with exponential backoff
6. ✅ Cross-references: Map Entrez Gene ID to cross_references.entrez
7. ✅ Performance: P95 < 3 seconds target

**Next Step**: Phase 1 (Design) - Create data-model.md, contracts/, and quickstart.md
