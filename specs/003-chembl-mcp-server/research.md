# Research: ChEMBL MCP Server

**Phase**: 0 (Outline & Research)
**Date**: 2025-12-22
**Purpose**: Resolve technical unknowns before design phase

## R1: ChEMBL SDK Analysis

**Question**: How should we wrap the synchronous `chembl_webresource_client` SDK for async usage?

**Decision**: Use `asyncio.run_in_executor` with ThreadPoolExecutor

**Rationale**:
- ChEMBL's official Python client is synchronous (PyPI: `chembl-webresource-client`)
- ADR-001 §2 explicitly permits this exception for legacy SDKs
- `run_in_executor` is the standard asyncio pattern for wrapping synchronous I/O

**Implementation Pattern**:
```python
import asyncio
from chembl_webresource_client.new_client import new_client

class ChEMBLClient(LifeSciencesClient):
    def __init__(self):
        super().__init__(base_url="https://www.ebi.ac.uk/chembl/api/data")
        self._molecule = new_client.molecule  # Synchronous SDK instance
        self._executor = None  # Lazy-init ThreadPoolExecutor

    async def search_compounds(self, query: str, **kwargs):
        loop = asyncio.get_event_loop()
        # Wrap synchronous SDK call
        results = await loop.run_in_executor(
            self._executor,
            lambda: self._molecule.search(query)
        )
        return self._transform_to_pagination_envelope(results)
```

**Alternatives Considered**:
- **Pure REST with httpx**: Rejected because ChEMBL's REST API requires complex query DSL for search. The SDK handles this complexity.
- **Forking SDK to make async**: Rejected because maintaining a fork is unsustainable (SDK updates, breaking changes)

**References**:
- ChEMBL Web Services: https://www.ebi.ac.uk/chembl/ws
- SDK GitHub: https://github.com/chembl/chembl_webresource_client
- Python asyncio docs: https://docs.python.org/3/library/asyncio-eventloop.html#executing-code-in-thread-or-process-pools

---

## R2: Rate Limiting Strategy

**Question**: What rate limit should we enforce for ChEMBL API calls?

**Decision**: 10 requests/second with exponential backoff (matching HGNC/UniProt pattern)

**Rationale**:
- ChEMBL public API has no published rate limit
- Anecdotal evidence suggests ~10 req/s is safe for sustained usage
- Exponential backoff prevents thundering herd if we hit hidden limits
- Matches rate limiting in LifeSciencesClient base class

**Implementation**:
```python
# In ChEMBLClient.__init__
self._rate_limiter = AsyncLimiter(max_rate=10, time_period=1.0)
self._backoff_config = {
    "max_retries": 3,
    "base_delay": 1.0,
    "max_delay": 60.0,
    "exponential_base": 2
}
```

**Fallback**: If users report 429 errors, reduce to 5 req/s and document in quickstart.md

---

## R3: Cross-Reference Mapping

**Question**: Which cross-reference databases does ChEMBL support, and how do we map them to our 22-key registry?

**Decision**: Map ChEMBL's cross-reference fields to our canonical keys

**Mapping Table**:

| ChEMBL Field | Our 22-Key Registry | Example Value | Notes |
|--------------|---------------------|---------------|-------|
| `molecule_structures.canonical_smiles` | (not a xref) | `CC(=O)Oc1ccccc1C(=O)O` | Molecular structure |
| `cross_references.xref_name = "UniProt"` | `uniprot` | `["UniProtKB:P23219"]` | Protein targets |
| `cross_references.xref_name = "PDBe"` | `pdb` | `["1PTY", "3LN1"]` | Crystal structures |
| `cross_references.xref_name = "PubChem"` | `pubchem_compound` | `["2244"]` | PubChem CID |
| `cross_references.xref_name = "DrugBank"` | `drugbank` | `["DB:00945"]` | Approved drugs |
| `molecule_hierarchy.parent_chembl_id` | `chembl` | `["CHEMBL:25"]` | Parent compound |

**Key Insights**:
- ChEMBL uses `cross_references` array with `{xref_name, xref_id}` structure
- We must transform to flat object: `{"uniprot": ["UniProtKB:P23219"], ...}`
- Omit keys entirely if xref_name not in mapping table (Constitution Principle III)

**CURIE Normalization**:
- ChEMBL IDs: Add `CHEMBL:` prefix (e.g., `25` → `CHEMBL:25`)
- UniProt IDs: Add `UniProtKB:` prefix (e.g., `P23219` → `UniProtKB:P23219`)
- DrugBank IDs: Add `DB:` prefix if missing (e.g., `00945` → `DB:00945`)

---

## R4: CURIE Format Validation

**Question**: What regex pattern should we use to validate ChEMBL CURIEs?

**Decision**: `^CHEMBL:[0-9]+$`

**Rationale**:
- ChEMBL IDs are always numeric (e.g., CHEMBL25, CHEMBL1201583)
- No letters, no zero-padding, no hyphens
- Must enforce `CHEMBL:` prefix to match our CURIE standard

**Validation Examples**:

