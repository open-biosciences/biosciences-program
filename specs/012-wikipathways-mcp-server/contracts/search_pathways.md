# Tool Contract: search_pathways

**Type**: Fuzzy Discovery
**Purpose**: Search for biological pathways by keyword/topic with optional organism filtering
**Returns**: Paginated list of PathwaySearchCandidate objects

---

## Input Parameters

| Parameter | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| `query` | string | ✅ Yes | N/A | Length >= 2 | Search query (pathway name, biological process, disease) |
| `organism` | string | ❌ No | `null` | Scientific name | Organism filter (e.g., "Homo sapiens", "Mus musculus") |
| `cursor` | string | ❌ No | `null` | Base64 string | Pagination cursor (opaque token) |
| `page_size` | integer | ❌ No | `50` | 1-100 | Number of results per page |
| `slim` | boolean | ❌ No | `true` | N/A | Return minimal fields (true) or full records (false) |

---

## Output Schema

### Success Response (PaginationEnvelope)

```json
{
  "items": [
    {
      "id": "WP:WP534",
      "title": "Glycolysis and gluconeogenesis",
      "organism": "Homo sapiens",
      "description": "Glycolysis is the metabolic pathway that converts glucose C6H12O6, into pyruvate...",
      "score": 0.95
    }
  ],
  "pagination": {
    "cursor": "eyJvZmZzZXQiOiA1MH0=",
    "total_count": 142,
    "page_size": 50
  }
}
```

### Error Response (ErrorEnvelope)

```json
{
  "success": false,
  "error": {
    "code": "AMBIGUOUS_QUERY",
    "message": "Query 'a' is too short (minimum 2 characters required)",
    "recovery_hint": "Use a more specific search term with at least 2 characters",
    "invalid_input": "a"
  }
}
```

---

## Behavior Specification

### Query Processing
1. Validate query length >= 2 characters
2. Normalize query (lowercase, trim whitespace)
3. If organism provided, validate scientific name format
4. Call WikiPathways `/findPathwaysByText` endpoint
5. Parse results and calculate relevance scores

### Score Calculation
- Base score from API response (`score` field, string to float)
- Apply position decay: `score = base_score - (position * 0.05)`
- Normalize to 0.0-1.0 range
- Sort results descending by score

### Pagination Logic
- Client-side cursor-based pagination (no API support)
- Fetch all results, paginate in memory
- Cursor = base64-encoded offset: `{"offset": 50}`
- Return `page_size` items starting at offset
- Set cursor to `null` if no more results

### Organism Filtering
- If organism provided: filter results where `species == organism`
- Exact match only (no fuzzy organism matching)
- Scientific names only: "Homo sapiens", "Mus musculus", "Rattus norvegicus"
- Common names rejected: "human", "mouse", "rat"

---

## Error Codes

| Code | Trigger | Recovery Hint | HTTP Analogy |
|------|---------|---------------|--------------|
| `AMBIGUOUS_QUERY` | Query length < 2 | "Use more specific search term (min 2 chars)" | 400 Bad Request |
| `RATE_LIMITED` | Rate limit exceeded | "Retry after 1 second" | 429 Too Many Requests |
| `UPSTREAM_ERROR` | WikiPathways API error | "Retry later, upstream service unavailable" | 502 Bad Gateway |

---

## Example Calls

### Scenario 1: Basic Search
```python
result = await client.call_tool("search_pathways", {"query": "glycolysis"})
# Returns pathways related to glycolysis across all organisms
```

### Scenario 2: Organism-Filtered Search
```python
result = await client.call_tool("search_pathways", {
    "query": "apoptosis",
    "organism": "Homo sapiens"
})
# Returns human apoptosis pathways only
```

### Scenario 3: Paginated Search
```python
# Page 1
page1 = await client.call_tool("search_pathways", {
    "query": "metabolism",
    "page_size": 20
})
cursor = page1["pagination"]["cursor"]

# Page 2
page2 = await client.call_tool("search_pathways", {
    "query": "metabolism",
    "page_size": 20,
    "cursor": cursor
})
```

### Scenario 4: Error - Query Too Short
```python
result = await client.call_tool("search_pathways", {"query": "a"})
# Returns AMBIGUOUS_QUERY error
```

---

## Performance Targets

- **Response Time**: <2s for 95th percentile (SC-001)
- **Token Budget**: ~20 tokens per candidate (slim mode)
- **Rate Limit**: 1 req/s with exponential backoff
- **Throughput**: ~50-200 pathways per query (typical)

---

## Integration Points

### Fuzzy-to-Fact Workflow
1. **Phase 1 (Fuzzy)**: User calls `search_pathways` with query "glycolysis"
2. **Returns**: Top-ranked PathwaySearchCandidate: `{"id": "WP:WP534", ...}`
3. **Phase 2 (Strict)**: User calls `get_pathway` with `{"pathway_id": "WP:WP534"}`
4. **Returns**: Complete Pathway entity with cross_references

### Cross-Database Integration
- Results can be filtered by gene using `get_pathways_for_gene` (reverse lookup)
- Pathway IDs can be passed to `get_pathway_components` for detailed analysis

---

## Testing Requirements

### Unit Tests
- ✅ Query validation (minimum length)
- ✅ Score calculation and normalization
- ✅ Organism filtering logic
- ✅ Cursor-based pagination
- ✅ Error envelope generation

### Integration Tests
- ✅ Live API call with keyword "glycolysis"
- ✅ Organism filter "Homo sapiens"
- ✅ Pagination through 100+ results
- ✅ Empty result handling
- ✅ Rate limit enforcement
- ✅ Upstream error recovery

---

## References

- [Feature Spec: User Story 1](../spec.md#user-story-1---pathway-discovery-by-topic-priority-p1)
- [Data Model: PathwaySearchCandidate](../data-model.md#2-pathwaysearchcandidate-fuzzy-discovery)
- [Research: API Endpoints](../research.md#research-question-1-api-endpoint-structure)
