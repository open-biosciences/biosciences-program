# PubChem MCP Server - Quick Start Guide

**Status**: Specification Complete (Implementation Pending)
**Version**: 0.1.0 (Draft)
**Spec Date**: 2026-01-01

## Overview

The PubChem MCP Server provides AI agents with access to PubChem's chemical compound database through a **Fuzzy-to-Fact protocol**. Agents can search for compounds by name, synonym, or identifier, then retrieve complete compound records including SMILES, InChI, InChIKey, molecular properties, and cross-references to ChEMBL, DrugBank, and other databases.

**Key Features**:
- Access to ~118 million chemical compounds
- SMILES, InChI, and InChIKey structure representations
- Cross-references to ChEMBL and DrugBank
- Dual rate limiting (5 req/s + 400 req/min) with automatic backoff

## Installation

```bash
# Clone repository
git clone https://github.com/donbr/lifesciences-research.git
cd lifesciences-research

# Install dependencies
uv sync

# Verify installation
uv run pytest tests/integration/test_pubchem_api.py -v -m integration
```

## Running the Server

### Development Mode (with live reload)

```bash
uv run fastmcp dev src/lifesciences_mcp/servers/pubchem.py
```

### Production Mode

```bash
uv run fastmcp run src/lifesciences_mcp/servers/pubchem.py
```

## Available Tools

### 1. search_compounds (Fuzzy Search)

**Purpose**: Search for compounds by name, synonym, or chemical identifier.

**Parameters**:
- `query` (required): Search query (minimum 2 characters)
- `slim` (optional): Return minimal data for token efficiency (default: false)
- `cursor` (optional): Pagination cursor from previous response
- `page_size` (optional): Results per page, 1-100 (default: 50)

**Returns**: PaginationEnvelope with PubChemSearchCandidate items

**Example 1: Basic Search**
```python
# Search for aspirin
{
  "query": "aspirin",
  "page_size": 10
}

# Response
{
  "items": [
    {
      "id": "PubChem:CID2244",
      "name": "Aspirin",
      "molecular_formula": "C9H8O4",
      "score": 1.0
    },
    {
      "id": "PubChem:CID135565885",
      "name": "Aspirin-d4",
      "molecular_formula": "C9H4D4O4",
      "score": 0.95
    }
  ],
  "pagination": {
    "cursor": "eyJvZmZzZXQiOjEwfQ==",
    "total_count": 42,
    "page_size": 10
  }
}
```

**Example 2: Search by IUPAC Name**
```python
# Search by systematic name
{
  "query": "acetylsalicylic acid",
  "page_size": 5
}
```

**Example 3: Pagination**
```python
# Get next page of results
{
  "query": "caffeine",
  "cursor": "eyJvZmZzZXQiOjUwfQ==",
  "page_size": 50
}
```

### 2. get_compound (Strict Lookup)

**Purpose**: Retrieve complete compound record with SMILES, InChI, and cross-references.

**Parameters**:
- `pubchem_id` (required): PubChem CURIE (format: `PubChem:CID{number}`)
- `slim` (optional): Return minimal data for token efficiency (default: false)

**Returns**: PubChemCompound model with full annotations or ErrorEnvelope

**Example 1: Get Complete Compound Record**
```python
# Get aspirin
{
  "pubchem_id": "PubChem:CID2244"
}

# Response
{
  "id": "PubChem:CID2244",
  "name": "Aspirin",
  "iupac_name": "2-acetoxybenzoic acid",
  "molecular_formula": "C9H8O4",
  "molecular_weight": 180.16,
  "canonical_smiles": "CC(=O)OC1=CC=CC=C1C(=O)O",
  "isomeric_smiles": "CC(=O)OC1=CC=CC=C1C(=O)O",
  "inchi": "InChI=1S/C9H8O4/c1-6(10)13-8-5-3-2-4-7(8)9(11)12/h2-5H,1H3,(H,11,12)",
  "inchikey": "BSYNRYMUTXBXSQ-UHFFFAOYSA-N",
  "synonyms": [
    "acetylsalicylic acid",
    "2-acetoxybenzoic acid",
    "ASA",
    "Ecotrin"
  ],
  "cross_references": {
    "pubchem_compound": ["2244"],
    "chembl": ["CHEMBL:25"],
    "drugbank": ["DB00945"]
  }
}
```

**Example 2: Slim Mode (Token Efficient)**
```python
# Get minimal compound data (~20 tokens vs ~115-150 tokens)
{
  "pubchem_id": "PubChem:CID3672",
  "slim": true
}

# Response
{
  "id": "PubChem:CID3672",
  "name": "Ibuprofen",
  "molecular_formula": "C13H18O2"
}
```

