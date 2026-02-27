# WikiPathways MCP Server Research

**Research Date**: 2026-01-03
**Researcher**: Claude Code
**Purpose**: Resolve unknowns for implementing WikiPathways MCP Server specification

---

## Research Question 1: API Endpoint Structure

### Decision
Use **WikiPathways REST API** at `http://webservice.wikipathways.org` with JSON format responses.

### Endpoints Mapping

| MCP Tool | REST Endpoint | HTTP Method | Parameters |
|----------|---------------|-------------|------------|
| `search_pathways` | `/findPathwaysByText` | GET | `query` (required), `species` (optional), `format=json` |
| `get_pathway` | `/getPathwayInfo` | GET | `pwId` (required, WP format), `format=json` |
| `get_pathways_for_gene` | `/findPathwaysByXref` | GET | `ids` (required), `codes` (optional), `format=json` |
| `get_pathway_components` | `/getXrefList` | GET | `pwId` (required), `code` (required), `format=json` |

### Rationale
- **Official REST API**: WikiPathways provides a documented REST API with OpenAPI specification at `https://www.wikipathways.org/openapi/openapi.json`
- **Stable base URL**: `http://webservice.wikipathways.org` (redirects to HTTPS automatically)
- **JSON support**: All endpoints support `format=json` parameter for structured responses
- **Active maintenance**: API is actively used by R client (rWikiPathways) and Python client (pywikipathways)

### Alternatives Considered
1. **JSON API files** (`https://www.wikipathways.org/json/`):
   - **Pros**: Bulk data access, single query returns all pathways
   - **Cons**: No search/filtering, static files updated periodically, forces client-side filtering
   - **Verdict**: Not suitable for fuzzy search or dynamic queries

2. **SPARQL endpoint** (`https://sparql.wikipathways.org/`):
   - **Pros**: Powerful graph queries, flexible filtering
   - **Cons**: Complex query construction, requires SPARQL expertise, overkill for simple lookups
   - **Verdict**: Unnecessary complexity for MCP tool use case

### Code Examples

**Keyword Search (search_pathways):**
```bash
# Search for glycolysis pathways across all organisms
curl "http://webservice.wikipathways.org/findPathwaysByText?query=glycolysis&format=json"

# Search with organism filter
curl "http://webservice.wikipathways.org/findPathwaysByText?query=glycolysis&species=Homo%20sapiens&format=json"
```

**Response Format:**
```json
{
  "result": [
    {
      "score": {"0": "4.639924"},
      "fields": [],
      "id": "WP534",
      "url": "https://classic.wikipathways.org/index.php/Pathway:WP534",
      "name": "Glycolysis and gluconeogenesis",
      "species": "Homo sapiens",
      "revision": "141823"
    }
  ]
}
```

**Pathway Lookup (get_pathway):**
```bash
curl "http://webservice.wikipathways.org/getPathwayInfo?pwId=WP534&format=json"
```

**Response Format:**
```json
{
  "pathwayInfo": {
    "id": "WP534",
    "url": "https://classic.wikipathways.org/index.php/Pathway:WP534",
    "name": "Glycolysis and gluconeogenesis",
    "species": "Homo sapiens",
    "revision": "141823"
  }
}
```

**Gene Lookup (get_pathways_for_gene):**
```bash
# Find pathways containing BRCA1
curl "http://webservice.wikipathways.org/findPathwaysByXref?ids=BRCA1&format=json"

# Specify database code for NCBI Gene (L = Entrez)
curl "http://webservice.wikipathways.org/findPathwaysByXref?ids=672&codes=L&format=json"
```

**Response Format:**
```json
{
  "result": [
    {
      "score": {"0": "1.4421465"},
      "fields": {
        "graphId": {"name": "graphId", "values": ["d89"]},
        "id": {"name": "id", "values": ["373983"]},
        "id.database": {"name": "id.database", "values": ["373983:L"]},
        "x.id": {"name": "x.id", "values": ["BRCA1"]}
      },
      "id": "WP783",
      "url": "https://classic.wikipathways.org/index.php/Pathway:WP783",
      "name": "Androgen receptor signaling pathway",
      "species": "Gallus gallus",
      "revision": "117996"
    }
  ]
}
```

