# IUPHAR/GtoPdb MCP Server Quickstart

A FastMCP server providing access to the IUPHAR/BPS Guide to PHARMACOLOGY database for drug discovery and pharmacology research.

## Prerequisites

- Python 3.11+
- uv package manager

## Installation

```bash
# Clone and install
cd lifesciences-research
uv sync
```

## Running the Server

```bash
uv run fastmcp run src/lifesciences_mcp/servers/iuphar.py
```

For development with auto-reload:
```bash
uv run fastmcp dev src/lifesciences_mcp/servers/iuphar.py
```

## Available Tools

The server provides 4 tools following the Fuzzy-to-Fact protocol:

| Tool | Type | Description |
|------|------|-------------|
| `search_ligands` | Fuzzy | Search for drugs, chemicals, peptides, antibodies |
| `get_ligand` | Strict | Get complete ligand record by IUPHAR CURIE |
| `search_targets` | Fuzzy | Search for receptors, enzymes, ion channels, transporters |
| `get_target` | Strict | Get complete target record by IUPHAR CURIE |

---

## Workflow 1: Finding a Drug (Ligand)

### Step 1: Fuzzy Search

```python
# Search for NSAIDs
result = await search_ligands(query="ibuprofen", page_size=10)

# Response:
{
  "items": [
    {
      "id": "IUPHAR:2713",
      "name": "ibuprofen",
      "type": "Synthetic organic",
      "approved": true,
      "score": 0.95
    },
    {
      "id": "IUPHAR:7089",
      "name": "ibuprofen lysine",
      "type": "Synthetic organic",
      "approved": true,
      "score": 0.90
    }
  ],
  "pagination": {
    "cursor": null,
    "total_count": 2,
    "page_size": 10
  }
}
```

### Step 2: Strict Lookup

```python
# Get complete ibuprofen record using CURIE from search
ligand = await get_ligand(iuphar_id="IUPHAR:2713")

# Response:
{
  "id": "IUPHAR:2713",
  "ligand_id": 2713,
  "name": "ibuprofen",
  "approved_name": "ibuprofen",
  "type": "Synthetic organic",
  "approved": true,
  "approval_source": "FDA (1974), EMA (2004)",
  "who_essential": true,
  "synonyms": ["Advil", "Brufen", "Motrin", "Nurofen"],
  "cross_references": {
    "chembl": "CHEMBL521",
    "drugbank": "DB01050",
    "pubchem_compound": "3672"
  }
}
```

---

## Workflow 2: Finding a Receptor (Target)

### Step 1: Fuzzy Search

```python
# Search for dopamine receptors
result = await search_targets(query="dopamine receptor", page_size=10)

# Response:
{
  "items": [
    {
      "id": "IUPHAR:214",
      "name": "D1 receptor",
      "family": "GPCR",
      "type": "GPCR",
      "score": 0.95
    },
    {
      "id": "IUPHAR:215",
      "name": "D2 receptor",
      "family": "GPCR",
      "type": "GPCR",
      "score": 0.90
    },
    {
      "id": "IUPHAR:216",
      "name": "D3 receptor",
      "family": "GPCR",
      "type": "GPCR",
      "score": 0.85
    }
  ],
  "pagination": {
    "cursor": null,
    "total_count": 7,
    "page_size": 10
  }
}
```

### Step 2: Strict Lookup

```python
# Get complete D2 receptor record
target = await get_target(iuphar_id="IUPHAR:215")

# Response:
{
  "id": "IUPHAR:215",
  "target_id": 215,
  "name": "D2 receptor",
  "target_family": "GPCR",
  "family_ids": [20],
  "species": "Homo sapiens",
  "gene_symbol": "DRD2",
  "cross_references": {
    "uniprot": ["P14416"],
    "ensembl_gene": "ENSG00000149295",
    "entrez": "1813",
    "refseq": ["NM_000795"],
    "chembl": "CHEMBL217",
    "omim": "126450"
  }
}
```

---

## Workflow 3: Search by Gene Symbol

```python
# Find target by gene symbol
result = await search_targets(query="DRD2", page_size=5)

# Returns D2 receptor as top result
top_candidate = result["items"][0]
assert top_candidate["name"] == "D2 receptor"
```

---

## Workflow 4: Filtering Approved Drugs Only

```python
# Search for approved COX inhibitors
result = await search_ligands(
    query="COX inhibitor",
    approved_only=True,
    page_size=20
)

# All results have approved=True
for item in result["items"]:
    assert item["approved"] == True
```

---

## Workflow 5: Filter by Target Type

