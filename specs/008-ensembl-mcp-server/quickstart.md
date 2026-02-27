# Ensembl MCP Server - Quick Start Guide

**Status**: Draft (Planning Complete)
**Version**: 0.1.0
**Target**: Implementation Ready

## Overview

The Ensembl MCP Server provides AI agents with access to genomic data through a **Fuzzy-to-Fact protocol**. Agents can search for genes by name/symbol, then retrieve complete gene and transcript records with cross-references to 22+ biological databases.

## Installation

```bash
# Clone repository
git clone https://github.com/donbr/lifesciences-research.git
cd lifesciences-research

# Install dependencies
uv sync

# Verify installation (after implementation)
uv run pytest tests/integration/test_ensembl_api.py -v -m integration
```

## Running the Server

### Development Mode (with live reload)

```bash
uv run fastmcp dev src/lifesciences_mcp/servers/ensembl.py
```

### Production Mode

```bash
uv run fastmcp run src/lifesciences_mcp/servers/ensembl.py
```

## Available Tools

### 1. search_genes (Fuzzy Search)

**Purpose**: Search for genes by symbol, name, or description.

**Parameters**:
- `query` (required): Search query (minimum 2 characters)
- `species` (optional): Species name or alias (default: "homo_sapiens")
- `page_size` (optional): Results per page, 1-100 (default: 50)
- `cursor` (optional): Pagination cursor from previous response
- `slim` (optional): Return minimal data for token efficiency (default: false)

**Returns**: PaginationEnvelope with GeneSearchCandidate items

**Example 1: Basic Search**
```python
# Search for TP53 gene
{
  "query": "TP53",
  "species": "human"
}

# Response
{
  "items": [
    {
      "id": "ENSG00000141510",
      "symbol": "TP53",
      "name": "tumor protein p53",
      "biotype": "protein_coding",
      "species": "homo_sapiens",
      "score": 1.0
    }
  ],
  "pagination": {
    "cursor": null,
    "total_count": 1,
    "page_size": 50
  }
}
```

**Example 2: Broad Search**
```python
# Search for tumor-related genes
{
  "query": "tumor",
  "page_size": 10
}

# Returns multiple candidates with relevance scores
```

**Example 3: Mouse Genes**
```python
# Search in mouse genome
{
  "query": "Brca1",
  "species": "mus_musculus"
}
```

### 2. get_gene (Strict Lookup)

**Purpose**: Retrieve complete gene record with cross-references by Ensembl Gene ID.

**Parameters**:
- `ensembl_id` (required): Ensembl Gene ID (format: ENSG + 11 digits)
- `slim` (optional): Return minimal data without cross_references (default: false)

**Returns**: EnsemblGene model with full annotations or ErrorEnvelope

**Example 1: Get Complete Gene Record**
```python
# Get human TP53 gene
{
  "ensembl_id": "ENSG00000141510"
}

# Response
{
  "id": "ENSG00000141510",
  "symbol": "TP53",
  "name": "tumor protein p53",
  "biotype": "protein_coding",
  "species": "homo_sapiens",
  "assembly_name": "GRCh38",
  "chromosome": "17",
  "start": 7661779,
  "end": 7687538,
  "strand": -1,
  "transcripts": [
    "ENST00000269305",
    "ENST00000445888",
    "ENST00000420246"
  ],
  "cross_references": {
    "hgnc": "HGNC:11998",
    "uniprot": ["P04637"],
    "entrez": "7157",
    "refseq": ["NM_000546"],
    "omim": "191170",
    "pdb": ["1TUP", "2OCJ", "1TSR"],
    "kegg": "hsa:7157",
    "string": "9606.ENSP00000269305"
  }
}
```

**Example 2: Slim Mode (Token Efficient)**
```python
# Get minimal gene data (~50 tokens vs ~150-350 tokens)
{
  "ensembl_id": "ENSG00000012048",
  "slim": true
}

# Response (cross_references omitted)
{
  "id": "ENSG00000012048",
  "symbol": "BRCA1",
  "name": "BRCA1 DNA repair associated",
  "biotype": "protein_coding",
  "species": "homo_sapiens",
  "assembly_name": "GRCh38",
  "chromosome": "17",
  "start": 43044295,
  "end": 43170245,
  "strand": -1,
  "transcripts": ["ENST00000357654"]
}
```

### 3. get_transcript (Strict Lookup)

**Purpose**: Retrieve transcript details by Ensembl Transcript ID.

**Parameters**:
- `transcript_id` (required): Ensembl Transcript ID (format: ENST + 11 digits)
- `slim` (optional): Return minimal data without cross_references (default: false)

**Returns**: EnsemblTranscript model with annotations or ErrorEnvelope