**Component Extraction (get_pathway_components):**
```bash
# Get NCBI Gene IDs from pathway WP534 (code 'L' = Entrez)
curl "http://webservice.wikipathways.org/getXrefList?pwId=WP534&code=L&format=json"
```

**Response Format:**
```json
{
  "xrefs": [
    "1737", "1738", "2023", "2026", "2027", "2203", "226", "229", "230",
    "2538", "25874", "2597", "2645", "2805", "2806", "2821", "3098"
  ]
}
```

---

## Research Question 2: Response Format

### Decision
Use **JSON format exclusively** for all API calls. Cross-references are available in JSON responses without GPML parsing.

### Response Structure by Endpoint

#### findPathwaysByText (search_pathways)
**Format**: JSON array wrapped in `{"result": [...]}`
**Key Fields**:
- `id`: Pathway ID (e.g., "WP534")
- `name`: Pathway title
- `species`: Scientific organism name (e.g., "Homo sapiens")
- `revision`: Revision number (string)
- `score`: Relevance score object (e.g., `{"0": "4.639924"}`)
- `url`: Pathway webpage URL
- `fields`: Additional metadata (usually empty for text search)

#### getPathwayInfo (get_pathway)
**Format**: JSON object `{"pathwayInfo": {...}}`
**Key Fields**:
- `id`, `name`, `species`, `revision`, `url` (same as search)
- **No built-in cross-references in this endpoint**

#### findPathwaysByXref (get_pathways_for_gene)
**Format**: JSON array `{"result": [...]}`
**Key Fields**:
- Same as findPathwaysByText
- `fields`: Contains matched datanode metadata:
  - `graphId`: Internal node identifier
  - `id`: Database-specific ID (e.g., "373983" for Entrez)
  - `id.database`: ID with database code (e.g., "373983:L")
  - `x.id`: Original query identifier (e.g., "BRCA1")

#### getXrefList (get_pathway_components)
**Format**: JSON object `{"xrefs": [...]}`
**Key Fields**:
- `xrefs`: Array of database-specific identifiers (strings)
- **Note**: Returns only IDs, not full entity metadata

### Cross-Reference Extraction

**Source for Cross-References**: Use **JSON API bulk files** at `https://www.wikipathways.org/json/` for comprehensive cross-references.

**Key File**: `findPathwaysByXref.json`
**Structure**:
```json
{
  "pathwayInfo": [
    {
      "id": "WP534",
      "name": "Glycolysis and gluconeogenesis",
      "species": "Homo sapiens",
      "ncbigene": "ncbigene:4191, ncbigene:7167, ...",
      "ensembl": "ensembl:ENSG00000146701, ensembl:ENSG00000111669, ...",
      "hgnc": "hgnc.symbol:MDH2, hgnc.symbol:TPI1, ...",
      "uniprot": "uniprot:U3KQ63;uniprot:A0A024R4K3, ...",
      "wikidata": "wikidata:Q27131127, ...",
      "chebi": "chebi:15903, chebi:30744, ...",
      "inchikey": "inchikey:WQZGKKKJIJFFOK-VFUOTHLCSA-N, ..."
    }
  ]
}
```

**Available Cross-Reference Keys**:
- `ncbigene`: NCBI Gene (Entrez) identifiers
- `ensembl`: Ensembl gene IDs
- `hgnc`: HGNC gene symbols (format: `hgnc.symbol:GENE`)
- `uniprot`: UniProt protein IDs (semicolon-separated for isoforms)
- `wikidata`: Wikidata entity IDs
- `chebi`: ChEBI compound IDs
- `inchikey`: InChIKey identifiers for metabolites

