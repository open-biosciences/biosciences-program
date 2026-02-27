# NCBI E-utilities API Research

**Date**: 2026-01-01
**Purpose**: Phase 0 research for NCBI Entrez MCP Server implementation

## R1: NCBI E-utilities API Endpoints

### Base URL
```
https://eutils.ncbi.nlm.nih.gov/entrez/eutils
```

### Key Endpoints

**1. ESearch - Database Search**
```
GET /esearch.fcgi?db={database}&term={query}&retmax={count}&retstart={offset}&retmode=json
```

Returns list of IDs matching the query. Supports pagination via `retstart` and `retmax`.

Example:
```bash
curl "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=gene&term=BRCA1[gene]+AND+human[organism]&retmax=10&retmode=json"
```

Response structure:
```json
{
  "header": { "type": "esearch", "version": "0.3" },
  "esearchresult": {
    "count": "2",
    "retmax": "10",
    "retstart": "0",
    "idlist": ["672", "100506742"],
    "translationset": [...],
    "translationstack": [...],
    "querytranslation": "BRCA1[gene] AND human[organism]"
  }
}
```

**2. ESummary - Document Summaries**
```
GET /esummary.fcgi?db={database}&id={id_list}&retmode=json
```

Returns document summaries (lightweight data). Good for search candidates.

Example:
```bash
curl "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esummary.fcgi?db=gene&id=7157,672&retmode=json"
```

Response structure:
```json
{
  "result": {
    "uids": ["7157", "672"],
    "7157": {
      "uid": "7157",
      "name": "TP53",
      "description": "tumor protein p53",
      "organism": { "scientificname": "Homo sapiens", "commonname": "human", "taxid": 9606 },
      "geneticsource": "genomic",
      "maplocation": "17p13.1",
      "chromosome": "17",
      "otheraliases": "BCC7, LFS1, P53, TRP53",
      "otherdesignations": "cellular tumor antigen p53; antigen NY-CO-13; mutant tumor protein 53; ...",
      "summary": "This gene encodes a tumor suppressor protein..."
    },
    "672": { ... }
  }
}
```

**3. EFetch - Full Records**
```
GET /efetch.fcgi?db={database}&id={id_list}&rettype={type}&retmode={mode}
```

Returns full records. For Gene database, returns XML (no JSON available).

Example:
```bash
curl "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi?db=gene&id=7157&retmode=xml"
```

**4. ELink - Related Records**
```
GET /elink.fcgi?dbfrom={source_db}&db={target_db}&id={id_list}&retmode=json
```

Finds related records across databases. Use for gene -> pubmed links.

Example:
```bash
curl "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/elink.fcgi?dbfrom=gene&db=pubmed&id=7157&retmode=json"
```

Response structure:
```json
{
  "linksets": [{
    "dbfrom": "gene",
    "ids": [{"value": "7157"}],
    "linksetdbs": [{
      "dbto": "pubmed",
      "linkname": "gene_pubmed",
      "links": ["39804234", "39804163", "39803982", ...]
    }]
  }]
}
```

### Authentication

**Decision**: Optional NCBI_API_KEY for higher rate limits
**Rationale**:
- Without API key: 3 requests/second
- With API key: 10 requests/second
- API key is free and recommended for production use

**API Key Parameter**:
```
&api_key={NCBI_API_KEY}
```

---

## R2: esearch + esummary Two-Step Pattern

### Why Two Steps?

NCBI E-utilities uses a two-step pattern:
1. **esearch**: Find matching IDs (cheap, fast)
2. **esummary/efetch**: Get data for those IDs (more expensive)

This prevents expensive full-text searches and enables efficient pagination.

### Implementation Strategy

**For Fuzzy Search (`search_genes`)**:
1. Call `esearch` with query -> returns ID list
2. Call `esummary` with IDs -> returns candidate data
3. Wrap in PaginationEnvelope