## Common Workflows

### Workflow 1: Fuzzy-to-Fact (Recommended)

**Use Case**: Agent doesn't know the exact PubChem CID but knows compound name.

```python
# Step 1: Fuzzy search by name
search_result = search_compounds({"query": "ibuprofen", "page_size": 5})

# Step 2: Extract top candidate CURIE
top_result = search_result["items"][0]
compound_id = top_result["id"]  # "PubChem:CID3672"

# Step 3: Get complete compound record
compound = get_compound({"pubchem_id": compound_id})

# Step 4: Access structure data
smiles = compound["canonical_smiles"]  # "CC(C)CC1=CC=C(C=C1)C(C)C(=O)O"
inchikey = compound["inchikey"]  # "HEFNNWSXXWATRW-UHFFFAOYSA-N"
```

### Workflow 2: Cross-Database Navigation

**Use Case**: Agent needs to link PubChem compounds to ChEMBL or DrugBank.

```python
# Get compound with cross-references
compound = get_compound({"pubchem_id": "PubChem:CID2244"})

# Navigate to ChEMBL
chembl_id = compound["cross_references"]["chembl"][0]  # "CHEMBL:25"
# Use chembl_id with ChEMBL MCP server

# Navigate to DrugBank
drugbank_id = compound["cross_references"]["drugbank"][0]  # "DB00945"
# Use drugbank_id with DrugBank MCP server (when API key available)
```

### Workflow 3: Structure-Based Reasoning

**Use Case**: Agent needs chemical structure for similarity analysis.

```python
# Get compound with structure representations
compound = get_compound({"pubchem_id": "PubChem:CID2244"})

# Access structure data
smiles = compound["canonical_smiles"]    # For 2D structure
inchi = compound["inchi"]                # For unique identification
inchikey = compound["inchikey"]          # For database searching

# InChIKey is useful for cross-database queries
# First 14 chars = connectivity layer (structure)
# Middle 10 chars = stereochemistry
# Last char = protonation state
```

### Workflow 4: Error Recovery (Autonomous Correction)

**Use Case**: Agent makes a mistake and receives actionable recovery hints.

```python
# Step 1: Agent tries invalid CURIE (missing prefix)
result = get_compound({"pubchem_id": "CID2244"})

# Step 2: Error response with recovery hint
{
  "success": false,
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "Invalid PubChem CURIE format: 'CID2244'",
    "recovery_hint": "Use format 'PubChem:CID{number}' (e.g., 'PubChem:CID2244'). Call search_compounds to find valid CIDs.",
    "invalid_input": "CID2244"
  }
}

# Step 3: Agent follows hint - search first
search_result = search_compounds({"query": "aspirin", "page_size": 5})
valid_curie = search_result["items"][0]["id"]  # "PubChem:CID2244"

# Step 4: Retry with valid CURIE
compound = get_compound({"pubchem_id": valid_curie})  # Success!
```

## Error Codes and Recovery

### AMBIGUOUS_QUERY
**Cause**: Query too short (< 2 characters)
**Recovery**: Provide more specific query terms

```python
# Error
search_compounds({"query": "a"})

# Recovery
search_compounds({"query": "aspirin"})
```

### UNRESOLVED_ENTITY
**Cause**: Invalid CURIE format (missing "PubChem:CID" prefix)
**Recovery**: Use search_compounds to find valid CURIE

```python
# Error
get_compound({"pubchem_id": "2244"})

# Recovery
search_result = search_compounds({"query": "aspirin"})
get_compound({"pubchem_id": search_result["items"][0]["id"]})
```

### ENTITY_NOT_FOUND
**Cause**: Valid CURIE format but CID doesn't exist
**Recovery**: Verify CID or search by compound name

```python
# Error
get_compound({"pubchem_id": "PubChem:CID999999999999"})

# Recovery
search_compounds({"query": "compound name"})
```

### RATE_LIMITED
**Cause**: Exceeded 5 req/s or 400 req/min limit
**Recovery**: Wait and retry (exponential backoff handled automatically)

```python
# Error (after many rapid requests)
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "PubChem rate limit exceeded",
    "recovery_hint": "PubChem API rate limit exceeded. Wait 60s before retrying."
  }
}

# Recovery: Client handles automatically with exponential backoff
```

### UPSTREAM_ERROR
**Cause**: PubChem API unavailable or returned error
**Recovery**: Retry request after brief delay