**Mapping to Agentic Biolink Schema (22-key)**:
| WikiPathways Key | Agentic Biolink Key | Example |
|------------------|---------------------|---------|
| `ncbigene` | `entrez` | `ncbigene:672` → `entrez` = `672` |
| `ensembl` | `ensembl_gene` | `ensembl:ENSG00000012048` → `ensembl_gene` = `ENSG00000012048` |
| `hgnc` | `hgnc` | `hgnc.symbol:BRCA1` → `hgnc` = `BRCA1` |
| `uniprot` | `uniprot` | `uniprot:P38398` → `uniprot` = `P38398` |
| `chebi` | `chembl` (if ChEMBL mapping exists) | Via secondary lookup |
| `wikidata` | (no direct mapping) | Omit or use as `wikidata` custom key |

### Rationale
- **JSON preferred over GPML**: GPML is XML-based and requires complex parsing. JSON responses provide structured data immediately.
- **Bulk JSON files for cross-references**: REST API endpoints lack comprehensive cross-reference data. The `findPathwaysByXref.json` file contains all cross-references for all pathways.
- **No GPML parsing needed**: All required data (pathway metadata, gene/protein lists, cross-references) is available in JSON format.

### Alternatives Considered
1. **GPML XML parsing**:
   - **Pros**: Complete pathway structure with visual layout, interaction details
   - **Cons**: Requires XML parsing library, complex XPath queries, performance overhead
   - **Verdict**: Unnecessary; JSON API provides sufficient data for MCP tools

2. **REST API only** (no JSON bulk files):
   - **Pros**: Simple HTTP client, no file downloads
   - **Cons**: Missing cross-reference data, requires multiple API calls per pathway
   - **Verdict**: Hybrid approach needed (REST for search/lookup, JSON files for cross-references)

### Code Examples

**Fetch bulk cross-references:**
```python
import httpx

async def get_pathway_cross_references(pathway_id: str) -> dict:
    """Fetch cross-references from JSON API bulk file."""
    url = "https://www.wikipathways.org/json/findPathwaysByXref.json"
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        data = response.json()

        # Find pathway by ID
        for pathway in data["pathwayInfo"]:
            if pathway["id"] == pathway_id:
                return {
                    "entrez": pathway.get("ncbigene", "").split(", ") if "ncbigene" in pathway else [],
                    "ensembl_gene": pathway.get("ensembl", "").split(", ") if "ensembl" in pathway else [],
                    "hgnc": [s.replace("hgnc.symbol:", "") for s in pathway.get("hgnc", "").split(", ")] if "hgnc" in pathway else [],
                    "uniprot": pathway.get("uniprot", "").split(", ") if "uniprot" in pathway else [],
                    "chebi": pathway.get("chebi", "").split(", ") if "chebi" in pathway else [],
                }
        return {}
```

---

## Research Question 3: Pagination Mechanism

### Decision
WikiPathways API **does NOT support native pagination**. Implement **client-side batching** with cursor-based pagination simulation for consistency with other MCP servers.

### Evidence
**Testing Results**:
```bash
# Test pagination parameters (not documented in OpenAPI spec)
curl "http://webservice.wikipathways.org/findPathwaysByText?query=glycolysis&format=json&limit=5"
# Response: Returns ALL results (no limit applied)

curl "http://webservice.wikipathways.org/findPathwaysByText?query=glycolysis&format=json&offset=10"
# Response: Returns ALL results from beginning (no offset applied)

# Verify result count
curl -s "http://webservice.wikipathways.org/findPathwaysByText?query=glycolysis&format=json" | jq '.result | length'
# Output: 148 (all matching pathways returned in single response)
```

### Implementation Strategy

**Client-Side Cursor Pagination**:
1. Fetch all results from REST API
2. Store results in memory (or cache temporarily)
3. Return paginated subsets using opaque cursor (base64-encoded offset)
4. Return `cursor=null` when no more results available

**Cursor Format**:
```python
import base64

def encode_cursor(offset: int) -> str:
    """Encode offset as opaque cursor."""
    return base64.b64encode(str(offset).encode()).decode()

def decode_cursor(cursor: str) -> int:
    """Decode cursor to offset."""
    return int(base64.b64decode(cursor).decode())

# Example:
# Page 1 (offset 0, page_size 50): cursor = None
# Page 2 (offset 50, page_size 50): cursor = "NTA=" (base64 of "50")
# Page 3 (offset 100, page_size 50): cursor = "MTAw" (base64 of "100")
```

