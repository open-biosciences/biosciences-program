# Quickstart: BioGRID MCP Server

**Purpose**: Get started with the BioGRID genetic and protein interaction server
**Audience**: Developers integrating BioGRID into LLM agent workflows
**Time**: 5 minutes

## Prerequisites

```bash
# Python 3.11+ with uv package manager
python --version  # 3.11+

# Install dependencies
uv sync

# Get free BioGRID API key
# 1. Register at https://webservice.thebiogrid.org/
# 2. Check email for confirmation and API key
# 3. Set environment variable
export BIOGRID_API_KEY="your-key-here"
```

## Running the Server

### Standalone Mode

```bash
uv run fastmcp run src/lifesciences_mcp/servers/biogrid.py
```

Server starts on stdio (default MCP transport).

### Development Mode

```bash
uv run fastmcp dev src/lifesciences_mcp/servers/biogrid.py
```

Interactive testing UI available at `http://localhost:8000`

## Basic Workflows

### Workflow 1: Gene Symbol Validation → Interaction Query

**Goal**: Find TP53 interaction partners with experimental evidence

```python
# Step 1: Search and confirm gene exists in BioGRID
search_result = await client.call_tool("search_genes", {
    "query": "TP53",
    "organism": 9606  # Human
})

# Result: PaginationEnvelope with confirmed gene and interaction count
{
  "items": [
    {
      "symbol": "TP53",
      "organism": "Homo sapiens",
      "taxon_id": 9606,
      "interaction_count": 14523
    }
  ],
  "pagination": {"cursor": null, "total_count": 1, "page_size": 1}
}

# Step 2: Get interaction network with confirmed symbol
interactions = await client.call_tool("get_interactions", {
    "gene_symbol": "TP53",
    "organism": 9606,
    "max_results": 10
})

# Result: InteractionResult with MDM2, ATM, BRCA1, etc.
{
  "query_gene": "TP53",
  "interactions": [
    {
      "biogrid_interaction_id": 1234567,
      "symbol_a": "TP53",
      "symbol_b": "MDM2",
      "experimental_system": "Affinity Capture-Western",
      "experimental_system_type": "physical",
      "pubmed_id": 12345678,
      "throughput": "Low Throughput",
      "organism_a_id": 9606,
      "organism_b_id": 9606,
      "entrez_gene_a": 7157,
      "entrez_gene_b": 4193
    },
    ...
  ],
  "cross_references": {
    "entrez": "7157"
  },
  "physical_count": 8,
  "genetic_count": 2,
  "total_count": 10
}
```

### Workflow 2: Evidence-Filtered Interaction Discovery

**Goal**: Find low-throughput (high-confidence) interactions only

```python
# Get all interactions
interactions = await client.call_tool("get_interactions", {
    "gene_symbol": "TP53",
    "max_results": 1000
})

# Filter for low-throughput studies
high_confidence = [
    interaction for interaction in interactions["interactions"]
    if interaction.get("throughput") == "Low Throughput"
]

# Result: Only focused experimental validations
# (excludes systematic screens with higher false positive rates)
```

### Workflow 3: Physical vs Genetic Interaction Analysis

**Goal**: Compare physical binding vs genetic epistasis for TP53

```python
# Get interactions
result = await client.call_tool("get_interactions", {
    "gene_symbol": "TP53",
    "max_results": 1000
})

# Separate by type
physical = [
    i for i in result["interactions"]
    if i["experimental_system_type"] == "physical"
]
genetic = [
    i for i in result["interactions"]
    if i["experimental_system_type"] == "genetic"
]

# Analyze experimental systems
from collections import Counter

physical_methods = Counter([i["experimental_system"] for i in physical])
genetic_methods = Counter([i["experimental_system"] for i in genetic])

print(f"Physical methods: {physical_methods.most_common(3)}")
# [('Affinity Capture-Western', 120), ('Two-hybrid', 85), ('Co-purification', 45)]

print(f"Genetic methods: {genetic_methods.most_common(3)}")
# [('Synthetic Lethality', 30), ('Negative Genetic', 15), ('Dosage Rescue', 5)]
```

### Workflow 4: Cross-Species Comparison

**Goal**: Compare TP53 interactions in human vs. mouse

```python
# Human TP53
human_result = await client.call_tool("get_interactions", {
    "gene_symbol": "TP53",
    "organism": 9606,  # Homo sapiens
    "max_results": 100
})

# Mouse Tp53 (note: BioGRID normalizes to uppercase)
mouse_result = await client.call_tool("get_interactions", {
    "gene_symbol": "TP53",
    "organism": 10090,  # Mus musculus
    "max_results": 100
})

# Compare conserved interactions
human_partners = {i["symbol_b"] for i in human_result["interactions"]}
mouse_partners = {i["symbol_b"] for i in mouse_result["interactions"]}
conserved = human_partners & mouse_partners

print(f"Conserved interactions: {conserved}")
# {'MDM2', 'ATM', 'BRCA1', ...}
```

## Error Handling

### Example: Missing API Key Recovery

```python
# Mistake: Forgot to set BIOGRID_API_KEY
result = await client.call_tool("search_genes", {
    "query": "TP53"
})

# Error response:
{
  "success": false,
  "error": {
    "code": "UPSTREAM_ERROR",
    "message": "BioGRID API returned: Invalid accesskey",
    "recovery_hint": "Set BIOGRID_API_KEY environment variable. Get free key at https://webservice.thebiogrid.org/",
    "invalid_input": ""
  }
}

# Fix: Set API key
export BIOGRID_API_KEY="your-key-here"
```

