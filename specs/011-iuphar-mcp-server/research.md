# Research: IUPHAR/GtoPdb API

**Feature**: 011-iuphar-mcp-server
**Date**: 2026-01-01
**Status**: Complete

## Overview

This document captures the API research findings for the IUPHAR/BPS Guide to PHARMACOLOGY (GtoPdb) REST API. GtoPdb is an expert-curated database of pharmacological targets and their experimental ligands, supporting drug discovery and pharmacology research.

## R1: API Base Endpoints

### Base URL

```
https://www.guidetopharmacology.org/services/
```

### Authentication

None required. GtoPdb is a public open-access resource.

### Data Format

JSON only (no XML or other formats)

### Licensing

- Data: Open Data Commons Open Database License (ODbL)
- Content: Creative Commons Attribution-ShareAlike 4.0 International License

### Core Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/ligands` | GET | List/search ligands with optional filters |
| `/ligands/{ligandId}` | GET | Get single ligand by ID |
| `/ligands/{ligandId}/databaseLinks` | GET | Get ligand cross-references |
| `/ligands/{ligandId}/synonyms` | GET | Get ligand synonyms/brand names |
| `/targets` | GET | List/search targets with optional filters |
| `/targets/{targetId}` | GET | Get single target by ID |
| `/targets/{targetId}/databaseLinks` | GET | Get target cross-references |
| `/interactions` | GET | Get target-ligand interactions |

---

## R2: Search API Query Parameters

### Ligand Search Parameters

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `name` | string | Fuzzy substring match on ligand name | `?name=ibuprofen` |
| `type` | string | Filter by ligand classification | `?type=Synthetic%20organic` |
| `accession` | string | External database ID | `?accession=DB01050` |
| `database` | string | Database for accession lookup (default: PubChemCID) | `?database=DrugBank` |
| `approved` | boolean | Filter for approved drugs only | `?approved=true` |
| `inchikey` | string | Structure search by InChIKey | `?inchikey=HEFNNWSXXWATRW-UHFFFAOYSA-N` |
| `immuno` | boolean | Immunopharmacology compounds | `?immuno=true` |
| `malaria` | boolean | Antimalarial compounds | `?malaria=true` |
| `antibacterial` | boolean | Antibacterial compounds | `?antibacterial=true` |

#### Ligand Type Values

- Synthetic organic
- Metabolite
- Natural product
- Endogenous peptide
- Peptide
- Antibody
- Inorganic
- Approved
- Withdrawn
- Labelled
- INN

### Target Search Parameters

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `name` | string | Fuzzy substring match on target name | `?name=dopamine` |
| `type` | string | Filter by target classification | `?type=GPCR` |
| `geneSymbol` | string | Human/mouse/rat gene symbol | `?geneSymbol=DRD2` |
| `accession` | string | External database ID | `?accession=P14416` |
| `database` | string | Database for accession lookup (default: UniProt) | `?database=UniProt` |
| `ecNumber` | string | Enzyme Commission number (enzymes only) | `?ecNumber=3.4.21.4` |
| `immuno` | boolean | Immunopharmacology targets | `?immuno=true` |
| `malaria` | boolean | Antimalarial targets | `?malaria=true` |

#### Target Type Values

- GPCR (G protein-coupled receptors)
- NHR (Nuclear hormone receptors)
- LGIC (Ligand-gated ion channels)
- VGIC (Voltage-gated ion channels)
- OtherIC (Other ion channels)
- Enzyme
- CatalyticReceptor
- Transporter
- OtherProtein
- AccessoryProtein

### Search Behavior

- **Matching**: Substring match (case-insensitive)
- **Ranking**: Results returned in database order (no explicit relevance ranking)
- **Combination**: Multiple parameters combined with AND logic
- **Empty Results**: Returns HTTP 204 No Content (not an error)

---

## R3: Pagination Patterns

### Findings

**GtoPdb does NOT support server-side pagination.** All matching results are returned in a single response.

### Implementation Strategy

Use client-side pagination:

1. Fetch all results from API
2. Store in memory
3. Return page_size items per request
4. Use base64-encoded cursor with offset

### Cursor Format