**PaginationEnvelope Response**:
```json
{
  "items": [
    {"id": "WP534", "name": "Glycolysis...", "score": 4.64, ...},
    {"id": "WP5049", "name": "Glycolysis in...", "score": 4.61, ...}
  ],
  "pagination": {
    "cursor": "NTA=",
    "total_count": 148,
    "page_size": 50
  }
}
```

### Rationale
- **No native pagination**: WikiPathways API returns complete result sets (verified by testing)
- **Consistency with ADR-001**: All MCP servers must use cursor-based pagination per Architecture Decision Record
- **Performance acceptable**: Typical search queries return 50-200 results (manageable in memory)
- **Large result sets**: For queries returning 1000+ pathways, implement optional caching (per ADR-001 §7)

### Alternatives Considered
1. **Offset-based pagination**:
   - **Pros**: Simpler implementation
   - **Cons**: Violates ADR-001 §5 (cursor-based required), not future-proof if API adds pagination
   - **Verdict**: Rejected (non-compliant with architecture)

2. **Server-side pagination** (if API supported it):
   - **Pros**: Efficient memory usage, no client caching
   - **Cons**: API does not support it (verified by testing and documentation review)
   - **Verdict**: Not available

3. **No pagination** (return all results):
   - **Pros**: Simple implementation
   - **Cons**: Violates ADR-001, poor UX for large result sets, excessive token usage for LLMs
   - **Verdict**: Rejected (non-compliant)

### Code Example

```python
from typing import Optional

async def search_pathways_paginated(
    query: str,
    organism: Optional[str] = None,
    cursor: Optional[str] = None,
    page_size: int = 50
) -> dict:
    """Search pathways with client-side pagination."""
    # Fetch all results from API
    params = {"query": query, "format": "json"}
    if organism:
        params["species"] = organism

    response = await client.get(
        "http://webservice.wikipathways.org/findPathwaysByText",
        params=params
    )
    all_results = response.json()["result"]

    # Apply client-side pagination
    offset = decode_cursor(cursor) if cursor else 0
    page_results = all_results[offset:offset + page_size]

    # Calculate next cursor
    next_offset = offset + page_size
    next_cursor = encode_cursor(next_offset) if next_offset < len(all_results) else None

    return {
        "items": page_results,
        "pagination": {
            "cursor": next_cursor,
            "total_count": len(all_results),
            "page_size": page_size
        }
    }
```

---

## Research Question 4: Rate Limiting

### Decision
Implement **conservative 1 request/second rate limit** with internal throttling using `asyncio.Lock` and exponential backoff.

### Evidence
**No documented rate limits found**:
- OpenAPI specification: No rate limit headers documented
- API responses: No `X-RateLimit-*` or `Retry-After` headers observed
- Official documentation: No rate limit policy published
- Client libraries: rWikiPathways and pywikipathways do not implement rate limiting

**Testing Results**:
```bash
# Send 10 rapid requests
for i in {1..10}; do
  curl -s "http://webservice.wikipathways.org/listOrganisms?format=json" > /dev/null &
done
wait

# Result: All requests succeeded, no 429 (Too Many Requests) responses
# No rate limit headers in response
```

### Implementation Strategy

**Rate Limit Configuration**:
- **Default**: 1 request/second (1000ms delay between requests)
- **Mechanism**: `asyncio.Lock` with timestamp tracking
- **Backoff**: Exponential backoff on 503/429 responses (base 2s, max 3 retries)
- **Headers**: Monitor for `Retry-After` header (if added in future)

