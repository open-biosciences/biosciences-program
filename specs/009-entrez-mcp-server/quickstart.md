# NCBI Entrez MCP Server - Quick Start Guide

**Status**: Specification Complete
**Version**: 0.1.0 (Draft)
**Tier**: 4 (Genomics & Identifiers)

## Overview

The NCBI Entrez MCP Server provides AI agents with access to NCBI Gene database through a **Fuzzy-to-Fact protocol**. Agents can search for genes by symbol/name/description, retrieve complete gene records with cross-references to 22+ biological databases, and discover associated PubMed literature for evidence gathering.

## Installation

```bash
# Clone repository
git clone https://github.com/donbr/lifesciences-research.git
cd lifesciences-research

# Install dependencies (includes defusedxml for secure XML parsing)
uv sync

# Optional: Set NCBI API key for higher rate limits (10 req/s vs 3 req/s)
export NCBI_API_KEY=your_api_key_here

# Verify installation
uv run pytest tests/integration/test_entrez_api.py -v -m integration
```

## Running the Server

### Development Mode (with live reload)

```bash
uv run fastmcp dev src/lifesciences_mcp/servers/entrez.py
```

### Production Mode

```bash
uv run fastmcp run src/lifesciences_mcp/servers/entrez.py
```

## Available Tools

### 1. search_genes (Fuzzy Search)

**Purpose**: Search for genes by symbol, name, description, or natural language query.

**Parameters**:
- `query` (required): Search term (minimum 2 characters)
- `organism` (optional): Filter by organism name (e.g., "human", "mouse")
- `page_size` (optional): Results per page, 1-100 (default: 50)
- `cursor` (optional): Pagination cursor from previous response

**Returns**: PaginationEnvelope with GeneSearchCandidate items

**Example 1: Basic Search**
```python
# Search for BRCA1 gene
{
  "query": "BRCA1",
  "organism": "human",
  "page_size": 10
}

# Response
{
  "items": [
    {
      "id": "NCBIGene:672",
      "symbol": "BRCA1",
      "name": "BRCA1 DNA repair associated",
      "description": "breast cancer 1, early onset",
      "organism": "Homo sapiens",
      "score": 1.0
    },
    ...
  ],
  "pagination": {
    "cursor": null,
    "total_count": 1,
    "page_size": 10
  }
}
```

**Example 2: Broad Search with Pagination**
```python
# Search for kinases
{
  "query": "kinase",
  "organism": "human",
  "page_size": 50
}

# Response with pagination cursor
{
  "items": [...],  # 50 kinase genes
  "pagination": {
    "cursor": "eyJvZmZzZXQiOiA1MH0=",
    "total_count": 1250,
    "page_size": 50
  }
}

# Get next page
{
  "query": "kinase",
  "organism": "human",
  "cursor": "eyJvZmZzZXQiOiA1MH0=",
  "page_size": 50
}
```

### 2. get_gene (Strict Lookup)

**Purpose**: Retrieve complete gene record with cross-references by NCBIGene CURIE.

**Parameters**:
- `entrez_id` (required): NCBIGene CURIE (format: `NCBIGene:NNNNN`)
- `slim` (optional): Return minimal data for token efficiency (default: false)

**Returns**: EntrezGene model with full annotations or ErrorEnvelope

**Example 1: Get Complete Gene Record**
```python
# Get TP53 gene (human tumor suppressor)
{
  "entrez_id": "NCBIGene:7157"
}

# Response
{
  "id": "NCBIGene:7157",
  "symbol": "TP53",
  "name": "tumor protein p53",
  "description": "cellular tumor antigen p53; phosphoprotein p53; antigen NY-CO-13",
  "summary": "This gene encodes a tumor suppressor protein containing transcriptional activation, DNA binding, and oligomerization domains...",
  "map_location": "17p13.1",
  "chromosome": "17",
  "aliases": ["P53", "TRP53", "LFS1", "BCC7"],
  "organism": "Homo sapiens",
  "taxon_id": 9606,
  "cross_references": {
    "hgnc": "HGNC:11998",
    "ensembl_gene": "ENSG00000141510",
    "uniprot": "UniProtKB:P04637",
    "refseq": "NM_000546.6",
    "omim": "191170"
  }
}
```

**Example 2: Slim Mode (Token Efficient)**
```python
# Get minimal gene data (~25 tokens vs ~115-300 tokens)
{
  "entrez_id": "NCBIGene:672",
  "slim": true
}

# Response (summary, description, cross_references omitted)
{
  "id": "NCBIGene:672",
  "symbol": "BRCA1",
  "name": "BRCA1 DNA repair associated",
  "organism": "Homo sapiens"
}
```

### 3. get_pubmed_links (Literature Discovery)

**Purpose**: Get PubMed IDs associated with a gene for evidence gathering.