```python
# Find only GPCRs
result = await search_targets(
    query="serotonin",
    type_filter="GPCR",
    page_size=20
)

# All results are GPCRs
for item in result["items"]:
    assert item["type"] == "GPCR"
```

---

## Error Handling

### UNRESOLVED_ENTITY Error

Passing a raw string to a strict tool:

```python
# Wrong: raw name instead of CURIE
ligand = await get_ligand(iuphar_id="ibuprofen")

# Error response:
{
  "success": false,
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "Invalid IUPHAR CURIE format: 'ibuprofen'",
    "recovery_hint": "Use search_ligands to find valid IUPHAR IDs, then call get_ligand with the resolved CURIE (e.g., 'IUPHAR:2713')",
    "invalid_input": "ibuprofen"
  }
}

# Recovery: Use fuzzy search first
search_result = await search_ligands(query="ibuprofen")
curie = search_result["items"][0]["id"]  # "IUPHAR:2713"
ligand = await get_ligand(iuphar_id=curie)  # Success!
```

### ENTITY_NOT_FOUND Error

Valid CURIE format but ID doesn't exist:

```python
# Valid format but non-existent ID
ligand = await get_ligand(iuphar_id="IUPHAR:999999")

# Error response:
{
  "success": false,
  "error": {
    "code": "ENTITY_NOT_FOUND",
    "message": "Ligand 'IUPHAR:999999' not found in GtoPdb",
    "recovery_hint": "Verify the IUPHAR ID or use search_ligands to find valid identifiers. Note: This may be a target ID - try get_target instead.",
    "invalid_input": "IUPHAR:999999"
  }
}
```

### RATE_LIMITED Error

Too many requests:

```python
{
  "success": false,
  "error": {
    "code": "RATE_LIMITED",
    "message": "GtoPdb API rate limit exceeded",
    "recovery_hint": "Retry after 2 seconds. Consider using batch operations or reducing request frequency."
  }
}
```

---

## Cross-Database Integration

### Ligand Cross-References

After getting a ligand, you can follow cross-references to other databases:

```python
ligand = await get_ligand(iuphar_id="IUPHAR:2713")

# Available cross-references:
cross_refs = ligand["cross_references"]

# ChEMBL compound lookup
chembl_id = cross_refs.get("chembl")  # "CHEMBL521"
# -> Use ChEMBL MCP server: get_compound(chembl_id="CHEMBL:521")

# DrugBank lookup
drugbank_id = cross_refs.get("drugbank")  # "DB01050"
# -> Use DrugBank MCP server: get_drug(drugbank_id="DrugBank:DB01050")

# PubChem lookup
pubchem_cid = cross_refs.get("pubchem_compound")  # "3672"
# -> Use PubChem API or web interface
```

### Target Cross-References

After getting a target, you can follow cross-references:

```python
target = await get_target(iuphar_id="IUPHAR:215")

# Available cross-references:
cross_refs = target["cross_references"]

# UniProt protein lookup
uniprot_ids = cross_refs.get("uniprot")  # ["P14416"]
# -> Use UniProt MCP server: get_protein(uniprot_id="UniProtKB:P14416")

# Ensembl gene lookup
ensembl_id = cross_refs.get("ensembl_gene")  # "ENSG00000149295"

# HGNC gene lookup (via Entrez)
entrez_id = cross_refs.get("entrez")  # "1813"
# -> Use HGNC MCP server: search_genes(query="DRD2")
```

---

## Pagination

For large result sets, use cursor-based pagination:

```python
# First page
result = await search_ligands(query="kinase inhibitor", page_size=50)
process_items(result["items"])

# Get next page if available
cursor = result["pagination"]["cursor"]
if cursor:
    result = await search_ligands(
        query="kinase inhibitor",
        page_size=50,
        cursor=cursor
    )
    process_items(result["items"])
```

---

## Rate Limiting

The server implements conservative rate limiting (1 req/s) to respect GtoPdb as a community resource:

- Requests are automatically throttled
- Exponential backoff on 429/503 errors
- Recommended: batch similar queries together
- Avoid rapid sequential requests

---

## Testing

Run integration tests:

```bash
# All IUPHAR tests
uv run pytest tests/integration/test_iuphar_api.py -v -m integration

# Specific test
uv run pytest tests/integration/test_iuphar_api.py::test_search_ligands -v -m integration
```

Run unit tests (no network required):

```bash
uv run pytest tests/unit/test_pharmacology_models.py -v
```

---

## Resources

- [GtoPdb Web Services Documentation](https://www.guidetopharmacology.org/webServices.jsp)
- [Guide to PHARMACOLOGY](https://www.guidetopharmacology.org/)
- [ADR-001: Agentic-First Architecture](../../docs/adr/accepted/adr-001-v1.2.md)