```python
async def search_genes(query: str, page_size: int = 50, cursor: str | None = None):
    # Step 1: Search for IDs
    offset = decode_cursor(cursor) or 0
    esearch_response = await self._esearch(
        db="gene",
        term=query,
        retmax=page_size,
        retstart=offset
    )

    id_list = esearch_response["esearchresult"]["idlist"]
    total_count = int(esearch_response["esearchresult"]["count"])

    if not id_list:
        return PaginationEnvelope.create(items=[], cursor=None, total_count=0, page_size=page_size)

    # Step 2: Get summaries for IDs
    esummary_response = await self._esummary(db="gene", ids=id_list)

    # Step 3: Transform to search candidates
    candidates = [
        GeneSearchCandidate(
            id=f"NCBIGene:{uid}",
            symbol=data["name"],
            name=data["description"],
            description=data.get("otherdesignations", ""),
            organism=data["organism"]["scientificname"],
            score=1.0 - (i * 0.05)  # Position-based scoring
        )
        for i, (uid, data) in enumerate(esummary_response["result"].items())
        if uid != "uids"
    ]

    # Step 4: Pagination
    next_cursor = None
    if offset + page_size < total_count:
        next_cursor = encode_cursor(offset + page_size)

    return PaginationEnvelope.create(
        items=candidates,
        cursor=next_cursor,
        total_count=total_count,
        page_size=page_size
    )
```

**For Strict Lookup (`get_gene`)**:
1. Validate CURIE format
2. Call `efetch` with single ID -> returns full XML
3. Parse XML to extract gene data
4. Return EntrezGene entity

```python
async def get_gene(entrez_id: str) -> EntrezGene | ErrorEnvelope:
    # Validate CURIE
    if not NCBI_GENE_CURIE_PATTERN.match(entrez_id):
        return ErrorEnvelope.unresolved_entity(entrez_id)

    numeric_id = entrez_id.replace("NCBIGene:", "")

    # Fetch full record (XML)
    xml_response = await self._efetch(db="gene", ids=[numeric_id], retmode="xml")

    # Parse XML
    gene_data = self._parse_gene_xml(xml_response)

    if not gene_data:
        return ErrorEnvelope.entity_not_found(entrez_id)

    return EntrezGene(
        id=entrez_id,
        symbol=gene_data["symbol"],
        name=gene_data["name"],
        # ... rest of fields
        cross_references=self._build_cross_references(gene_data["xrefs"])
    )
```

### Pagination

**esearch pagination parameters**:
- `retstart`: Offset (0-based)
- `retmax`: Page size (max 10000)
- `count`: Total results (in response)

**Cursor encoding**:
```python
import base64
import json

def encode_cursor(offset: int) -> str:
    return base64.b64encode(json.dumps({"offset": offset}).encode()).decode()

def decode_cursor(cursor: str | None) -> int | None:
    if not cursor:
        return None
    try:
        return json.loads(base64.b64decode(cursor).decode())["offset"]
    except (ValueError, KeyError):
        return None
```

---

## R3: XML Parsing with defusedxml

### Security Rationale

NCBI E-utilities return XML for `efetch` on the Gene database. Standard `xml.etree.ElementTree` is vulnerable to XML External Entity (XXE) attacks. Use `defusedxml` for secure parsing.

### Installation

```bash
uv add defusedxml
```

### Gene XML Structure

```xml
<?xml version="1.0"?>
<Entrezgene-Set>
  <Entrezgene>
    <Entrezgene_track-info>
      <Gene-track>
        <Gene-track_geneid>7157</Gene-track_geneid>
        <Gene-track_status value="live">0</Gene-track_status>
      </Gene-track>
    </Entrezgene_track-info>

    <Entrezgene_gene>
      <Gene-ref>
        <Gene-ref_locus>TP53</Gene-ref_locus>
        <Gene-ref_desc>tumor protein p53</Gene-ref_desc>
        <Gene-ref_maploc>17p13.1</Gene-ref_maploc>
        <Gene-ref_syn>
          <Gene-ref_syn_E>P53</Gene-ref_syn_E>
          <Gene-ref_syn_E>TRP53</Gene-ref_syn_E>
          <Gene-ref_syn_E>LFS1</Gene-ref_syn_E>
        </Gene-ref_syn>
      </Gene-ref>
    </Entrezgene_gene>

    <Entrezgene_summary>This gene encodes a tumor suppressor protein...</Entrezgene_summary>

    <Entrezgene_xref>
      <Dbtag>
        <Dbtag_db>HGNC</Dbtag_db>
        <Dbtag_tag>
          <Object-id>
            <Object-id_id>11998</Object-id_id>
          </Object-id>
        </Dbtag_tag>
      </Dbtag>
      <Dbtag>
        <Dbtag_db>Ensembl</Dbtag_db>
        <Dbtag_tag>
          <Object-id>
            <Object-id_str>ENSG00000141510</Object-id_str>
          </Object-id>
        </Dbtag_tag>
      </Dbtag>
      <Dbtag>
        <Dbtag_db>MIM</Dbtag_db>
        <Dbtag_tag>
          <Object-id>
            <Object-id_id>191170</Object-id_id>
          </Object-id>
        </Dbtag_tag>
      </Dbtag>
    </Entrezgene_xref>

    <Entrezgene_source>
      <BioSource>
        <BioSource_org>
          <Org-ref>
            <Org-ref_taxname>Homo sapiens</Org-ref_taxname>
            <Org-ref_common>human</Org-ref_common>
            <Org-ref_db>
              <Dbtag>
                <Dbtag_db>taxon</Dbtag_db>
                <Dbtag_tag>
                  <Object-id>
                    <Object-id_id>9606</Object-id_id>
                  </Object-id>
                </Dbtag_tag>
              </Dbtag>
            </Org-ref_db>
          </Org-ref>
        </BioSource_org>
      </BioSource>
    </Entrezgene_source>
  </Entrezgene>
</Entrezgene-Set>
```

