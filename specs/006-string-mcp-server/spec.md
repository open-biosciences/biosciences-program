# Feature Specification: STRING MCP Server

**Feature Branch**: `feature/006-string-007-biogrid`
**Created**: 2025-12-23
**Status**: Complete (10 integration tests passing)
**Updated**: 2025-12-23
**Linear**: AGE-74
**Input**: User description: "Build the STRING MCP Server for protein-protein interaction data"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Fuzzy Protein Search (Priority: P1)

As a systems biology researcher, I need to find proteins by gene symbol or protein name so that I can identify STRING identifiers for interaction network queries.

**Why this priority**: This is the entry point for all interaction network workflows. Without fuzzy search, users cannot obtain STRING protein identifiers needed for network queries.

**Independent Test**: Can be fully tested by querying "TP53" and verifying that ranked candidates include STRING:9606.ENSP00000269305 with the correct species. Delivers immediate value by enabling protein discovery.

**Acceptance Scenarios**:

1. **Given** a researcher knows a gene symbol, **When** they search for "TP53", **Then** the system returns ranked candidates including the STRING identifier with species information
2. **Given** a researcher enters a partial protein name, **When** they search with minimum 2 characters, **Then** the system returns relevant matches from the STRING database
3. **Given** a researcher wants human proteins only, **When** they specify species=9606, **Then** results are filtered to Homo sapiens
4. **Given** a researcher enters ambiguous input, **When** the query matches multiple proteins, **Then** the system returns top matches ranked by relevance with scores
5. **Given** a researcher provides invalid input, **When** they search with 1 character, **Then** the system returns error code AMBIGUOUS_QUERY with recovery hint

---

### User Story 2 - Protein Interaction Network (Priority: P2)

As a systems biology researcher, once I have identified the correct STRING identifier, I need to retrieve the protein interaction network including evidence scores for each interaction type so that I can analyze functional relationships.

**Why this priority**: This is the core functionality of STRING - providing evidence-weighted interaction networks. The 7 evidence channels (neighborhood, fusion, phyletic, co-expression, experimental, database, textmining) are essential for filtering by evidence type.

**Independent Test**: Can be tested by directly calling with "STRING:9606.ENSP00000269305" (TP53) and verifying interaction network with MDM2, ATM, BRCA1 including evidence scores. Delivers value by providing authoritative interaction data.

**Acceptance Scenarios**:

1. **Given** a researcher has a STRING identifier from search results, **When** they request interactions for "STRING:9606.ENSP00000269305", **Then** the system returns the interaction network with partner proteins
2. **Given** a researcher needs high-confidence interactions only, **When** they set required_score=700, **Then** only interactions with combined score >= 0.7 are returned
3. **Given** a researcher needs evidence breakdown, **When** they retrieve interactions, **Then** each interaction includes 7 evidence channel scores (nscore, fscore, pscore, ascore, escore, dscore, tscore)
4. **Given** a researcher provides an invalid STRING CURIE format, **When** they request "9606.ENSP00000269305" (missing prefix), **Then** the system returns error code UNRESOLVED_ENTITY with recovery hint
5. **Given** a researcher requests a non-existent protein, **When** they lookup an invalid STRING ID, **Then** the system returns error code ENTITY_NOT_FOUND
6. **Given** a researcher wants to limit results, **When** they set limit=10, **Then** only the top 10 interactions by combined score are returned

---

### User Story 3 - Network Visualization (Priority: P3)

As a systems biology researcher, I need to generate network visualization URLs so that I can view and share interaction networks using STRING's web interface.

**Why this priority**: While not essential for data retrieval, visualization URLs provide immediate value for presentations, publications, and exploratory analysis.

**Independent Test**: Can be tested by generating a URL for TP53 and verifying it opens a valid STRING network visualization page.

**Acceptance Scenarios**:

1. **Given** a researcher has a list of proteins, **When** they request a network image URL, **Then** the system returns a valid STRING network visualization URL
2. **Given** a researcher needs a specific image format, **When** they specify image type (SVG, PNG), **Then** the URL generates the correct format
3. **Given** a researcher needs high resolution, **When** they adjust resolution parameters, **Then** the generated URL reflects the settings

---

### User Story 4 - Error Recovery and Resilience (Priority: P4)

As a systems biology researcher, when I encounter errors during STRING queries (rate limits, invalid input), I need clear error messages with actionable recovery hints.

**Why this priority**: Good error handling enables self-service resolution and reduces user frustration.

**Acceptance Scenarios**:

