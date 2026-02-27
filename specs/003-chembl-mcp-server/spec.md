# Feature Specification: ChEMBL MCP Server

**Feature Branch**: `003-chembl-mcp-server`
**Created**: 2025-12-22
**Status**: In Progress (112 tests passing)
**Updated**: 2025-12-23
**Linear**: Pending - child of AGE-65
**Input**: User description: "Build the ChEMBL MCP Server for compound and bioactivity data"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Fuzzy Compound Search (Priority: P1)

As a drug discovery researcher, I need to find chemical compounds using common names, trade names, or partial identifiers so that I can quickly locate candidate compounds for further analysis without knowing exact ChEMBL IDs.

**Why this priority**: This is the entry point for all compound discovery workflows. Without fuzzy search, users cannot begin their research journey. It enables exploration and discovery when exact identifiers are unknown.

**Independent Test**: Can be fully tested by querying "aspirin" and verifying that ranked candidates include acetylsalicylic acid with ChEMBL IDs. Delivers immediate value by enabling compound discovery.

**Acceptance Scenarios**:

1. **Given** a researcher knows a drug's common name, **When** they search for "imatinib", **Then** the system returns ranked candidates including CHEMBL941 with molecular properties
2. **Given** a researcher enters a partial compound name, **When** they search with minimum 2 characters, **Then** the system returns relevant matches without requiring exact spelling
3. **Given** a researcher needs to browse results, **When** they request the next page using a cursor, **Then** the system returns the next batch of ranked results seamlessly
4. **Given** a researcher wants quick triage, **When** they enable slim mode, **Then** each result contains minimal fields suitable for list display
5. **Given** a researcher enters ambiguous input, **When** the query matches hundreds of compounds, **Then** the system returns top 50 matches ranked by relevance with pagination support

---

### User Story 2 - Strict Compound Lookup (Priority: P2)

As a drug discovery researcher, once I have identified the correct ChEMBL ID from search results, I need to retrieve the complete compound record including molecular structure, properties, and cross-references so that I can perform detailed analysis and navigate to related databases.

**Why this priority**: This is the natural second step after discovery. It transforms a search candidate into actionable data with full molecular details and cross-database links. Critical for the Fuzzy-to-Fact workflow.

**Independent Test**: Can be tested by directly calling with "CHEMBL:25" and verifying complete compound record with SMILES, molecular formula, and cross-references to UniProt, PDB, PubChem, DrugBank. Delivers value by providing authoritative compound data.

**Acceptance Scenarios**:

1. **Given** a researcher has a ChEMBL CURIE from search results, **When** they request compound details for "CHEMBL:25", **Then** the system returns complete molecular data including SMILES, InChI, molecular weight, and formula
2. **Given** a researcher needs to navigate to related proteins, **When** they retrieve a compound record, **Then** the cross_references field contains UniProt IDs for target proteins
3. **Given** a researcher needs structural data, **When** they retrieve a compound record, **Then** the cross_references field contains PDB and PubChem links
4. **Given** a researcher provides an invalid CURIE format, **When** they request "CHEMBL123" (missing colon), **Then** the system returns an error with code UNRESOLVED_ENTITY and a recovery hint to use "CHEMBL:123" format
5. **Given** a researcher requests a non-existent compound, **When** they lookup "CHEMBL:99999999", **Then** the system returns an error with code ENTITY_NOT_FOUND and suggestions for similar valid IDs
6. **Given** a researcher needs minimal token usage for batch processing, **When** they enable slim mode, **Then** the system returns core fields only (ID, name, formula) omitting verbose descriptions

---

### User Story 3 - Cross-Database Integration (Priority: P3)

As a drug discovery researcher, I need ChEMBL compound records to include standardized cross-references to other biological databases (UniProt, PDB, PubChem, DrugBank, KEGG) so that I can seamlessly navigate multi-hop workflows across databases without manual identifier translation.

**Why this priority**: This enables advanced multi-database research workflows (compound → protein → gene → disease). While not essential for basic compound lookup, it's critical for modern systems biology approaches to drug discovery.

**Independent Test**: Can be tested by retrieving any well-studied drug compound and verifying that cross_references contains mapped identifiers for UniProt targets, PDB structures, PubChem entries, DrugBank records. Delivers value by eliminating manual cross-reference lookup.

**Acceptance Scenarios**:

