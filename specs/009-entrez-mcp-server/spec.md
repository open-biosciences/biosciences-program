# Feature Specification: NCBI Entrez MCP Server

**Feature Branch**: `009-entrez-mcp-server`
**Created**: 2026-01-01
**Status**: Draft
**Input**: Build the NCBI Entrez MCP Server with Fuzzy-to-Fact workflow, Agentic Biolink schema, and Canonical Envelopes.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Fuzzy Gene Search (Priority: P1)

An LLM agent needs to find genes in the NCBI Gene database using natural language queries (gene symbols, names, or descriptions). The agent receives ranked candidates with Entrez Gene IDs that can be used for strict lookups.

**Why this priority**: This is the entry point for all NCBI Gene interactions. Without fuzzy search, agents cannot discover Entrez Gene identifiers, making all downstream operations impossible.

**Independent Test**: Can be fully tested by searching for "BRCA1" and verifying ranked candidates are returned with valid Entrez Gene IDs.

**Acceptance Scenarios**:

1. **Given** an agent searching for genes, **When** they call `search_genes("BRCA1")`, **Then** they receive a PaginationEnvelope with ranked GeneSearchCandidate items containing Entrez IDs.
2. **Given** an agent searching with a partial name, **When** they call `search_genes("tumor suppressor")`, **Then** they receive multiple ranked candidates matching the query.
3. **Given** an agent searching with organism filter, **When** they call `search_genes("TP53", organism="human")`, **Then** only human genes are returned.

---

### User Story 2 - Strict Gene Lookup (Priority: P2)

An LLM agent needs to retrieve complete gene information using a resolved Entrez Gene ID. The agent receives full gene details including summary, map location, aliases, and cross-references to other databases.

**Why this priority**: After fuzzy discovery, strict lookup provides the detailed entity data needed for reasoning. This completes the Fuzzy-to-Fact protocol.

**Independent Test**: Can be fully tested by calling `get_gene("NCBIGene:7157")` (TP53) and verifying all required fields are populated.

**Acceptance Scenarios**:

1. **Given** an agent with a valid Entrez Gene ID, **When** they call `get_gene("NCBIGene:7157")`, **Then** they receive a complete Gene entity with summary, map_location, aliases, and cross_references.
2. **Given** an agent with an invalid ID format, **When** they call `get_gene("TP53")` (raw string, not CURIE), **Then** they receive an UNRESOLVED_ENTITY error with recovery_hint pointing to search_genes.
3. **Given** an agent with a non-existent ID, **When** they call `get_gene("NCBIGene:999999999")`, **Then** they receive an ENTITY_NOT_FOUND error.

---

### User Story 3 - PubMed Literature Links (Priority: P3)

An LLM agent needs to find literature associated with a gene. The agent receives PubMed IDs linking the gene to relevant publications for evidence gathering.

**Why this priority**: Literature links provide evidence for gene function and disease associations, supporting multi-hop reasoning workflows.

**Independent Test**: Can be fully tested by calling `get_pubmed_links("NCBIGene:7157")` and verifying PubMed IDs are returned.

**Acceptance Scenarios**:

1. **Given** an agent with a valid Entrez Gene ID, **When** they call `get_pubmed_links("NCBIGene:7157")`, **Then** they receive a list of PubMed IDs associated with the gene.
2. **Given** an agent requesting limited results, **When** they call `get_pubmed_links("NCBIGene:7157", limit=10)`, **Then** at most 10 PubMed IDs are returned.

---

### User Story 4 - Error Recovery (Priority: P4)

An LLM agent encounters errors and needs actionable guidance to self-correct. All errors include recovery hints that guide the agent to the correct tool or action.

**Why this priority**: Autonomous agents must recover from errors without human intervention. This enables reliable multi-hop reasoning.

**Independent Test**: Can be fully tested by triggering each error type and verifying recovery hints are actionable.

**Acceptance Scenarios**:

1. **Given** an agent receives UNRESOLVED_ENTITY error, **When** they follow the recovery_hint, **Then** they can call search_genes to resolve the entity.
2. **Given** an agent receives RATE_LIMITED error, **When** they follow the recovery_hint, **Then** they know to retry with backoff or add NCBI_API_KEY.
3. **Given** an agent receives UPSTREAM_ERROR, **When** they examine the error, **Then** they understand the NCBI API is temporarily unavailable.

---

### Edge Cases

