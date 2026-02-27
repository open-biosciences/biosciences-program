# Tool Contract: get_pathway

**Type**: Strict Lookup
**Purpose**: Retrieve complete pathway details by resolved pathway ID
**Returns**: Single Pathway entity with metadata and cross-references

---

## Input Parameters

| Parameter | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| `pathway_id` | string | ✅ Yes | N/A | Pattern: `^WP:WP\d+$` | WikiPathways CURIE (e.g., "WP:WP534") |

---

## Output Schema

### Success Response (Pathway Entity)

```json
{
  "id": "WP:WP534",
  "title": "Glycolysis and gluconeogenesis",
  "organism": "Homo sapiens",
  "description": "Glycolysis is the metabolic pathway that converts glucose C6H12O6, into pyruvate. The free energy released in this process is used to form the high-energy molecules ATP and NADH...",
  "revision": {
    "version": "141823",
    "last_modified": "2024-11-15T10:30:00Z",
    "curators": ["AlexanderPico", "MaintBot"]
  },
  "component_counts": {
    "gene_count": 47,
    "protein_count": 52,
    "metabolite_count": 23,
    "interaction_count": 89
  },
  "cross_references": {
    "entrez": ["5213", "5214", "2203", "2204"],
    "ensembl_gene": ["ENSG00000152556", "ENSG00000067057"],
    "hgnc": ["HGNC:8877", "HGNC:8878"],
    "uniprot": ["P08237", "Q01813"],
    "kegg_pathway": "hsa00010",
    "reactome": "R-HSA-70171",
    "gene_ontology": "GO:0006096"
  },
  "url": "https://classic.wikipathways.org/index.php/Pathway:WP534"
}
```

### Error Response: UNRESOLVED_ENTITY

```json
{
  "success": false,
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "Invalid pathway ID format 'glycolysis'. Expected WP:WPNNNNN format (e.g., WP:WP534)",
    "recovery_hint": "Call search_pathways to resolve pathway identifier first, then use the returned pathway ID",
    "invalid_input": "glycolysis"
  }
}
```

### Error Response: ENTITY_NOT_FOUND

```json
{
  "success": false,
  "error": {
    "code": "ENTITY_NOT_FOUND",
    "message": "Pathway WP:WP99999 not found in WikiPathways database",
    "recovery_hint": "Verify pathway ID is correct or use search_pathways to find valid pathway",
    "invalid_input": "WP:WP99999"
  }
}
```

---

## Behavior Specification

### Input Validation
1. **Format Check**: Validate `pathway_id` matches regex `^WP:WP\d+$`
   - Valid: `WP:WP534`, `WP:WP12345`
   - Invalid: `WP534` (no prefix), `glycolysis` (raw name), `WP:534` (missing WP)
2. **Error on Invalid Format**: Return `UNRESOLVED_ENTITY` with recovery hint
3. **Extract Numeric ID**: Strip `WP:WP` prefix for API call (`WP:WP534` → `534`)

### API Call Sequence
1. **Pathway Info**: Call `/getPathwayInfo?pwId=WP534&format=json`
2. **Component Counts**: Call `/getXrefList` with codes L, S, Ce, H, En (aggregate counts)
3. **Cross-References**: Load from bulk JSON file (`findPathwaysByXref.json`)
4. **Merge Data**: Combine responses into single Pathway entity

### Cross-Reference Mapping
- **Source**: `https://www.wikipathways.org/json/findPathwaysByXref.json`
- **Mapping**:
  - `ncbigene` → `entrez` (split on comma)
  - `ensembl` → `ensembl_gene` (split on comma)
  - `hgnc.symbol` → `hgnc` (strip prefix, split)
  - `uniprot` → `uniprot` (split on comma, handle semicolon isoforms)
- **Omit Empty Keys**: Never include `null` or empty string values