### Example: Invalid Gene Symbol Recovery

```python
# Mistake: Query too short
result = await client.call_tool("search_genes", {
    "query": "A"
})

# Error response:
{
  "success": false,
  "error": {
    "code": "AMBIGUOUS_QUERY",
    "message": "Query must be at least 2 characters",
    "recovery_hint": "Provide a more specific gene symbol (minimum 2 characters)",
    "invalid_input": "A"
  }
}

# Fix: Use at least 2 characters
result = await client.call_tool("search_genes", {
    "query": "TP53"
})
```

### Example: Gene Not Found in BioGRID

```python
# Mistake: Gene doesn't exist in BioGRID
result = await client.call_tool("search_genes", {
    "query": "ZZZZZ99"
})

# Error response (search_genes catches this at Phase 1):
{
  "success": false,
  "error": {
    "code": "ENTITY_NOT_FOUND",
    "message": "Gene not found in BioGRID: ZZZZZ99",
    "recovery_hint": "Verify gene symbol is correct. Use HGNC search_genes for official symbol resolution.",
    "invalid_input": "ZZZZZ99"
  }
}

# Fix: Use a valid gene symbol
search = await client.call_tool("search_genes", {
    "query": "TP53"
})
# Returns interaction_count confirming gene exists
# Then use confirmed symbol for get_interactions
```

## Common Patterns

### Pattern: High-Confidence Physical Interactions Only

```python
# Get interactions
result = await client.call_tool("get_interactions", {
    "gene_symbol": "TP53",
    "max_results": 1000
})

# Filter for low-throughput physical interactions
high_confidence_physical = [
    i for i in result["interactions"]
    if i["experimental_system_type"] == "physical"
    and i.get("throughput") == "Low Throughput"
]
```

### Pattern: Literature-Backed Interactions

```python
# Get interactions with PubMed references
result = await client.call_tool("get_interactions", {
    "gene_symbol": "TP53",
    "max_results": 1000
})

# Filter for interactions with PubMed IDs
literature_backed = [
    i for i in result["interactions"]
    if i.get("pubmed_id") is not None
]
```

### Pattern: Cross-Database Integration via Entrez

```python
# Get BioGRID interactions
biogrid_result = await client.call_tool("get_interactions", {
    "gene_symbol": "TP53"
})

# Extract Entrez Gene ID
entrez_id = biogrid_result["cross_references"].get("entrez")

if entrez_id:
    # Use with other tools that accept Entrez IDs
    # (e.g., NCBI Gene, ClinVar, PubMed)
    print(f"Query NCBI with Entrez ID: {entrez_id}")
```

### Pattern: Experimental System Filtering

```python
# Get interactions
result = await client.call_tool("get_interactions", {
    "gene_symbol": "TP53",
    "max_results": 1000
})

# Filter by specific experimental system
two_hybrid = [
    i for i in result["interactions"]
    if i["experimental_system"] == "Two-hybrid"
]

affinity_capture = [
    i for i in result["interactions"]
    if "Affinity Capture" in i["experimental_system"]
]
```

## Rate Limiting

BioGRID enforces rate limits (estimated 2 req/sec). The client handles this automatically:

```python
# These requests are automatically rate-limited (2 req/sec)
for gene in ["TP53", "BRCA1", "EGFR", "KRAS"]:
    search = await client.call_tool("search_genes", {"query": gene})
    interactions = await client.call_tool("get_interactions", {
        "gene_symbol": search["items"][0]["symbol"],
        "max_results": 100
    })

# Client ensures 500ms between requests + exponential backoff on 429
```

## Testing

```bash
# Set API key for integration tests
export BIOGRID_API_KEY="your-key-here"

# Run integration tests (requires internet + API key)
uv run pytest tests/integration/test_biogrid_api.py -v -m integration

# Run all tests
uv run pytest tests/ -v
```

## API Key Management

### Getting an API Key

1. Visit https://webservice.thebiogrid.org/
2. Click "Request an API Key"
3. Fill registration form (free, no credit card)
4. Check email for confirmation and API key
5. Set environment variable:

```bash
# Linux/Mac
export BIOGRID_API_KEY="your-key-here"

# Add to .env file (recommended)
echo "BIOGRID_API_KEY=your-key-here" >> .env
```

### Troubleshooting API Key Issues

| Symptom | Cause | Solution |
|---------|-------|----------|
| "Invalid accesskey" | API key not set | Set BIOGRID_API_KEY environment variable |
| "Invalid accesskey" | Wrong API key format | Copy key exactly from BioGRID email (no spaces) |
| "Invalid accesskey" | Email not confirmed | Check spam folder for BioGRID confirmation email |

## Next Steps

- **Advanced**: Filter by specific evidence types (Affinity Capture, Two-hybrid, etc.)
- **Integration**: Combine with HGNC for gene symbol validation across databases
- **Integration**: Combine with UniProt for protein function annotation
- **Analysis**: Export to NetworkX/igraph for network analysis
- **Visualization**: Use Cytoscape or similar tools for network visualization

## References

- BioGRID Database: https://thebiogrid.org
- API Documentation: https://wiki.thebiogrid.org/doku.php/biogridrest
- Experimental Evidence Types: https://wiki.thebiogrid.org/doku.php/experimental_systems
- Tool Contracts: See [contracts/](contracts/)
- Data Models: See [data-model.md](data-model.md)