**Parameters**:
- `entrez_id` (required): NCBIGene CURIE
- `limit` (optional): Maximum PubMed IDs to return, 1-100 (default: 10)

**Returns**: List of PubMed IDs as strings

**Example 1: Get Literature for TP53**
```python
# Get top 10 PubMed citations for TP53
{
  "entrez_id": "NCBIGene:7157",
  "limit": 10
}

# Response
["39804234", "39804163", "39803982", "39803897", "39803721",
 "39803654", "39803512", "39803498", "39803421", "39803387"]
```

**Example 2: Limited Literature Query**
```python
# Get just 5 citations
{
  "entrez_id": "NCBIGene:672",
  "limit": 5
}

# Response
["39801234", "39801098", "39800987", "39800876", "39800765"]
```

## Common Workflows

### Workflow 1: Fuzzy-to-Fact (Recommended)

**Use Case**: Agent doesn't know the exact NCBI Gene ID but knows gene symbol or name.

```python
# Step 1: Fuzzy search by symbol
search_result = search_genes({"query": "BRCA1", "organism": "human", "page_size": 5})

# Step 2: Extract top candidate CURIE
top_result = search_result["items"][0]
gene_id = top_result["id"]  # "NCBIGene:672"

# Step 3: Get complete gene record
gene = get_gene({"entrez_id": gene_id})

# Step 4: Access cross-references for multi-hop queries
hgnc_id = gene["cross_references"]["hgnc"]        # "HGNC:1100"
ensembl = gene["cross_references"]["ensembl_gene"] # "ENSG00000012048"
uniprot = gene["cross_references"]["uniprot"]      # "UniProtKB:P38398"
```

### Workflow 2: Cross-Database Navigation

**Use Case**: Agent needs to connect gene data to other databases.

```python
# Get gene with all cross-references
gene = get_gene({"entrez_id": "NCBIGene:7157"})

# Navigate to other databases
hgnc_id = gene["cross_references"]["hgnc"]          # "HGNC:11998" -> HGNC MCP
ensembl = gene["cross_references"]["ensembl_gene"]  # "ENSG00000141510" -> Ensembl
uniprot = gene["cross_references"]["uniprot"]       # "UniProtKB:P04637" -> UniProt MCP
omim_id = gene["cross_references"]["omim"]          # "191170" -> Disease info
refseq = gene["cross_references"]["refseq"]         # "NM_000546.6" -> Sequence data

# Use these IDs with other MCP servers
hgnc_gene = call_hgnc_server("get_gene", {"hgnc_id": hgnc_id})
protein = call_uniprot_server("get_protein", {"uniprot_id": uniprot})
```

### Workflow 3: Evidence Gathering with Literature

**Use Case**: Agent needs scientific evidence for gene function claims.

```python
# Step 1: Resolve gene
search_result = search_genes({"query": "TP53 tumor suppressor", "organism": "human"})
gene_id = search_result["items"][0]["id"]

# Step 2: Get gene summary
gene = get_gene({"entrez_id": gene_id})
print(gene["summary"])  # "This gene encodes a tumor suppressor protein..."

# Step 3: Get supporting literature
pubmed_ids = get_pubmed_links({"entrez_id": gene_id, "limit": 5})
# ["39804234", "39804163", "39803982", "39803897", "39803721"]

# Step 4: Use PubMed MCP server to get article details
for pmid in pubmed_ids:
    article = call_pubmed_server("get_article", {"pmid": pmid})
    print(f"{article['title']} ({article['year']})")
```

### Workflow 4: Error Recovery (Autonomous Correction)

**Use Case**: Agent makes a mistake and receives actionable recovery hints.

```python
# Step 1: Agent tries invalid CURIE (missing prefix)
result = get_gene({"entrez_id": "7157"})

# Step 2: Error response with recovery hint
{
  "success": false,
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "The input '7157' is not a valid NCBIGene CURIE.",
    "recovery_hint": "Call search_genes to resolve the identifier first. Expected format: NCBIGene:<numeric_id>",
    "invalid_input": "7157"
  }
}

# Step 3: Agent follows hint
search_result = search_genes({"query": "7157", "page_size": 5})
# or
search_result = search_genes({"query": "TP53", "page_size": 5})
valid_curie = search_result["items"][0]["id"]  # "NCBIGene:7157"

# Step 4: Retry with valid CURIE
gene = get_gene({"entrez_id": valid_curie})  # Success!
```

## Error Codes and Recovery

### AMBIGUOUS_QUERY
**Cause**: Query too short (< 2 characters)
**Recovery**: Provide more specific query terms or add organism filter

```python
# Error
search_genes({"query": "A"})

# Recovery
search_genes({"query": "AKT1", "organism": "human"})
```

### UNRESOLVED_ENTITY
**Cause**: Invalid CURIE format (missing "NCBIGene:" prefix)
**Recovery**: Use search_genes to find valid CURIE

