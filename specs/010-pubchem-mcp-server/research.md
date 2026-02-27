# Research: PubChem MCP Server

**Phase**: 0 (Outline & Research)
**Date**: 2026-01-01
**Purpose**: Resolve technical unknowns before design phase

## R1: PUG REST API Structure

**Question**: What is the structure of PubChem's PUG REST API and which endpoints should we use?

**Decision**: Use PubChem PUG REST API with compound domain

**API Overview**:
- **Base URL**: `https://pubchem.ncbi.nlm.nih.gov/rest/pug`
- **Protocol**: REST (no authentication required)
- **Documentation**: https://pubchem.ncbi.nlm.nih.gov/docs/pug-rest

**URL Structure**:
```
{prolog}/{domain}/{namespace}/{identifiers}/{operation}/{output}
```

Components:
- `prolog`: Base URL (https://pubchem.ncbi.nlm.nih.gov/rest/pug)
- `domain`: compound, substance, or assay
- `namespace`: cid, name, smiles, inchi, inchikey, formula
- `identifiers`: Specific identifier value(s)
- `operation`: record, property, synonyms, xrefs, description
- `output`: JSON, XML, CSV, TXT, SDF

**Key Endpoints for Compound Operations**:

| Operation | Endpoint | Example |
|-----------|----------|---------|
| Search by name | `/compound/name/{name}/cids/JSON` | `/compound/name/aspirin/cids/JSON` |
| Get properties | `/compound/cid/{cid}/property/{props}/JSON` | `/compound/cid/2244/property/MolecularFormula/JSON` |
| Get synonyms | `/compound/cid/{cid}/synonyms/JSON` | `/compound/cid/2244/synonyms/JSON` |
| Get cross-refs | `/compound/cid/{cid}/xrefs/RegistryID/JSON` | `/compound/cid/2244/xrefs/RegistryID/JSON` |
| Get record | `/compound/cid/{cid}/record/JSON` | `/compound/cid/2244/record/JSON` |

**Response Examples**:

Search by name:
```json
{
  "IdentifierList": {
    "CID": [2244, 123456, 789012]
  }
}
```

Get properties:
```json
{
  "PropertyTable": {
    "Properties": [
      {
        "CID": 2244,
        "MolecularFormula": "C9H8O4",
        "MolecularWeight": 180.16,
        "CanonicalSMILES": "CC(=O)OC1=CC=CC=C1C(=O)O",
        "IsomericSMILES": "CC(=O)OC1=CC=CC=C1C(=O)O",
        "InChI": "InChI=1S/C9H8O4/c1-6(10)13-8-5-3-2-4-7(8)9(11)12/h2-5H,1H3,(H,11,12)",
        "InChIKey": "BSYNRYMUTXBXSQ-UHFFFAOYSA-N",
        "IUPACName": "2-acetoxybenzoic acid",
        "Title": "Aspirin"
      }
    ]
  }
}
```

Get synonyms:
```json
{
  "InformationList": {
    "Information": [
      {
        "CID": 2244,
        "Synonym": [
          "aspirin",
          "acetylsalicylic acid",
          "2-acetoxybenzoic acid",
          "CHEMBL25",
          "DB00945",
          ...
        ]
      }
    ]
  }
}
```

**References**:
- [PUG REST Documentation](https://pubchem.ncbi.nlm.nih.gov/docs/pug-rest)
- [PUG REST Tutorial](https://pubchem.ncbi.nlm.nih.gov/docs/pug-rest-tutorial)
- [IUPAC FAIR Chemistry Cookbook](https://iupac.github.io/WFChemCookbook/datasources/pubchem_pugrest.html)

---

## R2: Rate Limiting Strategy

**Question**: What rate limits does PubChem enforce and how should we implement them?

**Decision**: Implement dual rate limiting (5 req/s + 400 req/min) with exponential backoff

**PubChem Rate Limits** (from official documentation):
- **Per-second limit**: No more than 5 requests per second
- **Per-minute limit**: No more than 400 requests per minute
- **Running time limit**: No more than 300 seconds of running time per minute
- **Dynamic throttling**: Limits may be lowered during high load periods

**HTTP Response Headers**:
PubChem returns throttling information in headers:
- `X-Throttling-Control`: Current system load state and per-user limits

**Error Response** (HTTP 503):
```json
{
  "Fault": {
    "Code": "PUGREST.ServerBusy",
    "Message": "Server is busy. Try again later.",
    "Details": ["..."]
  }
}
```

**Implementation Strategy**:

```python
class PubChemClient(LifeSciencesClient):
    # Rate limiting configuration (Constitution v1.1.0 MANDATORY)
    RATE_LIMIT_PER_SECOND = 5    # Max 5 req/s
    RATE_LIMIT_PER_MINUTE = 400  # Max 400 req/min

    # Exponential backoff configuration
    MAX_RETRIES = 3
    BASE_DELAY = 1.0  # seconds
    MAX_DELAY = 60.0  # seconds

    def __init__(self):
        super().__init__(base_url="https://pubchem.ncbi.nlm.nih.gov/rest/pug")

        # Per-second rate limiter
        self._rate_lock = asyncio.Lock()
        self._last_request_time: float = 0.0

        # Per-minute rate limiter (rolling window)
        self._minute_window: deque[float] = deque(maxlen=400)

    async def _rate_limited_get(self, url: str) -> httpx.Response:
        """Execute GET with dual rate limiting."""
        async with self._rate_lock:
            now = time.monotonic()

            # Per-second limit (200ms minimum interval for 5 req/s)
            min_interval = 1.0 / self.RATE_LIMIT_PER_SECOND
            elapsed = now - self._last_request_time
            if elapsed < min_interval:
                await asyncio.sleep(min_interval - elapsed)

            # Per-minute limit (rolling window)
            while len(self._minute_window) >= self.RATE_LIMIT_PER_MINUTE:
                oldest = self._minute_window[0]
                if now - oldest < 60.0:
                    await asyncio.sleep(60.0 - (now - oldest))
                    now = time.monotonic()
                else:
                    self._minute_window.popleft()

            self._last_request_time = time.monotonic()
            self._minute_window.append(self._last_request_time)

        # Execute request with backoff
        return await self._request_with_backoff(url)
```

**Thundering Herd Prevention**:
- Acquire lock before timing check
- Re-check timing after acquiring lock
- Add jitter to backoff delay

**References**:
- [PubChem Programmatic Access](https://pubchem.ncbi.nlm.nih.gov/docs/programmatic-access)
- Kim et al. (2018) "An update on PUG-REST" Nucleic Acids Research

---

## R3: Cross-Reference Mapping

**Question**: How do we extract cross-references to ChEMBL, DrugBank, and other databases?

**Decision**: Extract from synonyms array and xrefs endpoint

**Cross-Reference Sources in PubChem**:

1. **Synonyms** (most reliable for ChEMBL/DrugBank):
   - ChEMBL IDs appear as synonyms (e.g., "CHEMBL25")
   - DrugBank IDs appear as synonyms (e.g., "DB00945")

2. **XRefs endpoint** (`/compound/cid/{cid}/xrefs/RegistryID/JSON`):
   - Returns registry IDs from depositors
   - Contains some UniProt, PDB references

3. **Description/Annotation** (via PUG-View, more complex):
   - Additional cross-references in annotations

**Mapping Table** (PubChem -> 22-Key Registry):

| PubChem Source | Pattern/Source | Our Registry Key | Notes |
|----------------|----------------|------------------|-------|
| Synonyms | `CHEMBL\d+` | `chembl` | Common for drugs |
| Synonyms | `DB\d{5}` | `drugbank` | DrugBank IDs |
| xrefs/RegistryID | UniProt entries | `uniprot` | When available |
| xrefs/RegistryID | PDB entries | `pdb` | Structure refs |
| xrefs/RegistryID | ChEBI entries | `chebi` | ChEBI ontology |
| CID itself | Numeric | `pubchem_compound` | Self-reference |

**Implementation**:

```python
def _extract_cross_references(self, cid: int, synonyms: list[str],
                               xrefs: list[dict]) -> dict[str, list[str]]:
    """Extract cross-references from synonyms and xrefs."""
    refs: dict[str, list[str]] = {}

    # Self-reference
    refs["pubchem_compound"] = [str(cid)]

    # ChEMBL from synonyms
    chembl_pattern = re.compile(r'^CHEMBL(\d+)$')
    for syn in synonyms:
        match = chembl_pattern.match(syn)
        if match:
            if "chembl" not in refs:
                refs["chembl"] = []
            refs["chembl"].append(f"CHEMBL:{match.group(1)}")

    # DrugBank from synonyms
    drugbank_pattern = re.compile(r'^DB\d{5}$')
    for syn in synonyms:
        if drugbank_pattern.match(syn):
            if "drugbank" not in refs:
                refs["drugbank"] = []
            refs["drugbank"].append(syn)

    # UniProt/PDB from xrefs
    for xref in xrefs:
        source = xref.get("SourceName", "")
        registry_id = xref.get("RegistryID", "")
        if "UniProt" in source:
            if "uniprot" not in refs:
                refs["uniprot"] = []
            refs["uniprot"].append(f"UniProtKB:{registry_id}")

    # Omit empty keys (Constitution Principle III)
    return {k: v for k, v in refs.items() if v}
```

**Limitations**:
- Not all compounds have ChEMBL/DrugBank cross-references
- xrefs coverage varies by compound source
- Some cross-references may be outdated

**References**:
- [PubChem PUG REST - XRefs](https://pubchem.ncbi.nlm.nih.gov/docs/pug-rest#section=XRefs)
- [Drug ID Mapping GitHub](https://github.com/iit-Demokritos/drug_id_mapping)

---

## R4: CURIE Format Validation

**Question**: What CURIE format should we use for PubChem compound IDs?

**Decision**: Use `PubChem:CID{number}` format

**Rationale**:
- PubChem uses numeric CIDs (Compound IDs)
- CURIE prefix clarifies source database
- "CID" suffix distinguishes from SID (Substance ID)
- Matches ADR-001 key registry format (`pubchem_compound`)

**Regex Pattern**: `^PubChem:CID\d+$`

**Validation Examples**:

| Input | Valid? | Reason |
|-------|--------|--------|
| `PubChem:CID2244` | PASS | Correct format (aspirin) |
| `PubChem:CID123456789` | PASS | Large CID valid |
| `PubChem:CID1` | PASS | Single digit valid |
| `CID2244` | FAIL | Missing "PubChem:" prefix |
| `2244` | FAIL | Raw number, no CURIE |
| `PubChem:2244` | FAIL | Missing "CID" |
| `PubChem:SID2244` | FAIL | SID not CID |
| `pubchem:CID2244` | FAIL | Lowercase prefix |

**Error Response** (UNRESOLVED_ENTITY):
```json
{
  "success": false,
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "Invalid PubChem CURIE format: 'CID2244'",
    "recovery_hint": "Use format 'PubChem:CID{number}' (e.g., 'PubChem:CID2244'). Call search_compounds to find valid CIDs.",
    "invalid_input": "CID2244"
  }
}
```

**Implementation**:
```python
import re

PUBCHEM_CURIE_PATTERN = re.compile(r"^PubChem:CID\d+$")

def _validate_pubchem_curie(self, pubchem_id: str) -> int | ErrorEnvelope:
    """Validate PubChem CURIE and extract numeric CID."""
    if not PUBCHEM_CURIE_PATTERN.match(pubchem_id):
        return ErrorEnvelope(
            error=ErrorDetail(
                code=ErrorCode.UNRESOLVED_ENTITY,
                message=f"Invalid PubChem CURIE format: '{pubchem_id}'",
                recovery_hint="Use format 'PubChem:CID{number}' (e.g., 'PubChem:CID2244'). Call search_compounds to find valid CIDs.",
                invalid_input=pubchem_id,
            )
        )
    # Extract numeric CID after "PubChem:CID"
    return int(pubchem_id.split(":CID")[1])
```

---

## R5: Property Retrieval

**Question**: Which properties should we retrieve and in what format?

**Decision**: Single API call for all required properties

**Available Properties** (PUG REST):

| Property | Description | Example Value |
|----------|-------------|---------------|
| `MolecularFormula` | Chemical formula | `C9H8O4` |
| `MolecularWeight` | Molecular weight (g/mol) | `180.16` |
| `CanonicalSMILES` | Unique SMILES string | `CC(=O)OC1=CC=CC=C1C(=O)O` |
| `IsomericSMILES` | SMILES with stereochemistry | `CC(=O)OC1=CC=CC=C1C(=O)O` |
| `InChI` | International Chemical Identifier | `InChI=1S/C9H8O4/...` |
| `InChIKey` | Hashed InChI (27 chars) | `BSYNRYMUTXBXSQ-UHFFFAOYSA-N` |
| `IUPACName` | IUPAC systematic name | `2-acetoxybenzoic acid` |
| `Title` | Common name | `Aspirin` |
| `XLogP` | Partition coefficient | `1.2` |
| `ExactMass` | Exact mass | `180.042259` |
| `TPSA` | Topological polar surface area | `63.6` |
| `Complexity` | Molecular complexity | `212` |
| `HBondDonorCount` | H-bond donor count | `1` |
| `HBondAcceptorCount` | H-bond acceptor count | `4` |
| `RotatableBondCount` | Rotatable bond count | `3` |
| `HeavyAtomCount` | Non-hydrogen atoms | `13` |
| `Charge` | Formal charge | `0` |

**Selected Properties** (for Compound model):
```
MolecularFormula,MolecularWeight,CanonicalSMILES,IsomericSMILES,InChI,InChIKey,IUPACName,Title
```

**API Call**:
```
GET /compound/cid/2244/property/MolecularFormula,MolecularWeight,CanonicalSMILES,IsomericSMILES,InChI,InChIKey,IUPACName,Title/JSON
```

**Response**:
```json
{
  "PropertyTable": {
    "Properties": [
      {
        "CID": 2244,
        "MolecularFormula": "C9H8O4",
        "MolecularWeight": 180.16,
        "CanonicalSMILES": "CC(=O)OC1=CC=CC=C1C(=O)O",
        "IsomericSMILES": "CC(=O)OC1=CC=CC=C1C(=O)O",
        "InChI": "InChI=1S/C9H8O4/c1-6(10)13-8-5-3-2-4-7(8)9(11)12/h2-5H,1H3,(H,11,12)",
        "InChIKey": "BSYNRYMUTXBXSQ-UHFFFAOYSA-N",
        "IUPACName": "2-acetoxybenzoic acid",
        "Title": "Aspirin"
      }
    ]
  }
}
```

**Mapping to Our Model**:

| PubChem Field | Our Field | Notes |
|---------------|-----------|-------|
| `CID` | `id` | Convert to CURIE `PubChem:CID{n}` |
| `Title` | `name` | Common name |
| `IUPACName` | `iupac_name` | Systematic name |
| `MolecularFormula` | `molecular_formula` | As-is |
| `MolecularWeight` | `molecular_weight` | In Daltons (g/mol) |
| `CanonicalSMILES` | `canonical_smiles` | As-is |
| `IsomericSMILES` | `isomeric_smiles` | As-is |
| `InChI` | `inchi` | As-is |
| `InChIKey` | `inchikey` | As-is |

---

## R6: Error Handling

**Question**: How do we map PubChem errors to our canonical error codes?

**Decision**: Map HTTP status codes to canonical ErrorEnvelope

**PubChem Error Responses**:

HTTP 400 - Bad Request:
```json
{
  "Fault": {
    "Code": "PUGREST.BadRequest",
    "Message": "Unable to standardize the given structure",
    "Details": ["..."]
  }
}
```

HTTP 404 - Not Found:
```json
{
  "Fault": {
    "Code": "PUGREST.NotFound",
    "Message": "No CID found",
    "Details": ["..."]
  }
}
```

HTTP 503 - Server Busy:
```json
{
  "Fault": {
    "Code": "PUGREST.ServerBusy",
    "Message": "Server is busy. Try again later.",
    "Details": ["..."]
  }
}
```

**Error Code Mapping**:

| HTTP Status | PubChem Code | Our Error Code | Recovery Hint |
|-------------|--------------|----------------|---------------|
| 400 | PUGREST.BadRequest | `UNRESOLVED_ENTITY` | "Check CURIE format. Use 'PubChem:CID{number}'" |
| 404 | PUGREST.NotFound | `ENTITY_NOT_FOUND` | "CID not found. Verify from search_compounds results." |
| 503 | PUGREST.ServerBusy | `RATE_LIMITED` | "PubChem rate limit exceeded. Wait {backoff}s before retry." |
| 500 | Server Error | `UPSTREAM_ERROR` | "PubChem API error. Retry in 60 seconds." |
| 502 | Bad Gateway | `UPSTREAM_ERROR` | "PubChem temporarily unavailable. Retry in 60 seconds." |
| Timeout | - | `UPSTREAM_ERROR` | "Request timeout. Simplify query or try later." |

**Implementation**:

```python
def _map_http_error(self, status: int, response_text: str,
                    input_value: str | None = None) -> ErrorEnvelope:
    """Map HTTP error to canonical ErrorEnvelope."""

    if status == 400:
        return ErrorEnvelope(
            error=ErrorDetail(
                code=ErrorCode.UNRESOLVED_ENTITY,
                message=f"PubChem bad request: {response_text[:100]}",
                recovery_hint="Check CURIE format. Use 'PubChem:CID{number}' (e.g., 'PubChem:CID2244')",
                invalid_input=input_value,
            )
        )

    if status == 404:
        return ErrorEnvelope(
            error=ErrorDetail(
                code=ErrorCode.ENTITY_NOT_FOUND,
                message=f"PubChem compound not found: {input_value}",
                recovery_hint="CID not found in PubChem. Verify CURIE from search_compounds results.",
                invalid_input=input_value,
            )
        )

    if status == 503:
        return ErrorEnvelope(
            error=ErrorDetail(
                code=ErrorCode.RATE_LIMITED,
                message="PubChem rate limit exceeded",
                recovery_hint="PubChem API rate limit exceeded. Wait 60s before retrying.",
                invalid_input=input_value,
            )
        )

    # Default: upstream error
    return ErrorEnvelope(
        error=ErrorDetail(
            code=ErrorCode.UPSTREAM_ERROR,
            message=f"PubChem API error (HTTP {status})",
            recovery_hint="PubChem API temporarily unavailable. Retry in 60 seconds.",
            invalid_input=input_value,
        )
    )
```

---

## R7: Search Result Ranking

**Question**: How do we rank fuzzy search results?

**Decision**: Use position-based linear decay scoring

**PubChem Search Behavior**:
- `/compound/name/{name}/cids/JSON` returns CIDs matching the name
- Results are ordered by relevance (exact matches first)
- May return multiple CIDs for synonyms

**Ranking Strategy**:

1. Query PubChem for CIDs matching the search term
2. For each CID, retrieve Title (common name)
3. Calculate score using linear decay:
   - Position 0 (first result): score = 1.0
   - Each subsequent position: score -= 0.05
   - Minimum score: 0.1

**Implementation**:

```python
async def search_compounds(self, query: str, slim: bool = False,
                          cursor: str | None = None,
                          page_size: int = 50) -> PaginationEnvelope | ErrorEnvelope:
    """Fuzzy search for compounds by name."""

    # Validate query
    if len(query) < 2:
        return ErrorEnvelope(
            error=ErrorDetail(
                code=ErrorCode.AMBIGUOUS_QUERY,
                message=f"Query too short: '{query}'",
                recovery_hint="Minimum query length is 2 characters",
                invalid_input=query,
            )
        )

    # Search for CIDs by name
    url = f"{self.PUBCHEM_BASE_URL}/compound/name/{quote(query)}/cids/JSON"
    response = await self._rate_limited_get(url)

    if response.status_code == 404:
        # No results
        return PaginationEnvelope.create(
            items=[],
            cursor=None,
            total_count=0,
            page_size=page_size,
        )

    cids = response.json()["IdentifierList"]["CID"]

    # Apply pagination
    offset = self._decode_cursor(cursor)
    paginated_cids = cids[offset:offset + page_size]

    # Get properties for paginated CIDs
    candidates = []
    for i, cid in enumerate(paginated_cids):
        props = await self._get_properties(cid)
        score = max(0.1, 1.0 - ((offset + i) * 0.05))

        candidates.append(PubChemSearchCandidate(
            id=f"PubChem:CID{cid}",
            name=props.get("Title"),
            molecular_formula=props.get("MolecularFormula"),
            score=score,
        ))

    # Calculate next cursor
    next_offset = offset + page_size
    next_cursor = self._encode_cursor(next_offset) if next_offset < len(cids) else None

    return PaginationEnvelope.create(
        items=candidates,
        cursor=next_cursor,
        total_count=len(cids),
        page_size=page_size,
    )
```

**Score Examples**:

| Position | Score | Description |
|----------|-------|-------------|
| 0 | 1.00 | Best match (exact name match) |
| 1 | 0.95 | Second best |
| 5 | 0.75 | Fifth result |
| 10 | 0.50 | Tenth result |
| 18+ | 0.10 | Minimum score |

---

## Summary

All technical unknowns resolved. Key decisions:

1. **API Structure**: Use PUG REST with compound domain, native async httpx
2. **Rate Limiting**: Dual limits (5 req/s + 400 req/min) with exponential backoff
3. **Cross-References**: Extract from synonyms (ChEMBL, DrugBank) and xrefs endpoint
4. **CURIE Validation**: `^PubChem:CID\d+$` regex pattern
5. **Property Retrieval**: Single API call for 8 core properties
6. **Error Mapping**: Map HTTP status to 5 canonical error codes
7. **Search Ranking**: Linear decay scoring from position

**Next Phase**: Design data models and contracts (Phase 1)

---

## References

- [PubChem PUG REST Documentation](https://pubchem.ncbi.nlm.nih.gov/docs/pug-rest)
- [PubChem PUG REST Tutorial](https://pubchem.ncbi.nlm.nih.gov/docs/pug-rest-tutorial)
- [PubChem Programmatic Access](https://pubchem.ncbi.nlm.nih.gov/docs/programmatic-access)
- [IUPAC FAIR Chemistry Cookbook - PubChem](https://iupac.github.io/WFChemCookbook/datasources/pubchem_pugrest.html)
- [PubChemPy Documentation](https://pubchempy.readthedocs.io/)
- Kim et al. (2018) "An update on PUG-REST" Nucleic Acids Research 46(W1):W563-W570