**Code Pattern** (from ADR-001 §6):
```python
import asyncio
from datetime import datetime, timedelta

class WikiPathwaysClient:
    def __init__(self):
        self._rate_limit_lock = asyncio.Lock()
        self._last_request_time: Optional[datetime] = None
        self._min_request_interval = timedelta(seconds=1)  # 1 req/s

    async def _enforce_rate_limit(self):
        """Enforce rate limit with thundering herd prevention."""
        async with self._rate_limit_lock:
            if self._last_request_time:
                elapsed = datetime.now() - self._last_request_time
                if elapsed < self._min_request_interval:
                    sleep_time = (self._min_request_interval - elapsed).total_seconds()
                    await asyncio.sleep(sleep_time)

            # Re-check after lock acquisition (thundering herd prevention)
            if self._last_request_time:
                elapsed = datetime.now() - self._last_request_time
                if elapsed < self._min_request_interval:
                    sleep_time = (self._min_request_interval - elapsed).total_seconds()
                    await asyncio.sleep(sleep_time)

            self._last_request_time = datetime.now()
```

**Exponential Backoff** (for 503/429 responses):
```python
async def _request_with_retry(self, method: str, url: str, **kwargs):
    """Make HTTP request with exponential backoff."""
    max_retries = 3
    base_delay = 2.0

    for attempt in range(max_retries + 1):
        try:
            await self._enforce_rate_limit()
            response = await self.client.request(method, url, **kwargs)

            if response.status_code in (429, 503):
                if attempt == max_retries:
                    raise RateLimitError("Max retries exceeded")

                # Exponential backoff
                delay = base_delay * (2 ** attempt)
                await asyncio.sleep(delay)
                continue

            return response
        except httpx.RequestError as e:
            if attempt == max_retries:
                raise UpstreamError(f"Request failed: {e}")
            await asyncio.sleep(base_delay * (2 ** attempt))

    raise UpstreamError("Request failed after retries")
```

### Rationale
- **Conservative approach**: 1 req/s is safe for public API with no documented limits
- **Follows proven pattern**: HGNC, UniProt, Ensembl servers use similar rate limiting (10-15 req/s)
- **Future-proof**: If WikiPathways adds rate limiting, our implementation is already compliant
- **Prevents abuse**: Protects public resource from excessive load

### Alternatives Considered
1. **No rate limiting**:
   - **Pros**: Simpler code, maximum performance
   - **Cons**: Risk of IP ban, unethical use of public resource, violates ADR-001 §6
   - **Verdict**: Rejected (non-compliant, unethical)

2. **Higher rate limit** (5-10 req/s):
   - **Pros**: Better performance for batch operations
   - **Cons**: No evidence API can handle this load, risk of errors
   - **Verdict**: Rejected (too aggressive without documented limits)

3. **Adaptive rate limiting** (monitor response times):
   - **Pros**: Self-tuning based on API performance
   - **Cons**: Complex implementation, unnecessary for single-user MCP server
   - **Verdict**: Over-engineering for this use case

### Code Example

**Complete Rate-Limited Request**:
```python
async def search_pathways(self, query: str, organism: Optional[str] = None) -> list:
    """Search pathways with rate limiting and error handling."""
    params = {"query": query, "format": "json"}
    if organism:
        params["species"] = organism

    try:
        response = await self._request_with_retry(
            "GET",
            "http://webservice.wikipathways.org/findPathwaysByText",
            params=params,
            timeout=10.0
        )

        data = response.json()
        return data.get("result", [])

    except httpx.TimeoutException:
        raise UpstreamError(
            code="UPSTREAM_ERROR",
            message="WikiPathways API timeout",
            recovery_hint="Retry request after 5 seconds"
        )
    except RateLimitError:
        raise ErrorEnvelope(
            code="RATE_LIMITED",
            message="WikiPathways rate limit exceeded",
            recovery_hint="Wait 60 seconds before retrying"
        )
```

---

## Research Question 5: Cross-Reference Extraction

### Decision
Use **hybrid approach**: REST API for search/lookup + JSON bulk files for cross-references. No GPML parsing required.

### Cross-Reference Storage Locations

**Option 1: JSON Bulk Files (RECOMMENDED)**
- **File**: `https://www.wikipathways.org/json/findPathwaysByXref.json`
- **Content**: All pathway cross-references in single JSON file
- **Update frequency**: Real-time (updated with every pathway change per documentation)
- **Size**: ~50-100 MB (manageable for in-memory processing)
- **Available databases**: ncbigene, ensembl, hgnc, uniprot, wikidata, chebi, inchikey

