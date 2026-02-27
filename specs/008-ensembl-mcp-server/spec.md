# Feature Specification: Ensembl MCP Server

**Feature Branch**: `008-ensembl-mcp-server`
**Created**: 2026-01-01
**Status**: Draft
**Input**: Build the Ensembl MCP Server with Fuzzy-to-Fact workflow, Agentic Biolink schema, and Canonical Envelopes.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Fuzzy Gene Search (Priority: P1)

An LLM agent needs to find genes in the Ensembl database using natural language queries (gene symbols, names, or partial matches). The agent receives ranked candidates with Ensembl Gene IDs that can be used for strict lookups.

**Why this priority**: This is the entry point for all Ensembl interactions. Without fuzzy search, agents cannot discover gene identifiers, making all downstream operations impossible.

**Independent Test**: Can be fully tested by searching for "BRCA1" and verifying ranked candidates are returned with valid Ensembl Gene IDs (ENSG*).

**Acceptance Scenarios**:

1. **Given** an agent searching for genes, **When** they call `search_genes("BRCA1")`, **Then** they receive a PaginationEnvelope with ranked GeneSearchCandidate items containing Ensembl IDs.
2. **Given** an agent searching with a partial name, **When** they call `search_genes("tumor protein")`, **Then** they receive multiple ranked candidates matching the query.
3. **Given** an agent searching with species filter, **When** they call `search_genes("TP53", species="human")`, **Then** only human (Homo sapiens) genes are returned.

---

### User Story 2 - Strict Gene Lookup (Priority: P2)

An LLM agent needs to retrieve complete gene information using a resolved Ensembl Gene ID. The agent receives full gene details including biotype, genomic coordinates, assembly information, and cross-references to other databases.

**Why this priority**: After fuzzy discovery, strict lookup provides the detailed entity data needed for reasoning. This completes the Fuzzy-to-Fact protocol.

**Independent Test**: Can be fully tested by calling `get_gene("ENSG00000141510")` (TP53) and verifying all required fields are populated.

**Acceptance Scenarios**:

1. **Given** an agent with a valid Ensembl Gene ID, **When** they call `get_gene("ENSG00000141510")`, **Then** they receive a complete Gene entity with biotype, strand, coordinates, and cross_references.
2. **Given** an agent with an invalid ID format, **When** they call `get_gene("BRCA1")` (raw string, not CURIE), **Then** they receive an UNRESOLVED_ENTITY error with recovery_hint pointing to search_genes.
3. **Given** an agent with a non-existent ID, **When** they call `get_gene("ENSG99999999999")`, **Then** they receive an ENTITY_NOT_FOUND error.

---

### User Story 3 - Transcript Lookup (Priority: P3)

An LLM agent needs to retrieve transcript details for a specific Ensembl Transcript ID. The agent receives transcript information including biotype, parent gene reference, and genomic coordinates.

**Why this priority**: Transcript-level queries are common in genomics workflows but depend on gene-level discovery first.

**Independent Test**: Can be fully tested by calling `get_transcript("ENST00000269305")` and verifying transcript details are returned.

**Acceptance Scenarios**:

1. **Given** an agent with a valid Ensembl Transcript ID, **When** they call `get_transcript("ENST00000269305")`, **Then** they receive transcript details including biotype and parent gene reference.
2. **Given** an agent with an invalid transcript ID, **When** they call `get_transcript("ENSG00000141510")` (gene ID, not transcript), **Then** they receive an appropriate error with recovery hint.

---

### User Story 4 - Error Recovery (Priority: P4)

An LLM agent encounters errors and needs actionable guidance to self-correct. All errors include recovery hints that guide the agent to the correct tool or action.

**Why this priority**: Autonomous agents must recover from errors without human intervention. This enables reliable multi-hop reasoning.

**Independent Test**: Can be fully tested by triggering each error type and verifying recovery hints are actionable.

**Acceptance Scenarios**:

1. **Given** an agent receives UNRESOLVED_ENTITY error, **When** they follow the recovery_hint, **Then** they can call search_genes to resolve the entity.
2. **Given** an agent receives RATE_LIMITED error, **When** they follow the recovery_hint, **Then** they know to retry with backoff.
3. **Given** an agent receives UPSTREAM_ERROR, **When** they examine the error, **Then** they understand the Ensembl API is temporarily unavailable.

---

### Edge Cases

