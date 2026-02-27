# Quickstart: Open Targets MCP Server

**Version**: 0.1.0 (Specification)
**Status**: Planning Phase
**Target Users**: Drug discovery researchers, computational biologists

## Overview

The Open Targets MCP Server provides access to target-disease association data from the Open Targets Platform using the Fuzzy-to-Fact protocol. Use this server to search for gene targets, retrieve detailed target profiles, and navigate target-disease associations for drug repurposing.

## Installation

```bash
# Install dependencies
uv sync

# Run the Open Targets MCP server
uv run fastmcp run src/lifesciences_mcp/servers/opentargets.py
```

## Quick Start: 4 Core Workflows

### Workflow 1: Fuzzy-to-Fact Target Discovery

**Use Case**: Find a gene target when you only know its symbol or common name

```python
from mcp import ClientSession

async with ClientSession() as session:
    # Step 1: Fuzzy search for "TP53"
    search_result = await session.call_tool("search_targets", {
        "query": "TP53",
        "slim": False,
        "page_size": 10
    })

    # Step 2: Extract Ensembl ID from top result
    top_match = search_result["items"][0]
    ensembl_id = top_match["id"]  # "ENSG00000141510"

    # Step 3: Get complete target record
    target = await session.call_tool("get_target", {
        "ensembl_id": ensembl_id,
        "slim": False
    })

    # Step 4: Access cross-references
    print(f"HGNC: {target['cross_references'].get('hgnc', [])}")
    print(f"UniProt: {target['cross_references'].get('uniprot', [])}")
    print(f"Disease Associations: {target['associated_diseases_count']}")
```

**Expected Output**:
```
HGNC: ['HGNC:11998']
UniProt: ['UniProtKB:P04637']
Disease Associations: 245
```

---

### Workflow 2: Cross-Database Navigation (Target → Protein → Gene)

**Use Case**: Navigate from Open Targets target to UniProt protein to HGNC gene

```python
# Start with target (from Workflow 1)
target = await session.call_tool("get_target", {"ensembl_id": "ENSG00000141510"})

# Extract UniProt IDs from cross-references
uniprot_ids = target["cross_references"].get("uniprot", [])

# For each protein, get full UniProt record
for uniprot_id in uniprot_ids:
    protein = await session.call_tool("uniprot.get_protein", {"uniprot_id": uniprot_id})
    hgnc_ids = protein["cross_references"].get("hgnc", [])

    for hgnc_id in hgnc_ids:
        gene = await session.call_tool("hgnc.get_gene", {"hgnc_id": hgnc_id})
        print(f"Target: {target['approved_symbol']} → Protein: {protein['name']} → Gene: {gene['symbol']}")
```

**Expected Output**:
```
Target: TP53 → Protein: Cellular tumor antigen p53 → Gene: TP53
```

---

### Workflow 3: Token Budgeting for Batch Operations

**Use Case**: Search for multiple targets efficiently without exhausting context window

```python
# Scenario: Find targets for multiple gene symbols
gene_symbols = ["TP53", "BRCA1", "EGFR", "MYC", "KRAS"]  # 5 symbols

# ✅ CORRECT: Use slim mode for search (~100 tokens total)
for symbol in gene_symbols:
    result = await session.call_tool("search_targets", {
        "query": symbol,
        "slim": True,  # Returns only id, symbol, name, score
        "page_size": 1
    })
    top_match = result["items"][0]
    print(f"{symbol}: {top_match['id']} (score: {top_match['score']})")

# Slim mode returns: id, approved_symbol, approved_name, score (~20 tokens each)
```

**Token Savings**: 80% reduction (500 → 100 tokens for 5 targets)

---

### Workflow 4: Target-Disease Association Discovery

**Use Case**: Find diseases associated with a specific gene target

```python
# Query associations for TP53
associations = await session.call_tool("get_associations", {
    "target_id": "ENSG00000141510",  # TP53
    "page_size": 10
})

# Display top disease associations
for assoc in associations["items"][:5]:
    print(f"{assoc['disease_name']}: score={assoc['score']:.2f}, evidence={assoc['evidence_count']}")
```

**Expected Output**:
```
neoplasm: score=0.85, evidence=142
Li-Fraumeni syndrome: score=0.92, evidence=38
breast carcinoma: score=0.78, evidence=56
lung carcinoma: score=0.72, evidence=64
colorectal carcinoma: score=0.69, evidence=48
```

---

## Common Patterns

### Pagination for Large Result Sets

```python
# Search for broad term (e.g., "kinase")
cursor = None
all_results = []

while True:
    result = await session.call_tool("search_targets", {
        "query": "kinase",
        "page_size": 50,
        "cursor": cursor
    })

    all_results.extend(result["items"])

    # Check if more pages available
    cursor = result["pagination"]["cursor"]
    if cursor is None:
        break
```

### Filtering Associations by Disease

```python
# Get only breast cancer associations for BRCA1
associations = await session.call_tool("get_associations", {
    "target_id": "ENSG00000012048",  # BRCA1
    "disease_id": "EFO_0000305",  # breast carcinoma
    "page_size": 50
})
```

---

## Performance Expectations

| Success Criterion | Target | Typical Performance |
|-------------------|--------|---------------------|
| Search query time (SC-001) | <2s for 95% | ~0.5-1.5s |
| Target lookup time (SC-002) | <1s for 100% | ~0.3-0.8s |
| Relevance accuracy (SC-003) | >90% for well-known genes | ~95% |
| Concurrent requests (SC-004) | 100 without degradation | Validated in tests |
| Cross-ref accuracy (SC-006) | >95% for well-studied targets | ~98% |

---

## Troubleshooting

### Issue: "AMBIGUOUS_QUERY" error

**Cause**: Query too short (minimum 2 characters)

**Solution**: Use longer search terms

```python
# ❌ WRONG: Single character
result = await session.call_tool("search_targets", {"query": "a"})

# ✅ CORRECT: At least 2 characters
result = await session.call_tool("search_targets", {"query": "TP"})
```

### Issue: "UNRESOLVED_ENTITY" error

**Cause**: Invalid Ensembl ID format

**Solution**: Use correct format (ENSG + 11 digits)

```python
# ❌ WRONG: Gene symbol instead of Ensembl ID
result = await session.call_tool("get_target", {"ensembl_id": "TP53"})

# ✅ CORRECT: Valid Ensembl ID
result = await session.call_tool("get_target", {"ensembl_id": "ENSG00000141510"})
```

### Issue: Missing cross-references for a target

**Cause**: Open Targets may not have mappings for obscure or newly added targets

**Solution**: Check `cross_references` keys exist before accessing

```python
target = await session.call_tool("get_target", {"ensembl_id": "ENSG00000012345"})

# ✅ CORRECT: Safe access with .get()
uniprot_ids = target["cross_references"].get("uniprot", [])
if uniprot_ids:
    # Process protein targets
    pass
else:
    print("No UniProt mappings for this target")
```

---

## Next Steps

- **Implement ChEMBL MCP Server**: Navigate from targets to compounds
- **Implement DrugBank MCP Server**: Get drug approval status and clinical trial data
- **Explore 22-Key Cross-Reference Registry**: See `docs/adr/accepted/adr-001-v1.2.md` Appendix A

For implementation details, see [plan.md](./plan.md) and [data-model.md](./data-model.md).
