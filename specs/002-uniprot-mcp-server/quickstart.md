# UniProt MCP Server - Quick Start Guide

**Status**: Complete (User Stories 1-4)
**Version**: 0.1.0
**Test Coverage**: 47/47 tests passing

## Overview

The UniProt MCP Server provides AI agents with access to protein sequence and annotation data through a **Fuzzy-to-Fact protocol**. Agents can search for proteins by name/gene/organism, then retrieve complete protein records with cross-references to 22+ biological databases.

## Installation

```bash
# Clone repository
git clone https://github.com/donbr/lifesciences-research.git
cd lifesciences-research

# Install dependencies
uv sync

# Verify installation
uv run pytest tests/integration/test_uniprot_api.py -v -m integration
```

## Running the Server

### Development Mode (with live reload)

```bash
uv run fastmcp dev src/lifesciences_mcp/servers/uniprot.py
```

### Production Mode

```bash
uv run fastmcp run src/lifesciences_mcp/servers/uniprot.py
```

## Available Tools

### 1. search_proteins (Fuzzy Search)

**Purpose**: Search for proteins by name, gene symbol, organism, or function.

**Parameters**:
- `query` (required): Search query (minimum 2 characters)
- `slim` (optional): Return minimal data for token efficiency (default: false)
- `cursor` (optional): Pagination cursor from previous response
- `page_size` (optional): Results per page, 1-500 (default: 50)

**Returns**: PaginationEnvelope with ProteinSearchCandidate items

**Example 1: Basic Search**
```python
# Search for p53 protein
{
  "query": "p53",
  "page_size": 10
}

# Response
{
  "items": [
    {
      "id": "UniProtKB:P04637",
      "name": "Cellular tumor antigen p53",
      "gene_symbol": "TP53",
      "organism": "Homo sapiens",
      "relevance_score": 1.0
    },
    ...
  ],
  "pagination": {
    "cursor": "next_page_cursor_string",
    "total_count": 45,
    "page_size": 10
  }
}
```

**Example 2: Organism-Specific Search**
```python
# Search for insulin in humans
{
  "query": "insulin human",
  "page_size": 5
}
```

**Example 3: Pagination**
```python
# Get next page of results
{
  "query": "kinase",
  "cursor": "next_page_cursor_from_previous_response",
  "page_size": 50
}
```

**Example 4: Token-Efficient Search**
```python
# Use slim mode for batch operations
{
  "query": "BRCA",
  "slim": true,
  "page_size": 100
}
```

### 2. get_protein (Strict Lookup)

**Purpose**: Retrieve complete protein record with cross-references by UniProt CURIE.

**Parameters**:
- `uniprot_id` (required): UniProt CURIE (format: `UniProtKB:P12345`)
- `slim` (optional): Return minimal data for token efficiency (default: false)

**Returns**: Protein model with full annotations or ErrorEnvelope

**Example 1: Get Complete Protein Record**
```python
# Get human p53 protein
{
  "uniprot_id": "UniProtKB:P04637"
}

# Response
{
  "id": "UniProtKB:P04637",
  "name": "Cellular tumor antigen p53",
  "gene_symbol": "TP53",
  "organism": "Homo sapiens",
  "sequence": "MEEPQSDPSVEPPLSQETFSDLWKLLPENNVLS...",
  "length": 393,
  "mass": 43653,
  "function": "Acts as a tumor suppressor...",
  "cross_references": {
    "hgnc": "HGNC:11998",
    "ensembl_gene": "ENSG00000141510",
    "refseq": "NP_000537.3",
    "pdb": ["1TUP", "1TSR", "2OCJ"],
    "omim": "191170",
    "kegg": "hsa:7157"
  }
}
```

**Example 2: Slim Mode (Token Efficient)**
```python
# Get minimal protein data (~20 tokens vs ~115-300 tokens)
{
  "uniprot_id": "UniProtKB:P38398",
  "slim": true
}

# Response (sequence and function omitted)
{
  "id": "UniProtKB:P38398",
  "name": "Breast cancer type 1 susceptibility protein",
  "gene_symbol": "BRCA1",
  "organism": "Homo sapiens",
  "length": 1863,
  "mass": 207721,
  "cross_references": {
    "hgnc": "HGNC:1100",
    "ensembl_gene": "ENSG00000012048",
    ...
  }
}
```

