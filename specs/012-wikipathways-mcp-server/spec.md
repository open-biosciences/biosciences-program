# Feature Specification: WikiPathways MCP Server

**Feature Branch**: `012-wikipathways-mcp-server`
**Created**: 2026-01-03
**Status**: Draft
**Input**: User description: "Build the WikiPathways MCP Server. Core Requirements: 1. Architecture: Implement WikiPathwaysClient extending LifeSciencesClient using httpx and native asyncio (ADR-001 §2). Base URL: https://www.wikipathways.org/api/. Open CC0 license - no restrictions. 2. Protocol: Implement the Fuzzy-to-Fact workflow (ADR-001 §3): Tool 1: search_pathways(query, organism) (Fuzzy) returning ranked pathway candidates with organism filtering. Tool 2: get_pathway(pathway_id) (Strict) accepting ONLY resolved pathway IDs (WP format). Tool 3: get_pathways_for_gene(gene_id) for reverse gene→pathway lookup. Tool 4: get_pathway_components(pathway_id) extracting genes, proteins, metabolites from pathway. 3. Schema: All outputs must use the Agentic Biolink schema with cross_references (ADR-001 §4). Include pathway reactions, participants, data-nodes, organism, revision info. 4. Envelopes: Must use Canonical Pagination and Error envelopes (ADR-001 §8). 5. Testing: Include a pytest-asyncio test plan covering the Junior Dev ambiguity cases. Rate limit: Conservative 1 req/sec."

## User Scenarios & Testing

### User Story 1 - Pathway Discovery by Topic (Priority: P1)

A researcher wants to find biological pathways related to a specific disease, process, or concept (e.g., "glycolysis", "cancer metabolism", "insulin signaling") to understand relevant biological mechanisms and pathway compositions.

**Why this priority**: Core value proposition enabling users to discover pathways from natural language queries without knowing exact pathway IDs. This is the primary entry point for all WikiPathways interactions and must work first.

**Independent Test**: Can be fully tested by executing keyword search "glycolysis" with organism filter "Homo sapiens" and verifying ranked pathway results are returned with valid WP IDs. Delivers immediate value by surfacing relevant pathways.

**Acceptance Scenarios**:

1. **Given** a user wants to research glycolysis, **When** they search for "glycolysis" with organism filter "Homo sapiens", **Then** they receive a paginated list of pathway candidates with pathway IDs (WP format), names, organisms, and relevance scores
2. **Given** a user searches with a very broad term like "metabolism", **When** results exceed page_size, **Then** the system returns paginated results with cursor-based navigation for accessing additional pages
3. **Given** a user searches for "nonexistent pathway XYZ123", **When** no matches are found, **Then** the system returns an empty items array with total_count 0 and helpful guidance
4. **Given** a user searches without organism filter, **When** they query "apoptosis", **Then** they receive apoptosis pathways across all organisms (human, mouse, rat, etc.)

---

### User Story 2 - Detailed Pathway Information Retrieval (Priority: P1)

After discovering a pathway of interest, a researcher needs to retrieve complete pathway details including title, description, organism, revision metadata, component counts, and cross-references to other pathway databases.

**Why this priority**: Critical for understanding pathway mechanisms. Without detailed information, the discovery in Story 1 has limited value. Must be independently functional with resolved pathway IDs.

**Independent Test**: Can be tested by providing a known pathway ID (e.g., "WP:WP4868") and verifying complete pathway record is returned with all required fields, cross-references, and component counts. Works standalone if pathway IDs are known.

**Acceptance Scenarios**:

1. **Given** a user has a pathway ID (WP:WP4868), **When** they request pathway details, **Then** they receive complete pathway with title, organism, description, revision info (date, version), component counts, and cross-references to Reactome, KEGG, Gene Ontology
2. **Given** a user provides an invalid pathway ID format like "apoptosis" instead of "WP:WPNNNNN", **When** they attempt to retrieve pathway details, **Then** the system returns an UNRESOLVED_ENTITY error with recovery hint pointing to search_pathways tool
3. **Given** a user provides a valid pathway ID format (WP:WP99999) that doesn't exist in database, **When** they request pathway details, **Then** the system returns an ENTITY_NOT_FOUND error with appropriate recovery guidance
4. **Given** a pathway with no cross-references to external databases, **When** pathway details are retrieved, **Then** the cross_references object omits empty keys entirely (never returns null values)

---

### User Story 3 - Reverse Gene-to-Pathway Lookup (Priority: P2)

A researcher studying a specific gene (e.g., "BRCA1", "TP53") wants to identify all biological pathways in which that gene participates to understand its broader biological context, interactions, and disease associations.

