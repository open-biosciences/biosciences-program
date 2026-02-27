# Feature Specification: UniProt MCP Server

**Feature Branch**: `002-uniprot-mcp-server`
**Created**: 2025-12-22
**Status**: Draft
**Input**: User description: "Build the UniProt MCP Server with Fuzzy-to-Fact protocol, Agentic Biolink schema, and canonical envelopes per ADR-001"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Fuzzy Protein Search (Priority: P1)

An AI agent needs to find protein information when given an ambiguous query like "p53 tumor suppressor" or "BRCA1". The system should return ranked candidates without requiring exact accession numbers.

**Why this priority**: Core capability - agents can't query UniProt without first finding the right protein identifier. This is the entry point for all protein lookups.

**Independent Test**: Can be fully tested by querying "insulin human" and verifying multiple ranked candidates are returned with accession IDs, scores, and basic metadata.

**Acceptance Scenarios**:

1. **Given** an agent has a protein name like "p53", **When** searching for proteins, **Then** receive top 10 ranked candidates with UniProt accessions, protein names, organism, and relevance scores
2. **Given** an agent searches for "kinase", **When** results exceed 100 matches, **Then** receive paginated results with a cursor to fetch the next page
3. **Given** an agent provides a query shorter than 2 characters like "a", **When** searching, **Then** receive an error explaining the query is too ambiguous
4. **Given** an agent searches for a common protein like "actin", **When** results are returned, **Then** candidates are ranked by relevance with human proteins prioritized

---

### User Story 2 - Strict Protein Lookup (Priority: P2)

After identifying the correct protein from search results, an agent needs to retrieve complete protein data including sequence, function, cross-references to other databases, and structural information.

**Why this priority**: Essential for retrieving actionable data once the right protein is identified. Depends on US1 for discovery.

**Independent Test**: Can be tested independently by providing a known UniProt ID like "UniProtKB:P04637" (human p53) and verifying complete protein record is returned with all cross-references.

**Acceptance Scenarios**:

1. **Given** an agent has a resolved UniProt CURIE like "UniProtKB:P04637", **When** requesting protein details, **Then** receive complete protein record including gene names, function, organism, and cross-references to HGNC/Ensembl/PDB/etc
2. **Given** an agent provides an invalid CURIE format like "invalid-id", **When** requesting protein details, **Then** receive an error indicating the identifier format is unresolved
3. **Given** an agent provides a valid but non-existent CURIE like "UniProtKB:NOTFOUND", **When** requesting protein details, **Then** receive an error indicating the protein was not found
4. **Given** an agent needs minimal data for batch operations, **When** requesting with slim=true, **Then** receive core fields only (id, name, organism) reducing token usage by 80%

---

### User Story 3 - Cross-Database Integration (Priority: P3)

An agent working across multiple biological databases needs to navigate from UniProt proteins to related entities in HGNC (genes), Ensembl (genomic data), PDB (protein structures), and other databases.

**Why this priority**: Enables multi-database workflows. Less critical than basic search/lookup but essential for comprehensive biological analysis.

**Independent Test**: Can be tested by retrieving a well-annotated protein like "UniProtKB:P38398" (BRCA1) and verifying cross-references are present for HGNC, Ensembl, PDB, and other databases.

**Acceptance Scenarios**:

1. **Given** an agent retrieves a protein record, **When** examining cross-references, **Then** find links to gene databases (HGNC, Ensembl), sequence databases (RefSeq), structure databases (PDB), and pathway databases (KEGG)
2. **Given** an agent needs to map from protein to gene, **When** accessing cross_references.hgnc, **Then** receive the corresponding HGNC gene identifier if available
3. **Given** a protein has no cross-reference to a specific database, **When** accessing that field, **Then** the key is omitted entirely (not null or empty string)

---

### User Story 4 - Error Recovery and Guidance (Priority: P3)

When an agent makes mistakes (ambiguous queries, invalid IDs, rate limits), the system provides actionable guidance to correct the error and continue the workflow.

**Why this priority**: Improves agent autonomy and reduces retry loops. Less critical than core functionality but significantly improves user experience.

**Independent Test**: Can be tested by deliberately triggering each error condition and verifying the error response includes recovery hints.

**Acceptance Scenarios**:

1. **Given** an agent's query returns too many results (>100 with short query), **When** receiving the error, **Then** the error includes a recovery hint to refine the query with more specific terms
2. **Given** an agent hits rate limits, **When** receiving a 429 error, **Then** the error includes the retry-after time and suggests using batch operations
3. **Given** an agent provides a fuzzy query to get_protein instead of a CURIE, **When** receiving the error, **Then** the error explains to use search_proteins first then get_protein with the resolved ID