## Common Workflows

### Workflow 1: Fuzzy-to-Fact (Recommended)

**Use Case**: Agent doesn't know the exact UniProt ID but knows protein name or gene symbol.

```python
# Step 1: Fuzzy search by name
search_result = search_proteins({"query": "BRCA1", "page_size": 5})

# Step 2: Extract top candidate CURIE
top_result = search_result["items"][0]
protein_id = top_result["id"]  # "UniProtKB:P38398"

# Step 3: Get complete protein record
protein = get_protein({"uniprot_id": protein_id})

# Step 4: Access cross-references
hgnc_id = protein["cross_references"]["hgnc"]  # "HGNC:1100"
```

### Workflow 2: Cross-Database Navigation

**Use Case**: Agent needs to connect protein data to other databases.

```python
# Get protein with all cross-references
protein = get_protein({"uniprot_id": "UniProtKB:P04637"})

# Navigate to other databases
hgnc_id = protein["cross_references"]["hgnc"]        # "HGNC:11998" → Gene nomenclature
ensembl = protein["cross_references"]["ensembl_gene"] # "ENSG00000141510" → Genomics
pdb_ids = protein["cross_references"]["pdb"]          # ["1TUP", ...] → 3D structures
omim_id = protein["cross_references"]["omim"]         # "191170" → Disease associations
kegg_id = protein["cross_references"]["kegg"]         # "hsa:7157" → Pathways

# Use these IDs with other MCP servers (HGNC, Ensembl, PDB, etc.)
```

### Workflow 3: Batch Processing with Token Budgeting

**Use Case**: Agent needs to process many proteins efficiently.

```python
# Search in slim mode
search_result = search_proteins({
  "query": "kinase human",
  "slim": true,
  "page_size": 100
})

# Process all candidates (minimal token usage)
for candidate in search_result["items"]:
    protein_id = candidate["id"]
    # Get full details only for selected proteins
    protein = get_protein({"uniprot_id": protein_id, "slim": false})
```

### Workflow 4: Error Recovery (Autonomous Correction)

**Use Case**: Agent makes a mistake and receives actionable recovery hints.

```python
# Step 1: Agent tries invalid CURIE (missing prefix)
result = get_protein({"uniprot_id": "P04637"})

# Step 2: Error response with recovery hint
{
  "success": false,
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "Invalid UniProt CURIE format: 'P04637'",
    "recovery_hint": "Use search_proteins to find valid UniProt IDs, then call get_protein with the resolved CURIE (e.g., 'UniProtKB:P04637')",
    "invalid_input": "P04637"
  }
}

# Step 3: Agent follows hint
search_result = search_proteins({"query": "P04637", "page_size": 5})
valid_curie = search_result["items"][0]["id"]  # "UniProtKB:P04637"

# Step 4: Retry with valid CURIE
protein = get_protein({"uniprot_id": valid_curie})  # Success!
```

## Error Codes and Recovery

### AMBIGUOUS_QUERY
**Cause**: Query too short (< 2 characters)
**Recovery**: Provide more specific query terms

```python
# Error
search_proteins({"query": "p"})

# Recovery
search_proteins({"query": "p53"})
```

### UNRESOLVED_ENTITY
**Cause**: Invalid CURIE format (missing "UniProtKB:" prefix)
**Recovery**: Use search_proteins to find valid CURIE

```python
# Error
get_protein({"uniprot_id": "P04637"})

# Recovery
search_result = search_proteins({"query": "P04637"})
get_protein({"uniprot_id": search_result["items"][0]["id"]})
```

### ENTITY_NOT_FOUND
**Cause**: Valid CURIE format but protein doesn't exist
**Recovery**: Verify accession or search by protein name

```python
# Error
get_protein({"uniprot_id": "UniProtKB:A0A0B0B0C0"})

# Recovery
search_proteins({"query": "protein name"})
```

### RATE_LIMITED
**Cause**: Exceeded 10 requests/second limit
**Recovery**: Wait and retry (exponential backoff handled automatically)