**Why this priority**: Essential for gene-centric research enabling reverse lookups from genes to pathways. Provides unique capability complementing forward pathway search. Requires Stories 1 and 2 infrastructure but delivers independent value.

**Independent Test**: Can be tested by providing gene identifier "TP53" and verifying all pathways containing TP53 are returned with pathway IDs, names, and organisms. Works independently if gene IDs are known.

**Acceptance Scenarios**:

1. **Given** a researcher investigating BRCA1, **When** they query pathways for gene "BRCA1" with organism "Homo sapiens", **Then** they receive all human pathways containing BRCA1 with pathway IDs, names, organisms, and descriptions
2. **Given** a user queries an organism-specific gene, **When** organism filter doesn't match gene's typical organism, **Then** the system returns an empty result with clarification about no pathways found in specified organism
3. **Given** a user queries a gene that exists but has no pathway associations, **When** they request pathways, **Then** the system returns an empty items array with total_count 0
4. **Given** a user queries without organism filter, **When** they search for "TP53", **Then** they receive TP53 pathways across all organisms (human, mouse, rat, etc.)

---

### User Story 4 - Pathway Component Extraction (Priority: P3)

After identifying a pathway of interest, a researcher needs to extract all participating biological entities (genes, proteins, metabolites) with their identifiers and relationships to perform downstream analysis such as enrichment testing or network analysis.

**Why this priority**: Enables data export and integration with other analytical tools. Lower priority because it builds on Stories 1-2 and serves more specialized analytical workflows requiring detailed pathway composition.

**Independent Test**: Can be tested by providing pathway ID "WP:WP4868" and verifying all data-nodes (genes, proteins, metabolites) are extracted with entity types, labels, and database identifiers. Independently useful for pathway ID inputs.

**Acceptance Scenarios**:

1. **Given** a user wants to analyze pathway components, **When** they request components for pathway WP:WP4868, **Then** they receive all genes, proteins, and metabolites with entity types (Gene/Protein/Metabolite), labels, and cross-reference identifiers
2. **Given** a user requests components for a pathway containing only genes (no metabolites), **When** components are extracted, **Then** the system returns only gene entities without placeholder metabolite entries (omit empty arrays)
3. **Given** a user requests components using an invalid pathway ID format, **When** extraction is attempted, **Then** the system returns an UNRESOLVED_ENTITY error consistent with Story 2 validation
4. **Given** a pathway with interaction data, **When** components are extracted, **Then** the system includes interaction relationships (source, target, type) between components

---

### Edge Cases

- What happens when a search query is less than 2 characters (e.g., "a")? System returns AMBIGUOUS_QUERY error with guidance to use more specific search terms (minimum 2 characters required).
- How does the system handle rate limiting when API limit (1 req/sec) is exceeded? System implements internal rate limiting with lock-based throttling and exponential backoff on 429 responses, returning RATE_LIMITED error if retry limit exceeded.
- What happens when WikiPathways API is temporarily unavailable (503/504 errors)? System returns UPSTREAM_ERROR with recovery hint to retry later and implements exponential backoff for transient failures.
- How does pagination work when total result count is unknown? System uses cursor-based pagination with opaque cursor tokens, avoiding offset assumptions. Client iterates until cursor is null.
- What happens when a gene ID is valid but case-sensitive (e.g., "tp53" vs "TP53")? System normalizes gene symbols to uppercase for matching and returns matches across all relevant organisms unless organism filter is specified.
- How are cross-references handled when a pathway exists in multiple databases? System returns all available cross-references as individual keys (reactome, kegg_pathway, gene_ontology) with arrays for multi-value references, omitting keys with no values.
- What happens when organism name is misspelled or uses non-standard format? System performs exact match on organism parameter and returns empty results if no match found. No fuzzy organism matching is performed.
- How does the system handle deprecated or replaced pathway IDs? System returns ENTITY_NOT_FOUND error for deprecated IDs. No automatic redirect to replacement IDs (user must search for current version).

## Requirements

### Functional Requirements

#### Architecture Requirements

- **FR-001**: System MUST implement WikiPathwaysClient extending LifeSciencesClient base class with httpx async HTTP client
- **FR-002**: Client MUST use native asyncio (no synchronous SDK dependencies per ADR-001 §2)
- **FR-003**: Client MUST connect to WikiPathways REST API base URL: https://www.wikipathways.org/api/
- **FR-004**: Client MUST implement context manager protocol (__aenter__, __aexit__) for resource cleanup

#### Fuzzy Search Tool (search_pathways)

