# Tool Contract: get_pathway_components

**Type**: Strict Extraction
**Purpose**: Extract all biological entities (genes, proteins, metabolites) and interactions from a pathway
**Returns**: PathwayComponents entity with structured component lists

---

## Input Parameters

| Parameter | Type | Required | Default | Validation | Description |
|-----------|------|----------|---------|------------|-------------|
| `pathway_id` | string | ✅ Yes | N/A | Pattern: `^WP:WP\d+$` | WikiPathways CURIE (e.g., "WP:WP534") |

---

## Output Schema

### Success Response (PathwayComponents Entity)

```json
{
  "genes": [
    {
      "id": "ncbigene:5213",
      "label": "PFKM",
      "type": "Gene",
      "database": "Entrez Gene",
      "cross_references": {
        "entrez": "5213",
        "hgnc": "HGNC:8877",
        "ensembl_gene": "ENSG00000152556"
      }
    }
  ],
  "proteins": [
    {
      "id": "uniprot:P08237",
      "label": "PFKM",
      "type": "Protein",
      "database": "UniProt",
      "cross_references": {
        "uniprot": "P08237",
        "hgnc": "HGNC:8877"
      }
    }
  ],
  "metabolites": [
    {
      "id": "chebi:17234",
      "label": "Glucose",
      "type": "Metabolite",
      "database": "ChEBI",
      "cross_references": {
        "pubchem_compound": "5793"
      }
    }
  ],
  "interactions": []
}
```

