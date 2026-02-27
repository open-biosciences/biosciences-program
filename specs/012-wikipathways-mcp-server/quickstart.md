# Quickstart Guide: WikiPathways MCP Server

**Audience**: Life sciences researchers, bioinformaticians, AI agent developers
**Prerequisites**: Basic understanding of biological pathways, Python 3.11+, FastMCP CLI
**Time to Complete**: 10-15 minutes

---

## Installation

```bash
# Clone the repository
git clone https://github.com/your-org/lifesciences-research.git
cd lifesciences-research

# Install dependencies with uv
uv sync

# Verify installation
uv run fastmcp run src/lifesciences_mcp/servers/wikipathways.py
```

**Expected Output**:
```
FastMCP server running at stdio://
4 tools registered: search_pathways, get_pathway, get_pathways_for_gene, get_pathway_components
```

---

## Core Workflows

### Workflow 1: Discover Pathways by Topic (Fuzzy Search)

**Goal**: Find pathways related to "glycolysis" in humans

```python
# Step 1: Search for pathways
result = await client.call_tool("search_pathways", {
    "query": "glycolysis",
    "organism": "Homo sapiens",
    "page_size": 10
})

# Expected response:
{
  "items": [
    {
      "id": "WP:WP534",
      "title": "Glycolysis and gluconeogenesis",
      "organism": "Homo sapiens",
      "description": "Glycolysis is the metabolic pathway...",
      "score": 0.95
    }
  ],
  "pagination": {
    "cursor": null,
    "total_count": 3,
    "page_size": 10
  }
}
```

**Key Points**:
- Returns ranked PathwaySearchCandidate results
- Use `organism` parameter for species-specific searches
- Pagination cursor is `null` when no more results
- Score ranges from 0.0-1.0 (higher is better)

---

### Workflow 2: Get Complete Pathway Details (Strict Lookup)

**Goal**: Retrieve full pathway information including cross-references

```python
# Step 2: Get pathway details using ID from search
pathway = await client.call_tool("get_pathway", {
    "pathway_id": "WP:WP534"
})

# Expected response:
{
  "id": "WP:WP534",
  "title": "Glycolysis and gluconeogenesis",
  "organism": "Homo sapiens",
  "description": "Glycolysis is the metabolic pathway that converts glucose...",
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
    "entrez": ["5213", "5214", "2203"],
    "kegg_pathway": "hsa00010",
    "reactome": "R-HSA-70171",
    "gene_ontology": "GO:0006096"
  },
  "url": "https://classic.wikipathways.org/index.php/Pathway:WP534"
}
```

**Key Points**:
- Requires exact pathway ID format: `WP:WP534` (not `WP534`)
- Returns complete metadata including curators and revision info
- Cross-references enable lookup in other databases (KEGG, Reactome, GO)
- Component counts summarize pathway composition

---

### Workflow 3: Reverse Gene-to-Pathway Lookup

**Goal**: Find all pathways containing the BRCA1 gene

```python
# Step 3: Find pathways by gene
pathways = await client.call_tool("get_pathways_for_gene", {
    "gene_id": "BRCA1",
    "organism": "Homo sapiens"
})

# Expected response:
{
  "items": [
    {
      "id": "WP:WP4868",
      "title": "RAC1/PAK1/p38/MMP2 pathway",
      "organism": "Homo sapiens",
      "description": "This pathway is based on figure 4 from Cheung et al...",
      "score": 0.95
    },
    {
      "id": "WP:WP4239",
      "title": "EMT in colorectal cancer",
      "organism": "Homo sapiens",
      "description": "Epithelial-to-mesenchymal transition...",
      "score": 0.90
    }
  ],
  "pagination": {
    "cursor": "eyJvZmZzZXQiOiA1MH0=",
    "total_count": 127,
    "page_size": 50
  }
}
```

**Key Points**:
- Accepts gene symbols (BRCA1), Entrez IDs (672), or Ensembl IDs (ENSG00000012048)
- Gene symbols are normalized to uppercase automatically
- Returns all pathways containing the gene across specified organism
- Use pagination cursor for large result sets (>50 pathways)

---

### Workflow 4: Extract Pathway Components (Genes, Proteins, Metabolites)

**Goal**: Get all molecular entities participating in glycolysis pathway

