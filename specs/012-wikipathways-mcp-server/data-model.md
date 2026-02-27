# Data Model: WikiPathways MCP Server

**Created**: 2026-01-03
**Source**: Feature specification + API research
**Purpose**: Define Pydantic models for WikiPathways entities following Agentic Biolink schema

---

## Entity Overview

| Entity | Purpose | Fields | Relationships |
|--------|---------|--------|---------------|
| `Pathway` | Complete pathway record from strict lookup | 10+ fields | Has cross_references |
| `PathwaySearchCandidate` | Lightweight search result for fuzzy discovery | 5 fields | Subset of Pathway |
| `PathwayComponents` | Structured pathway composition | 4 lists | Contains DataNodes, Interactions |
| `DataNode` | Pathway component (gene/protein/metabolite) | 5 fields | Has cross_references |
| `Interaction` | Relationship between DataNodes | 3 fields | References DataNodes |

---

## 1. Pathway (Complete Entity)

**Usage**: Returned by `get_pathway` strict lookup tool

**Source Fields** (from `/getPathwayInfo` endpoint):
- `id`: WP### (numeric portion only, no prefix)
- `name`: Pathway title
- `species`: Organism scientific name
- `revision`: Revision number (string)
- `url`: Classic WikiPathways URL

**Derived Fields**:
- `description`: Extracted from pathway description or name (first 200 chars)
- `cross_references`: Populated from `findPathwaysByXref.json` bulk file
- `revision_metadata`: Composite object with version, last_modified, curators
- `component_counts`: Populated by aggregating `getXrefList` results

### Pydantic Model

```python
from pydantic import BaseModel, Field
from typing import Optional

class RevisionMetadata(BaseModel):
    """Pathway revision and curation information."""
    version: str = Field(..., description="Pathway revision number")
    last_modified: Optional[str] = Field(None, description="Last modification date (ISO 8601)")
    curators: list[str] = Field(default_factory=list, description="List of pathway curators")

class ComponentCounts(BaseModel):
    """Counts of pathway components."""
    gene_count: int = Field(0, description="Number of gene entities")
    protein_count: int = Field(0, description="Number of protein entities")
    metabolite_count: int = Field(0, description="Number of metabolite entities")
    interaction_count: int = Field(0, description="Number of interactions")

class Pathway(BaseModel):
    """
    Complete pathway entity from WikiPathways following Agentic Biolink schema.

    Returned by get_pathway strict lookup tool.
    """
    id: str = Field(..., description="WikiPathways CURIE (WP:WP###)", pattern=r"^WP:WP\d+$")
    title: str = Field(..., description="Pathway name")
    organism: str = Field(..., description="Scientific organism name (e.g., 'Homo sapiens')")
    description: str = Field(..., description="Pathway description or summary")
    revision: RevisionMetadata = Field(..., description="Revision metadata")
    component_counts: ComponentCounts = Field(..., description="Component counts")
    cross_references: dict[str, str | list[str]] = Field(
        default_factory=dict,
        description="Cross-references to external databases (reactome, kegg_pathway, gene_ontology, etc.)"
    )
    url: str = Field(..., description="WikiPathways pathway URL")

    class Config:
        json_schema_extra = {
            "example": {
                "id": "WP:WP534",
                "title": "Glycolysis and gluconeogenesis",
                "organism": "Homo sapiens",
                "description": "Glycolysis is the metabolic pathway that converts glucose into pyruvate...",
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
                    "kegg_pathway": "hsa00010",
                    "reactome": "R-HSA-70171",
                    "gene_ontology": "GO:0006096"
                },
                "url": "https://classic.wikipathways.org/index.php/Pathway:WP534"
            }
        }
```

**Cross-Reference Mapping** (from research.md):
- `ncbigene` → `entrez`
- `ensembl` → `ensembl_gene`
- `hgnc.symbol` → `hgnc`
- `uniprot` → `uniprot`
- `chebi` → (map to `pubchem_compound` if possible, else omit)
- `wikidata` → (omit - not in 22-key registry)
- `inchikey` → (omit - not in 22-key registry)