**Cross-Reference Format**:
```json
{
  "id": "WP534",
  "ncbigene": "ncbigene:4191, ncbigene:7167, ncbigene:5105, ...",
  "ensembl": "ensembl:ENSG00000146701, ensembl:ENSG00000111669, ...",
  "hgnc": "hgnc.symbol:MDH2, hgnc.symbol:TPI1, hgnc.symbol:PCK1, ...",
  "uniprot": "uniprot:U3KQ63;uniprot:A0A024R4K3, uniprot:P60174, ...",
  "chebi": "chebi:15903, chebi:30744, chebi:29052, ..."
}
```

**Option 2: getXrefList Endpoint (Per-Database)**
- **Endpoint**: `http://webservice.wikipathways.org/getXrefList?pwId=WP534&code=L&format=json`
- **Content**: Identifiers for single database (requires multiple calls)
- **Database codes**: L=NCBI Gene, S=UniProt, En=Ensembl, H=HGNC, Ce=ChEBI
- **Response**: `{"xrefs": ["4191", "7167", "5105", ...]}`

**BridgeDb System Codes** (for getXrefList):
| Database | Code | Example ID | Agentic Biolink Key |
|----------|------|------------|---------------------|
| NCBI Gene (Entrez) | `L` | `672` | `entrez` |
| UniProt | `S` | `P38398` | `uniprot` |
| Ensembl | `En` | `ENSG00000012048` | `ensembl_gene` |
| HGNC | `H` | `BRCA1` | `hgnc` |
| ChEBI | `Ce` | `15903` | (map to `chembl` if possible) |
| RefSeq | `Q` | `NM_007294` | `refseq` |
| PDB | `Pd` | `1JNX` | `pdb` |
| KEGG Genes | `Kg` | `hsa:672` | (extract gene ID) |
| KEGG Pathways | `Kp` | `hsa04110` | `kegg_pathway` |

**Full BridgeDb datasources**: https://github.com/bridgedb/datasources/blob/main/datasources.ts

### Mapping to Agentic Biolink Schema (22-key)

**Direct Mappings** (available in WikiPathways):
| WikiPathways | Agentic Biolink | Parsing Required |
|--------------|-----------------|------------------|
| `ncbigene:672` | `entrez` = `"672"` | Strip prefix |
| `ensembl:ENSG...` | `ensembl_gene` = `"ENSG..."` | Strip prefix |
| `hgnc.symbol:BRCA1` | `hgnc` = `"BRCA1"` | Strip prefix |
| `uniprot:P38398` | `uniprot` = `"P38398"` | Strip prefix, handle isoforms (`;`) |
| `chebi:15903` | (no direct mapping) | Requires ChEMBL lookup |

**Indirect Mappings** (require secondary lookup):
| Agentic Biolink Key | WikiPathways Source | Secondary Lookup Required |
|---------------------|---------------------|---------------------------|
| `chembl` | `chebi` or `inchikey` | PubChem/ChEMBL API |
| `drugbank` | (not in WikiPathways) | N/A |
| `string` | (not in WikiPathways) | Gene symbol → STRING API |
| `kegg` | `kegg` codes | Extract from BridgeDb |
| `omim` | (not in WikiPathways) | Gene symbol → OMIM API |
| `pdb` | getXrefList code `Pd` | Requires separate API call |
| `refseq` | getXrefList code `Q` | Requires separate API call |

**Omit Keys** (per ADR-001 §4 omit-if-null):
- `drugbank`, `omim`, `orphanet`, `mondo`, `efo` → Not available in WikiPathways
- `biogrid`, `stitch`, `iuphar` → Not available in WikiPathways
- `pubchem_compound`, `pubchem_substance` → Requires InChIKey → PubChem mapping

### Implementation Strategy