```python
import base64
import json

# Encode cursor
cursor_data = {"offset": 50}
cursor = base64.b64encode(json.dumps(cursor_data).encode()).decode()

# Decode cursor
cursor_data = json.loads(base64.b64decode(cursor).decode())
offset = cursor_data.get("offset", 0)
```

### Pattern Reference

Same approach used in HGNC server implementation.

---

## R4: Rate Limiting Policy

### Official Documentation

**None found.** GtoPdb does not publish rate limit documentation.

### Recommendation

Implement conservative rate limiting:

- **Rate**: 1 request per second (1 req/s)
- **Rationale**: GtoPdb is a community academic resource with limited infrastructure
- **Backoff**: Exponential (2^attempt seconds)
- **Max Retries**: 3

### Error Handling

GtoPdb does not return standard rate limit headers:

- No `Retry-After` header
- No `X-RateLimit-*` headers
- Rate limit violations likely return 503 or connection timeout

### Implementation

```python
RATE_LIMIT_DELAY = 1.0  # 1 req/s = 1000ms between requests
MAX_RETRIES = 3

async def _rate_limited_get(self, path: str) -> httpx.Response:
    async with self._lock:
        now = asyncio.get_event_loop().time()
        elapsed = now - self._last_request_time
        if elapsed < self.RATE_LIMIT_DELAY:
            await asyncio.sleep(self.RATE_LIMIT_DELAY - elapsed)
        response = await self._get(path)
        self._last_request_time = asyncio.get_event_loop().time()
    return response
```

---

## R5: Ligand Cross-Reference Mappings

### Endpoint

```
GET /ligands/{ligandId}/databaseLinks
```

### Sample Response (ibuprofen, ID 2713)

```json
[
  {
    "database": "ChEMBL Ligand",
    "accession": "CHEMBL521",
    "url": "https://www.ebi.ac.uk/chembl/compound_report_card/CHEMBL521"
  },
  {
    "database": "DrugBank Ligand",
    "accession": "DB01050",
    "url": "https://www.drugbank.ca/drugs/DB01050"
  },
  {
    "database": "PubChem CID",
    "accession": "3672",
    "url": "https://pubchem.ncbi.nlm.nih.gov/compound/3672"
  },
  {
    "database": "ChEBI",
    "accession": "CHEBI:5855",
    "url": "https://www.ebi.ac.uk/chebi/searchId.do?chebiId=CHEBI:5855"
  },
  {
    "database": "BindingDB Ligand",
    "accession": "50009859",
    "url": "https://www.bindingdb.org/bind/chemsearch/marvin/MolStructure.jsp?monession=50009859"
  },
  {
    "database": "CAS Registry No.",
    "accession": "15687-27-1",
    "url": null
  },
  {
    "database": "DrugCentral Ligand",
    "accession": "1407",
    "url": "https://drugcentral.org/?q=1407"
  },
  {
    "database": "Wikipedia",
    "accession": "Ibuprofen",
    "url": "https://en.wikipedia.org/wiki/Ibuprofen"
  }
]
```

### Mapping to 22-Key Registry

| GtoPdb Database | Registry Key | Notes |
|-----------------|--------------|-------|
| ChEMBL Ligand | `chembl` | Strip "CHEMBL" prefix if present |
| DrugBank Ligand | `drugbank` | Format: DB\d{5} |
| PubChem CID | `pubchem_compound` | Numeric ID |
| ChEBI | - | Not in current registry (consider adding) |
| BindingDB Ligand | - | Not in registry |
| CAS Registry No. | - | Not in registry |
| DrugCentral Ligand | - | Not in registry |
| Wikipedia | - | Not in registry |

### Implementation

```python
def _map_ligand_cross_references(self, db_links: list[dict]) -> CrossReferences:
    refs_dict = {}
    for link in db_links:
        db = link.get("database", "")
        acc = link.get("accession", "")
        if db == "ChEMBL Ligand":
            refs_dict["chembl"] = acc
        elif db == "DrugBank Ligand":
            refs_dict["drugbank"] = acc
        elif db == "PubChem CID":
            refs_dict["pubchem_compound"] = acc
    return CrossReferences(**refs_dict)
```

---

## R6: Target Cross-Reference Mappings

