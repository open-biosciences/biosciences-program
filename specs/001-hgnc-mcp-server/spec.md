# Feature Specification: HGNC MCP Server

**Feature Branch**: `001-hgnc-mcp-server`
**Created**: 2025-12-21
**Status**: Draft
**Input**: User description: "Build the HGNC MCP Server as the foundation of the Life Sciences project"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Fuzzy Gene Search (Priority: P1)

An AI agent needs to find genes related to a user's natural language query. The agent
sends a search term (e.g., "breast cancer", "BRCA", or "tumor suppressor") and receives
a ranked list of candidate genes with enough information to select the correct one.

**Why this priority**: This is the entry point for all gene-related queries. Without fuzzy
search, agents cannot discover gene identifiers and the entire Fuzzy-to-Fact workflow fails.

**Independent Test**: Can be fully tested by sending gene names, synonyms, and disease terms
to the search endpoint and verifying ranked results are returned with identifiers.

**Acceptance Scenarios**:

1. **Given** an AI agent with no prior gene knowledge, **When** searching for "BRCA1",
   **Then** receive a ranked list of candidates where BRCA1 (HGNC:1100) is the top result
   with its official symbol, name, and HGNC ID.

2. **Given** an AI agent searching for a disease term, **When** searching for "breast cancer
   susceptibility", **Then** receive candidates including BRCA1, BRCA2, and other relevant
   genes with scores indicating relevance.

3. **Given** an AI agent searching with a typo, **When** searching for "BRCA" (incomplete),
   **Then** receive fuzzy matches including BRCA1, BRCA2, and related genes rather than
   an error.

4. **Given** an AI agent with limited context budget, **When** searching with `slim=True`,
   **Then** receive only id, name, and score for each candidate (~20 tokens per entity
   instead of ~115).

---

### User Story 2 - Strict Gene Lookup (Priority: P1)

An AI agent has resolved a gene identifier from fuzzy search and needs the complete gene
record with all cross-references for downstream reasoning. The agent passes a validated
CURIE (e.g., "HGNC:1100") and receives the full Agentic Biolink entity.

**Why this priority**: This completes the Fuzzy-to-Fact protocol. Without strict lookup,
agents cannot retrieve authoritative gene data for verification and triangulation.

**Independent Test**: Can be fully tested by passing known HGNC IDs and verifying complete
records with cross-references are returned.

**Acceptance Scenarios**:

1. **Given** a valid HGNC CURIE "HGNC:1100", **When** calling get_gene, **Then** receive
   the complete gene record for BRCA1 including cross_references to Ensembl, UniProt,
   Entrez, and other databases.

2. **Given** a raw gene symbol "BRCA1" (not a CURIE), **When** calling get_gene, **Then**
   receive an UNRESOLVED_ENTITY error with recovery_hint pointing to search_genes.

3. **Given** a valid CURIE format but non-existent ID "HGNC:9999999", **When** calling
   get_gene, **Then** receive an ENTITY_NOT_FOUND error with appropriate message.

4. **Given** a gene with multiple protein isoforms, **When** retrieving via get_gene,
   **Then** receive cross_references.uniprot as a list containing all isoform accessions.

---

### User Story 3 - Error Recovery (Priority: P2)

An AI agent receives structured errors that allow it to self-correct without human
intervention. Errors include actionable recovery hints that guide the agent to the
correct workflow.

**Why this priority**: Robust error handling enables autonomous agent operation and
prevents cascading failures in multi-step reasoning chains.

**Independent Test**: Can be tested by triggering each error condition and verifying
the error envelope contains the correct code, message, and recovery_hint.

**Acceptance Scenarios**:

1. **Given** a raw string passed to a strict tool, **When** the error is returned,
   **Then** the error code is "UNRESOLVED_ENTITY" and recovery_hint says "Call
   search_genes to resolve the identifier first".

2. **Given** an ambiguous search returning >100 results, **When** the warning is
   issued, **Then** the response includes "AMBIGUOUS_QUERY" indicator with hint to
   refine search terms.

3. **Given** HGNC API is temporarily unavailable, **When** the error is returned,
   **Then** the error code is "UPSTREAM_ERROR" with appropriate message about
   service availability.

---

### User Story 4 - Pagination for Large Results (Priority: P2)

An AI agent searching for common terms receives paginated results to prevent context
window overflow. The agent can traverse pages using opaque cursors.