**Example: Get Transcript**
```python
# Get canonical TP53 transcript
{
  "transcript_id": "ENST00000269305"
}

# Response
{
  "id": "ENST00000269305",
  "display_name": "TP53-201",
  "biotype": "protein_coding",
  "parent_gene": "ENSG00000141510",
  "is_canonical": true,
  "species": "homo_sapiens",
  "assembly_name": "GRCh38",
  "chromosome": "17",
  "start": 7669609,
  "end": 7687538,
  "strand": -1,
  "cross_references": {
    "ensembl_gene": "ENSG00000141510",
    "uniprot": ["P04637"],
    "refseq": ["NM_000546.6", "NP_000537.3"]
  }
}
```

## Common Workflows

### Workflow 1: Fuzzy-to-Fact (Recommended)

**Use Case**: Agent doesn't know the exact Ensembl ID but knows gene name or symbol.

```python
# Step 1: Fuzzy search by name
search_result = search_genes({"query": "BRCA1", "species": "human"})

# Step 2: Extract top candidate Ensembl ID
top_result = search_result["items"][0]
gene_id = top_result["id"]  # "ENSG00000012048"

# Step 3: Get complete gene record
gene = get_gene({"ensembl_id": gene_id})

# Step 4: Access cross-references
hgnc_id = gene["cross_references"]["hgnc"]  # "HGNC:1100"
uniprot_ids = gene["cross_references"]["uniprot"]  # ["P38398"]

# Step 5: Optionally get transcript details
canonical_transcript = gene["transcripts"][0]  # "ENST00000357654"
transcript = get_transcript({"transcript_id": canonical_transcript})
```

### Workflow 2: Cross-Database Navigation

**Use Case**: Agent needs to connect Ensembl gene data to other databases.

```python
# Get gene with all cross-references
gene = get_gene({"ensembl_id": "ENSG00000141510"})

# Navigate to other databases
hgnc_id = gene["cross_references"]["hgnc"]        # "HGNC:11998" -> HGNC server
uniprot = gene["cross_references"]["uniprot"]     # ["P04637"] -> UniProt server
pdb_ids = gene["cross_references"]["pdb"]         # ["1TUP", ...] -> Structural data
omim_id = gene["cross_references"]["omim"]        # "191170" -> Disease associations
kegg_id = gene["cross_references"]["kegg"]        # "hsa:7157" -> Pathway data
string_id = gene["cross_references"]["string"]    # "9606.ENSP00000269305" -> PPI

# Use these IDs with other MCP servers
```

### Workflow 3: Gene-to-Transcript Exploration

**Use Case**: Agent needs to explore transcript variants for a gene.

```python
# Step 1: Get gene with transcript list
gene = get_gene({"ensembl_id": "ENSG00000141510"})
transcripts = gene["transcripts"]  # ["ENST00000269305", "ENST00000445888", ...]

# Step 2: Get details for each transcript
for transcript_id in transcripts:
    transcript = get_transcript({"transcript_id": transcript_id})
    print(f"{transcript['display_name']}: {transcript['biotype']}")
    # TP53-201: protein_coding
    # TP53-202: protein_coding
```

### Workflow 4: Error Recovery (Autonomous Correction)

**Use Case**: Agent makes a mistake and receives actionable recovery hints.

```python
# Step 1: Agent tries invalid ID (gene symbol instead of Ensembl ID)
result = get_gene({"ensembl_id": "TP53"})

# Step 2: Error response with recovery hint
{
  "success": false,
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "Invalid Ensembl Gene ID format",
    "recovery_hint": "Use search_genes to find valid Ensembl Gene IDs. Format: ENSG + 11 digits (e.g., ENSG00000141510)",
    "invalid_input": "TP53"
  }
}

# Step 3: Agent follows hint
search_result = search_genes({"query": "TP53", "species": "human"})
valid_id = search_result["items"][0]["id"]  # "ENSG00000141510"

# Step 4: Retry with valid ID
gene = get_gene({"ensembl_id": valid_id})  # Success!
```

## Error Codes and Recovery

### AMBIGUOUS_QUERY
**Cause**: Query too short (< 2 characters) or too broad
**Recovery**: Provide more specific query terms

```python
# Error
search_genes({"query": "p"})

# Recovery
search_genes({"query": "p53"})
```

### UNRESOLVED_ENTITY
**Cause**: Invalid Ensembl ID format (not ENSG/ENST + 11 digits)
**Recovery**: Use search_genes to find valid ID

```python
# Error
get_gene({"ensembl_id": "BRCA1"})

# Recovery
search_result = search_genes({"query": "BRCA1"})
get_gene({"ensembl_id": search_result["items"][0]["id"]})
```