**Validation Rules**:
- `id` MUST match regex `^WP:WP\d+$` (WP: prefix + WP + digits)
- `organism` MUST be scientific name (not common name)
- `cross_references` MUST omit keys with no values (never use `null` or empty string)
- `curators` MAY be empty list if data unavailable

---

## 2. PathwaySearchCandidate (Fuzzy Discovery)

**Usage**: Returned by `search_pathways` and `get_pathways_for_gene` fuzzy tools

**Source Fields** (from `/findPathwaysByText` and `/findPathwaysByXref` endpoints):
- `id`: WP### (numeric portion)
- `name`: Pathway title
- `species`: Organism scientific name
- `score`: Relevance score (string, needs float conversion)

**Derived Fields**:
- `description`: Truncated pathway description (first 200 chars)
- `relevance_score`: Normalized score (0.0-1.0)

### Pydantic Model

```python
class PathwaySearchCandidate(BaseModel):
    """
    Lightweight pathway search result for fuzzy discovery.

    Returned by search_pathways and get_pathways_for_gene tools.
    Uses slim representation (~20 tokens vs ~115 tokens for full Pathway).
    """
    id: str = Field(..., description="WikiPathways CURIE (WP:WP###)", pattern=r"^WP:WP\d+$")
    title: str = Field(..., description="Pathway name")
    organism: str = Field(..., description="Scientific organism name")
    description: str = Field(..., description="Brief description snippet (first 200 chars)")
    score: float = Field(..., description="Relevance score (0.0-1.0, higher is better)")

    class Config:
        json_schema_extra = {
            "example": {
                "id": "WP:WP534",
                "title": "Glycolysis and gluconeogenesis",
                "organism": "Homo sapiens",
                "description": "Glycolysis is the metabolic pathway that converts glucose C6H12O6, into pyruvate...",
                "score": 0.95
            }
        }
```

**Score Calculation** (per FR-009):
- Base score from API (`score` field in response)
- Apply position decay: `score - (position * 0.05)`
- Normalize to 0.0-1.0 range
- Sort descending by score

**Token Budget**:
- Slim mode (default): ~20 tokens
- Full Pathway: ~115-300 tokens
- Reduction: 80-90% token savings

---

## 3. PathwayComponents (Component Extraction)

**Usage**: Returned by `get_pathway_components` tool

**Source**: Aggregates multiple `getXrefList` API calls with different BridgeDb codes:
- `code=L`: NCBI Gene (Entrez)
- `code=S`: UniProt
- `code=En`: Ensembl
- `code=H`: HGNC
- `code=Ce`: ChEMBL

**Classification Logic**:
- Genes: Entities with `code=L` (NCBI Gene) or `code=H` (HGNC)
- Proteins: Entities with `code=S` (UniProt)
- Metabolites: Entities with `code=Ce` (ChEMBL) or `code=Ch` (CHEBI)
- Interactions: Derived from pathway topology (if available)

### Pydantic Model

```python
class PathwayComponents(BaseModel):
    """
    Structured pathway composition data.

    Returned by get_pathway_components tool.
    Contains genes, proteins, metabolites, and interactions.
    """
    genes: list["DataNode"] = Field(default_factory=list, description="Gene entities in pathway")
    proteins: list["DataNode"] = Field(default_factory=list, description="Protein entities in pathway")
    metabolites: list["DataNode"] = Field(default_factory=list, description="Metabolite entities in pathway")
    interactions: list["Interaction"] = Field(default_factory=list, description="Interactions between components")

    class Config:
        json_schema_extra = {
            "example": {
                "genes": [
                    {
                        "id": "ncbigene:5213",
                        "label": "PFKM",
                        "type": "Gene",
                        "database": "Entrez Gene",
                        "cross_references": {"entrez": "5213", "hgnc": "HGNC:8877"}
                    }
                ],
                "proteins": [
                    {
                        "id": "uniprot:P08237",
                        "label": "PFKM",
                        "type": "Protein",
                        "database": "UniProt",
                        "cross_references": {"uniprot": "P08237", "hgnc": "HGNC:8877"}
                    }
                ],
                "metabolites": [
                    {
                        "id": "chebi:17234",
                        "label": "Glucose",
                        "type": "Metabolite",
                        "database": "ChEBI",
                        "cross_references": {"pubchem_compound": "5793"}
                    }
                ],
                "interactions": [
                    {
                        "source": "ncbigene:5213",
                        "target": "chebi:17234",
                        "type": "catalysis"
                    }
                ]
            }
        }
```