- What happens when search returns no results? → Empty items array with total_count: 0
- What happens when NCBI API is unavailable? → UPSTREAM_ERROR with recovery_hint
- What happens when rate limit is exceeded? → RATE_LIMITED error with retry guidance and API key suggestion
- What happens when organism filter is invalid? → Appropriate error with guidance
- What happens when gene has no PubMed links? → Empty list returned
- What happens when XML parsing fails? → UPSTREAM_ERROR with details about malformed response

## Requirements *(mandatory)*

### Functional Requirements

#### Search Tool (Fuzzy Discovery)

- **FR-001**: System MUST provide `search_genes(query, organism?, page_size?, cursor?)` tool accepting natural language queries
- **FR-002**: System MUST return GeneSearchCandidate objects with id (NCBIGene:*), name, description, and relevance score
- **FR-003**: System MUST support organism filtering with default to all organisms
- **FR-004**: System MUST support cursor-based pagination with default page_size of 50
- **FR-005**: System MUST wrap results in Canonical PaginationEnvelope
- **FR-006**: System MUST parse XML responses from NCBI E-utilities

#### Lookup Tools (Strict Execution)

- **FR-007**: System MUST provide `get_gene(entrez_id)` tool accepting ONLY valid Entrez Gene IDs (regex: `^NCBIGene:\d+$`)
- **FR-008**: System MUST provide `get_pubmed_links(entrez_id, limit?)` tool returning associated PubMed IDs
- **FR-009**: System MUST return UNRESOLVED_ENTITY error when raw strings are passed to strict tools
- **FR-010**: System MUST return ENTITY_NOT_FOUND error when valid ID format yields no results

#### Schema Requirements

- **FR-011**: Gene entities MUST include: id, symbol, name, description, summary, map_location, chromosome, aliases
- **FR-012**: Gene entities MUST include cross_references object using 22-key registry (hgnc, ensembl_gene, uniprot, refseq, etc.)
- **FR-013**: System MUST omit cross-reference keys entirely when no reference exists (never use null)
- **FR-014**: PubMed links MUST return list of PubMed IDs (PMID format)

#### Error Handling

- **FR-015**: All errors MUST use Canonical ErrorEnvelope with code, message, recovery_hint, invalid_input
- **FR-016**: Error codes MUST be from standard registry: UNRESOLVED_ENTITY, ENTITY_NOT_FOUND, AMBIGUOUS_QUERY, RATE_LIMITED, UPSTREAM_ERROR
- **FR-017**: Recovery hints MUST be actionable (e.g., "Call search_genes to resolve entity first")
- **FR-018**: RATE_LIMITED errors MUST suggest using NCBI_API_KEY for higher limits

#### Rate Limiting & Reliability

- **FR-019**: Client MUST implement rate limiting at 3 requests/second (default without API key)
- **FR-020**: Client MUST support 10 requests/second when NCBI_API_KEY environment variable is set
- **FR-021**: Client MUST implement exponential backoff on 429 responses
- **FR-022**: Client MUST implement thundering herd prevention for concurrent requests

### Key Entities

- **Gene**: NCBI Gene record with symbol, name, description, summary (functional description), map_location (cytogenetic), chromosome, aliases (OtherAliases), and cross-references to HGNC, Ensembl, UniProt, RefSeq
- **GeneSearchCandidate**: Slim representation for search results with id, name, description, and relevance score
- **PubMedLink**: Association between gene and literature with PubMed ID
- **CrossReferences**: Object with optional keys from 22-key registry mapping to external database identifiers

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 95% of search queries return results in under 2 seconds
- **SC-002**: Rate limiting prevents HTTP 429 errors during normal operation
- **SC-003**: Cross-references are populated for genes with known external database mappings (HGNC, Ensembl, UniProt)
- **SC-004**: Agents can complete Fuzzy-to-Fact workflow (search → select → get) without human intervention
- **SC-005**: Error recovery hints enable agents to self-correct in 90% of error scenarios
- **SC-006**: All integration tests pass against live NCBI E-utilities API
- **SC-007**: XML responses are correctly parsed into structured Gene entities

## Assumptions

- NCBI E-utilities API (https://eutils.ncbi.nlm.nih.gov) remains publicly accessible
- Rate limits are 3 req/s without API key, 10 req/s with API key
- NCBI_API_KEY can be obtained free from NCBI (optional but recommended)
- E-utilities return XML by default; JSON is not available for all endpoints
- Cross-reference mappings in gene2refseq and gene2ensembl files are accessible via E-utilities

## Out of Scope

- Batch gene retrieval (efetch with multiple IDs - may be added in future iteration)
- Sequence data retrieval (nucleotide/protein sequences)
- Variation/SNP data
- Homology/orthologs data
- Full PubMed abstract retrieval (only IDs returned, not content)