- **FR-005**: System MUST provide search_pathways tool accepting query parameter (string, minimum 2 characters)
- **FR-006**: System MUST support optional organism filter parameter for species-specific searches
- **FR-007**: System MUST support cursor-based pagination with cursor and page_size parameters (default page_size: 50, max: 100)
- **FR-008**: System MUST return PathwaySearchCandidate objects with id (WP:WPNNNNN format), title, organism, description snippet, and relevance score
- **FR-009**: System MUST calculate relevance scores based on query match quality and result position (score decay: 0.05 per position)
- **FR-010**: System MUST wrap results in Canonical PaginationEnvelope with items, cursor, total_count, and page_size
- **FR-011**: System MUST support slim mode returning minimal fields (id, title, organism, score) to reduce token usage (~20 tokens vs ~115 tokens per entity)
- **FR-012**: System MUST return AMBIGUOUS_QUERY error when query length is less than 2 characters

#### Strict Lookup Tool (get_pathway)

- **FR-013**: System MUST provide get_pathway tool accepting ONLY pathway_id parameter in WP:WPNNNNN format
- **FR-014**: System MUST validate pathway_id against regex pattern ^WP:WP\d+$ before API call
- **FR-015**: System MUST return UNRESOLVED_ENTITY error when raw pathway names or invalid formats are passed
- **FR-016**: System MUST return ENTITY_NOT_FOUND error when valid ID format yields no results
- **FR-017**: System MUST return complete Pathway entity with id, title, description, organism, revision metadata (version, last_modified, curators), and component_counts
- **FR-018**: System MUST include cross_references object following Agentic Biolink schema with external database mappings (reactome, kegg_pathway, gene_ontology, etc.)
- **FR-019**: System MUST omit cross-reference keys entirely when no reference exists (never use null or empty string)

#### Reverse Gene Lookup Tool (get_pathways_for_gene)

- **FR-020**: System MUST provide get_pathways_for_gene tool accepting gene_id parameter (gene symbol, Entrez ID, or Ensembl ID)
- **FR-021**: System MUST support optional organism filter parameter to narrow results to specific species
- **FR-022**: System MUST support cursor-based pagination with cursor and page_size parameters
- **FR-023**: System MUST normalize gene symbols to uppercase for matching consistency
- **FR-024**: System MUST return PathwaySearchCandidate objects for all pathways containing the specified gene
- **FR-025**: System MUST wrap results in Canonical PaginationEnvelope
- **FR-026**: System MUST return empty items array when gene exists but has no pathway associations

#### Component Extraction Tool (get_pathway_components)

- **FR-027**: System MUST provide get_pathway_components tool accepting pathway_id parameter in WP:WPNNNNN format
- **FR-028**: System MUST validate pathway_id format before API call (same validation as get_pathway)
- **FR-029**: System MUST return PathwayComponents entity with separate lists for genes, proteins, metabolites, and interactions
- **FR-030**: System MUST include DataNode metadata: id, label, type (Gene/Protein/Metabolite/Complex), database source
- **FR-031**: System MUST include Interaction metadata: source (node id), target (node id), type (activation/inhibition/binding/conversion)
- **FR-032**: System MUST include cross-references for each DataNode using 22-key registry (HGNC for genes, UniProt for proteins, ChEMBL/PubChem for metabolites)
- **FR-033**: System MUST omit empty component lists (if pathway has no metabolites, omit metabolites key entirely)

#### Schema Requirements

- **FR-034**: All pathway entities MUST follow Agentic Biolink schema with flattened JSON structure
- **FR-035**: CrossReferences object MUST use 22-key registry: hgnc, ensembl_gene, uniprot, entrez, refseq, chembl, drugbank, string, kegg, kegg_pathway, omim, orphanet, mondo, efo, pdb, pubchem_compound, pubchem_substance, reactome, gene_ontology
- **FR-036**: System MUST omit cross-reference keys when no value exists (never use null, empty string, or empty array)
- **FR-037**: Component counts MUST include: gene_count, protein_count, metabolite_count, interaction_count
- **FR-038**: Revision metadata MUST include: version (string), last_modified (ISO 8601 date), curators (array of strings)

#### Error Handling

- **FR-039**: All errors MUST use Canonical ErrorEnvelope with success (false), error object containing code, message, recovery_hint, and optional invalid_input
- **FR-040**: Error codes MUST be from standard registry: UNRESOLVED_ENTITY, ENTITY_NOT_FOUND, AMBIGUOUS_QUERY, RATE_LIMITED, UPSTREAM_ERROR
- **FR-041**: Recovery hints MUST be actionable and agent-directed (e.g., "Call search_pathways to resolve pathway identifier first")
- **FR-042**: System MUST return UPSTREAM_ERROR on API failures (503, 504, network timeouts) with retry guidance

