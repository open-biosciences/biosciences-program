# Feature Specification: Open Targets MCP Server

**Feature Branch**: `004-opentargets-mcp-server`
**Created**: 2025-12-22
**Status**: In Progress (699 lines, GraphQL API validated)
**Updated**: 2025-12-23
**Linear**: Pending - child of AGE-65
**Input**: User description: "Build the Open Targets MCP Server.

Core Requirements:
1. **Architecture:** Implement `OpenTargetsClient` extending `LifeSciencesClient` using `httpx` and native `asyncio` (ADR-001 §2). Use GraphQL API.
2. **Protocol:** Implement the 'Fuzzy-to-Fact' workflow (ADR-001 §3):
   - Tool 1: `search_targets(query)` (Fuzzy) returning ranked target candidates.
   - Tool 2: `get_target(ensembl_id)` (Strict) accepting ONLY resolved Ensembl gene IDs.
   - Tool 3: `get_associations(target_id, disease_id)` for target-disease evidence.
3. **Schema:** All outputs must use the 'Agentic Biolink' schema with `cross_references` (ADR-001 §4).
4. **Envelopes:** Must use Canonical Pagination and Error envelopes (ADR-001 §8).
5. **Testing:** Include a `pytest-asyncio` test plan covering the 'Junior Dev' ambiguity cases."

## User Scenarios & Testing

### User Story 1 - Fuzzy Target Search (Priority: P1)

As a drug discovery researcher, I need to find gene/protein targets using common names, symbols, or partial identifiers so that I can quickly locate candidate targets for disease association analysis without knowing exact Ensembl gene IDs.

**Why this priority**: Enables the critical first step of target discovery - researchers rarely know exact Ensembl IDs but need to start from familiar gene symbols (e.g., "BRCA1", "TP53") or disease-related search terms. Without this, the entire workflow is blocked.

**Independent Test**: Can be fully tested by searching for well-known gene symbols like "BRCA1" or "TP53" and verifying that ranked candidates are returned with Ensembl IDs that can be used in subsequent lookups. Delivers immediate value for target identification.

**Acceptance Scenarios**:

1. **Given** a researcher knows a gene symbol, **When** they search for "BRCA1", **Then** the system returns ranked candidates including ENSG00000012048 with relevance scores
2. **Given** a researcher uses a partial search, **When** they search for "kinase", **Then** the system returns paginated results with multiple kinase-related targets
3. **Given** a researcher searches for a well-characterized target, **When** they search for "TP53", **Then** the top result has a score of 1.0 (exact match)
4. **Given** a researcher needs to browse large result sets, **When** they use pagination cursors, **Then** they can navigate through all matching targets without duplicates

---

### User Story 2 - Strict Target Lookup (Priority: P2)

As a drug discovery researcher, I need to retrieve complete target records by Ensembl gene ID so that I can access detailed gene information, cross-references to other databases, and downstream association data for targets identified through fuzzy search.

**Why this priority**: Provides the factual "ground truth" data needed after target identification. This is the second step in the Fuzzy-to-Fact workflow, enabling researchers to access comprehensive target profiles required for informed decision-making.

**Independent Test**: Can be fully tested by looking up known Ensembl IDs like ENSG00000141510 (TP53) and verifying that complete target records with cross-references are returned. Delivers standalone value for researchers who already have Ensembl IDs from external sources.

**Acceptance Scenarios**:

1. **Given** a resolved Ensembl ID from search results, **When** they call get_target("ENSG00000141510"), **Then** the system returns complete TP53 target data with HGNC, UniProt, and disease cross-references
2. **Given** a researcher needs minimal data for token efficiency, **When** they call get_target with slim=True, **Then** the system returns only id, approved_symbol, approved_name, and biotype
3. **Given** an invalid Ensembl ID format, **When** they call get_target("INVALID"), **Then** the system returns an error envelope with code UNRESOLVED_ENTITY and recovery hint
4. **Given** a non-existent but valid Ensembl ID, **When** they call get_target("ENSG00000000000"), **Then** the system returns an error envelope with code ENTITY_NOT_FOUND