---

### Edge Cases

- What happens when a protein has multiple isoforms or splice variants?
- How does the system handle obsolete UniProt IDs that have been merged?
- What if an agent queries with a gene name that maps to multiple proteins?
- How are non-English protein names (Chinese, Japanese) handled?
- What happens when UniProt API is unavailable or returns incomplete data?
- How does pagination handle large result sets (10,000+ proteins)?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST provide fuzzy search accepting natural language protein queries (names, genes, organisms, functions)
- **FR-002**: System MUST return ranked search candidates with UniProt accessions as resolvable CURIEs (format: "UniProtKB:XXXXXX")
- **FR-003**: System MUST provide strict lookup accepting only validated UniProt CURIEs
- **FR-004**: System MUST reject fuzzy queries in strict lookup tools with actionable error messages
- **FR-005**: System MUST include cross-references to at minimum: HGNC, Ensembl, RefSeq, PDB, OMIM, KEGG
- **FR-006**: System MUST omit cross-reference keys when data is unavailable (never use null or empty strings)
- **FR-007**: System MUST support cursor-based pagination for search results exceeding page_size
- **FR-008**: System MUST support slim mode reducing token usage by 80% for batch operations
- **FR-009**: System MUST return canonical error envelopes with error codes: UNRESOLVED_ENTITY, ENTITY_NOT_FOUND, AMBIGUOUS_QUERY, RATE_LIMITED, UPSTREAM_ERROR
- **FR-010**: System MUST provide recovery hints in all error responses explaining how to correct the issue
- **FR-011**: System MUST validate minimum query length (2 characters) to prevent overly ambiguous searches
- **FR-012**: System MUST handle rate limiting with exponential backoff and respect Retry-After headers
- **FR-013**: System MUST calculate relevance scores for search results based on query match quality
- **FR-014**: System MUST support concurrent requests without data corruption or race conditions
- **FR-015**: System MUST clean up resources (close connections) when server shuts down

### Key Entities

- **Protein**: A biological molecule with a unique UniProt accession, name, organism, function, sequence, and structural data. Cross-references link to related entities in other databases (genes, pathways, structures). May have isoforms or variants.

- **SearchCandidate**: A lightweight protein match from fuzzy search containing only the UniProt CURIE, protein name, organism, and relevance score. Used to present options before fetching full data.

- **Cross-Reference**: A link from a protein to an external database identifier (HGNC gene ID, Ensembl ID, PDB structure ID, etc). Enables multi-database workflows and entity resolution across biological data sources.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Agents can find relevant proteins in under 2 seconds for 95% of common protein queries (top 1000 human proteins)
- **SC-002**: Fuzzy search returns the biologically correct protein as the top result for 90% of unambiguous queries (e.g., "human p53" → TP53/P04637)
- **SC-003**: System handles 100 concurrent search requests without degradation or errors
- **SC-004**: Agents successfully complete the fuzzy-to-fact workflow (search → validate → fetch) in under 5 seconds for 95% of cases
- **SC-005**: Token usage for batch operations (slim=true) is 80% lower than full records
- **SC-006**: Error recovery hints enable agents to correct mistakes without human intervention in 90% of error scenarios
- **SC-007**: Cross-references are present for at least 80% of well-annotated human proteins (covering major databases: HGNC, Ensembl, RefSeq)
- **SC-008**: System maintains 99.9% uptime measured as successful responses vs total requests (excluding upstream UniProt failures)

## Assumptions

- UniProt REST API supports search queries with relevance ranking
- UniProt provides cross-references in API responses for major databases
- Default rate limit is 10 requests/second (to be verified with UniProt documentation)
- Most agent workflows involve 1-10 protein lookups per task (batch operations rare)
- Agents prefer human proteins when queries are ambiguous (e.g., "insulin" → human insulin prioritized over mouse/rat)
- UniProt API pagination uses standard offset/limit or cursor patterns
- Cross-reference availability varies by protein - well-studied proteins have more links
- Protein names and descriptions are primarily in English in UniProt

## Dependencies

- UniProt REST API availability and stability
- Existing ADR-001 architecture patterns (LifeSciencesClient, canonical envelopes, Fuzzy-to-Fact protocol)
- HGNC cross-reference format compatibility (to enable gene-to-protein workflows)
- Network connectivity to rest.uniprot.org

## Out of Scope

- Protein sequence alignment or BLAST searches
- Protein structure visualization or 3D rendering
- Protein interaction network analysis
- Integration with non-UniProt protein databases (e.g., PDB-specific features)
- Real-time monitoring of protein data updates
- User accounts, authentication, or personalization
- Data caching or local storage (all queries are live to UniProt API)
