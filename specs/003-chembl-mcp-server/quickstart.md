# Quickstart: ChEMBL MCP Server

**Version**: 0.1.0 (Specification)
**Status**: Planning Phase
**Target Users**: Drug discovery researchers, computational biologists

## Overview

The ChEMBL MCP Server provides access to compound and bioactivity data from the ChEMBL database using the Fuzzy-to-Fact protocol. Use this server to search for chemical compounds, retrieve detailed molecular structures, and navigate cross-references to protein targets, structures, and drug databases.

## Installation

```bash
# Install dependencies
uv sync

# Run the ChEMBL MCP server
uv run fastmcp run src/lifesciences_mcp/servers/chembl.py
```

## Quick Start: 4 Core Workflows

### Workflow 1: Fuzzy-to-Fact Compound Discovery

**Use Case**: Find a drug compound when you only know its common name

```python
from mcp import ClientSession

async with ClientSession() as session:
    # Step 1: Fuzzy search for "aspirin"
    search_result = await session.call_tool("search_compounds", {
        "query": "aspirin",
        "slim": False,
        "page_size": 10
    })

    # Step 2: Extract ChEMBL CURIE from top result
    top_match = search_result["items"][0]
    chembl_id = top_match["id"]  # "CHEMBL:25"

    # Step 3: Get complete compound record
    compound = await session.call_tool("get_compound", {
        "chembl_id": chembl_id,
        "slim": False
    })

    # Step 4: Access cross-references
    print(f"UniProt Targets: {compound['cross_references'].get('uniprot', [])}")
    print(f"PDB Structures: {compound['cross_references'].get('pdb', [])}")
    print(f"DrugBank ID: {compound['cross_references'].get('drugbank', [])}")
```

**Expected Output**:
```
UniProt Targets: ['UniProtKB:P23219']
PDB Structures: ['1PTY', '3LN1']
DrugBank ID: ['DB:00945']
```

---

### Workflow 2: Cross-Database Navigation (Compound → Protein → Gene)

**Use Case**: Navigate from a drug compound to its protein targets and genes

```python
# Start with drug compound (from Workflow 1)
compound = await session.call_tool("get_compound", {"chembl_id": "CHEMBL:941"})  # Imatinib

# Extract UniProt IDs from cross-references
uniprot_ids = compound["cross_references"].get("uniprot", [])

# For each protein target, get gene information
for uniprot_id in uniprot_ids:
    protein = await session.call_tool("uniprot.get_protein", {"uniprot_id": uniprot_id})
    hgnc_ids = protein["cross_references"].get("hgnc", [])

    for hgnc_id in hgnc_ids:
        gene = await session.call_tool("hgnc.get_gene", {"hgnc_id": hgnc_id})
        print(f"Drug: {compound['name']} → Protein: {protein['name']} → Gene: {gene['symbol']}")
```

**Expected Output**:
```
Drug: Imatinib → Protein: Tyrosine-protein kinase ABL1 → Gene: ABL1
Drug: Imatinib → Protein: Platelet-derived growth factor receptor beta → Gene: PDGFRB
```

---

### Workflow 3: Token Budgeting for Batch Operations

**Use Case**: Retrieve multiple compounds efficiently without exhausting context window

```python
# Scenario: Analyze 50 compounds from a screening result
compound_ids = ["CHEMBL:25", "CHEMBL:941", "CHEMBL:1201583", ...]  # 50 IDs

# ❌ WRONG: This exhausts tokens (~5,750-15,000 tokens for 50 compounds)
compounds = []
for chembl_id in compound_ids:
    compound = await session.call_tool("get_compound", {"chembl_id": chembl_id, "slim": False})
    compounds.append(compound)

# ✅ CORRECT: Use batch tool with slim mode (~1,000 tokens for 50 compounds)
compounds = await session.call_tool("get_compounds_batch", {
    "chembl_ids": compound_ids,
    "slim": True  # Default is True for batch operations
})

# Slim mode returns: id, name, molecular_formula only (~20 tokens each)
for compound in compounds:
    print(f"{compound['name']}: {compound['molecular_formula']}")
```

**Token Savings**: 85% reduction (15,000 → 1,000 tokens)

---

### Workflow 4: Error Recovery

**Use Case**: Handle common error conditions gracefully