### XML Parsing Implementation

```python
import defusedxml.ElementTree as ET
from typing import Any

def parse_gene_xml(xml_content: str) -> dict[str, Any] | None:
    """Parse NCBI Gene XML to extract gene data.

    Uses defusedxml for security (prevents XXE attacks).
    """
    root = ET.fromstring(xml_content)
    gene_elem = root.find(".//Entrezgene")

    if gene_elem is None:
        return None

    # Extract gene ID
    gene_id = gene_elem.findtext(".//Gene-track_geneid")

    # Extract gene reference data
    gene_ref = gene_elem.find(".//Gene-ref")
    symbol = gene_ref.findtext("Gene-ref_locus") if gene_ref is not None else None
    name = gene_ref.findtext("Gene-ref_desc") if gene_ref is not None else None
    map_location = gene_ref.findtext("Gene-ref_maploc") if gene_ref is not None else None

    # Extract aliases
    aliases = []
    syn_elem = gene_elem.find(".//Gene-ref_syn")
    if syn_elem is not None:
        aliases = [e.text for e in syn_elem.findall("Gene-ref_syn_E") if e.text]

    # Extract summary
    summary = gene_elem.findtext(".//Entrezgene_summary")

    # Extract organism
    organism = gene_elem.findtext(".//Org-ref_taxname")

    # Extract cross-references
    xrefs = {}
    for dbtag in gene_elem.findall(".//Entrezgene_xref/Dbtag"):
        db_name = dbtag.findtext("Dbtag_db")
        obj_id = dbtag.findtext(".//Object-id_id") or dbtag.findtext(".//Object-id_str")
        if db_name and obj_id:
            xrefs[db_name] = obj_id

    return {
        "gene_id": gene_id,
        "symbol": symbol,
        "name": name,
        "map_location": map_location,
        "aliases": aliases,
        "summary": summary,
        "organism": organism,
        "xrefs": xrefs
    }
```

### Cross-Reference Mapping

| NCBI Dbtag_db | Our 22-Key Registry | Example Value | Notes |
|---------------|---------------------|---------------|-------|
| `HGNC` | `hgnc` | `HGNC:11998` | Add HGNC: prefix |
| `Ensembl` | `ensembl_gene` | `ENSG00000141510` | Already in correct format |
| `MIM` | `omim` | `191170` | Numeric only |
| `UniProtKB/Swiss-Prot` | `uniprot` | `UniProtKB:P04637` | Add UniProtKB: prefix |
| `RefSeq` | `refseq` | `NM_000546` | Multiple entries possible |

---

## R4: Rate Limit Policy

### Official NCBI Rate Limits

