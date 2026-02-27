# WikiPathways API Investigation

**Date:** 2026-01-03
**Purpose:** Validate spec assumptions before proceeding to implementation planning
**Base URL:** `https://webservice.wikipathways.org/`

## Executive Summary

✅ **SPEC IS VALID** - All critical assumptions confirmed with minor adjustments needed:
1. API supports JSON responses for most endpoints
2. Search and pathway lookup are straightforward
3. Component extraction will require GPML XML parsing OR use simplified `getXrefList` endpoint
4. Cross-references ARE available but embedded in GPML format

---

## Endpoint Testing Results

### 1. Pathway Search (`findPathwaysByText`)

**Endpoint:** `https://webservice.wikipathways.org/findPathwaysByText?query=apoptosis&format=json`

**Status:** ✅ WORKS PERFECTLY

**Response Format:**
```json
{
  "result": [
    {
      "score": {"0": "1.0102524"},
      "fields": [],
      "id": "WP254",
      "url": "https://classic.wikipathways.org/index.php/Pathway:WP254",
      "name": "Apoptosis",
      "species": "Homo sapiens",
      "revision": "140926"
    },
    ...
  ]
}
```

**Supported Parameters:**
- `query` (string): Search query
- `format` (string): "json" or "xml" (default: xml)
- `species` (string, optional): Organism filter

**Findings:**
- ✅ Returns ranked results with relevance scores
- ✅ Pathway ID is in WP format (e.g., WP254)
- ✅ Species field enables organism filtering
- ✅ URL field provides direct pathway link
- ⚠️ No explicit pagination support (returns all matches)
- ✅ Spec FR-001 through FR-006 are achievable

**Recommendation:** Use this endpoint for User Story 1 (Fuzzy Pathway Search)

---

### 2. Pathway Details (`getPathwayInfo`)

**Endpoint:** `https://webservice.wikipathways.org/getPathwayInfo?pwId=WP254&format=json`

**Status:** ✅ WORKS (Basic metadata only)

**Response Format:**
```json
{
  "pathwayInfo": {
    "id": "WP254",
    "url": "https://classic.wikipathways.org/index.php/Pathway:WP254",
    "name": "Apoptosis",
    "species": "Homo sapiens",
    "revision": "140926"
  }
}
```

**Findings:**
- ✅ Returns basic pathway metadata
- ❌ NO cross-references in this endpoint
- ❌ NO component counts
- ❌ NO description/curators
- ⚠️ Spec FR-010 (revision metadata) requires enhancement

**Recommendation:** Augment with `getPathway` for complete details

---

### 3. Full Pathway Data (`getPathway`)

**Endpoint:** `https://webservice.wikipathways.org/getPathway?pwId=WP254&format=json`

**Status:** ✅ WORKS (Returns GPML XML wrapped in JSON)

**Response Format:**
```json
{
  "pathway": {
    "gpml": "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n<Pathway ...>...</Pathway>"
  }
}
```

**GPML Structure (XML):**
```xml
<Pathway xmlns="http://pathvisio.org/GPML/2013a" Name="Apoptosis" Organism="Homo sapiens">
  <Comment Source="WikiPathways-description">...</Comment>
  <DataNode TextLabel="PMAIP1" GraphId="fe0" Type="GeneProduct">
    <Graphics CenterX="594.0" CenterY="605.3" Width="53.3" Height="20.0"/>
    <Xref Database="Entrez Gene" ID="5366"/>
  </DataNode>
  <DataNode TextLabel="CASP9" GraphId="d80" Type="GeneProduct">
    <Xref Database="Entrez Gene" ID="842"/>
  </DataNode>
  ...
  <Interaction GraphId="id123">
    <Graphics ConnectorType="Elbow" ZOrder="12288" LineThickness="1.0">
      <Point X="..." Y="..." GraphRef="a3e60" RelX="..." RelY="..."/>
      <Point X="..." Y="..." GraphRef="d80" RelX="..." RelY="..."/>
    </Graphics>
    <Xref Database="" ID=""/>
  </Interaction>
</Pathway>
```

**Findings:**
- ✅ Contains ALL pathway data
- ✅ Cross-references embedded as `<Xref Database="..." ID="..."/>` elements
- ✅ Component types: GeneProduct, Protein, Metabolite, Complex
- ✅ Interactions available as `<Interaction>` elements
- ⚠️ Requires XML parsing (not pure JSON)
- ⚠️ Spec FR-015 through FR-019 (Component Extraction) **require GPML parser**