```python
# Step 4: Extract pathway components
components = await client.call_tool("get_pathway_components", {
    "pathway_id": "WP:WP534"
})

# Expected response:
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

**Key Points**:
- Returns structured lists: genes, proteins, metabolites, interactions
- Each component has cross-references for lookup in other databases
- Empty component lists are OMITTED (not returned as empty arrays)
- Interactions list currently empty (future enhancement)

---

## Common Error Recovery Scenarios

### Scenario 1: Invalid Pathway ID Format

**Error**:
```python
result = await client.call_tool("get_pathway", {"pathway_id": "glycolysis"})
```

**Response**:
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

**Recovery**:
```python
# Step 1: Search to resolve identifier
search_result = await client.call_tool("search_pathways", {"query": "glycolysis"})
pathway_id = search_result["items"][0]["id"]  # "WP:WP534"

# Step 2: Use resolved ID for strict lookup
pathway = await client.call_tool("get_pathway", {"pathway_id": pathway_id})
```

---

### Scenario 2: Pathway Not Found

**Error**:
```python
result = await client.call_tool("get_pathway", {"pathway_id": "WP:WP99999"})
```

**Response**:
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

**Recovery**:
```python
# Verify pathway ID by searching
search_result = await client.call_tool("search_pathways", {"query": "your search term"})
# Use one of the returned pathway IDs
```

---

### Scenario 3: Query Too Short (Ambiguous)

**Error**:
```python
result = await client.call_tool("search_pathways", {"query": "a"})
```

**Response**:
```json
{
  "success": false,
  "error": {
    "code": "AMBIGUOUS_QUERY",
    "message": "Query 'a' is too short (minimum 2 characters required)",
    "recovery_hint": "Use a more specific search term with at least 2 characters",
    "invalid_input": "a"
  }
}
```

**Recovery**:
```python
# Use more specific query
result = await client.call_tool("search_pathways", {"query": "apoptosis"})
```

---

### Scenario 4: Empty Search Results

**Query**:
```python
result = await client.call_tool("search_pathways", {
    "query": "nonexistent pathway XYZ123"
})
```

**Response**:
```json
{
  "items": [],
  "pagination": {
    "cursor": null,
    "total_count": 0,
    "page_size": 50
  }
}
```

**Note**: Empty results are NOT errors. They indicate no matching pathways found.

---

## End-to-End Example: Research Workflow

**Research Question**: "What pathways involve BRCA1 in humans, and what other genes participate?"

```python
# Step 1: Find pathways containing BRCA1
brca1_pathways = await client.call_tool("get_pathways_for_gene", {
    "gene_id": "BRCA1",
    "organism": "Homo sapiens",
    "page_size": 10
})

print(f"Found {brca1_pathways['pagination']['total_count']} pathways containing BRCA1")

# Step 2: Select most relevant pathway
top_pathway_id = brca1_pathways["items"][0]["id"]  # e.g., "WP:WP4868"
print(f"Analyzing pathway: {brca1_pathways['items'][0]['title']}")

# Step 3: Get complete pathway details
pathway_details = await client.call_tool("get_pathway", {
    "pathway_id": top_pathway_id
})

print(f"Pathway has {pathway_details['component_counts']['gene_count']} genes")
print(f"Cross-references: {pathway_details['cross_references'].keys()}")

# Step 4: Extract all pathway components
components = await client.call_tool("get_pathway_components", {
    "pathway_id": top_pathway_id
})

# Step 5: Analyze co-participating genes
gene_symbols = [gene["label"] for gene in components["genes"]]
print(f"Genes in pathway: {', '.join(gene_symbols[:10])}...")

# Step 6: Cross-reference to other databases
for gene in components["genes"]:
    if "uniprot" in gene["cross_references"]:
        print(f"{gene['label']}: UniProt {gene['cross_references']['uniprot']}")
```

**Expected Output**:
```
Found 127 pathways containing BRCA1
Analyzing pathway: RAC1/PAK1/p38/MMP2 pathway
Pathway has 23 genes
Cross-references: dict_keys(['kegg_pathway', 'reactome', 'gene_ontology'])
Genes in pathway: BRCA1, RAC1, PAK1, MMP2, TIAM1, ...
BRCA1: UniProt P38398
RAC1: UniProt P63000
...
```

---

## Advanced Features

### Pagination for Large Result Sets

```python
# Initial query
page1 = await client.call_tool("search_pathways", {
    "query": "metabolism",
    "page_size": 50
})

# Iterate through all pages
all_pathways = page1["items"]
cursor = page1["pagination"]["cursor"]

while cursor is not None:
    next_page = await client.call_tool("search_pathways", {
        "query": "metabolism",
        "page_size": 50,
        "cursor": cursor
    })
    all_pathways.extend(next_page["items"])
    cursor = next_page["pagination"]["cursor"]