---

### User Story 3 - Target-Disease Association Discovery (Priority: P3)

As a drug discovery researcher, I need to query target-disease associations and evidence scores so that I can identify which diseases are linked to specific gene targets and prioritize targets for drug repurposing or development.

**Why this priority**: Enables the critical translational step from target biology to disease relevance. This is the unique value proposition of Open Targets - connecting genetic evidence to disease. Essential for drug repurposing and target prioritization but depends on having targets identified first (US1+US2).

**Independent Test**: Can be fully tested by querying associations for known disease genes like ENSG00000141510 (TP53) and verifying that cancer-related diseases appear with evidence scores. Delivers standalone value for researchers exploring disease connections for known targets.

**Acceptance Scenarios**:

1. **Given** a target with known disease associations, **When** they call get_associations("ENSG00000141510"), **Then** the system returns paginated associations with breast cancer, lung cancer, etc., each with evidence scores
2. **Given** a researcher wants to filter by specific disease, **When** they call get_associations("ENSG00000141510", disease_id="EFO_0000311"), **Then** only breast cancer associations are returned
3. **Given** a target with many associations, **When** they use pagination cursors, **Then** they can navigate through all associations without missing data
4. **Given** a target with no known associations, **When** they query associations, **Then** the system returns an empty items array with pagination metadata

---

### User Story 4 - Error Recovery & Validation (Priority: P4)

As a drug discovery researcher, I need clear error messages with recovery hints when queries fail so that I can quickly correct mistakes and continue my analysis workflow without frustration or support escalation.

**Why this priority**: Improves user experience and reduces support burden, but does not block core functionality. Researchers can still use the system with trial-and-error, though less efficiently. This is a quality-of-life improvement built on top of the core features.

**Independent Test**: Can be fully tested by deliberately triggering each error condition (invalid IDs, malformed queries, etc.) and verifying that error envelopes contain actionable recovery hints. Delivers standalone value for reducing friction in the user experience.

**Acceptance Scenarios**:

1. **Given** a researcher provides a short query, **When** they search for "a", **Then** the system returns AMBIGUOUS_QUERY error with hint "Minimum query length is 2 characters"
2. **Given** a researcher uses invalid Ensembl ID format, **When** they call get_target("TP53"), **Then** the system returns UNRESOLVED_ENTITY with hint showing correct format (ENSG...)
3. **Given** the Open Targets API is temporarily unavailable, **When** any query is made, **Then** the system returns UPSTREAM_ERROR with hint "Retry in 60 seconds"
4. **Given** a rate limit is exceeded, **When** queries are made too quickly, **Then** the system returns RATE_LIMITED error with backoff time in recovery hint

---

### Edge Cases

- What happens when a target has zero disease associations? (Return empty pagination envelope, not error)
- How does the system handle targets with hundreds of disease associations? (Cursor pagination required)
- What happens when Open Targets GraphQL schema changes? (Error handling for unexpected response structure)
- How does the system handle Unicode in gene symbols? (UTF-8 encoding throughout, proper validation)
- What happens when searching for extremely broad terms like "protein"? (Return top 100 results with AMBIGUOUS_QUERY warning if total_count >> page_size)
- How does the system handle Ensembl IDs from non-human organisms? (Accept if valid format, return ENTITY_NOT_FOUND if not in Open Targets)

## Requirements

### Functional Requirements