| Input | Valid? | Reason |
|-------|--------|--------|
| `CHEMBL:25` | ✅ | Correct format |
| `CHEMBL:1201583` | ✅ | Correct format |
| `CHEMBL25` | ❌ | Missing colon (return UNRESOLVED_ENTITY) |
| `25` | ❌ | Missing prefix (return UNRESOLVED_ENTITY) |
| `CHEMBL:ABC` | ❌ | Non-numeric ID (return UNRESOLVED_ENTITY) |
| `chembl:25` | ❌ | Lowercase prefix (return UNRESOLVED_ENTITY) |

**Recovery Hint**: "Invalid ChEMBL CURIE format. Use 'CHEMBL:NNNNN' (e.g., 'CHEMBL:25')"

---

## R5: Thread Pool Configuration

**Question**: How many threads should we allocate for `run_in_executor` to handle concurrent ChEMBL SDK calls?

**Decision**: Use default ThreadPoolExecutor (max_workers = min(32, (os.cpu_count() or 1) + 4))

**Rationale**:
- Python's default is well-tuned for I/O-bound tasks
- ChEMBL SDK calls are network I/O, not CPU-bound
- With 10 req/s rate limit, we won't spawn >10-15 concurrent SDK calls in practice
- Default thread pool size (typically 8-12 on modern machines) is sufficient

**Monitoring**: If users report "ThreadPoolExecutor queue full" errors, add:
```python
self._executor = ThreadPoolExecutor(max_workers=20)  # Explicit override
```

**Batch Tool Mitigation**: `get_compounds_batch` amortizes thread pool usage by making a single SDK call for multiple CURIEs instead of N calls.

---

## R6: Search Result Ranking

**Question**: How should we rank search results for `search_compounds`?

**Decision**: Preserve ChEMBL SDK's default relevance ranking, add `score` field

**Rationale**:
- ChEMBL SDK returns results sorted by relevance (ElasticSearch backend)
- We don't need to re-rank; trust the API's algorithm
- Add synthetic `score` field (1.0 for first result, decreasing linearly) for consistency with HGNC/UniProt pattern

**Implementation**:
```python
def _transform_to_search_candidates(sdk_results):
    candidates = []
    for i, result in enumerate(sdk_results):
        score = max(0.1, 1.0 - (i * 0.05))  # Linear decay
        candidates.append(CompoundSearchCandidate(
            id=f"CHEMBL:{result['molecule_chembl_id']}",
            name=result['pref_name'] or result['molecule_chembl_id'],
            molecular_formula=result.get('molecule_properties', {}).get('full_mwt'),
            score=score
        ))
    return candidates
```

---

## R7: Error Code Mapping

**Question**: How do we map ChEMBL SDK exceptions to our 6 canonical error codes?

**Decision**: Map SDK exceptions to canonical errors with recovery hints

**Mapping Table**:

| ChEMBL SDK Exception | Our Error Code | Recovery Hint |
|----------------------|----------------|---------------|
| `requests.exceptions.HTTPError (404)` | `ENTITY_NOT_FOUND` | "ChEMBL ID not found. Verify CURIE format 'CHEMBL:NNNNN'" |
| `requests.exceptions.HTTPError (429)` | `RATE_LIMITED` | "ChEMBL API rate limit exceeded. Wait {backoff_seconds}s before retrying" |
| `requests.exceptions.HTTPError (500/502/503)` | `UPSTREAM_ERROR` | "ChEMBL API temporarily unavailable. Retry in 60 seconds" |
| `requests.exceptions.Timeout` | `UPSTREAM_ERROR` | "ChEMBL API timeout. Retry with smaller query or check network" |
| `ValueError` (invalid CURIE format) | `UNRESOLVED_ENTITY` | "Invalid ChEMBL CURIE format. Use 'CHEMBL:NNNNN' (e.g., 'CHEMBL:25')" |
| Query too short (<2 chars) | `AMBIGUOUS_QUERY` | "Minimum query length is 2 characters" |

**Implementation Pattern**:
```python
try:
    result = await loop.run_in_executor(self._executor, lambda: self._molecule.get(chembl_id))
except HTTPError as e:
    if e.response.status_code == 404:
        return ErrorEnvelope(code="ENTITY_NOT_FOUND", ...)
    elif e.response.status_code == 429:
        return ErrorEnvelope(code="RATE_LIMITED", ...)
```

---

## Summary

All technical unknowns resolved. Key decisions:

1. **SDK Wrapping**: Use `run_in_executor` with default ThreadPoolExecutor
2. **Rate Limiting**: 10 req/s with exponential backoff
3. **Cross-References**: Transform ChEMBL's array format to flat object with 22-key registry
4. **CURIE Validation**: `^CHEMBL:[0-9]+$` regex pattern
5. **Thread Pool**: Use Python defaults (8-12 workers), mitigate with batch tool
6. **Search Ranking**: Preserve ChEMBL's relevance ordering, add synthetic scores
7. **Error Mapping**: Map 6 SDK exceptions to canonical error codes

**Next Phase**: Design data models and contracts (Phase 1)