**Why this priority**: Essential for handling broad searches without flooding the
agent's context window.

**Independent Test**: Can be tested by performing broad searches and verifying
pagination envelope structure with working cursor traversal.

**Acceptance Scenarios**:

1. **Given** a search returning 150 results, **When** requesting the first page,
   **Then** receive exactly 50 items with pagination.cursor for the next page.

2. **Given** a valid cursor from a previous response, **When** passing it to the
   next request, **Then** receive the next 50 items and a new cursor (or null if
   final page).

3. **Given** the final page of results, **When** pagination is returned, **Then**
   cursor is null indicating no more pages.

---

### Edge Cases

- **Empty query**: Return AMBIGUOUS_QUERY error with hint to provide at least 2
  characters.
- **Rate limiting**: Return RATE_LIMITED error with retry hint and exponential
  backoff recommendation.
- **Special characters**: Normalize queries like "TP53/P53" and search both variants,
  returning unified results.
- **Missing cross-references**: Omit keys entirely from cross_references (never null
  or empty string).
- **Deprecated genes**: Include status field indicating gene state (approved, withdrawn).
- **Unicode in symbols**: Handle Greek letters and special characters in gene symbols.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST provide a fuzzy search tool (`search_genes`) that accepts
  natural language queries and returns ranked gene candidates.

- **FR-002**: System MUST provide a strict lookup tool (`get_gene`) that accepts
  ONLY validated HGNC CURIEs in the format "HGNC:\d+".

- **FR-003**: All list responses MUST use the Canonical Pagination Envelope with
  `items`, `pagination.cursor`, `pagination.total_count`, and `pagination.page_size`.

- **FR-004**: All error responses MUST use the Canonical Error Envelope with
  `success: false`, `error.code`, `error.message`, `error.recovery_hint`, and
  `error.invalid_input`.

- **FR-005**: All entity responses MUST include a `cross_references` object using
  the 22-key registry defined in ADR-001 Appendix A.

- **FR-006**: Cross-reference keys with no value MUST be omitted entirely (never
  set to null or empty string).

- **FR-007**: All batch/list tools MUST support a `slim=True` parameter that returns
  only id, name, and score fields (~20 tokens per entity).

- **FR-008**: Strict tools MUST return UNRESOLVED_ENTITY error when passed raw
  strings instead of valid CURIEs.

- **FR-009**: All network I/O MUST be asynchronous using non-blocking patterns.

- **FR-010**: Default page size MUST be 50 items; maximum page size MUST NOT exceed
  100 items.

### Key Entities

- **Gene**: Represents an HGNC gene record with official symbol, full name, HGNC ID,
  chromosomal location, gene group, locus type, and status (approved/withdrawn).
  Related to multiple cross-references across external databases.

- **CrossReferences**: A flattened object containing identifiers from external
  databases (ensembl_gene, uniprot, entrez, chembl, etc.). Keys are omitted if
  no reference exists for that database.

- **SearchCandidate**: A lightweight representation of a gene for search results,
  containing id, name, symbol, and relevance score. Can be expanded to full Gene
  via strict lookup.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Agents can resolve gene names to valid CURIEs in under 2 tool calls
  (one fuzzy search, one strict lookup).

- **SC-002**: 95% of fuzzy searches for known gene symbols return the correct gene
  in the top 3 results.

- **SC-003**: Error responses enable agent self-recovery without human intervention
  in 90% of recoverable error cases.

- **SC-004**: Slim mode reduces token usage by at least 80% compared to full mode
  for batch operations.

- **SC-005**: Pagination enables traversal of result sets up to 10,000 items without
  context overflow.

- **SC-006**: Cross-reference data enables triangulation verification with at least
  3 external databases per gene entity.

- **SC-007**: System handles 100 concurrent agent requests without degradation.

## Assumptions

The following reasonable defaults have been applied:

1. **HGNC API**: Uses the public HGNC REST API at genenames.org with no API key
   required for basic usage.

2. **Rate Limiting**: Standard polite client behavior with exponential backoff;
   no aggressive rate limiting expected for reasonable usage patterns.

3. **Data Freshness**: Gene data is fetched live from HGNC; no local caching in
   initial implementation (can be added as optimization).

4. **Error Recovery**: Agents are expected to handle errors programmatically;
   recovery hints are machine-actionable, not human-readable.

5. **Character Encoding**: All text is UTF-8; gene symbols may contain Greek
   letters and special characters.