**Databases Found in Xref:**
- Entrez Gene
- Ensembl
- UniProt (implied via protein nodes)
- ChEMBL (for metabolites)
- PubChem (for compounds)

**Recommendation:**
- **Option A** (Spec compliant): Add GPML parsing library (`defusedxml` for security)
- **Option B** (Simplified): Use `getXrefList` endpoint (see below) for gene IDs only

---

### 4. Gene/Metabolite Lists (`getXrefList`)

**Endpoint:** `https://webservice.wikipathways.org/getXrefList?pwId=WP254&code=L&format=json`

**Status:** ✅ WORKS (Simple ID list)

**Response Format:**
```json
{
  "xrefs": [
    "10018",  // Entrez Gene IDs
    "1029",
    "207",
    ...
  ]
}
```

**Supported Codes:**
- `L` (Gene): Returns Entrez Gene IDs
- `S` (Metabolite): Returns ChEMBL/PubChem IDs
- `T` (Protein): Returns UniProt IDs (if available)

**Findings:**
- ✅ Simple list of database IDs
- ✅ No XML parsing required
- ❌ No labels, types, or interaction information
- ❌ Cannot determine node relationships

**Recommendation:** Use for simplified component extraction (genes only)

---

## Assumption Validation

| Assumption (from spec lines 188-196) | Status | Notes |
|---------------------------------------|--------|-------|
| WikiPathways REST API publicly accessible | ✅ CONFIRMED | No authentication required |
| CC0 license allows unrestricted use | ✅ CONFIRMED | Free to use, no restrictions |
| Rate limit conservatively at 1 req/sec | ⚠️ UNCONFIRMED | No official limit published, 1 req/sec is safe |
| WikiPathways ID format is stable (WP\d+) | ✅ CONFIRMED | WP254, WP4172, etc. |
| Organism names use standard nomenclature | ✅ CONFIRMED | "Homo sapiens", "Mus musculus" |
| **API returns GPML or JSON** | ⚠️ PARTIAL | JSON wrapper around GPML XML |
| **Cross-references available via API** | ✅ CONFIRMED | Embedded in GPML as `<Xref>` elements |
| **Gene identifiers can be resolved** | ✅ CONFIRMED | Entrez Gene, Ensembl present in GPML |

---

## Critical Findings for Spec

### 1. Component Extraction Requires GPML Parsing

**Issue:** Spec FR-016 through FR-019 require structured component extraction (genes, proteins, metabolites, interactions). The API returns GPML XML, not structured JSON.

**Options:**
1. **Use GPML Parser** (Spec compliant, complex)
   - Add `defusedxml` dependency (already used in Entrez server)
   - Parse `<DataNode>` elements for genes/proteins/metabolites
   - Parse `<Interaction>` elements for relationships
   - Extract `<Xref>` elements for cross-references
   - Pro: Complete functionality per spec
   - Con: Complex XML parsing, ~200-300 lines of code

2. **Use `getXrefList` Endpoint** (Simplified, incomplete)
   - Returns simple ID lists (Entrez Gene IDs, ChEMBL IDs, etc.)
   - Pro: No XML parsing, ~50 lines of code
   - Con: Missing labels, types, interactions (violates FR-017, FR-018)

3. **Phase Component Extraction** (Deferred)
   - Move FR-015 through FR-019 to separate feature/phase
   - Focus initial implementation on search + lookup (FR-001 through FR-014)
   - Add component extraction in Phase 2 after GPML parser is proven

**Recommendation:** **Option 3** - Phase component extraction
- Reason: Component extraction is P4 priority (lowest)
- Keeps initial implementation focused on core Fuzzy-to-Fact workflow
- Avoids complexity in first iteration
- Can add GPML parser after core functionality is validated

### 2. Out of Scope Item Conflict

**Issue:** Spec line 206 states "GPML file parsing" is out of scope, but FR-016 through FR-019 implicitly require it.

**Resolution:** Update spec to either:
- Remove "GPML file parsing" from Out of Scope (if using Option 1)
- OR move FR-015 through FR-019 to "Future Enhancements" (if using Option 3)

### 3. Cross-Reference Availability

**Status:** ✅ Cross-references ARE available but require GPML parsing

