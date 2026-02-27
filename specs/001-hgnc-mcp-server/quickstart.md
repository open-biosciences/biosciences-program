# Quickstart: HGNC MCP Server

**Feature**: 001-hgnc-mcp-server
**Date**: 2025-12-21

## Overview

The HGNC MCP Server provides gene resolution tools for AI agents, implementing the
Fuzzy-to-Fact protocol from ADR-001. It exposes two tools:

1. `search_genes` — Fuzzy search returning ranked candidates
2. `get_gene` — Strict lookup by HGNC CURIE

## Installation

```bash
# Install dependencies
uv sync

# Run the server (stdio transport)
uv run fastmcp run src/lifesciences_mcp/servers/hgnc.py
```

## Usage Examples

### Example 1: Basic Gene Resolution (Fuzzy-to-Fact)

The standard workflow for resolving a gene name to its authoritative record:

```python
# Step 1: Fuzzy search for candidates
result = await client.call_tool("search_genes", {"query": "BRCA1"})

# Response:
{
  "items": [
    {"id": "HGNC:1100", "symbol": "BRCA1", "name": "BRCA1 DNA repair associated", "score": 1.0},
    {"id": "HGNC:1101", "symbol": "BRCA2", "name": "BRCA2 DNA repair associated", "score": 0.8}
  ],
  "pagination": {"cursor": null, "total_count": 2, "page_size": 50}
}

# Step 2: Strict lookup for the resolved ID
gene = await client.call_tool("get_gene", {"hgnc_id": "HGNC:1100"})

# Response:
{
  "id": "HGNC:1100",
  "symbol": "BRCA1",
  "name": "BRCA1 DNA repair associated",
  "status": "Approved",
  "location": "17q21.31",
  "cross_references": {
    "ensembl_gene": "ENSG00000012048",
    "uniprot": ["P38398"],
    "entrez": "672",
    "omim": "113705"
  }
}
```

### Example 2: Token-Efficient Batch Search

For broad searches, use slim mode to reduce token usage:

```python
# Slim mode returns only id, name, score (~20 tokens per entity)
result = await client.call_tool("search_genes", {
    "query": "kinase",
    "slim": True,
    "page_size": 50
})

# Response (compact):
{
  "items": [
    {"id": "HGNC:...", "symbol": "...", "name": "...", "score": 0.95},
    # ... 49 more items, each ~20 tokens
  ],
  "pagination": {"cursor": "eyJvZmZzZXQiOjUwfQ==", "total_count": 1234, "page_size": 50}
}

# Get next page
next_page = await client.call_tool("search_genes", {
    "query": "kinase",
    "slim": True,
    "cursor": "eyJvZmZzZXQiOjUwfQ=="
})
```

### Example 3: Error Handling

Agents receive structured errors with recovery hints:

```python
# Passing raw string to strict tool
result = await client.call_tool("get_gene", {"hgnc_id": "BRCA1"})

# Response:
{
  "success": false,
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "The input 'BRCA1' is not a valid HGNC CURIE.",
    "recovery_hint": "Call search_genes to resolve the identifier first.",
    "invalid_input": "BRCA1"
  }
}

# Agent can self-correct by calling search_genes
```

### Example 4: Cross-Reference Triangulation

Verify gene identity across multiple databases:

```python
gene = await client.call_tool("get_gene", {"hgnc_id": "HGNC:1100"})

# Triangulation check: verify BRCA1 exists in UniProt
uniprot_ids = gene["cross_references"].get("uniprot", [])
# ["P38398"] — can be verified against UniProt API

# Triangulation check: verify in Ensembl
ensembl_id = gene["cross_references"].get("ensembl_gene")
# "ENSG00000012048" — can be verified against Ensembl API
```

## Agent System Prompt Integration

Add this to the agent's system prompt for optimal usage:

```text
When resolving gene names:
1. ALWAYS call search_genes first to get valid HGNC CURIEs
2. Select the highest-scoring candidate that matches context
3. Call get_gene with the resolved CURIE for full details
4. Use slim=True for broad searches to conserve tokens
5. If get_gene returns UNRESOLVED_ENTITY, call search_genes first
```

## Error Code Reference

| Code | Meaning | Agent Action |
|------|---------|--------------|
| UNRESOLVED_ENTITY | Raw string to strict tool | Call search_genes first |
| ENTITY_NOT_FOUND | CURIE not in HGNC | Verify ID, try synonym |
| AMBIGUOUS_QUERY | Too many results (>100) | Refine query terms |
| RATE_LIMITED | HGNC API throttled | Retry with backoff |
| UPSTREAM_ERROR | HGNC API unavailable | Report to user |

## Testing

```bash
# Run all tests
uv run pytest tests/ -v

# Run only unit tests (no network)
uv run pytest tests/unit/ -v

# Run contract tests
uv run pytest tests/contract/ -v

# Run integration tests (requires network)
uv run pytest tests/integration/ -v -m integration
```

## Performance Expectations

| Operation | Expected Latency |
|-----------|------------------|
| search_genes (first page) | <500ms |
| get_gene (single lookup) | <300ms |
| Pagination (subsequent pages) | <100ms (cached) |

Note: Latency depends on HGNC API response time. The server adds minimal overhead.