**Omit Empty Lists**: If pathway has no metabolites, omit `metabolites` key entirely (ADR-001 §4).

---

## 4. DataNode (Pathway Component)

**Usage**: Represents a gene, protein, metabolite, or complex within a pathway

**Source Fields** (from `getXrefList` responses):
- `id`: External database identifier (e.g., `672` for NCBI Gene)
- `label`: Entity label (gene symbol, protein name, metabolite name)
- Database code determines `type` and `database` fields

### Pydantic Model

```python
class DataNode(BaseModel):
    """
    Pathway component entity (gene, protein, metabolite, complex).

    Used within PathwayComponents.
    """
    id: str = Field(..., description="Internal node identifier or external database ID")
    label: str = Field(..., description="Display name (gene symbol, protein name, etc.)")
    type: str = Field(..., description="Entity type: Gene, Protein, Metabolite, Complex")
    database: str = Field(..., description="Source database (e.g., 'Entrez Gene', 'UniProt', 'ChEMBL')")
    cross_references: dict[str, str | list[str]] = Field(
        default_factory=dict,
        description="Cross-references to external databases"
    )

    class Config:
        json_schema_extra = {
            "example": {
                "id": "ncbigene:672",
                "label": "BRCA1",
                "type": "Gene",
                "database": "Entrez Gene",
                "cross_references": {
                    "entrez": "672",
                    "hgnc": "HGNC:1100",
                    "ensembl_gene": "ENSG00000012048",
                    "uniprot": "P38398"
                }
            }
        }
```

**Type Classification**:
- `Gene`: BridgeDb code `L` (NCBI Gene) or `H` (HGNC)
- `Protein`: BridgeDb code `S` (UniProt)
- `Metabolite`: BridgeDb code `Ce` (ChEMBL) or `Ch` (CHEBI)
- `Complex`: Special case (multiple proteins, identified by label pattern)

---

## 5. Interaction (Pathway Relationship)

**Usage**: Represents a relationship between two DataNodes

**Source**: Derived from pathway topology data (if available from API)

**Note**: WikiPathways REST API may not provide interaction data directly. If unavailable, return empty `interactions` list.

### Pydantic Model

```python
class Interaction(BaseModel):
    """
    Pathway relationship entity connecting two DataNodes.

    Used within PathwayComponents.
    """
    source: str = Field(..., description="Source DataNode ID")
    target: str = Field(..., description="Target DataNode ID")
    type: str = Field(..., description="Interaction type: activation, inhibition, binding, conversion, catalysis")

    class Config:
        json_schema_extra = {
            "example": {
                "source": "ncbigene:5213",
                "target": "chebi:17234",
                "type": "catalysis"
            }
        }
```

**Interaction Types**:
- `activation`: Positive regulation
- `inhibition`: Negative regulation
- `binding`: Physical association
- `conversion`: Metabolic transformation
- `catalysis`: Enzymatic reaction

---

## Cross-Reference Mapping Strategy

### Research Finding: Bulk JSON File

**Source**: `https://www.wikipathways.org/json/findPathwaysByXref.json`

**Structure**:
```json
{
  "pathways": {
    "WP534": {
      "ncbigene": "672,675,5213,5214,...",
      "ensembl": "ENSG00000012048,ENSG00000013374,...",
      "hgnc.symbol": "BRCA1,BRCA2,PFKM,...",
      "uniprot": "P38398;P51587,Q01813,...",
      "chebi": "17234,16108,..."
    }
  }
}
```

### Mapping Algorithm

1. **Load bulk file on client initialization** (singleton pattern)
2. **Parse comma-separated values** for each database
3. **Strip prefixes**: `ncbigene:` → `` (empty), `hgnc.symbol:` → `` (empty)
4. **Handle UniProt isoforms**: `P38398;P51587` → Use first (canonical)
5. **Map to 22-key registry**:
   - `ncbigene` → `entrez`
   - `ensembl` → `ensembl_gene`
   - `hgnc.symbol` → `hgnc`
   - `uniprot` → `uniprot`