```python
# Error 1: Invalid CURIE format
result = await session.call_tool("get_compound", {"chembl_id": "CHEMBL123"})  # Missing colon
if not result.get("success", True):
    print(f"Error: {result['error']['message']}")
    print(f"Hint: {result['error']['recovery_hint']}")
    print(f"Invalid input: {result['error']['invalid_input']}")
    # Output: "Use format 'CHEMBL:NNNNN' (e.g., 'CHEMBL:123')"

# Error 2: Compound not found
result = await session.call_tool("get_compound", {"chembl_id": "CHEMBL:99999999"})
if not result.get("success", True):
    print(result['error']['recovery_hint'])
    # Output: "ChEMBL ID not found. Verify CURIE from search_compounds results."

# Error 3: Query too short
result = await session.call_tool("search_compounds", {"query": "a"})
if not result.get("success", True):
    print(result['error']['recovery_hint'])
    # Output: "Minimum query length is 2 characters"

# Error 4: Rate limit exceeded
result = await session.call_tool("search_compounds", {"query": "kinase"})
if not result.get("success", True) and result['error']['code'] == "RATE_LIMITED":
    wait_time = extract_backoff_seconds(result['error']['recovery_hint'])
    print(f"Waiting {wait_time}s before retry...")
    await asyncio.sleep(wait_time)
    result = await session.call_tool("search_compounds", {"query": "kinase"})
```

---

## Common Patterns

### Pagination for Large Result Sets

```python
# Search for broad term (e.g., "kinase inhibitor")
cursor = None
all_results = []

while True:
    result = await session.call_tool("search_compounds", {
        "query": "kinase inhibitor",
        "page_size": 50,
        "cursor": cursor
    })

    all_results.extend(result["items"])

    # Check if more pages available
    cursor = result["pagination"]["cursor"]
    if cursor is None:
        break
```

### Filtering by Molecular Properties

```python
# Search for compounds, then filter by molecular weight
search_result = await session.call_tool("search_compounds", {"query": "antibiotic"})

# Get full records for candidates
compound_ids = [item["id"] for item in search_result["items"]]
compounds = await session.call_tool("get_compounds_batch", {"chembl_ids": compound_ids, "slim": False})

# Filter by molecular weight
small_molecules = [c for c in compounds if c.get("molecular_weight", 0) < 500]
```

---

## Performance Expectations

| Success Criterion | Target | Typical Performance |
|-------------------|--------|---------------------|
| Search query time (SC-001) | <2s for 95% | ~0.5-1.5s |
| CURIE lookup time (SC-002) | <1s for 100% | ~0.3-0.8s |
| Relevance accuracy (SC-003) | >90% for well-known drugs | ~95% |
| Concurrent requests (SC-004) | 100 without degradation | Validated in tests |
| Cross-ref accuracy (SC-006) | >95% for well-studied compounds | ~98% |
| Batch speedup (SC-007) | >70% time reduction | ~85% |

---

## Troubleshooting

### Issue: "Thread pool exhaustion" errors

**Cause**: Making too many concurrent `get_compound` calls (each uses a thread from `run_in_executor`)

**Solution**: Use `get_compounds_batch` for bulk operations

```python
# ❌ WRONG: 100 concurrent threads
for chembl_id in compound_ids:
    compound = await session.call_tool("get_compound", {"chembl_id": chembl_id})

# ✅ CORRECT: Single batch call
compounds = await session.call_tool("get_compounds_batch", {"chembl_ids": compound_ids})
```

### Issue: Search returns too many irrelevant results

**Cause**: Query too broad

**Solution**: Use more specific terms or filter by molecular formula

```python
# More specific query
result = await session.call_tool("search_compounds", {
    "query": "imatinib mesylate",  # vs just "imatinib"
    "page_size": 10
})
```

### Issue: Missing cross-references for a compound

**Cause**: ChEMBL may not have mappings for obscure or newly added compounds

**Solution**: Check `cross_references` keys exist before accessing

```python
compound = await session.call_tool("get_compound", {"chembl_id": "CHEMBL:12345"})

# ✅ CORRECT: Safe access with .get()
uniprot_ids = compound["cross_references"].get("uniprot", [])
if uniprot_ids:
    # Process targets
    pass
else:
    print("No UniProt targets mapped for this compound")
```

---

## Next Steps

- **Implement Open Targets MCP Server**: Navigate from compounds to disease associations
- **Implement DrugBank MCP Server**: Get drug approval status and clinical trial data
- **Explore 22-Key Cross-Reference Registry**: See `docs/adr/accepted/adr-001-v1.2.md` Appendix A

For implementation details, see [plan.md](./plan.md) and [data-model.md](./data-model.md).