```python
# Error (after many rapid requests)
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded",
    "recovery_hint": "Retry after 5 seconds"
  }
}

# Recovery: Client handles automatically with exponential backoff
```

### UPSTREAM_ERROR
**Cause**: UniProt API unavailable or returned error
**Recovery**: Retry request after brief delay

```python
# Error
{
  "error": {
    "code": "UPSTREAM_ERROR",
    "message": "UniProt API returned 503 status",
    "recovery_hint": "UniProt service may be temporarily unavailable. Retry after a few seconds."
  }
}
```

## Cross-Reference Key Registry

The UniProt server maps UniProt cross-references to the **22-key Agentic Biolink registry**:

| Key | Database | Example |
|-----|----------|---------|
| `hgnc` | HGNC Gene Nomenclature | HGNC:11998 |
| `ensembl_gene` | Ensembl Gene | ENSG00000141510 |
| `ensembl_transcript` | Ensembl Transcript | ENST00000269305 |
| `refseq` | RefSeq Protein | NP_000537.3 |
| `entrez` | NCBI Gene | 7157 |
| `pdb` | Protein Data Bank | 1TUP, 1TSR |
| `kegg` | KEGG Gene | hsa:7157 |
| `kegg_pathway` | KEGG Pathway | hsa04115 |
| `omim` | OMIM Disease | 191170 |
| `orphanet` | Orphanet Disease | - |
| `chembl` | ChEMBL Compound | - |
| `drugbank` | DrugBank Drug | - |
| `string` | STRING Protein | 9606.ENSP00000269305 |
| `biogrid` | BioGRID Interaction | - |
| `iuphar` | IUPHAR/GtoPdb | - |
| `stitch` | STITCH Interaction | - |
| `pubchem_compound` | PubChem Compound | - |
| `pubchem_substance` | PubChem Substance | - |
| `mondo` | MONDO Disease | - |
| `efo` | EFO Experimental Factor | - |

**Note**: Cross-references use the **omit-if-null pattern** - fields are excluded entirely if no cross-reference exists (never `null` or empty string).

## Performance Characteristics

- **Search latency**: <2s for 95% of common protein queries (SC-001)
- **Accuracy**: 90% correct top result for unambiguous queries (SC-002)
- **Concurrency**: 100 concurrent requests without degradation (SC-003)
- **Rate limiting**: 10 requests/second with exponential backoff
- **Token efficiency**: Slim mode reduces response size by ~80% (~20 vs ~115-300 tokens)

## Testing

```bash
# Run all UniProt tests
uv run pytest tests/integration/test_uniprot_api.py -v -m integration

# Run specific test suites
uv run pytest tests/integration/test_uniprot_api.py::TestUniProtFuzzySearch -v
uv run pytest tests/integration/test_uniprot_api.py::TestUniProtStrictLookup -v
uv run pytest tests/integration/test_cross_database_integration.py -v
uv run pytest tests/integration/test_error_recovery.py -v
uv run pytest tests/integration/test_performance.py -v

# Run error envelope tests
uv run pytest tests/unit/test_error_envelopes.py -v
```

## Architecture References

- **ADR-001 v1.2**: Constitution (6 core principles)
- **ADR-002 v1.0**: Project Skills (Platform Engineering)
- **ADR-003 v1.0**: SpecKit SDLC (Specification-driven development)
- **ADR-004 v1.0**: FastMCP Lifecycle Management (shutdown hook antipattern)

## Next Steps

1. **Integrate with HGNC server**: Combine gene and protein queries
2. **Add more APIs**: ChEMBL, STRING, BioGRID, Open Targets
3. **Implement caching**: Redis caching for frequently accessed proteins
4. **Add monitoring**: Prometheus metrics for query latency and error rates

## Support

- **Documentation**: `docs/adr/accepted/` for architecture decisions
- **Specification**: `specs/002-uniprot-mcp-server/spec.md` for requirements
- **Plan**: `specs/002-uniprot-mcp-server/plan.md` for implementation design
- **Tasks**: `specs/002-uniprot-mcp-server/tasks.md` for task breakdown
- **GitHub**: https://github.com/donbr/lifesciences-research