From [NCBI E-utilities documentation](https://www.ncbi.nlm.nih.gov/books/NBK25497/):

| Condition | Rate Limit |
|-----------|------------|
| Without API key | 3 requests/second |
| With API key | 10 requests/second |
| Per IP address | Applies to all requests from same IP |

### API Key

**How to obtain**:
1. Create NCBI account: https://www.ncbi.nlm.nih.gov/account/
2. Go to Settings: https://www.ncbi.nlm.nih.gov/account/settings/
3. Generate API key (free, no approval needed)

**Usage**:
```
&api_key=YOUR_API_KEY
```

### Rate Limiting Implementation

```python
import asyncio
import os
import time

class EntrezClient(LifeSciencesClient):
    ENTREZ_BASE_URL = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils"
    MAX_RETRIES = 3

    def __init__(self):
        super().__init__(base_url=self.ENTREZ_BASE_URL)
        self.api_key = os.getenv("NCBI_API_KEY")
        # 10 req/s with key, 3 req/s without
        self.rate_limit_delay = 0.1 if self.api_key else 0.333
        self._last_request_time: float = 0.0
        self._lock = asyncio.Lock()

    async def _rate_limited_get(self, path: str, params: dict | None = None) -> httpx.Response:
        """Make a rate-limited GET request with exponential backoff."""
        if params is None:
            params = {}

        # Add API key if available
        if self.api_key:
            params["api_key"] = self.api_key

        # Rate limiting with lock
        async with self._lock:
            now = time.monotonic()
            elapsed = now - self._last_request_time
            if elapsed < self.rate_limit_delay:
                await asyncio.sleep(self.rate_limit_delay - elapsed)

            response = await self._get(path, params=params)
            self._last_request_time = time.monotonic()

        # Exponential backoff on 429
        for attempt in range(self.MAX_RETRIES):
            if response.status_code != 429:
                break

            retry_after = response.headers.get("Retry-After")
            wait_time = int(retry_after) if retry_after else (2 ** attempt)

            await asyncio.sleep(wait_time)

            async with self._lock:
                # Re-check timing (thundering herd prevention)
                now = time.monotonic()
                elapsed = now - self._last_request_time
                if elapsed < self.rate_limit_delay:
                    await asyncio.sleep(self.rate_limit_delay - elapsed)

                response = await self._get(path, params=params)
                self._last_request_time = time.monotonic()

        return response
```

### Error Response Handling

| HTTP Status | NCBI Error | Our ErrorEnvelope Code | Recovery Hint |
|-------------|------------|------------------------|---------------|
| 400 | Bad request | `AMBIGUOUS_QUERY` | "Refine query or check syntax" |
| 404 | Not found | `ENTITY_NOT_FOUND` | "Verify the Entrez Gene ID" |
| 429 | Too many requests | `RATE_LIMITED` | "Wait and retry, or add NCBI_API_KEY" |
| 500/502/503 | Server error | `UPSTREAM_ERROR` | "NCBI E-utilities temporarily unavailable" |

---

## R5: elink for PubMed Links

### Endpoint

```
GET /elink.fcgi?dbfrom=gene&db=pubmed&id={gene_id}&retmode=json&linkname=gene_pubmed
```

### Response Structure

```json
{
  "header": { "type": "elink", "version": "0.3" },
  "linksets": [{
    "dbfrom": "gene",
    "ids": [{"value": "7157"}],
    "linksetdbs": [{
      "dbto": "pubmed",
      "linkname": "gene_pubmed",
      "links": ["39804234", "39804163", "39803982", ...]
    }]
  }]
}
```

### Implementation

```python
async def get_pubmed_links(entrez_id: str, limit: int = 10) -> list[str] | ErrorEnvelope:
    """Get PubMed IDs associated with a gene."""
    # Validate CURIE
    if not NCBI_GENE_CURIE_PATTERN.match(entrez_id):
        return ErrorEnvelope.unresolved_entity(entrez_id)

    numeric_id = entrez_id.replace("NCBIGene:", "")

    response = await self._rate_limited_get(
        "/elink.fcgi",
        params={
            "dbfrom": "gene",
            "db": "pubmed",
            "id": numeric_id,
            "retmode": "json",
            "linkname": "gene_pubmed"
        }
    )

    if response.status_code != 200:
        return ErrorEnvelope.upstream_error(response.status_code)

    data = response.json()

    # Extract PubMed IDs
    linksets = data.get("linksets", [])
    if not linksets:
        return []

    linksetdbs = linksets[0].get("linksetdbs", [])
    if not linksetdbs:
        return []

    pmids = linksetdbs[0].get("links", [])

    # Apply limit
    return pmids[:limit]
```

### Link Types Available

| linkname | Description |
|----------|-------------|
| `gene_pubmed` | All PubMed citations for a gene |
| `gene_pubmed_rif` | PubMed citations with GeneRIF (gene reference into function) |
| `gene_pubmed_pmc_nucleotide` | PMC articles with nucleotide sequences |

For MVP, use `gene_pubmed` (most comprehensive).

---

## R6: CURIE Format Validation

### Observed Entrez Gene ID Patterns

**Format**: Numeric integers only (no prefixes in raw form)
```
7157    # TP53
672     # BRCA1
1017    # CDK2
10000   # AKT3
```

### CURIE Format

**Decision**: `NCBIGene:{numeric_id}`
**Example**: `NCBIGene:7157`

**Rationale**:
- Follows Biolink/CURIE convention
- Distinguishes from HGNC, Ensembl, UniProt IDs
- Matches common biomedical ontology usage

### Validation Regex

```python
import re

NCBI_GENE_CURIE_PATTERN = re.compile(r'^NCBIGene:\d+$')
```

**Validation examples**:
```python
NCBI_GENE_CURIE_PATTERN.match("NCBIGene:7157")      # Valid
NCBI_GENE_CURIE_PATTERN.match("NCBIGene:672")       # Valid
NCBI_GENE_CURIE_PATTERN.match("7157")               # Invalid (no prefix)
NCBI_GENE_CURIE_PATTERN.match("NCBI:7157")          # Invalid (wrong prefix)
NCBI_GENE_CURIE_PATTERN.match("NCBIGene:TP53")      # Invalid (not numeric)
```

### Pydantic Validator

```python
from pydantic import BaseModel, field_validator

class EntrezGene(BaseModel):
    id: str

    @field_validator("id")
    @classmethod
    def validate_curie_format(cls, v: str) -> str:
        if not NCBI_GENE_CURIE_PATTERN.match(v):
            raise ValueError(
                f"Invalid NCBIGene CURIE format. Expected 'NCBIGene:NNNNN' "
                f"where N is a numeric digit, got: {v}"
            )
        return v
```

---

## R7: Error Response Formats

### Test Cases

**Invalid query** (empty):
```bash
curl "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?db=gene&term=&retmode=json"
```
Response: Returns empty result set (not an error)

**Not found** (non-existent ID):
```bash
curl "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esummary.fcgi?db=gene&id=999999999999&retmode=json"
```
Response: Returns "error" field in result:
```json
{
  "result": {
    "uids": ["999999999999"],
    "999999999999": {
      "error": "cannot get document summary"
    }
  }
}
```

**Rate limit exceeded**:
Response: HTTP 429 Too Many Requests
Headers: `Retry-After: N`

### Error Mapping Table

| Condition | HTTP Status | Our ErrorEnvelope Code | Recovery Hint |
|-----------|-------------|------------------------|---------------|
| Query too short | N/A (validation) | `AMBIGUOUS_QUERY` | "Provide at least 2 characters" |
| Invalid CURIE format | N/A (validation) | `UNRESOLVED_ENTITY` | "Call search_genes to resolve entity first" |
| Gene not found | 200 with error field | `ENTITY_NOT_FOUND` | "Verify the NCBIGene ID or search by gene symbol" |
| Rate limit | 429 | `RATE_LIMITED` | "Wait {retry_after}s. Add NCBI_API_KEY for 10 req/s" |
| Server error | 500/502/503 | `UPSTREAM_ERROR` | "NCBI E-utilities temporarily unavailable" |

---

## Summary of Decisions

| Research Task | Decision | Rationale |
|---------------|----------|-----------|
| **R1: API Endpoints** | esearch + esummary for search, efetch for full records, elink for pubmed | Standard E-utilities pattern |
| **R2: Two-Step Pattern** | esearch -> esummary/efetch | Required by NCBI API design |
| **R3: XML Parsing** | defusedxml for security | Prevents XXE attacks |
| **R4: Rate Limiting** | 3 req/s default, 10 req/s with API key | NCBI official policy |
| **R5: PubMed Links** | elink with gene_pubmed linkname | Most comprehensive citation coverage |
| **R6: CURIE Format** | `NCBIGene:\d+` regex | Follows Biolink/CURIE convention |
| **R7: Error Responses** | Map to canonical ErrorEnvelope codes | Consistent with ADR-001 |

---

## Next Steps

With research complete, proceed to Phase 1:
1. Create data-model.md with EntrezGene, GeneSearchCandidate, PubMedLink models
2. Create tool contracts (search_genes.yaml, get_gene.yaml, get_pubmed_links.yaml)
3. Create quickstart.md with usage examples

All research findings documented here should be referenced during implementation.