```python
# Error
get_gene({"entrez_id": "7157"})

# Recovery
search_result = search_genes({"query": "7157"})
get_gene({"entrez_id": search_result["items"][0]["id"]})
```

### ENTITY_NOT_FOUND
**Cause**: Valid CURIE format but gene doesn't exist
**Recovery**: Verify ID or search by gene symbol

```python
# Error
get_gene({"entrez_id": "NCBIGene:999999999999"})

# Recovery
search_genes({"query": "gene symbol or name"})
```

### RATE_LIMITED
**Cause**: Exceeded NCBI rate limit (3 req/s without key, 10 req/s with key)
**Recovery**: Wait and retry; add NCBI_API_KEY for higher limits

```python
# Error (after many rapid requests)
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Exceeded NCBI rate limit",
    "recovery_hint": "Wait and retry. Add NCBI_API_KEY env var for 10 req/s limit"
  }
}

# Recovery: Client handles automatically with exponential backoff
# Or: Set NCBI_API_KEY environment variable
```

### UPSTREAM_ERROR
**Cause**: NCBI E-utilities unavailable or returned error
**Recovery**: Retry request after brief delay

```python
# Error
{
  "error": {
    "code": "UPSTREAM_ERROR",
    "message": "NCBI E-utilities API returned 503 status",
    "recovery_hint": "NCBI service may be temporarily unavailable. Retry after a few seconds."
  }
}
```

## Cross-Reference Key Registry

The Entrez server extracts cross-references from NCBI Gene XML to the **22-key Agentic Biolink registry**:

| Key | Database | Example | NCBI Source |
|-----|----------|---------|-------------|
| `hgnc` | HGNC Gene Nomenclature | HGNC:11998 | Dbtag_db="HGNC" |
| `ensembl_gene` | Ensembl Gene | ENSG00000141510 | Dbtag_db="Ensembl" |
| `uniprot` | UniProt Protein | UniProtKB:P04637 | Dbtag_db="UniProtKB/Swiss-Prot" |
| `refseq` | RefSeq Transcript/Protein | NM_000546.6 | Dbtag_db="RefSeq" |
| `omim` | OMIM Disease/Phenotype | 191170 | Dbtag_db="MIM" |

**Note**: Cross-references use the **omit-if-null pattern** - fields are excluded entirely if no cross-reference exists (never `null` or empty string).

## Rate Limiting

| Condition | Rate Limit | Delay Between Requests |
|-----------|------------|------------------------|
| Without NCBI_API_KEY | 3 req/s | 333ms |
| With NCBI_API_KEY | 10 req/s | 100ms |

**How to get an API key** (free):
1. Create account: https://www.ncbi.nlm.nih.gov/account/
2. Go to Settings: https://www.ncbi.nlm.nih.gov/account/settings/
3. Generate API key
4. Set environment variable: `export NCBI_API_KEY=your_key`

## Performance Characteristics

- **Search latency**: <2s for 95% of queries (SC-001)
- **Rate limiting**: 3 req/s default, 10 req/s with API key
- **Pagination**: Cursor-based with retstart/retmax offsets
- **Token efficiency**: Slim mode reduces response size by ~80% (~25 vs ~115-300 tokens)

## Testing

```bash
# Run all Entrez tests
uv run pytest tests/integration/test_entrez_api.py -v -m integration

# Run specific test suites
uv run pytest tests/integration/test_entrez_api.py::TestEntrezFuzzySearch -v
uv run pytest tests/integration/test_entrez_api.py::TestEntrezStrictLookup -v
uv run pytest tests/integration/test_entrez_api.py::TestEntrezPubMedLinks -v
uv run pytest tests/integration/test_entrez_api.py::TestEntrezErrorRecovery -v

# Run performance tests
uv run pytest tests/integration/test_entrez_performance.py -v
```

## Architecture References

- **ADR-001 v1.2**: Constitution (6 core principles)
- **ADR-002 v1.0**: Project Skills (Platform Engineering)
- **ADR-003 v1.0**: SpecKit SDLC (Specification-driven development)
- **ADR-004 v1.0**: FastMCP Lifecycle Management

## Related MCP Servers

| Server | Use With | Cross-Reference Key |
|--------|----------|---------------------|
| HGNC | Gene nomenclature | `hgnc` |
| UniProt | Protein sequences | `uniprot` |
| Ensembl | Genomic features | `ensembl_gene` |
| Open Targets | Disease associations | via Ensembl ID |
| PubMed | Literature details | from `get_pubmed_links` |

## Support

- **Specification**: `specs/009-entrez-mcp-server/spec.md`
- **Research**: `specs/009-entrez-mcp-server/research.md`
- **Data Model**: `specs/009-entrez-mcp-server/data-model.md`
- **Contracts**: `specs/009-entrez-mcp-server/contracts/`
- **GitHub**: https://github.com/donbr/lifesciences-research