### Component Count Calculation
- **Genes**: Count from `getXrefList?code=L` (NCBI Gene) + `code=H` (HGNC)
- **Proteins**: Count from `getXrefList?code=S` (UniProt)
- **Metabolites**: Count from `getXrefList?code=Ce` (ChEMBL) + `code=Ch` (CHEBI)
- **Interactions**: Default 0 (API doesn't provide interaction count)

### Revision Metadata
- **Version**: From `pathwayInfo.revision` (string)
- **Last Modified**: Derive from revision history (if available, else `null`)
- **Curators**: Extract from pathway metadata (if available, else empty list)

---

## Error Codes

| Code | Trigger | Recovery Hint | HTTP Analogy |
|------|---------|---------------|--------------|
| `UNRESOLVED_ENTITY` | Invalid ID format (no WP: prefix, raw name) | "Call search_pathways to resolve identifier" | 400 Bad Request |
| `ENTITY_NOT_FOUND` | Valid format but ID doesn't exist | "Verify ID or search for valid pathway" | 404 Not Found |
| `RATE_LIMITED` | Rate limit exceeded | "Retry after 1 second" | 429 Too Many Requests |
| `UPSTREAM_ERROR` | WikiPathways API error | "Retry later, upstream service unavailable" | 502 Bad Gateway |

---

## Example Calls

### Scenario 1: Valid Pathway Lookup
```python
pathway = await client.call_tool("get_pathway", {"pathway_id": "WP:WP534"})
# Returns complete Pathway entity with cross_references
```

### Scenario 2: Invalid Format - Raw Name
```python
result = await client.call_tool("get_pathway", {"pathway_id": "glycolysis"})
# Returns UNRESOLVED_ENTITY error with recovery hint
```

### Scenario 3: Invalid Format - Missing Prefix
```python
result = await client.call_tool("get_pathway", {"pathway_id": "WP534"})
# Returns UNRESOLVED_ENTITY error (expected WP:WP534)
```

### Scenario 4: Valid Format - Non-Existent ID
```python
result = await client.call_tool("get_pathway", {"pathway_id": "WP:WP99999"})
# Returns ENTITY_NOT_FOUND error
```

---

## Performance Targets

- **Response Time**: <2s for 95th percentile (SC-001)
- **Token Budget**: ~300 tokens per full Pathway entity
- **Rate Limit**: 1 req/s with exponential backoff
- **Reliability**: Zero failed requests for valid IDs (SC-004)

---

## Integration Points

### Fuzzy-to-Fact Workflow
1. **Phase 1**: User calls `search_pathways("glycolysis")`
   - Returns: `{"id": "WP:WP534", "title": "Glycolysis and gluconeogenesis", "score": 0.95}`
2. **Phase 2**: User calls `get_pathway("WP:WP534")` ← THIS TOOL
   - Returns: Complete Pathway with cross_references
3. **Phase 3**: User calls `get_pathway_components("WP:WP534")` for detailed analysis

### Cross-Database Integration
- **Cross-references** enable lookups in other servers:
  - `entrez` → Entrez MCP server
  - `uniprot` → UniProt MCP server
  - `hgnc` → HGNC MCP server
  - `kegg_pathway` → KEGG server (future)
  - `reactome` → Reactome server (future)

---

## Testing Requirements

### Unit Tests
- ✅ Pathway ID format validation (regex match)
- ✅ UNRESOLVED_ENTITY error for invalid formats
- ✅ Cross-reference mapping algorithm
- ✅ Component count aggregation
- ✅ Omit-if-null pattern for empty cross-references

### Integration Tests
- ✅ Live API call for known pathway (WP:WP534)
- ✅ Verify all required fields populated
- ✅ Verify cross_references contain valid identifiers
- ✅ ENTITY_NOT_FOUND for non-existent ID (WP:WP99999)
- ✅ UNRESOLVED_ENTITY for raw pathway name
- ✅ Error→hint→recovery→success cycle validation

---

## References

- [Feature Spec: User Story 2](../spec.md#user-story-2---detailed-pathway-information-retrieval-priority-p1)
- [Data Model: Pathway](../data-model.md#1-pathway-complete-entity)
- [Research: Response Format](../research.md#research-question-2-response-format)
- [ADR-001 §3: Fuzzy-to-Fact Protocol](../../../docs/adr/accepted/adr-001-v1.2.md)