1. **Given** a researcher provides an invalid CURIE, **When** they attempt lookup without "STRING:" prefix, **Then** the system returns error code UNRESOLVED_ENTITY with message and recovery hint
2. **Given** a researcher sends too many requests, **When** they exceed STRING's rate limit (1 req/sec), **Then** the system handles backoff internally
3. **Given** the STRING API is temporarily unavailable, **When** a query fails, **Then** the system returns error code UPSTREAM_ERROR with recovery hint
4. **Given** a researcher provides a query that's too short, **When** they search with 1 character, **Then** the system returns error code AMBIGUOUS_QUERY with recovery hint

---

### Edge Cases

- What happens when a protein has no known interaction partners in STRING?
- How does the system handle proteins from non-model organisms with sparse data?
- What happens when STRING API returns interactions without all evidence channel scores?
- How does the system handle very large networks (proteins with 500+ interaction partners)?

## Requirements *(mandatory)*

### Functional Requirements

**Architecture & Integration**

- **FR-001**: System MUST implement STRINGClient as a subclass of LifeSciencesClient to inherit connection pooling, rate limiting, and error handling patterns (ADR-001 Section 2)
- **FR-002**: All HTTP calls MUST use httpx async client with native asyncio (ADR-001 Section 2)
- **FR-003**: Client MUST implement context manager protocol for proper resource cleanup

**Fuzzy-to-Fact Protocol**

- **FR-004**: System MUST implement `search_proteins(query)` tool returning ranked InteractionSearchCandidate results with STRING CURIEs (ADR-001 Section 3)
- **FR-005**: System MUST implement `get_interactions(string_id)` tool accepting ONLY resolved STRING CURIEs in format `STRING:TAXID.ENSPNNNNN` (ADR-001 Section 3)
- **FR-006**: System MUST implement `get_network_image_url(identifiers)` utility for visualization

**Evidence Scores**

- **FR-007**: All interactions MUST include 7 evidence channel scores:
  - nscore: Neighborhood (gene proximity)
  - fscore: Gene fusion events
  - pscore: Phyletic profiles (co-occurrence)
  - ascore: Co-expression (mRNA correlation)
  - escore: Experimental (physical binding)
  - dscore: Database (curated knowledge)
  - tscore: Textmining (literature mentions)
- **FR-008**: Combined score MUST be provided as normalized 0-1 value

**Envelopes & Schema**

- **FR-009**: Fuzzy search MUST return results wrapped in PaginationEnvelope with cursor support (ADR-001 Section 8)
- **FR-010**: All errors MUST use ErrorEnvelope with code, message, recovery_hint, and invalid_input fields (ADR-001 Section 8)
- **FR-011**: Interaction records MUST include cross_references following the 22-key registry (ADR-001 Section 4)

### Non-Functional Requirements

- **NFR-001**: Rate limit: 1 request per second to STRING API
- **NFR-002**: Response time: P95 < 3 seconds for network queries
- **NFR-003**: Max interactions per request: 10,000

## Technical Implementation

### API Details

- **Base URL**: `https://string-db.org/api`
- **Protocol**: REST (JSON output)
- **Authentication**: None required (public API)
- **Rate Limit**: 1 req/sec (conservative estimate)

### Endpoints Used

| Endpoint | Purpose |
|----------|---------|
| `/json/resolve` | Resolve gene symbols to STRING identifiers |
| `/json/network` | Get interaction network with evidence scores |
| `/json/get_link` | Generate network visualization URL |

### Models Implemented

- `InteractionSearchCandidate`: Fuzzy search result with STRING CURIE
- `EvidenceScores`: 7-channel evidence breakdown
- `Interaction`: Single protein-protein interaction with evidence
- `InteractionCrossReferences`: Cross-database links (UniProt, Ensembl, HGNC)
- `InteractionNetwork`: Complete network response with interactions list

### CURIE Format

```
STRING:9606.ENSP00000269305
       │    └─ Ensembl Protein ID
       └─ NCBI Taxonomy ID (9606 = Homo sapiens)
```

## Test Coverage

| Test | Status | Description |
|------|--------|-------------|
| test_search_proteins_tp53 | PASS | Fuzzy search for TP53 |
| test_search_proteins_ranking | PASS | Verify ranking by relevance |
| test_search_proteins_validation | PASS | Query validation errors |
| test_get_interactions_tp53 | PASS | Network for TP53 |
| test_get_interactions_evidence | PASS | Evidence score breakdown |
| test_get_interactions_score_filter | PASS | required_score filtering |
| test_get_interactions_limit | PASS | Interaction count limiting |
| test_get_interactions_validation | PASS | CURIE validation errors |
| test_get_network_image_url | PASS | Visualization URL generation |
| test_error_recovery | PASS | Error envelope format |

**Total: 10/10 integration tests passing**

## Cross-References

- **ADR-001**: Architecture Decision Record for Life Sciences MCP
- **Constitution**: Principles I (Async-First), II (Fuzzy-to-Fact), III (Schema Determinism)
- **Linear Issue**: AGE-74 (STRING Tier 1)