- What happens when search returns no results? → Empty items array with total_count: 0
- What happens when Ensembl API is unavailable? → UPSTREAM_ERROR with recovery_hint
- What happens when rate limit is exceeded? → RATE_LIMITED error with retry guidance
- What happens when species filter is invalid? → Appropriate error with valid species list hint
- What happens when gene has no cross-references? → Cross-reference keys are omitted (not null)

## Requirements *(mandatory)*

### Functional Requirements

#### Search Tool (Fuzzy Discovery)

- **FR-001**: System MUST provide `search_genes(query, species?, page_size?, cursor?)` tool accepting natural language queries
- **FR-002**: System MUST return GeneSearchCandidate objects with id (ENSG*), name, biotype, and relevance score
- **FR-003**: System MUST support species filtering with default to all species
- **FR-004**: System MUST support cursor-based pagination with default page_size of 50
- **FR-005**: System MUST wrap results in Canonical PaginationEnvelope

#### Lookup Tools (Strict Execution)

- **FR-006**: System MUST provide `get_gene(ensembl_id)` tool accepting ONLY valid Ensembl Gene IDs (regex: `^ENSG\d{11}$`)
- **FR-007**: System MUST provide `get_transcript(transcript_id)` tool accepting ONLY valid Ensembl Transcript IDs (regex: `^ENST\d{11}$`)
- **FR-008**: System MUST return UNRESOLVED_ENTITY error when raw strings are passed to strict tools
- **FR-009**: System MUST return ENTITY_NOT_FOUND error when valid ID format yields no results

#### Schema Requirements

- **FR-010**: Gene entities MUST include: id, symbol, name, biotype, assembly_name, strand, start, end, chromosome
- **FR-011**: Gene entities MUST include cross_references object using 22-key registry (hgnc, uniprot, entrez, refseq, etc.)
- **FR-012**: System MUST omit cross-reference keys entirely when no reference exists (never use null)
- **FR-013**: Transcript entities MUST include: id, biotype, parent_gene, start, end, strand

#### Error Handling

- **FR-014**: All errors MUST use Canonical ErrorEnvelope with code, message, recovery_hint, invalid_input
- **FR-015**: Error codes MUST be from standard registry: UNRESOLVED_ENTITY, ENTITY_NOT_FOUND, AMBIGUOUS_QUERY, RATE_LIMITED, UPSTREAM_ERROR
- **FR-016**: Recovery hints MUST be actionable (e.g., "Call search_genes to resolve entity first")

#### Rate Limiting & Reliability

- **FR-017**: Client MUST implement rate limiting at 15 requests/second (Ensembl limit)
- **FR-018**: Client MUST implement exponential backoff on 429 responses
- **FR-019**: Client MUST implement thundering herd prevention for concurrent requests

### Key Entities

- **Gene**: Genomic feature with symbol, name, biotype (protein_coding, lncRNA, etc.), assembly version, strand (+/-), chromosomal coordinates, and cross-references to HGNC, UniProt, Entrez, RefSeq
- **Transcript**: RNA product of a gene with biotype, parent gene reference, and genomic coordinates
- **GeneSearchCandidate**: Slim representation for search results with id, name, biotype, and relevance score
- **CrossReferences**: Object with optional keys from 22-key registry mapping to external database identifiers

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 95% of search queries return results in under 2 seconds
- **SC-002**: Rate limiting prevents HTTP 429 errors during normal operation
- **SC-003**: Cross-references are populated for genes with known external database mappings (HGNC, UniProt, Entrez)
- **SC-004**: Agents can complete Fuzzy-to-Fact workflow (search → select → get) without human intervention
- **SC-005**: Error recovery hints enable agents to self-correct in 90% of error scenarios
- **SC-006**: All integration tests pass against live Ensembl REST API

## Assumptions

- Ensembl REST API (https://rest.ensembl.org) remains publicly accessible without authentication
- Rate limit of 15 requests/second is current Ensembl policy
- Ensembl Gene ID format (ENSG + 11 digits) and Transcript ID format (ENST + 11 digits) are stable
- Cross-reference mappings in Ensembl xrefs endpoint follow standard identifier formats

## Out of Scope

- Variant/mutation data (VEP endpoints)
- Regulatory features
- Comparative genomics
- Archive/historical assemblies
- Batch endpoint for multiple genes (may be added in future iteration)