**get_pathway Cross-References**:
```python
async def get_pathway(self, pathway_id: str) -> dict:
    """Get pathway with cross-references from JSON bulk file."""
    # 1. Fetch pathway metadata from REST API
    response = await self.client.get(
        "http://webservice.wikipathways.org/getPathwayInfo",
        params={"pwId": pathway_id, "format": "json"}
    )
    pathway_info = response.json()["pathwayInfo"]

    # 2. Fetch cross-references from JSON bulk file
    xref_response = await self.client.get(
        "https://www.wikipathways.org/json/findPathwaysByXref.json"
    )
    xref_data = xref_response.json()

    # 3. Find pathway cross-references
    pathway_xrefs = next(
        (p for p in xref_data["pathwayInfo"] if p["id"] == pathway_id),
        {}
    )

    # 4. Map to Agentic Biolink schema
    cross_references = {}

    if "ncbigene" in pathway_xrefs:
        cross_references["entrez"] = [
            xref.replace("ncbigene:", "")
            for xref in pathway_xrefs["ncbigene"].split(", ")
        ]

    if "ensembl" in pathway_xrefs:
        cross_references["ensembl_gene"] = [
            xref.replace("ensembl:", "")
            for xref in pathway_xrefs["ensembl"].split(", ")
        ]

    if "hgnc" in pathway_xrefs:
        cross_references["hgnc"] = [
            xref.replace("hgnc.symbol:", "")
            for xref in pathway_xrefs["hgnc"].split(", ")
        ]

    if "uniprot" in pathway_xrefs:
        # Handle semicolon-separated isoforms
        uniprot_list = []
        for entry in pathway_xrefs["uniprot"].split(", "):
            primary = entry.replace("uniprot:", "").split(";")[0]
            uniprot_list.append(primary)
        cross_references["uniprot"] = uniprot_list

    # Omit empty keys
    cross_references = {k: v for k, v in cross_references.items() if v}

    return {
        **pathway_info,
        "cross_references": cross_references
    }
```

**get_pathway_components DataNode Cross-References**:
```python
async def get_pathway_components(self, pathway_id: str) -> dict:
    """Extract pathway components with per-node cross-references."""
    # 1. Get gene IDs from pathway
    gene_response = await self.client.get(
        "http://webservice.wikipathways.org/getXrefList",
        params={"pwId": pathway_id, "code": "L", "format": "json"}
    )
    gene_ids = gene_response.json()["xrefs"]

    # 2. Get corresponding HGNC symbols
    hgnc_response = await self.client.get(
        "http://webservice.wikipathways.org/getXrefList",
        params={"pwId": pathway_id, "code": "H", "format": "json"}
    )
    hgnc_symbols = hgnc_response.json()["xrefs"]

    # 3. Build DataNode list
    genes = []
    for i, gene_id in enumerate(gene_ids):
        genes.append({
            "id": f"datanode_{i}",
            "label": hgnc_symbols[i] if i < len(hgnc_symbols) else f"Gene_{gene_id}",
            "type": "Gene",
            "database": "Entrez Gene",
            "cross_references": {
                "entrez": gene_id,
                "hgnc": hgnc_symbols[i] if i < len(hgnc_symbols) else None
            }
        })

    return {
        "genes": genes,
        "proteins": [],  # Would require code='S' call
        "metabolites": [],  # Would require code='Ce' call
        "interactions": []  # Not available without GPML parsing
    }
```

### Rationale
- **JSON bulk files preferred**: Single file contains all cross-references, avoiding multiple API calls
- **Real-time updates**: WikiPathways documentation states JSON files are updated with every pathway change
- **No GPML parsing**: All required identifiers available in JSON format
- **Partial 22-key coverage**: WikiPathways provides 5-7 of 22 Agentic Biolink keys (acceptable per ADR-001 §4)

### Alternatives Considered
1. **GPML XML parsing**:
   - **Pros**: Complete interaction data, visual layout information
   - **Cons**: Complex XML parsing, performance overhead, unnecessary for identifier extraction
   - **Verdict**: Rejected (over-engineering)