1. **Given** a researcher retrieves a compound with known protein targets, **When** they access the cross_references field, **Then** it contains UniProt IDs using the standard 22-key registry format
2. **Given** a researcher needs to verify structural data, **When** they retrieve a compound with known crystal structures, **Then** the cross_references field includes PDB IDs
3. **Given** a researcher works with commercial drug databases, **When** they retrieve an approved drug compound, **Then** the cross_references field includes DrugBank IDs where available
4. **Given** a compound has no cross-references for a particular database, **When** retrieving the record, **Then** that database key is omitted entirely from cross_references (never null or empty string)
5. **Given** a researcher needs pathway analysis, **When** they retrieve a metabolite compound, **Then** the cross_references field includes KEGG compound IDs where available

---

### User Story 4 - Error Recovery and Resilience (Priority: P4)

As a drug discovery researcher, when I encounter errors during compound queries (rate limits, network issues, invalid input), I need clear error messages with actionable recovery hints so that I can quickly resolve issues and continue my research without interruption.

**Why this priority**: While error handling doesn't add new functionality, it's essential for production use. Good error messages prevent user frustration and reduce support burden. This is a polish layer that makes the system production-ready.

**Independent Test**: Can be tested by triggering each error condition (invalid CURIE, malformed query, rate limit exceeded) and verifying that error envelopes contain correct error codes, clear messages, and actionable recovery hints. Delivers value by enabling self-service error resolution.

**Acceptance Scenarios**:

1. **Given** a researcher provides an invalid CURIE, **When** they attempt lookup with "CHEMBL123" (no colon), **Then** the system returns error code UNRESOLVED_ENTITY with message "Invalid ChEMBL CURIE format" and recovery hint "Use format 'CHEMBL:NNNNN' (e.g., 'CHEMBL:123')"
2. **Given** a researcher sends too many requests, **When** they exceed the rate limit, **Then** the system returns error code RATE_LIMITED with the backoff duration and recovery hint "Wait 5 seconds before retrying"
3. **Given** the ChEMBL API is temporarily unavailable, **When** a query fails due to upstream error, **Then** the system returns error code UPSTREAM_ERROR with recovery hint "ChEMBL API unavailable - retry in 60 seconds"
4. **Given** a researcher provides a query that's too short, **When** they search with 1 character, **Then** the system returns error code AMBIGUOUS_QUERY with recovery hint "Minimum query length is 2 characters"
5. **Given** a researcher encounters any error, **When** they review the error envelope, **Then** it includes the invalid_input field showing exactly what they provided, helping them identify the mistake

---

### Edge Cases

- What happens when a compound has no approved name and only systematic IUPAC nomenclature?
- How does the system handle compounds with thousands of synonyms that would exceed token budgets?
- What happens when ChEMBL API returns partial data due to a database migration or schema change?
- How does batch lookup handle a mix of valid and invalid CURIEs in the same request?
- What happens when a compound exists in ChEMBL but has zero cross-references to any other database?
- How does the system handle ChEMBL SDK version updates that change response schemas?

## Requirements *(mandatory)*

### Functional Requirements

**Architecture & Integration**

- **FR-001**: System MUST implement ChEMBLClient as a subclass of LifeSciencesClient to inherit connection pooling, rate limiting, and error handling patterns
- **FR-002**: System MUST wrap all ChEMBL SDK calls using asyncio.run_in_executor to comply with async-first architecture (ADR-001 §2 exception for synchronous SDKs)
- **FR-003**: System MUST configure rate limiting at 10 requests/second with exponential backoff to prevent API throttling
- **FR-004**: System MUST use connection pooling with maximum 10 concurrent connections to ChEMBL API

**Fuzzy-to-Fact Protocol (US1 + US2)**

- **FR-005**: System MUST provide a search_compounds tool that accepts natural language queries (minimum 2 characters) and returns ranked CompoundSearchCandidate results
- **FR-006**: System MUST validate search queries and reject queries shorter than 2 characters with error code AMBIGUOUS_QUERY and recovery hint
- **FR-007**: System MUST rank search results by relevance score based on ChEMBL's internal ranking algorithm
- **FR-008**: System MUST provide a get_compound tool that accepts ONLY ChEMBL CURIEs in format "CHEMBL:NNNNN" (e.g., "CHEMBL:25", "CHEMBL:1201583")
- **FR-009**: System MUST validate CURIE format using regex pattern `^CHEMBL:[0-9]+$` and reject invalid formats with error code UNRESOLVED_ENTITY
- **FR-010**: System MUST return complete Compound records including molecular formula, molecular weight, SMILES notation, InChI, and canonical name

**Batch Operations (Prevents Thread Pool Exhaustion)**

- **FR-011**: System MUST provide a get_compounds_batch tool that accepts a list of ChEMBL CURIEs and returns compound records in a single API call
- **FR-012**: System MUST default to slim=True for batch operations to minimize token usage when processing multiple compounds
- **FR-013**: System MUST handle mixed valid/invalid CURIEs in batch requests gracefully, returning successful lookups and error envelopes for failures