print(f"Total pathways retrieved: {len(all_pathways)}")
```

---

### Multi-Organism Gene Comparison

```python
# Compare TP53 pathways across species
organisms = ["Homo sapiens", "Mus musculus", "Rattus norvegicus"]
tp53_pathways_by_organism = {}

for organism in organisms:
    result = await client.call_tool("get_pathways_for_gene", {
        "gene_id": "TP53",
        "organism": organism
    })
    tp53_pathways_by_organism[organism] = result["pagination"]["total_count"]

print(tp53_pathways_by_organism)
# {'Homo sapiens': 89, 'Mus musculus': 54, 'Rattus norvegicus': 12}
```

---

### Cross-Database Integration

```python
# Workflow: WikiPathways → HGNC → UniProt → PDB
# Step 1: Get pathway components
components = await client.call_tool("get_pathway_components", {
    "pathway_id": "WP:WP534"
})

# Step 2: Extract HGNC IDs
hgnc_ids = [
    gene["cross_references"]["hgnc"]
    for gene in components["genes"]
    if "hgnc" in gene["cross_references"]
]

# Step 3: Lookup in HGNC server (separate MCP server)
for hgnc_id in hgnc_ids[:5]:
    hgnc_gene = await hgnc_client.call_tool("get_gene", {"hgnc_id": hgnc_id})
    print(f"{hgnc_gene['symbol']}: {hgnc_gene['name']}")

# Step 4: Lookup in UniProt server (separate MCP server)
for gene in components["genes"]:
    if "uniprot" in gene["cross_references"]:
        uniprot_id = gene["cross_references"]["uniprot"]
        protein = await uniprot_client.call_tool("get_protein", {"uniprot_id": uniprot_id})
        print(f"{protein['name']}: {protein['function']}")
```

---

## Rate Limiting Best Practices

**Default Rate Limit**: 1 request/second (conservative)

### Pattern 1: Sequential Requests with Delay
```python
import asyncio

pathway_ids = ["WP:WP534", "WP:WP4868", "WP:WP4239"]
pathways = []

for pathway_id in pathway_ids:
    pathway = await client.call_tool("get_pathway", {"pathway_id": pathway_id})
    pathways.append(pathway)
    await asyncio.sleep(1)  # Respect rate limit
```

### Pattern 2: Batch with Exponential Backoff
```python
async def fetch_with_retry(pathway_id, max_retries=3):
    for attempt in range(max_retries):
        try:
            return await client.call_tool("get_pathway", {"pathway_id": pathway_id})
        except RateLimitError:
            wait_time = 2 ** attempt  # Exponential backoff
            await asyncio.sleep(wait_time)
    raise Exception(f"Failed to fetch {pathway_id} after {max_retries} retries")

pathways = await asyncio.gather(*[
    fetch_with_retry(pid) for pid in pathway_ids
])
```

---

## Troubleshooting

### Issue 1: "API rate limit exceeded"
**Symptom**: `RATE_LIMITED` error code
**Solution**: Implement exponential backoff and reduce request frequency

### Issue 2: "Organism filter returns no results"
**Symptom**: Empty items array despite valid gene
**Solution**: Verify organism is exact scientific name (not common name)
```python
# ❌ Wrong
result = await client.call_tool("get_pathways_for_gene", {
    "gene_id": "BRCA1",
    "organism": "human"  # Common name not supported
})

# ✅ Correct
result = await client.call_tool("get_pathways_for_gene", {
    "gene_id": "BRCA1",
    "organism": "Homo sapiens"  # Scientific name
})
```

### Issue 3: "Missing cross-references in pathway"
**Symptom**: `cross_references` object has fewer keys than expected
**Solution**: Not all pathways have all cross-references. Missing keys indicate no mapping exists.

---

## Next Steps

1. **Explore Tool Contracts**: Review detailed specifications in `contracts/` directory
2. **Run Integration Tests**: `uv run pytest tests/integration/test_wikipathways_api.py -v`
3. **Read ADR-001**: Understand Fuzzy-to-Fact protocol architecture
4. **Try Multi-Server Workflows**: Combine with HGNC, UniProt, ChEMBL servers

---

## Support & Resources

- **API Documentation**: [WikiPathways REST API](http://webservice.wikipathways.org/ui/)
- **GitHub Issues**: [lifesciences-research/issues](https://github.com/your-org/lifesciences-research/issues)
- **Tool Contracts**: See `contracts/*.md` for detailed specifications
- **Data Model**: See `data-model.md` for Pydantic schemas