**Architecture & Integration**
- **FR-001**: System MUST implement OpenTargetsClient as a subclass of LifeSciencesClient
- **FR-002**: System MUST use httpx with native asyncio for all API calls (no run_in_executor needed - GraphQL is async-friendly)
- **FR-003**: System MUST use Open Targets Platform GraphQL API (endpoint: https://api.platform.opentargets.org/api/v4/graphql)
- **FR-004**: System MUST configure connection pooling via LifeSciencesClient base class

**Fuzzy-to-Fact Protocol (US1 + US2 + US3)**
- **FR-005**: System MUST provide a search_targets tool that accepts natural language queries (gene names, symbols, disease terms)
- **FR-006**: System MUST rank search results by relevance score (1.0 = exact match, decreasing linearly by position)
- **FR-007**: System MUST return TargetSearchCandidate entities from search_targets with id, approved_symbol, approved_name, score
- **FR-008**: System MUST provide a get_target tool that accepts ONLY Ensembl gene IDs in format ENSG[0-9]{11}
- **FR-009**: System MUST reject non-Ensembl ID formats in get_target with UNRESOLVED_ENTITY error code
- **FR-010**: System MUST provide a get_associations tool for querying target-disease relationships
- **FR-011**: System MUST support optional disease_id filtering in get_associations

**Agentic Biolink Schema (US2)**
- **FR-012**: System MUST return Target entities conforming to Agentic Biolink schema
- **FR-013**: System MUST include cross_references object mapping to 22-key registry (hgnc, uniprot, chembl, drugbank, omim, mondo, efo, etc.)
- **FR-014**: System MUST omit cross_reference keys entirely when no mapping exists (never use null or empty string)
- **FR-015**: System MUST normalize cross-reference CURIEs to standard formats (HGNC:NNNNN, UniProtKB:ACCESSION, etc.)

**GraphQL Query Construction**
- **FR-016**: System MUST construct GraphQL queries dynamically based on tool parameters
- **FR-017**: System MUST use GraphQL search query for search_targets (fuzzy matching)
- **FR-018**: System MUST use GraphQL target query for get_target (strict lookup by Ensembl ID)
- **FR-019**: System MUST use GraphQL associatedDiseases query for get_associations
- **FR-020**: System MUST handle GraphQL errors and map to appropriate error codes

**Canonical Envelopes (US1 + US2 + US3 + US4)**
- **FR-021**: System MUST return PaginationEnvelope for all list-returning tools (search_targets, get_associations)
- **FR-022**: System MUST include opaque cursor for pagination (base64-encoded offset or GraphQL cursor)
- **FR-023**: System MUST return ErrorEnvelope for all failures with code, message, recovery_hint, invalid_input
- **FR-024**: System MUST use these error codes: UNRESOLVED_ENTITY, ENTITY_NOT_FOUND, AMBIGUOUS_QUERY, RATE_LIMITED, UPSTREAM_ERROR, INVALID_CROSS_REFERENCE
- **FR-025**: System MUST provide actionable recovery hints in all error responses

**Token Budgeting (US2)**
- **FR-026**: System MUST support slim mode for search_targets (returns id, approved_symbol, approved_name, score only, ~20 tokens per entity)
- **FR-027**: System MUST support slim mode for get_target (returns id, approved_symbol, approved_name, biotype only, ~20 tokens)
- **FR-028**: System MUST return full Target entity in get_target when slim=False (~115-300 tokens including cross_references)

**Validation & Error Handling (US4)**
- **FR-029**: System MUST validate query length in search_targets (minimum 2 characters) → AMBIGUOUS_QUERY error
- **FR-030**: System MUST validate Ensembl ID format in get_target using regex ^ENSG\d{11}$ → UNRESOLVED_ENTITY error
- **FR-031**: System MUST validate page_size parameter (1-100) in all paginated tools
- **FR-032**: System MUST implement rate limiting (10 requests/second with exponential backoff)
- **FR-033**: System MUST handle Open Targets API errors gracefully → UPSTREAM_ERROR with retry hint
- **FR-034**: System MUST handle network timeouts with 30-second default timeout
- **FR-035**: System MUST handle GraphQL query errors and map to appropriate error codes

**Testing (per ADR-003)**
- **FR-036**: System MUST include pytest-asyncio tests for all 4 user stories
- **FR-037**: System MUST include integration tests for "Junior Dev" ambiguity cases (e.g., passing gene symbol instead of Ensembl ID to get_target)
- **FR-038**: System MUST include tests for all 6 error codes with recovery scenarios
- **FR-039**: System MUST include performance tests validating SC-001 (<2s for 95% of queries)
- **FR-040**: System MUST include concurrency tests validating SC-004 (100 concurrent requests)

### Key Entities

- **TargetSearchCandidate**: Lightweight search result for fuzzy target discovery (id, approved_symbol, approved_name, score)
- **Target**: Complete target record with gene information (id, approved_symbol, approved_name, biotype, description, associated_diseases count, cross_references)
- **Association**: Target-disease association with evidence (target_id, disease_id, disease_name, score, evidence_count, evidence_sources)
- **CrossReferences**: 22-key registry mapping target to external databases (hgnc, uniprot, chembl, drugbank, omim, mondo, efo, ensembl_gene, entrez, refseq, string, biogrid, kegg, etc.)

## Success Criteria

### Measurable Outcomes

- **SC-001**: 95% of target search queries return results in under 2 seconds (measured from API call to response)
- **SC-002**: 100% of get_target lookups with valid Ensembl IDs return results in under 1 second
- **SC-003**: Search relevance accuracy >90% for well-known gene symbols (top result matches intended target for BRCA1, TP53, EGFR, etc.)
- **SC-004**: System supports 100 concurrent requests across all tools without performance degradation
- **SC-005**: Error recovery hints enable 95% of users to self-correct errors without documentation
- **SC-006**: Cross-reference mappings achieve >95% accuracy for well-studied targets (verified against manual curation)
- **SC-007**: Association queries return complete evidence chains (all data types: genetic, somatic, drug, etc.) for cancer-related targets
- **SC-008**: Pagination maintains cursor integrity (no duplicate or skipped results when navigating forward/backward)

## Dependencies

- **External Dependencies**:
  - Open Targets Platform GraphQL API (https://platform-docs.opentargets.org/data-access/graphql-api)
  - Ensembl gene ID format specification (11-digit numeric suffix after ENSG prefix)
  - EFO (Experimental Factor Ontology) for disease IDs

- **Internal Dependencies**:
  - LifeSciencesClient base class (client.py)
  - Canonical envelopes (models/envelopes.py)
  - 22-key CrossReferences schema (models/gene.py)

- **API Access**:
  - No API key required (public API)
  - Rate limiting expected (10 req/s default, to be validated in Phase 0 research)

## Assumptions

1. **GraphQL Schema Stability**: Assumes Open Targets GraphQL schema remains backward-compatible (query field names, response structure). Mitigation: Version GraphQL queries, handle unexpected fields gracefully.

2. **Ensembl ID Coverage**: Assumes Open Targets primarily covers human genes with Ensembl IDs. Non-human organisms or non-Ensembl IDs will return ENTITY_NOT_FOUND.

3. **Rate Limiting**: Assumes 10 req/s rate limit (same as HGNC/UniProt). Will be validated in Phase 0 research. May need adjustment if Open Targets enforces stricter limits.

4. **Disease ID Format**: Assumes EFO disease IDs are the primary identifier for filtering associations. Alternative disease ontologies (e.g., MONDO) may need additional mapping.

5. **Performance Expectations**: Assumes GraphQL API response times are similar to REST APIs (<2s for 95% of queries). Will be validated in performance tests.

6. **Token Budgeting**: Assumes slim mode reduces tokens by 80-85% (from ~115-300 to ~20 tokens per entity). Will be validated in implementation.

7. **Cross-Reference Availability**: Assumes well-studied targets have comprehensive cross-references (HGNC, UniProt, ChEMBL). Obscure targets may have sparse cross-references - handled via omit-if-null pattern.

## Out of Scope

- Disease-centric workflows (searching diseases to find targets) - Open Targets supports this, but our focus is target-centric workflows
- Drug-centric queries (searching drugs to find targets) - Defer to ChEMBL/DrugBank MCP servers
- Batch target lookup tool - Not needed due to async nature of GraphQL (unlike ChEMBL's synchronous SDK)
- Evidence detail expansion - Associations return summary scores, not full evidence chains (can be added in future iteration)
- Pathway analysis - Open Targets provides pathway data, but this requires separate tool design
- CRISPR screen data - Available in Open Targets, but requires specialized workflow beyond current scope