**Agentic Biolink Schema (ADR-001 §4)**

- **FR-014**: System MUST return all compound data in flattened JSON format compatible with the Agentic Biolink schema
- **FR-015**: System MUST include a cross_references object in every complete Compound record using the standardized 22-key registry (uniprot, pdb, pubchem_compound, drugbank, kegg, etc.)
- **FR-016**: System MUST omit cross-reference keys entirely when no mapping exists (never use null, empty string, or empty array)
- **FR-017**: System MUST populate cross_references.uniprot with UniProtKB CURIEs for known protein targets
- **FR-018**: System MUST populate cross_references.pdb with PDB IDs for compounds with structural data
- **FR-019**: System MUST populate cross_references.pubchem_compound with PubChem CIDs where available
- **FR-020**: System MUST populate cross_references.drugbank with DrugBank IDs for approved drugs

**Canonical Envelopes (ADR-001 §8)**

- **FR-021**: System MUST return all list results (search_compounds) wrapped in PaginationEnvelope with fields: items, pagination.cursor, pagination.total_count, pagination.page_size
- **FR-022**: System MUST implement cursor-based pagination for search results with default page_size=50 (configurable 1-100)
- **FR-023**: System MUST return all errors wrapped in ErrorEnvelope with fields: success=false, error.code, error.message, error.recovery_hint, error.invalid_input
- **FR-024**: System MUST use standard error codes: UNRESOLVED_ENTITY (invalid CURIE), ENTITY_NOT_FOUND (valid CURIE but no record), AMBIGUOUS_QUERY (query too short/broad), RATE_LIMITED (throttled), UPSTREAM_ERROR (ChEMBL API failure), INVALID_CROSS_REFERENCE (malformed xref)

**Token Budgeting (ADR-001 §5)**

- **FR-025**: System MUST support slim mode (slim=True parameter) that returns minimal fields suitable for list display (~20 tokens per compound)
- **FR-026**: System MUST include in slim mode: id, name, molecular_formula only
- **FR-027**: System MUST include in full mode (slim=False): id, name, molecular_formula, molecular_weight, smiles, inchi, canonical_name, synonyms, cross_references (~115-300 tokens per compound)

**Testing & Quality**

- **FR-028**: System MUST include pytest-asyncio integration tests covering the complete Fuzzy-to-Fact workflow (search → extract CURIE → get full record)
- **FR-029**: System MUST include unit tests for all error conditions (invalid CURIE, short query, rate limit, upstream failure)
- **FR-030**: System MUST include performance tests validating Success Criterion SC-001 (95% of queries <2s)
- **FR-031**: System MUST include cross-database integration tests verifying cross_references contain valid identifiers that can resolve in target databases (UniProt, PDB, PubChem, DrugBank)

### Key Entities *(include if feature involves data)*

- **CompoundSearchCandidate**: Lightweight search result containing id (ChEMBL CURIE), name (preferred compound name), molecular_formula, and score (relevance ranking 0.0-1.0). Used in fuzzy search results for fast browsing.

- **Compound**: Complete compound record containing id, name, molecular_formula, molecular_weight, smiles (simplified molecular-input line-entry system), inchi (International Chemical Identifier), canonical_name, synonyms (list of alternative names), cross_references (object mapping to 22-key registry). Represents the authoritative ChEMBL compound data.

- **CrossReferences**: Object mapping ChEMBL compounds to external databases using standardized keys from the 22-key registry. Core mappings: uniprot (protein targets), pdb (3D structures), pubchem_compound (chemical databases), drugbank (approved drugs), kegg (pathway databases), chembl (self-reference). Keys are omitted if no cross-reference exists.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 95% of compound search queries return results in under 2 seconds (measured from query submission to first result display)
- **SC-002**: 100% of valid ChEMBL CURIEs resolve to complete compound records within 1 second
- **SC-003**: Search results achieve >90% relevance accuracy for well-known drug names (e.g., "aspirin", "imatinib", "metformin") measured by target compound appearing in top 5 results
- **SC-004**: System successfully handles 100 concurrent search requests without degradation in response time or error rate increase
- **SC-005**: All error conditions return actionable recovery hints that enable users to self-resolve issues without external documentation in >80% of cases
- **SC-006**: Cross-reference mappings achieve >95% accuracy for well-studied compounds (validated against manual curation of 100 approved drugs)
- **SC-007**: Batch operations reduce total query time by >70% compared to sequential individual lookups (e.g., 10 compounds in batch should complete in <3s vs >10s for 10 sequential calls)