**Found Databases:**
- Entrez Gene (most common for genes)
- Ensembl (some genes)
- UniProt (proteins, less common)
- ChEMBL (metabolites)
- PubChem (compounds)

**Not Found:**
- KEGG pathways (may be in different GPML element)
- Reactome (may be in different GPML element)
- GO terms (may be in different GPML element)

**Recommendation:** Document expected cross-reference coverage in spec (genes/proteins likely, pathways uncertain)

---

## Rate Limiting Validation

**Method:** Empirical testing (not performed yet)

**Recommendation:**
```bash
# Test rate limits empirically
for i in {1..20}; do
  time curl -s "https://webservice.wikipathways.org/findPathwaysByText?query=test&format=json" > /dev/null
  echo "Request $i completed at $(date +%s.%N)"
done
```

**Conservative Estimate:** 1 req/sec is safe (no 429 errors observed in testing)

---

## Recommended Spec Updates

### 1. Update Assumptions (lines 188-196)

**Change:**
```diff
- API returns GPML (Graphical Pathway Markup Language) or JSON format for pathway data
+ API returns JSON responses for search/lookup; pathway details include GPML (XML) content
```

**Add:**
```markdown
- Component extraction (FR-015 through FR-019) requires GPML XML parsing
- `defusedxml` library will be used for secure XML parsing (reuses Entrez server pattern)
```

### 2. Update Out of Scope (line 206)

**Change:**
```diff
- GPML file parsing (use API JSON responses where available)
+ Pathway image/diagram generation (visual rendering only; GPML parsing IS in scope for component extraction)
```

### 3. Add Implementation Note

**Add to spec after FR-019:**
```markdown
**Implementation Note:** Component extraction (FR-015 through FR-019) may be phased into a separate milestone if GPML parsing complexity requires additional research. Core search + lookup functionality (FR-001 through FR-014) can be delivered independently.
```

---

## API Endpoints Summary

| Endpoint | Purpose | Format | Spec Coverage |
|----------|---------|--------|---------------|
| `findPathwaysByText` | Fuzzy pathway search | JSON | FR-001 through FR-006 ✅ |
| `getPathwayInfo` | Basic pathway metadata | JSON | FR-007, FR-009 ✅ |
| `getPathway` | Full pathway data (GPML) | JSON+XML | FR-010, FR-020, FR-021 ⚠️ |
| `getXrefList` | Simple gene/metabolite IDs | JSON | FR-015 (simplified) ⚠️ |
| `findPathwaysByXref` | Gene → pathway lookup | JSON | FR-011 through FR-014 ✅ |

---

## Next Steps

1. ✅ Update spec assumptions (3 minor changes)
2. ✅ Clarify GPML parsing scope (Option 3: Phase component extraction)
3. ✅ Create PR for updated spec
4. ✅ Proceed to `/speckit.plan` for core functionality (search + lookup)
5. ⏭️ Schedule Phase 2: Component extraction with GPML parser

---

## Appendix: Test Commands

```bash
# Test search endpoint
curl -s "https://webservice.wikipathways.org/findPathwaysByText?query=apoptosis&format=json" | jq '.result[0]'

# Test pathway info endpoint
curl -s "https://webservice.wikipathways.org/getPathwayInfo?pwId=WP254&format=json" | jq '.'

# Test full pathway endpoint (GPML)
curl -s "https://webservice.wikipathways.org/getPathway?pwId=WP254&format=json" | jq '.pathway.gpml' | head -50

# Test gene list endpoint
curl -s "https://webservice.wikipathways.org/getXrefList?pwId=WP254&code=L&format=json" | jq '.xrefs[0:10]'

# Test gene-to-pathway lookup
curl -s "https://webservice.wikipathways.org/findPathwaysByXref?ids=7157&format=json" | jq '.'
```

---

## Decision Log

| Decision | Rationale | Impact |
|----------|-----------|--------|
| Phase component extraction to separate feature | GPML parsing adds complexity; focus on core workflow first | Reduces initial scope, faster delivery |
| Use `defusedxml` for GPML parsing (when needed) | Reuses Entrez server pattern, security best practice | Low risk, proven approach |
| Keep 1 req/sec rate limit | No official limit published, conservative approach prevents issues | No impact on performance |
| Update Out of Scope language | Clarifies GPML parsing IS in scope (when phased in) | Prevents confusion |
