# Quickstart: DrugBank MCP Server

**Server**: DrugBank MCP Server
**Version**: 0.1.0 (in development)
**Date**: 2025-12-22

## Overview

The DrugBank MCP Server provides drug discovery researchers with tools to search and retrieve drug information from DrugBank. It implements the Fuzzy-to-Fact protocol for reliable entity resolution.

## Starting the Server

```bash
# Run the DrugBank MCP server
uv run fastmcp run src/lifesciences_mcp/servers/drugbank.py
```

## Available Tools

| Tool | Type | Description |
|------|------|-------------|
| `search_drugs` | Fuzzy | Search for drugs by name, brand name, or indication |
| `get_drug` | Strict | Get complete drug record by DrugBank CURIE |

## Workflow 1: Fuzzy-to-Fact Drug Discovery

The recommended workflow for finding and retrieving drug information:

```python
# Phase 1: Fuzzy Search - Find drug candidates
search_result = await client.call_tool("search_drugs", {
    "query": "aspirin",
    "page_size": 10
})

# Returns PaginationEnvelope with ranked candidates:
# {
#   "items": [
#     {"id": "DrugBank:DB00945", "name": "Aspirin", "score": 1.0, ...},
#     ...
#   ],
#   "pagination": {"cursor": "...", "total_count": 5, "page_size": 10}
# }

# Phase 2: Extract the best match
top_candidate = search_result["items"][0]
drugbank_id = top_candidate["id"]  # "DrugBank:DB00945"

# Phase 3: Strict Lookup - Get complete drug record
drug = await client.call_tool("get_drug", {
    "drugbank_id": drugbank_id
})

# Returns complete drug with cross-references:
# {
#   "id": "DrugBank:DB00945",
#   "name": "Aspirin",
#   "drug_type": "Small molecule",
#   "indication": "For the treatment of mild to moderate pain...",
#   "mechanism_of_action": "Irreversibly inhibits COX enzymes...",
#   "cross_references": {
#     "chembl": "CHEMBL:25",
#     "uniprot": ["UniProtKB:P23219", "UniProtKB:P35354"],
#     "kegg": "D00109"
#   }
# }
```

## Workflow 2: Cross-Database Navigation

Navigate from DrugBank to other databases using cross-references:

```python
# Get drug with cross-references
drug = await client.call_tool("get_drug", {
    "drugbank_id": "DrugBank:DB00945"
})

# Navigate to ChEMBL for compound data
chembl_id = drug["cross_references"].get("chembl")
if chembl_id:
    # Use ChEMBL MCP server to get compound details
    compound = await chembl_client.call_tool("get_compound", {
        "chembl_id": chembl_id
    })

# Navigate to UniProt for target proteins
uniprot_ids = drug["cross_references"].get("uniprot", [])
for uniprot_id in uniprot_ids:
    # Use UniProt MCP server to get protein details
    protein = await uniprot_client.call_tool("get_protein", {
        "uniprot_id": uniprot_id
    })
```

## Workflow 3: Token-Efficient Batch Operations

Use slim mode for efficient list processing:

```python
# Search with slim mode for quick triage
search_result = await client.call_tool("search_drugs", {
    "query": "diabetes",
    "slim": True,  # ~20 tokens per result
    "page_size": 50
})

# Each item contains only: id, name, drug_type, categories, score
# Total: ~1000 tokens for 50 drugs instead of ~5000+ tokens

# Only retrieve full details for selected candidates
selected_ids = ["DrugBank:DB00331", "DrugBank:DB01268"]
for drug_id in selected_ids:
    drug = await client.call_tool("get_drug", {
        "drugbank_id": drug_id,
        "slim": False  # Full details for selected drugs
    })
```

## Workflow 4: Error Recovery

Handle common error scenarios:

```python
# Example 1: Invalid CURIE format
result = await client.call_tool("get_drug", {
    "drugbank_id": "DB00945"  # Missing prefix!
})
# Returns:
# {
#   "success": false,
#   "error": {
#     "code": "UNRESOLVED_ENTITY",
#     "message": "Invalid DrugBank CURIE format",
#     "recovery_hint": "Use format DrugBank:DBXXXXX (e.g., DrugBank:DB00945)",
#     "invalid_input": "DB00945"
#   }
# }

# Fix: Use correct CURIE format
result = await client.call_tool("get_drug", {
    "drugbank_id": "DrugBank:DB00945"  # Correct!
})

# Example 2: Drug not found
result = await client.call_tool("get_drug", {
    "drugbank_id": "DrugBank:DB99999"
})
# Returns:
# {
#   "success": false,
#   "error": {
#     "code": "ENTITY_NOT_FOUND",
#     "message": "Drug DrugBank:DB99999 not found",
#     "recovery_hint": "Use search_drugs to find valid DrugBank IDs"
#   }
# }

# Example 3: Query too short
result = await client.call_tool("search_drugs", {
    "query": "a"  # Too short!
})
# Returns:
# {
#   "success": false,
#   "error": {
#     "code": "AMBIGUOUS_QUERY",
#     "message": "Query 'a' is too short",
#     "recovery_hint": "Minimum query length is 2 characters"
#   }
# }
```

## DrugBank ID Format

DrugBank uses a standardized ID format:

| Format | Example | Description |
|--------|---------|-------------|
| Raw ID | `DB00945` | Internal DrugBank identifier |
| CURIE | `DrugBank:DB00945` | Required format for `get_drug` |

**Important**: The `get_drug` tool requires the full CURIE format (`DrugBank:DBXXXXX`). The `search_drugs` tool returns CURIEs in this format.

## Cross-Reference Keys

DrugBank drugs may include cross-references to these databases:

| Key | Database | Example |
|-----|----------|---------|
| `chembl` | ChEMBL | `"CHEMBL:25"` |
| `uniprot` | UniProt | `["UniProtKB:P23219"]` |
| `kegg` | KEGG Drug | `"D00109"` |
| `pubchem_compound` | PubChem | `"2244"` |
| `pdb` | Protein Data Bank | `"1BNA"` |

**Note**: Keys are omitted if no cross-reference exists (never null or empty).

## Error Codes

| Code | Meaning | Recovery |
|------|---------|----------|
| `UNRESOLVED_ENTITY` | Invalid CURIE format | Use `search_drugs` to find valid IDs |
| `ENTITY_NOT_FOUND` | Drug not in database | Verify ID or search for alternatives |
| `AMBIGUOUS_QUERY` | Query too short/vague | Use more specific search terms |
| `RATE_LIMITED` | Too many requests | Wait and retry with backoff |
| `UPSTREAM_ERROR` | DrugBank API issue | Retry after delay |

## Performance Expectations

| Metric | Target | Notes |
|--------|--------|-------|
| Search latency | <2s (95th percentile) | For typical queries |
| Lookup latency | <1s (100%) | For valid CURIEs |
| Relevance | >90% top-5 accuracy | For common drug names |
| Rate limit | 10 req/s | Automatic backoff on 429 |

## Next Steps

After retrieving drug data, common next steps include:

1. **Navigate to ChEMBL** - Get compound structure and bioactivity data
2. **Navigate to UniProt** - Explore drug target proteins
3. **Navigate to KEGG** - Analyze metabolic pathways
4. **Navigate to Open Targets** - Find target-disease associations

See the respective MCP server documentation for cross-database workflows.
