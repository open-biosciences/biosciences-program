# Quickstart: STRING MCP Server

**Purpose**: Get started with the STRING protein interaction network server
**Audience**: Developers integrating STRING into LLM agent workflows
**Time**: 5 minutes

## Prerequisites

```bash
# Python 3.11+ with uv package manager
python --version  # 3.11+

# Install dependencies
uv sync
```

## Running the Server

### Standalone Mode

```bash
uv run fastmcp run src/lifesciences_mcp/servers/string.py
```

Server starts on stdio (default MCP transport).

### Development Mode

```bash
uv run fastmcp dev src/lifesciences_mcp/servers/string.py
```

Interactive testing UI available at `http://localhost:8000`

## Basic Workflows

### Workflow 1: Fuzzy-to-Fact Protein Interaction Lookup

**Goal**: Find TP53 interaction partners with high confidence

```python
# Step 1: Fuzzy search for TP53
search_result = await client.call_tool("search_proteins", {
    "query": "TP53",
    "species": 9606,  # Human
    "limit": 10
})

# Result: PaginationEnvelope with ranked candidates
{
  "items": [
    {
      "id": "STRING:9606.ENSP00000269305",
      "preferred_name": "TP53",
      "annotation": "Cellular tumor antigen p53",
      "taxon_id": 9606,
      "score": 0.95
    }
  ],
  "pagination": {"cursor": null, "total_count": 1, "page_size": 10}
}

# Step 2: Get interaction network with resolved CURIE
network = await client.call_tool("get_interactions", {
    "string_id": "STRING:9606.ENSP00000269305",
    "required_score": 700,  # High confidence
    "limit": 10
})

# Result: InteractionNetwork with MDM2, ATM, BRCA1, etc.
{
  "id": "STRING:9606.ENSP00000269305",
  "preferred_name": "TP53",
  "interaction_count": 10,
  "interactions": [
    {
      "string_id_a": "9606.ENSP00000269305",
      "string_id_b": "9606.ENSP00000258149",
      "preferred_name_a": "TP53",
      "preferred_name_b": "MDM2",
      "score": 0.999,
      "evidence": {
        "nscore": 0.0,
        "fscore": 0.0,
        "pscore": 0.12,
        "ascore": 0.45,
        "escore": 0.92,  # Strong experimental evidence
        "dscore": 0.90,  # High database curation
        "tscore": 0.99   # Extensive literature support
      }
    },
    ...
  ],
  "network_image_url": "https://string-db.org/api/highres_image/network?..."
}
```

### Workflow 2: Evidence-Filtered Interaction Discovery

**Goal**: Find experimental-only interactions (no text mining)

```python
# Get all interactions
network = await client.call_tool("get_interactions", {
    "string_id": "STRING:9606.ENSP00000269305",
    "required_score": 0,  # Get all
    "limit": 100
})

# Filter for experimental evidence > 0.7
experimental_interactions = [
    interaction for interaction in network["interactions"]
    if interaction["evidence"]["escore"] > 0.7
]

# Result: Only experimentally validated interactions
# (Y2H, affinity capture, co-IP, etc.)
```

### Workflow 3: Network Visualization

**Goal**: Generate shareable network diagram

```python
# Generate visualization URL
url = client.call_tool("get_network_image_url", {
    "identifiers": "TP53\nMDM2\nATM\nBRCA1",  # Multi-protein network
    "species": 9606,
    "add_nodes": 5,
    "network_flavor": "evidence"  # Color by evidence type
})

# Result: High-res PNG URL
"https://string-db.org/api/highres_image/network?identifiers=TP53%0dMDM2%0dATM%0dBRCA1&species=9606&add_nodes=5&network_flavor=evidence"
```

### Workflow 4: Cross-Species Comparison

**Goal**: Compare TP53 interactions in human vs. mouse

```python
# Human TP53
human_search = await client.call_tool("search_proteins", {
    "query": "TP53",
    "species": 9606
})
human_network = await client.call_tool("get_interactions", {
    "string_id": human_search["items"][0]["id"],
    "required_score": 700
})

# Mouse Tp53
mouse_search = await client.call_tool("search_proteins", {
    "query": "Tp53",
    "species": 10090
})
mouse_network = await client.call_tool("get_interactions", {
    "string_id": mouse_search["items"][0]["id"],
    "required_score": 700
})

# Compare conserved interactions
human_partners = {i["preferred_name_b"] for i in human_network["interactions"]}
mouse_partners = {i["preferred_name_b"] for i in mouse_network["interactions"]}
conserved = human_partners & mouse_partners
```

## Error Handling

### Example: Invalid CURIE Recovery

```python
# Mistake: Missing "STRING:" prefix
result = await client.call_tool("get_interactions", {
    "string_id": "9606.ENSP00000269305"  # Wrong!
})

# Error response:
{
  "success": false,
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "The input '9606.ENSP00000269305' is not a valid STRING CURIE.",
    "recovery_hint": "Call search_proteins to resolve the identifier first. Expected format: STRING:<taxid>.ENSP<id>",
    "invalid_input": "9606.ENSP00000269305"
  }
}

# Fix: Use fuzzy search first
search_result = await client.call_tool("search_proteins", {
    "query": "TP53"
})
correct_id = search_result["items"][0]["id"]  # "STRING:9606.ENSP00000269305"
```

## Common Patterns

### Pattern: High-Confidence Filter

```python
# Only highest confidence (combined score > 0.9)
network = await client.call_tool("get_interactions", {
    "string_id": "STRING:9606.ENSP00000269305",
    "required_score": 900,  # 0.9 threshold
    "limit": 50
})
```

### Pattern: Database-Curated Only

```python
# Only curated pathway/complex data
network = await client.call_tool("get_interactions", {
    "string_id": "STRING:9606.ENSP00000269305",
    "required_score": 0
})

curated = [
    i for i in network["interactions"]
    if i["evidence"]["dscore"] > 0.8
]
```

### Pattern: Pagination Through Search Results

```python
all_candidates = []
cursor = None

while True:
    page = await client.call_tool("search_proteins", {
        "query": "kinase",
        "limit": 50,
        "cursor": cursor
    })

    all_candidates.extend(page["items"])
    cursor = page["pagination"]["cursor"]

    if cursor is None:
        break  # No more pages
```

## Rate Limiting

STRING enforces **1 request/second**. The client handles this automatically:

```python
# These requests are automatically rate-limited (1 req/s)
for gene in ["TP53", "BRCA1", "EGFR", "KRAS"]:
    search = await client.call_tool("search_proteins", {"query": gene})
    network = await client.call_tool("get_interactions", {
        "string_id": search["items"][0]["id"]
    })

# Client ensures 1 second between requests + exponential backoff on 429
```

## Testing

```bash
# Run integration tests (requires internet)
uv run pytest tests/integration/test_string_api.py -v -m integration

# Run all tests
uv run pytest tests/ -v
```

## Next Steps

- **Advanced**: Filter by specific evidence types (experimental, co-expression, etc.)
- **Integration**: Combine with HGNC for gene symbol validation
- **Integration**: Combine with UniProt for protein function annotation
- **Visualization**: Use network_image_url for presentations/publications

## References

- STRING Database: https://string-db.org
- API Documentation: https://string-db.org/help/api/
- Evidence Score Interpretation: See [data-model.md](data-model.md)
- Tool Contracts: See [contracts/](contracts/)