### Endpoint

```
GET /targets/{targetId}/databaseLinks
```

### Sample Response (D2 receptor, ID 215)

```json
[
  {
    "species": "Human",
    "database": "UniProtKB",
    "accession": "P14416",
    "url": "https://www.uniprot.org/uniprot/P14416"
  },
  {
    "species": "Human",
    "database": "Ensembl Gene",
    "accession": "ENSG00000149295",
    "url": "https://www.ensembl.org/Homo_sapiens/Gene/Summary?g=ENSG00000149295"
  },
  {
    "species": "Human",
    "database": "Entrez Gene",
    "accession": "1813",
    "url": "https://www.ncbi.nlm.nih.gov/gene/1813"
  },
  {
    "species": "Human",
    "database": "RefSeq Nucleotide",
    "accession": "NM_000795",
    "url": "https://www.ncbi.nlm.nih.gov/nuccore/NM_000795"
  },
  {
    "species": "Human",
    "database": "ChEMBL Target",
    "accession": "CHEMBL217",
    "url": "https://www.ebi.ac.uk/chembl/target_report_card/CHEMBL217"
  },
  {
    "species": "Human",
    "database": "OMIM",
    "accession": "126450",
    "url": "https://omim.org/entry/126450"
  },
  {
    "species": "Human",
    "database": "Orphanet",
    "accession": "ORPHA121177",
    "url": "https://www.orpha.net/consor/cgi-bin/OC_Exp.php?Expert=121177"
  },
  {
    "species": "Mouse",
    "database": "UniProtKB",
    "accession": "P61168",
    "url": "https://www.uniprot.org/uniprot/P61168"
  }
]
```

### Mapping to 22-Key Registry (Human Only)

| GtoPdb Database | Registry Key | Format Regex |
|-----------------|--------------|--------------|
| UniProtKB | `uniprot` | `^[A-Z0-9]{6,10}$` |
| Ensembl Gene | `ensembl_gene` | `^ENSG\d{11}$` |
| Entrez Gene | `entrez` | `^\d+$` |
| RefSeq Nucleotide | `refseq` | `^[NX][MR]_\d+$` |
| ChEMBL Target | `chembl` | `^CHEMBL\d+$` |
| OMIM | `omim` | `^\d{6}$` |
| Orphanet | `orphanet` | `^ORPHA:\d+$` |

### Species Filtering

Target database links include multi-species data. Implementation must filter to human:

```python
def _map_target_cross_references(self, db_links: list[dict]) -> CrossReferences:
    refs_dict = {}
    for link in db_links:
        species = link.get("species", "")
        if species != "Human":
            continue  # Skip non-human entries
        db = link.get("database", "")
        acc = link.get("accession", "")
        if db == "UniProtKB":
            refs_dict["uniprot"] = [acc]  # List per ADR-001
        elif db == "Ensembl Gene":
            refs_dict["ensembl_gene"] = acc
        elif db == "Entrez Gene":
            refs_dict["entrez"] = acc
        # ... etc
    return CrossReferences(**refs_dict)
```

---

## R7: CURIE Format Validation

### Ligand IDs

- **Source Field**: `ligandId` (integer)
- **CURIE Format**: `IUPHAR:{ligandId}`
- **Examples**: `IUPHAR:2713` (ibuprofen), `IUPHAR:4139` (aspirin)
- **Regex**: `^IUPHAR:\d+$`

### Target IDs

- **Source Field**: `targetId` (integer)
- **CURIE Format**: `IUPHAR:{targetId}`
- **Examples**: `IUPHAR:214` (D1 receptor), `IUPHAR:215` (D2 receptor)
- **Regex**: `^IUPHAR:\d+$`

### Disambiguation

Both ligands and targets use the same CURIE prefix. Tools disambiguate by:
1. Context (tool name: `get_ligand` vs `get_target`)
2. API endpoint (`/ligands/{id}` vs `/targets/{id}`)
3. Recovery hints (if 404, suggest trying the other entity type)

### Validation Pattern