### ENTITY_NOT_FOUND
**Cause**: Valid ID format but gene/transcript doesn't exist
**Recovery**: Verify ID or search by name

```python
# Error
get_gene({"ensembl_id": "ENSG99999999999"})

# Recovery
search_genes({"query": "gene name"})
```

### RATE_LIMITED
**Cause**: Exceeded 15 requests/second limit
**Recovery**: Wait and retry (handled automatically with exponential backoff)

```python
# Error (after rapid requests)
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Exceeded rate limit",
    "recovery_hint": "Wait 5 seconds before retrying"
  }
}

# Recovery: Client handles automatically
```

### UPSTREAM_ERROR
**Cause**: Ensembl API unavailable or returned error
**Recovery**: Retry after brief delay

```python
# Error
{
  "error": {
    "code": "UPSTREAM_ERROR",
    "message": "Ensembl API returned 503 status",
    "recovery_hint": "Ensembl service may be temporarily unavailable. Retry after a few seconds."
  }
}
```

## Cross-Reference Key Registry

The Ensembl server maps cross-references to the **22-key Agentic Biolink registry**:

| Key | Database | Example |
|-----|----------|---------|
| `hgnc` | HGNC Gene Nomenclature | HGNC:11998 |
| `ensembl_gene` | Ensembl Gene | ENSG00000141510 |
| `ensembl_transcript` | Ensembl Transcript | ENST00000269305 |
| `uniprot` | UniProt Protein | P04637 |
| `entrez` | NCBI Gene | 7157 |
| `refseq` | RefSeq | NM_000546 |
| `omim` | OMIM Disease | 191170 |
| `pdb` | Protein Data Bank | 1TUP |
| `kegg` | KEGG Gene | hsa:7157 |
| `chembl` | ChEMBL Target | CHEMBL3927 |
| `string` | STRING Protein | 9606.ENSP00000269305 |
| `biogrid` | BioGRID | 113010 |

**Note**: Cross-references use the **omit-if-null pattern** - fields are excluded entirely if no cross-reference exists (never `null` or empty string).

## Performance Characteristics

- **Search latency**: <2s for 95% of common gene queries (SC-001)
- **Rate limiting**: 15 requests/second with exponential backoff
- **Supported species**: All Ensembl species (default: human)
- **Token efficiency**: Slim mode reduces response size by ~70% (~50 vs ~150-350 tokens)

## Testing

```bash
# Run all Ensembl tests
uv run pytest tests/integration/test_ensembl_api.py -v -m integration

# Run specific test suites
uv run pytest tests/integration/test_ensembl_api.py::TestEnsemblFuzzySearch -v
uv run pytest tests/integration/test_ensembl_api.py::TestEnsemblStrictLookup -v
uv run pytest tests/integration/test_ensembl_api.py::TestEnsemblTranscript -v
uv run pytest tests/integration/test_ensembl_api.py::TestErrorRecovery -v

# Run unit tests
uv run pytest tests/unit/test_ensembl_models.py tests/unit/test_ensembl_client.py -v
```

## Architecture References

- **ADR-001 v1.2**: Agentic-First Architecture (Fuzzy-to-Fact, cross-references)
- **ADR-002 v1.0**: Project Skills (Platform Engineering)
- **ADR-003 v1.0**: SpecKit SDLC (Specification-driven development)
- **ADR-004 v1.0**: FastMCP Lifecycle Management
- **Constitution v1.1**: Rate limiting (15 req/s), async-first

## Integration with Other Servers

The Ensembl server integrates with other Life Sciences MCP servers through cross-references:

```
Ensembl (genes) <---> HGNC (nomenclature)
    |                      |
    v                      v
UniProt (proteins) <---> STRING (interactions)
    |
    v
ChEMBL (compounds) <---> Open Targets (diseases)
```

**Example Multi-Server Workflow**:
1. Search for gene in Ensembl: `search_genes("TP53")`
2. Get HGNC ID from cross-references: `HGNC:11998`
3. Look up gene details in HGNC: `get_gene("HGNC:11998")`
4. Get UniProt ID from cross-references: `P04637`
5. Look up protein in UniProt: `get_protein("UniProtKB:P04637")`
6. Get STRING interactions: `get_interactions("STRING:9606.ENSP00000269305")`

## Support

- **Specification**: `specs/008-ensembl-mcp-server/spec.md`
- **Plan**: `specs/008-ensembl-mcp-server/plan.md`
- **Research**: `specs/008-ensembl-mcp-server/research.md`
- **Data Model**: `specs/008-ensembl-mcp-server/data-model.md`
- **Contracts**: `specs/008-ensembl-mcp-server/contracts/`
- **ADR-001**: `docs/adr/accepted/adr-001-v1.2.md`
- **Ensembl REST API**: https://rest.ensembl.org