#### Rate Limiting & Reliability

- **FR-043**: Client MUST implement conservative rate limiting at 1 request per second (1000ms delay between requests)
- **FR-044**: Client MUST use asyncio.Lock to prevent race conditions in rate limiting logic
- **FR-045**: Client MUST implement exponential backoff on 429 (rate limited) and 503 (service unavailable) responses with maximum 3 retry attempts
- **FR-046**: Client MUST implement thundering herd prevention by re-checking rate limit delay after lock acquisition
- **FR-047**: Client MUST use 10-second default timeout for API requests
- **FR-048**: Client MUST handle httpx.TimeoutException and httpx.RequestError gracefully with UPSTREAM_ERROR responses

### Key Entities

- **Pathway**: Biological pathway entity with id (WP:WPNNNNN format), title (string), description (string), organism (scientific name like "Homo sapiens"), revision object (version, last_modified, curators), component_counts object (gene_count, protein_count, metabolite_count, interaction_count), and cross_references object mapping to external databases
- **PathwaySearchCandidate**: Lightweight search result for fuzzy discovery with id (WP:WPNNNNN), title (string), organism (string), description snippet (first 200 chars), and relevance score (float 0.0-1.0)
- **PathwayComponents**: Structured pathway composition data with separate lists for genes (DataNode array), proteins (DataNode array), metabolites (DataNode array), and interactions (Interaction array)
- **DataNode**: Pathway component entity with id (internal node identifier), label (display name), type (Gene/Protein/Metabolite/Complex), database (source database like "Entrez Gene", "UniProt", "ChEMBL"), and cross_references object
- **Interaction**: Pathway relationship entity with source (DataNode id reference), target (DataNode id reference), and type (activation/inhibition/binding/conversion/catalysis)
- **CrossReferences**: Object with optional keys from 22-key registry (reactome, kegg_pathway, gene_ontology, hgnc, uniprot, etc.) mapping to external database identifier strings or arrays

## Success Criteria

### Measurable Outcomes

- **SC-001**: Users can discover relevant pathways from natural language queries in under 2 seconds (95th percentile response time)
- **SC-002**: Fuzzy search returns top-ranked pathway candidate matching user's intended topic in top 5 results for 90% of common biological terms (glycolysis, apoptosis, cell cycle, etc.)
- **SC-003**: System correctly rejects invalid pathway ID formats (raw pathway names like "apoptosis") with UNRESOLVED_ENTITY error in 100% of cases
- **SC-004**: Users can retrieve complete pathway details including all components and cross-references in a single lookup operation with zero failed requests
- **SC-005**: Gene-to-pathway reverse lookup returns all relevant pathways for well-studied genes (BRCA1, TP53, EGFR) with zero false negatives compared to WikiPathways website
- **SC-006**: System handles WikiPathways API rate limits gracefully with zero failed requests due to rate limiting (automatic throttling and backoff)
- **SC-007**: Pagination supports iterating through large result sets (100+ pathways) without data loss, duplication, or cursor failures
- **SC-008**: Error messages enable agent self-correction with actionable recovery hints in 100% of error scenarios (verified through error recovery test suite)
- **SC-009**: Cross-references are correctly populated for 95% of pathways that have documented external database mappings in WikiPathways
- **SC-010**: Component extraction identifies all genes, proteins, metabolites, and interactions with zero data loss compared to WikiPathways GPML/JSON source

## Assumptions

- WikiPathways REST API documentation at https://www.wikipathways.org/api/ is accurate and endpoints return data in documented JSON or GPML format
- WikiPathways API base URL (https://www.wikipathways.org/api/) is stable and publicly accessible without authentication
- Pathway IDs follow WP:WPNNNNN format consistently across API responses (WP prefix + digits, no version suffixes)
- CC0 license allows unrestricted API usage without requiring API keys, rate limit headers, or attribution in responses
- Organism names use standard scientific nomenclature (e.g., "Homo sapiens", "Mus musculus", "Rattus norvegicus")
- Cross-reference database identifiers (Reactome, KEGG, GO) are provided in pathway metadata or can be extracted from GPML annotations
- Rate limit of 1 req/sec is sufficient for typical research workflows (no official rate limit published, conservative assumption)
- Gene identifiers used in reverse lookup match gene symbols, Entrez IDs, or Ensembl IDs used in WikiPathways pathway annotations
- API responses include revision metadata (version, last_modified, curator list) for pathway provenance tracking
- WikiPathways API returns pathway component data (genes, proteins, metabolites, interactions) in structured format (JSON) or parseable GPML XML
- Deprecated pathway IDs return 404 errors (not automatically redirected to replacement pathway IDs)