**Note**: `interactions` list is typically empty (WikiPathways API doesn't provide interaction data via REST endpoints). Future enhancement may extract from GPML.

### Error Response: UNRESOLVED_ENTITY

```json
{
  "success": false,
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "Invalid pathway ID format 'glycolysis'. Expected WP:WPNNNNN format",
    "recovery_hint": "Call search_pathways to resolve pathway identifier first",
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
    "recovery_hint": "Verify pathway ID or use search_pathways to find valid pathway",
    "invalid_input": "WP:WP99999"
  }
}
```

---

## Behavior Specification

### Input Validation
1. **Format Check**: Validate `pathway_id` matches regex `^WP:WP\d+$`
2. **Extract Numeric ID**: Strip `WP:WP` prefix for API call (`WP:WP534` → `534`)
3. **Error Handling**: Return `UNRESOLVED_ENTITY` for invalid format

### API Call Sequence
Fetch components by BridgeDb system code:

| API Call | BridgeDb Code | Entity Type | Cross-Reference Key |
|----------|---------------|-------------|---------------------|
| `/getXrefList?pwId=WP534&code=L` | `L` | Genes | `entrez` |
| `/getXrefList?pwId=WP534&code=H` | `H` | Genes | `hgnc` |
| `/getXrefList?pwId=WP534&code=S` | `S` | Proteins | `uniprot` |
| `/getXrefList?pwId=WP534&code=Ce` | `Ce` | Metabolites | `chembl` |
| `/getXrefList?pwId=WP534&code=Ch` | `Ch` | Metabolites | `chebi` |
| `/getXrefList?pwId=WP534&code=En` | `En` | Genes | `ensembl_gene` |

### Component Classification

#### Genes
- **Sources**: BridgeDb codes `L` (Entrez), `H` (HGNC), `En` (Ensembl)
- **Type**: `"Gene"`
- **Database**: `"Entrez Gene"`, `"HGNC"`, or `"Ensembl"`
- **ID Format**: `ncbigene:5213`, `hgnc:HGNC:8877`, `ensembl:ENSG00000152556`

#### Proteins
- **Source**: BridgeDb code `S` (UniProt)
- **Type**: `"Protein"`
- **Database**: `"UniProt"`
- **ID Format**: `uniprot:P08237`
- **UniProt Isoforms**: Split on semicolon, use first (canonical) isoform

#### Metabolites
- **Sources**: BridgeDb codes `Ce` (ChEMBL), `Ch` (CHEBI)
- **Type**: `"Metabolite"`
- **Database**: `"ChEMBL"`, `"CHEBI"`, or `"PubChem"`
- **ID Format**: `chembl:CHEMBL25`, `chebi:17234`, `pubchem:5793`

### Cross-Reference Enrichment
- **Load bulk JSON**: Parse `findPathwaysByXref.json` for pathway-level cross-references
- **Merge DataNode XRefs**: Combine API results with bulk file data
- **Omit Empty Keys**: Never include `null` or empty cross-references

### Interaction Extraction (Future)
- **Current**: Return empty `interactions` list
- **Future Enhancement**: Parse GPML XML to extract:
  - Source/target DataNode IDs
  - Interaction types (activation, inhibition, binding, conversion, catalysis)
  - Evidence codes (if available)

### Omit Empty Lists
- **Pattern**: If pathway has no metabolites, OMIT `metabolites` key entirely
- **Rationale**: Token efficiency (ADR-001 §4)
- **Example**: Signaling pathway with only genes/proteins → omit metabolites

---

## Error Codes

| Code | Trigger | Recovery Hint | HTTP Analogy |
|------|---------|---------------|--------------|
| `UNRESOLVED_ENTITY` | Invalid ID format | "Call search_pathways to resolve identifier" | 400 Bad Request |
| `ENTITY_NOT_FOUND` | Valid format but ID doesn't exist | "Verify ID or search for valid pathway" | 404 Not Found |
| `RATE_LIMITED` | Rate limit exceeded | "Retry after 1 second" | 429 Too Many Requests |
| `UPSTREAM_ERROR` | WikiPathways API error | "Retry later, upstream service unavailable" | 502 Bad Gateway |

---

## Example Calls

### Scenario 1: Extract Components from Glycolysis Pathway
```python
components = await client.call_tool("get_pathway_components", {
    "pathway_id": "WP:WP534"
})
# Returns genes (PFKM, HK1, etc.), proteins (P08237, etc.), metabolites (glucose, pyruvate)
```

### Scenario 2: Pathway with No Metabolites
```python
components = await client.call_tool("get_pathway_components", {
    "pathway_id": "WP:WP4868"  # RAC1/PAK1 signaling pathway
})
# Returns {"genes": [...], "proteins": [...], "interactions": []}
# NOTE: "metabolites" key is OMITTED (not empty list)
```

### Scenario 3: Invalid Format Error
```python
result = await client.call_tool("get_pathway_components", {
    "pathway_id": "glycolysis"
})
# Returns UNRESOLVED_ENTITY error
```

### Scenario 4: Non-Existent Pathway
```python
result = await client.call_tool("get_pathway_components", {
    "pathway_id": "WP:WP99999"
})
# Returns ENTITY_NOT_FOUND error
```

---

## Performance Targets

- **Response Time**: <2s for 95th percentile (SC-001)
- **Token Budget**: ~500-1000 tokens per PathwayComponents (varies by pathway size)
- **Rate Limit**: 1 req/s with exponential backoff
- **Data Completeness**: Zero data loss vs WikiPathways GPML source (SC-010)

---

## Integration Points

### Pathway Analysis Workflow
1. **Phase 1**: User finds pathway via `search_pathways("glycolysis")`
2. **Phase 2**: User retrieves details via `get_pathway("WP:WP534")`
3. **Phase 3**: User extracts components via `get_pathway_components("WP:WP534")` ← THIS TOOL
4. **Phase 4**: User performs downstream analysis:
   - Gene enrichment testing
   - Cross-database lookups (UniProt, HGNC, ChEMBL servers)
   - Network visualization

### Cross-Database Integration
- **Genes**: `entrez` IDs → Entrez MCP server
- **Proteins**: `uniprot` IDs → UniProt MCP server
- **Metabolites**: `chembl` IDs → ChEMBL MCP server
- **Reverse Lookup**: Compare with `get_pathways_for_gene` results

---

## Testing Requirements

### Unit Tests
- ✅ Pathway ID format validation
- ✅ BridgeDb code to entity type mapping
- ✅ UniProt isoform handling (semicolon splitting)
- ✅ Cross-reference merging logic
- ✅ Omit empty lists pattern
- ✅ UNRESOLVED_ENTITY error generation

### Integration Tests
- ✅ Live API call for known pathway (WP:WP534)
- ✅ Verify all component types extracted
- ✅ Verify cross_references populated
- ✅ Pathway with no metabolites (omit check)
- ✅ ENTITY_NOT_FOUND for non-existent ID
- ✅ UNRESOLVED_ENTITY for invalid format

---

## BridgeDb System Codes Reference

| Code | Database | Entity Type | Example ID | Cross-Ref Key |
|------|----------|-------------|------------|---------------|
| `L` | NCBI Gene | Gene | `5213` | `entrez` |
| `H` | HGNC | Gene | `HGNC:8877` | `hgnc` |
| `En` | Ensembl | Gene | `ENSG00000152556` | `ensembl_gene` |
| `S` | UniProt | Protein | `P08237` | `uniprot` |
| `Ce` | ChEMBL | Metabolite | `CHEMBL25` | `chembl` |
| `Ch` | CHEBI | Metabolite | `17234` | `chebi` |
| `Cp` | PubChem Compound | Metabolite | `5793` | `pubchem_compound` |

**Source**: [BridgeDb datasources.ts](https://github.com/bridgedb/datasources/blob/main/datasources.ts)

---

## Edge Cases

### Case 1: Pathway with Only Genes (No Proteins or Metabolites)
- **Behavior**: Omit `proteins` and `metabolites` keys entirely
- **Rationale**: Token efficiency (ADR-001 §4)

### Case 2: UniProt Isoforms in Response
- **Input**: `P08237;P51587` (semicolon-separated isoforms)
- **Behavior**: Split on `;`, use first isoform (`P08237`)
- **Rationale**: Canonical isoform is typically first

### Case 3: Empty Component List from API
- **Trigger**: API returns `{"xrefs": []}`
- **Behavior**: Omit component type from response (don't return empty list)

### Case 4: Duplicate Identifiers Across Codes
- **Example**: Gene appears in both `code=L` (Entrez) and `code=H` (HGNC)
- **Behavior**: Deduplicate by merging cross_references, keep single DataNode

---

## References

- [Feature Spec: User Story 4](../spec.md#user-story-4---pathway-component-extraction-priority-p3)
- [Data Model: PathwayComponents](../data-model.md#3-pathwaycomponents-component-extraction)
- [Research: API Endpoints](../research.md#research-question-1-api-endpoint-structure)
- [BridgeDb System Codes](../research.md#bridgedb-system-codes)