2. **Multiple getXrefList calls**:
   - **Pros**: Uses official REST API, no file downloads
   - **Cons**: 5-7 API calls per pathway (L, S, En, H, Ce, Q, Pd), violates rate limiting
   - **Verdict**: Rejected (inefficient)

3. **SPARQL queries**:
   - **Pros**: Flexible graph queries, can extract relationships
   - **Cons**: Complex query construction, overkill for identifier mapping
   - **Verdict**: Rejected (unnecessary complexity)

### Code Example: Complete Cross-Reference Extraction

```python
async def _fetch_all_cross_references(self) -> dict:
    """Fetch and cache all pathway cross-references from JSON bulk file."""
    if self._xref_cache is not None:
        return self._xref_cache

    response = await self.client.get(
        "https://www.wikipathways.org/json/findPathwaysByXref.json"
    )
    data = response.json()

    # Build pathway_id → cross_references lookup
    self._xref_cache = {
        pathway["id"]: self._parse_cross_references(pathway)
        for pathway in data["pathwayInfo"]
    }

    return self._xref_cache

def _parse_cross_references(self, pathway: dict) -> dict:
    """Parse WikiPathways cross-references to Agentic Biolink format."""
    xrefs = {}

    # Entrez (NCBI Gene)
    if "ncbigene" in pathway:
        xrefs["entrez"] = [
            xref.replace("ncbigene:", "")
            for xref in pathway["ncbigene"].split(", ")
        ]

    # Ensembl
    if "ensembl" in pathway:
        xrefs["ensembl_gene"] = [
            xref.replace("ensembl:", "")
            for xref in pathway["ensembl"].split(", ")
        ]

    # HGNC
    if "hgnc" in pathway:
        xrefs["hgnc"] = [
            xref.replace("hgnc.symbol:", "")
            for xref in pathway["hgnc"].split(", ")
        ]

    # UniProt (handle isoforms)
    if "uniprot" in pathway:
        uniprot_list = []
        for entry in pathway["uniprot"].split(", "):
            # Split on semicolon, take first (canonical)
            primary = entry.replace("uniprot:", "").split(";")[0]
            uniprot_list.append(primary)
        xrefs["uniprot"] = uniprot_list

    # Omit empty keys per ADR-001 §4
    return {k: v for k, v in xrefs.items() if v}
```

---

## Summary of Decisions

| Question | Decision | Implementation |
|----------|----------|----------------|
| **API Endpoints** | WikiPathways REST API at `http://webservice.wikipathways.org` | Use documented endpoints with `format=json` |
| **Response Format** | JSON exclusively (no GPML parsing) | Parse JSON responses with httpx |
| **Pagination** | Client-side cursor-based (API has no native pagination) | Base64-encoded offset cursors |
| **Rate Limiting** | 1 req/s with exponential backoff | `asyncio.Lock` + timestamp tracking |
| **Cross-References** | Hybrid: REST API + JSON bulk files | Fetch `findPathwaysByXref.json` for cross-references |

---

## References

### Primary Sources
- [WikiPathways JSON API](https://www.wikipathways.org/json/) - Bulk JSON data files
- [WikiPathways REST API (Swagger)](https://webservice.wikipathways.org/ui/) - OpenAPI documentation
- [rWikiPathways Documentation](https://r.wikipathways.org/) - R client library
- [bioservices WikiPathway module](https://bioservices.readthedocs.io/en/main/_modules/bioservices/wikipathway.html) - Python client implementation
- [BridgeDb System Codes](https://github.com/bridgedb/datasources/blob/main/datasources.ts) - Database identifier mapping

### Related Documentation
- ADR-001 v1.2: FastMCP Project Structure (Architecture Decision Record)
- ADR-001 §3: Fuzzy-to-Fact Protocol
- ADR-001 §4: Agentic Biolink Schema (22-key cross-reference registry)
- ADR-001 §5: Cursor-Based Pagination
- ADR-001 §6: Rate Limiting and Exponential Backoff
- ADR-001 §8: Canonical Envelopes (PaginationEnvelope, ErrorEnvelope)

---

**End of Research Document**