6. **Omit unmapped keys** (wikidata, inchikey not in registry)

### Code Pattern

```python
def _map_cross_references(self, pathway_id: str) -> dict[str, str | list[str]]:
    """Map WikiPathways cross-references to Agentic Biolink schema."""
    xrefs = {}

    # Get cross-references from bulk JSON file
    pathway_xrefs = self._bulk_xrefs.get(pathway_id, {})

    # Map ncbigene → entrez
    if "ncbigene" in pathway_xrefs:
        xrefs["entrez"] = pathway_xrefs["ncbigene"].split(",")

    # Map ensembl → ensembl_gene
    if "ensembl" in pathway_xrefs:
        xrefs["ensembl_gene"] = pathway_xrefs["ensembl"].split(",")

    # Map hgnc.symbol → hgnc (strip prefix)
    if "hgnc.symbol" in pathway_xrefs:
        xrefs["hgnc"] = [s.replace("hgnc.symbol:", "") for s in pathway_xrefs["hgnc.symbol"].split(",")]

    # Map uniprot → uniprot (handle isoforms)
    if "uniprot" in pathway_xrefs:
        # Split on comma, then semicolon for isoforms, take first
        xrefs["uniprot"] = [id.split(";")[0] for id in pathway_xrefs["uniprot"].split(",")]

    # Omit empty values
    return {k: v for k, v in xrefs.items() if v}
```

---

## Validation Rules

### ID Format Validation

```python
import re

PATHWAY_ID_PATTERN = re.compile(r"^WP:WP\d+$")

def validate_pathway_id(pathway_id: str) -> bool:
    """Validate pathway ID matches WP:WPNNNNN format."""
    return bool(PATHWAY_ID_PATTERN.match(pathway_id))
```

### Error Cases

| Invalid Input | Error Code | Recovery Hint |
|---------------|------------|---------------|
| `"glycolysis"` | `UNRESOLVED_ENTITY` | "Call search_pathways to resolve pathway identifier first" |
| `"WP534"` (no prefix) | `UNRESOLVED_ENTITY` | "Pathway ID must be in WP:WPNNNNN format (e.g., WP:WP534)" |
| `"WP:WP99999"` (valid format, doesn't exist) | `ENTITY_NOT_FOUND` | "Pathway WP:WP99999 not found. Verify ID or use search_pathways" |

---

## Model File Organization

**File**: `src/lifesciences_mcp/models/pathway.py`
```python
from pydantic import BaseModel, Field
from typing import Optional

# All pathway-related models in single file
class RevisionMetadata(BaseModel): ...
class ComponentCounts(BaseModel): ...
class Pathway(BaseModel): ...
class PathwaySearchCandidate(BaseModel): ...
```

**File**: `src/lifesciences_mcp/models/pathway_components.py`
```python
from pydantic import BaseModel, Field

# Component-related models in separate file
class DataNode(BaseModel): ...
class Interaction(BaseModel): ...
class PathwayComponents(BaseModel): ...
```

**Rationale**: Separate files prevent circular imports and improve modularity.

---

## Token Budget Analysis

| Model | Fields | Avg Tokens (Full) | Avg Tokens (Slim) | Reduction |
|-------|--------|-------------------|-------------------|-----------|
| Pathway | 10 | ~300 | ~20 (id, title, organism, score) | 93% |
| PathwaySearchCandidate | 5 | ~20 | ~20 (already slim) | 0% |
| PathwayComponents | 4 lists | ~500-1000 | N/A (no slim mode) | N/A |
| DataNode | 5 | ~50 | N/A | N/A |

**Key Takeaway**: PathwaySearchCandidate is already token-optimized for fuzzy discovery phase.

---

## References

- [ADR-001 v1.2: Agentic-First Architecture](../../docs/adr/accepted/adr-001-v1.2.md)
- [WikiPathways REST API](http://webservice.wikipathways.org/ui/)
- [WikiPathways JSON Bulk Files](https://www.wikipathways.org/json/)
- [BridgeDb System Codes](https://github.com/bridgedb/datasources/blob/main/datasources.ts)