```python
# Error
{
  "error": {
    "code": "UPSTREAM_ERROR",
    "message": "PubChem API error (HTTP 503)",
    "recovery_hint": "PubChem API temporarily unavailable. Retry in 60 seconds."
  }
}
```

## Cross-Reference Key Registry

The PubChem server extracts cross-references from compound synonyms:

| Key | Database | Extraction Method | Example |
|-----|----------|-------------------|---------|
| `pubchem_compound` | PubChem | Self-reference | 2244 |
| `chembl` | ChEMBL | Pattern: `CHEMBL\d+` | CHEMBL:25 |
| `drugbank` | DrugBank | Pattern: `DB\d{5}` | DB00945 |
| `uniprot` | UniProt | From xrefs (rare) | UniProtKB:P23219 |
| `pdb` | PDB | From xrefs | 1PTY |

**Note**: Cross-references use the **omit-if-null pattern** - fields are excluded entirely if no cross-reference exists (never `null` or empty string).

## Performance Characteristics

- **Search latency**: <2s for 95% of queries (SC-001)
- **Rate limits**: 5 req/s, 400 req/min (with automatic backoff)
- **Request timeout**: 30 seconds per request
- **Database size**: ~118 million compounds
- **Token efficiency**: Slim mode reduces response size by ~80% (~20 vs ~115-150 tokens)

## Structure Representations

PubChem provides multiple structure representations:

| Format | Field | Description | Use Case |
|--------|-------|-------------|----------|
| **Canonical SMILES** | `canonical_smiles` | Unique SMILES string | 2D structure, fingerprinting |
| **Isomeric SMILES** | `isomeric_smiles` | SMILES with stereochemistry | Chirality-aware analysis |
| **InChI** | `inchi` | IUPAC identifier | Unique compound identification |
| **InChIKey** | `inchikey` | 27-char hash of InChI | Database searching |
| **Molecular Formula** | `molecular_formula` | Chemical formula | Composition analysis |

**InChIKey Structure**:
```
BSYNRYMUTXBXSQ-UHFFFAOYSA-N
^^^^^^^^^^^^^^ ^^^^^^^^^^ ^
|              |          |
|              |          +-- Protonation state (N=neutral)
|              +-- Stereochemistry (UHFFFAOYSA = undefined)
+-- Connectivity layer (first 14 chars)
```

## Testing

```bash
# Run all PubChem tests
uv run pytest tests/integration/test_pubchem_api.py -v -m integration

# Run specific test suites
uv run pytest tests/integration/test_pubchem_api.py::TestPubChemFuzzySearch -v
uv run pytest tests/integration/test_pubchem_api.py::TestPubChemStrictLookup -v
uv run pytest tests/integration/test_pubchem_api.py::TestCrossReferences -v
uv run pytest tests/integration/test_pubchem_api.py::TestErrorRecovery -v

# Run unit tests
uv run pytest tests/unit/test_pubchem_client.py -v
uv run pytest tests/unit/test_pubchem_models.py -v
```

## Architecture References

- **ADR-001 v1.2**: Agentic-First Architecture (async, rate limiting, cross-references)
- **ADR-004 v1.0**: FastMCP Lifecycle Management (module-level singleton pattern)
- **Constitution v1.1.0**: Rate limiting requirements, schema determinism

## Integration with Other Servers

PubChem compounds can be linked to other Life Sciences MCP servers:

```python
# Get PubChem compound
compound = get_compound({"pubchem_id": "PubChem:CID2244"})

# Link to ChEMBL (if cross-reference exists)
if "chembl" in compound["cross_references"]:
    chembl_id = compound["cross_references"]["chembl"][0]
    # Call ChEMBL MCP server with chembl_id

# Link to DrugBank (if cross-reference exists)
if "drugbank" in compound["cross_references"]:
    drugbank_id = compound["cross_references"]["drugbank"][0]
    # Call DrugBank MCP server with drugbank_id (requires API key)

# Use InChIKey for structure-based cross-database search
inchikey = compound["inchikey"]
# Search other databases by InChIKey for structure matches
```

## Support

- **Specification**: `specs/010-pubchem-mcp-server/spec.md`
- **Research**: `specs/010-pubchem-mcp-server/research.md`
- **Data Model**: `specs/010-pubchem-mcp-server/data-model.md`
- **Contracts**: `specs/010-pubchem-mcp-server/contracts/`
- **PubChem Docs**: https://pubchem.ncbi.nlm.nih.gov/docs/pug-rest
- **GitHub**: https://github.com/donbr/lifesciences-research
