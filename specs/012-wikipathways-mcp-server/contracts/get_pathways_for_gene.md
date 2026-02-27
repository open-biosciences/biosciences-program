# Tool Contract: get_pathways_for_gene

**Type**: Fuzzy Discovery (Reverse Lookup)
**Purpose**: Find all pathways containing a specific gene/protein
**Returns**: Paginated list of PathwaySearchCandidate objects

---

## Input Parameters

| Parameter | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| `gene_id` | string | ✅ Yes | N/A | Non-empty | Gene symbol (BRCA1), Entrez ID (672), or Ensembl ID (ENSG...) |
| `organism` | string | ❌ No | `null` | Scientific name | Organism filter (e.g., "Homo sapiens") |
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
      "id": "WP:WP4868",
      "title": "RAC1/PAK1/p38/MMP2 pathway",
      "organism": "Homo sapiens",
      "description": "This pathway is based on figure 4 from Cheung et al. RAC1 is activated by TIAM1...",
      "score": 0.95
    },
    {
      "id": "WP:WP4239",
      "title": "EMT in colorectal cancer",
      "organism": "Homo sapiens",
      "description": "Epithelial-to-mesenchymal transition (EMT) is a process where epithelial cells...",
      "score": 0.90
    }
  ],
  "pagination": {
    "cursor": "eyJvZmZzZXQiOiA1MH0=",
    "total_count": 127,
    "page_size": 50
  }
}
```

### Empty Result Response

```json
{
  "items": [],
  "pagination": {
    "cursor": null,
    "total_count": 0,
    "page_size": 50
  }
}
```

### Error Response (ErrorEnvelope)

```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded (1 req/s), retry after delay",
    "recovery_hint": "Wait 1 second before retrying request",
    "invalid_input": null
  }
}
```

---

## Behavior Specification

### Gene ID Normalization
1. **Input Types**:
   - Gene symbol: `BRCA1`, `TP53`, `EGFR`
   - Entrez ID: `672`, `7157`, `1956`
   - Ensembl ID: `ENSG00000012048`, `ENSG00000141510`
2. **Normalization**:
   - Convert gene symbols to UPPERCASE: `brca1` → `BRCA1`
   - Strip whitespace: ` TP53 ` → `TP53`
   - Preserve Entrez/Ensembl IDs as-is
3. **No Validation**: Accept any non-empty string (API will handle invalid IDs)

### API Call Sequence
1. **Call**: `/findPathwaysByXref?ids=BRCA1&format=json`
2. **Optional Filter**: If organism provided, filter results where `species == organism`
3. **Score Calculation**: Base score from API, apply position decay
4. **Pagination**: Client-side cursor-based (fetch all, paginate in memory)

### Organism Filtering
- **Exact Match**: `species` field must exactly match `organism` parameter
- **Scientific Names Only**: "Homo sapiens", "Mus musculus", "Rattus norvegicus"
- **No Fuzzy Matching**: "human" does NOT match "Homo sapiens"
- **Empty Results**: Return empty items array if no matches in specified organism

### Score Calculation
- **Base Score**: From API response (`score` field, string to float)
- **Position Decay**: `score = base_score - (position * 0.05)`
- **Normalization**: Clamp to 0.0-1.0 range
- **Sort**: Descending by score

### Pagination Logic
- **Client-side cursors** (no API support for pagination)
- **Cursor format**: Base64-encoded JSON `{"offset": 50}`
- **Behavior**: Fetch all results, return `page_size` items starting at offset
- **Termination**: Set cursor to `null` when no more results

---

## Error Codes

| Code | Trigger | Recovery Hint | HTTP Analogy |
|------|---------|---------------|--------------|
| `RATE_LIMITED` | Rate limit exceeded (>1 req/s) | "Wait 1 second before retrying" | 429 Too Many Requests |
| `UPSTREAM_ERROR` | WikiPathways API error | "Retry later, upstream service unavailable" | 502 Bad Gateway |

**Note**: Empty results (gene exists but has no pathway associations) are NOT errors. Return empty items array with `total_count: 0`.

---

## Example Calls

### Scenario 1: Gene Symbol Lookup (All Organisms)
```python
result = await client.call_tool("get_pathways_for_gene", {"gene_id": "BRCA1"})
# Returns all pathways containing BRCA1 across all organisms
```

### Scenario 2: Gene Symbol + Organism Filter
```python
result = await client.call_tool("get_pathways_for_gene", {
    "gene_id": "TP53",
    "organism": "Homo sapiens"
})
# Returns only human pathways containing TP53
```

### Scenario 3: Entrez ID Lookup
```python
result = await client.call_tool("get_pathways_for_gene", {"gene_id": "672"})
# Returns pathways containing Entrez Gene:672 (BRCA1)
```

### Scenario 4: Empty Result (Gene Has No Pathways)
```python
result = await client.call_tool("get_pathways_for_gene", {"gene_id": "FAKE123"})
# Returns {"items": [], "pagination": {"cursor": null, "total_count": 0, "page_size": 50}}
```

### Scenario 5: Organism Mismatch
```python
result = await client.call_tool("get_pathways_for_gene", {
    "gene_id": "BRCA1",
    "organism": "Drosophila melanogaster"  # BRCA1 is mammalian gene
})
# Returns empty items array (no fly pathways contain BRCA1)
```

---

## Performance Targets

- **Response Time**: <2s for 95th percentile (SC-001)
- **Token Budget**: ~20 tokens per candidate (slim mode)
- **Rate Limit**: 1 req/s with exponential backoff
- **Accuracy**: Zero false negatives for well-studied genes (SC-005)

---

## Integration Points

### Reverse Gene Lookup Workflow
1. **Phase 1**: User calls `get_pathways_for_gene("BRCA1")` ← THIS TOOL
   - Returns: Pathways containing BRCA1
2. **Phase 2**: User selects pathway `WP:WP4868`
   - Calls `get_pathway("WP:WP4868")` for details
3. **Phase 3**: User calls `get_pathway_components("WP:WP4868")` for full gene list

### Cross-Database Integration
- **Input**: Gene IDs from HGNC, Entrez, Ensembl servers
- **Output**: Pathway IDs for further lookup via `get_pathway`
- **Bidirectional**: Complements `get_pathway_components` (pathway → genes)

---

## Testing Requirements

### Unit Tests
- ✅ Gene symbol normalization (uppercase conversion)
- ✅ Organism filtering logic
- ✅ Score calculation and sorting
- ✅ Cursor-based pagination
- ✅ Empty result handling

### Integration Tests
- ✅ Live API call for BRCA1
- ✅ Organism filter "Homo sapiens"
- ✅ Gene with no pathway associations (empty result)
- ✅ Pagination through 100+ results
- ✅ Rate limit enforcement
- ✅ Upstream error recovery

---

## Edge Cases

### Case 1: Gene Symbol Case Sensitivity
- **Input**: `tp53` (lowercase)
- **Normalization**: `TP53` (uppercase)
- **Behavior**: Match all TP53 pathways

### Case 2: Organism Spelling Variations
- **Input**: `organism="Human"`
- **Expected**: "Homo sapiens"
- **Behavior**: No matches (exact scientific name required)
- **Note**: Do NOT implement fuzzy organism matching

### Case 3: Gene Symbol Ambiguity
- **Input**: `gene_id="AKT"` (matches AKT1, AKT2, AKT3)
- **Behavior**: API returns pathways for all AKT variants
- **Note**: User must specify exact gene symbol for precise lookup

### Case 4: Cross-Species Gene Orthologs
- **Input**: `gene_id="TP53"` (no organism filter)
- **Behavior**: Returns human, mouse, rat TP53 pathways
- **Note**: Use organism filter to narrow results

---

## References

- [Feature Spec: User Story 3](../spec.md#user-story-3---reverse-gene-to-pathway-lookup-priority-p2)
- [Data Model: PathwaySearchCandidate](../data-model.md#2-pathwaysearchcandidate-fuzzy-discovery)
- [Research: API Endpoints](../research.md#research-question-1-api-endpoint-structure)
- [ADR-001 §3: Fuzzy-to-Fact Protocol](../../../docs/adr/accepted/adr-001-v1.2.md)