```python
import re

IUPHAR_CURIE_PATTERN = re.compile(r"^IUPHAR:\d+$")

def validate_iuphar_curie(curie: str) -> bool:
    return bool(IUPHAR_CURIE_PATTERN.match(curie))

def extract_id(curie: str) -> int:
    """Extract numeric ID from IUPHAR CURIE."""
    return int(curie.replace("IUPHAR:", ""))
```

---

## R8: Response Structure Analysis

### Ligand Response Fields

```json
{
  "ligandId": 2713,
  "name": "ibuprofen",
  "type": "Synthetic organic",
  "abbreviation": "",
  "inn": "ibuprofen",
  "approvalSource": "FDA (1974), EMA (2004)",
  "approved": true,
  "whoEssential": true,
  "withdrawn": false,
  "antibacterial": false,
  "immuno": true,
  "malaria": false,
  "labelled": false,
  "radioactive": false,
  "activeDrugIds": [],
  "prodrugIds": [],
  "complexIds": [],
  "subunitIds": []
}
```

### Ligand Synonym Response

```json
[
  {"name": "Advil", "refs": []},
  {"name": "Brufen", "refs": []},
  {"name": "ibuprofen lysine", "refs": []},
  {"name": "Motrin", "refs": []},
  {"name": "Nurofen", "refs": []}
]
```

### Target Response Fields

```json
{
  "targetId": 215,
  "name": "D<sub>2</sub> receptor",
  "type": "GPCR",
  "familyIds": [20],
  "subunitIds": [],
  "complexIds": []
}
```

### HTML Stripping

Target names contain HTML subscript tags. Strip before storage:

```python
import re

def strip_html(text: str) -> str:
    """Remove HTML tags from text."""
    return re.sub(r"<[^>]+>", "", text)

# "D<sub>2</sub> receptor" -> "D2 receptor"
```

---

## R9: Error Response Formats

### HTTP Status Codes

| Code | Meaning | Agent Action |
|------|---------|--------------|
| 200 | Success | Process response |
| 204 | No Content (empty result) | Return empty PaginationEnvelope |
| 303 | Redirect | Follow Location header |
| 404 | Not Found | Return ENTITY_NOT_FOUND |
| 500+ | Server Error | Return UPSTREAM_ERROR |

### Error Mapping

```python
async def handle_response(self, response: httpx.Response, curie: str) -> ErrorEnvelope | None:
    if response.status_code == 200:
        return None  # Success
    if response.status_code == 204:
        return None  # Empty result, not an error
    if response.status_code == 404:
        return ErrorEnvelope(
            error=ErrorDetail(
                code=ErrorCode.ENTITY_NOT_FOUND,
                message=f"Entity '{curie}' not found in GtoPdb",
                recovery_hint="Verify the IUPHAR ID or use search tools to find valid identifiers",
                invalid_input=curie,
            )
        )
    # All other errors
    return ErrorEnvelope(
        error=ErrorDetail(
            code=ErrorCode.UPSTREAM_ERROR,
            message=f"GtoPdb API error: {response.status_code}",
            recovery_hint="GtoPdb may be temporarily unavailable. Retry later.",
        )
    )
```

### No Structured Error Body

GtoPdb does not return JSON error bodies. Errors must be inferred from HTTP status codes.

---

## Summary

### Key Decisions

| Decision | Value | Rationale |
|----------|-------|-----------|
| Rate Limit | 1 req/s | Conservative for community resource |
| Pagination | Client-side | API returns all results |
| CURIE Format | `IUPHAR:\d+` | Matches both ligands and targets |
| Species Filter | Human only | Prioritize human cross-references |
| HTML Handling | Strip tags | Clean target names |

### Cross-Reference Coverage

| Entity Type | Databases Mapped |
|-------------|------------------|
| Ligand | ChEMBL, DrugBank, PubChem |
| Target | UniProt, Ensembl, Entrez, RefSeq, ChEMBL, OMIM, Orphanet |

### Validation Patterns

| Field | Regex |
|-------|-------|
| IUPHAR CURIE | `^IUPHAR:\d+$` |
| ChEMBL | `^CHEMBL\d+$` |
| DrugBank | `^DB\d{5}$` |
| UniProt | `^[A-Z0-9]{6,10}$` |
| Ensembl Gene | `^ENSG\d{11}$` |
