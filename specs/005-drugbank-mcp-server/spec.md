# Feature Specification: DrugBank MCP Server

**Feature Branch**: `005-drugbank-mcp-server`
**Created**: 2025-12-22
**Status**: ⛔ Blocked (Public API Cloudflare-protected, requires API key)
**Updated**: 2025-12-23
**Linear**: Pending - child of AGE-65
**Blocker**: DrugBank public endpoint (go.drugbank.com) returns Cloudflare challenge. Commercial API (api.drugbank.com) returns 401. Contact: sales@drugbank.com
**Input**: User description: "Build the DrugBank MCP Server for drug and drug target information"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Fuzzy Drug Search (Priority: P1)

As a drug discovery researcher, I need to find drugs using common names, brand names, or therapeutic indications so that I can quickly locate candidate drugs for analysis without knowing exact DrugBank IDs.

**Why this priority**: This is the entry point for all drug discovery workflows. Without fuzzy search, users cannot begin their research journey. It enables exploration and discovery when exact identifiers are unknown.

**Independent Test**: Can be fully tested by querying "aspirin" and verifying that ranked candidates include acetylsalicylic acid with DrugBank ID DB00945. Delivers immediate value by enabling drug discovery.

**Acceptance Scenarios**:

1. **Given** a researcher knows a drug's common name, **When** they search for "metformin", **Then** the system returns ranked candidates including DrugBank:DB00331 with drug type and categories
2. **Given** a researcher enters a brand name, **When** they search for "Lipitor", **Then** the system returns candidates including the corresponding generic drug (atorvastatin)
3. **Given** a researcher needs to browse results, **When** they request the next page using a cursor, **Then** the system returns the next batch of ranked results seamlessly
4. **Given** a researcher wants quick triage, **When** they enable slim mode, **Then** each result contains minimal fields suitable for list display
5. **Given** a researcher enters ambiguous input, **When** the query matches hundreds of drugs, **Then** the system returns top 50 matches ranked by relevance with pagination support

---

### User Story 2 - Strict Drug Lookup (Priority: P2)

As a drug discovery researcher, once I have identified the correct DrugBank ID from search results, I need to retrieve the complete drug record including pharmacology, targets, interactions, and cross-references so that I can perform detailed analysis and navigate to related databases.

**Why this priority**: This is the natural second step after discovery. It transforms a search candidate into actionable data with full drug details and cross-database links. Critical for the Fuzzy-to-Fact workflow.

**Independent Test**: Can be tested by directly calling with "DrugBank:DB00945" (aspirin) and verifying complete drug record with description, categories, targets, and cross-references to ChEMBL, UniProt, KEGG, PubChem. Delivers value by providing authoritative drug data.

**Acceptance Scenarios**:

1. **Given** a researcher has a DrugBank CURIE from search results, **When** they request drug details for "DrugBank:DB00945", **Then** the system returns complete drug data including name, description, drug type, and categories
2. **Given** a researcher needs to navigate to related compounds, **When** they retrieve a drug record, **Then** the cross_references field contains ChEMBL IDs for the active compound
3. **Given** a researcher needs to find drug targets, **When** they retrieve a drug record, **Then** the cross_references field contains UniProt IDs for target proteins
4. **Given** a researcher provides an invalid CURIE format, **When** they request "DB00945" (missing prefix), **Then** the system returns an error with code UNRESOLVED_ENTITY and a recovery hint to use "DrugBank:DBXXXXX" format
5. **Given** a researcher requests a non-existent drug, **When** they lookup "DrugBank:DB99999999", **Then** the system returns an error with code ENTITY_NOT_FOUND and suggestions to verify the ID
6. **Given** a researcher needs minimal token usage, **When** they enable slim mode, **Then** the system returns core fields only (ID, name, type, categories) omitting verbose descriptions

---

### User Story 3 - Cross-Database Integration (Priority: P3)

As a drug discovery researcher, I need DrugBank drug records to include standardized cross-references to other biological databases (ChEMBL, UniProt, KEGG, PubChem) so that I can seamlessly navigate multi-hop workflows across databases without manual identifier translation.

**Why this priority**: This enables advanced multi-database research workflows (drug → compound → protein → gene → disease). While not essential for basic drug lookup, it's critical for modern systems pharmacology approaches.

**Independent Test**: Can be tested by retrieving any well-studied drug and verifying that cross_references contains mapped identifiers for ChEMBL compounds, UniProt targets, KEGG pathways, PubChem entries. Delivers value by eliminating manual cross-reference lookup.

**Acceptance Scenarios**:

1. **Given** a researcher retrieves a drug with known compound data, **When** they access the cross_references field, **Then** it contains ChEMBL IDs using the standard 22-key registry format
2. **Given** a researcher needs to find target proteins, **When** they retrieve a drug with known targets, **Then** the cross_references field includes UniProt IDs
3. **Given** a researcher works with pathway databases, **When** they retrieve a metabolized drug, **Then** the cross_references field includes KEGG compound IDs where available
4. **Given** a drug has no cross-references for a particular database, **When** retrieving the record, **Then** that database key is omitted entirely from cross_references (never null or empty string)
5. **Given** a researcher needs chemical structure data, **When** they retrieve a small molecule drug, **Then** the cross_references field includes PubChem compound IDs where available

---

### User Story 4 - Error Recovery and Resilience (Priority: P4)

As a drug discovery researcher, when I encounter errors during drug queries (rate limits, network issues, invalid input), I need clear error messages with actionable recovery hints so that I can quickly resolve issues and continue my research without interruption.

**Why this priority**: While error handling doesn't add new functionality, it's essential for production use. Good error messages prevent user frustration and reduce support burden. This is a polish layer that makes the system production-ready.

**Independent Test**: Can be tested by triggering each error condition (invalid CURIE, malformed query, rate limit exceeded) and verifying that error envelopes contain correct error codes, clear messages, and actionable recovery hints. Delivers value by enabling self-service error resolution.

**Acceptance Scenarios**:

1. **Given** a researcher provides an invalid CURIE, **When** they attempt lookup with "DB00945" (no prefix), **Then** the system returns error code UNRESOLVED_ENTITY with message "Invalid DrugBank CURIE format" and recovery hint "Use format 'DrugBank:DBXXXXX' (e.g., 'DrugBank:DB00945')"
2. **Given** a researcher sends too many requests, **When** they exceed the rate limit, **Then** the system returns error code RATE_LIMITED with the backoff duration and recovery hint "Wait {N} seconds before retrying"
3. **Given** the DrugBank API is temporarily unavailable, **When** a query fails due to upstream error, **Then** the system returns error code UPSTREAM_ERROR with recovery hint "DrugBank API unavailable - retry in 60 seconds"
4. **Given** a researcher provides a query that's too short, **When** they search with 1 character, **Then** the system returns error code AMBIGUOUS_QUERY with recovery hint "Minimum query length is 2 characters"
5. **Given** a researcher encounters any error, **When** they review the error envelope, **Then** it includes the invalid_input field showing exactly what they provided, helping them identify the mistake

---

### Edge Cases

- What happens when a drug has multiple brand names across different regions/countries?
- How does the system handle drugs with thousands of synonyms that would exceed token budgets?
- What happens when DrugBank API returns partial data due to tiered access restrictions (public vs commercial)?
- How does the system handle investigational drugs that may have limited data available?
- What happens when a drug exists in DrugBank but has zero cross-references to any other database?
- How does the system distinguish between small molecule drugs and biologic drugs (different data structures)?

## Requirements *(mandatory)*

### Functional Requirements

**Architecture & Integration**

- **FR-001**: System MUST implement DrugBankClient as a subclass of LifeSciencesClient to inherit connection pooling, rate limiting, and error handling patterns
- **FR-002**: System MUST use async httpx for all DrugBank API calls to comply with async-first architecture (ADR-001 §2)
- **FR-003**: System MUST configure rate limiting at 10 requests/second with exponential backoff to prevent API throttling
- **FR-004**: System MUST use connection pooling with maximum 10 concurrent connections to DrugBank API
- **FR-005**: System MUST gracefully handle tiered API access (public endpoints for basic data, with optional API key for enhanced access)

**Fuzzy-to-Fact Protocol (US1 + US2)**

- **FR-006**: System MUST provide a search_drugs tool that accepts natural language queries (minimum 2 characters) and returns ranked DrugSearchCandidate results
- **FR-007**: System MUST validate search queries and reject queries shorter than 2 characters with error code AMBIGUOUS_QUERY and recovery hint
- **FR-008**: System MUST rank search results by relevance score prioritizing exact name matches, then partial matches
- **FR-009**: System MUST provide a get_drug tool that accepts ONLY DrugBank CURIEs in format "DrugBank:DBXXXXX" (e.g., "DrugBank:DB00945")
- **FR-010**: System MUST validate CURIE format using regex pattern `^DrugBank:DB\d{5}$` and reject invalid formats with error code UNRESOLVED_ENTITY
- **FR-011**: System MUST return complete Drug records including name, description, drug_type (small molecule, biotech), categories, and indication

**Agentic Biolink Schema (ADR-001 §4)**

- **FR-012**: System MUST return all drug data in flattened JSON format compatible with the Agentic Biolink schema
- **FR-013**: System MUST include a cross_references object in every complete Drug record using the standardized 22-key registry (chembl, uniprot, kegg, pubchem_compound, etc.)
- **FR-014**: System MUST omit cross-reference keys entirely when no mapping exists (never use null, empty string, or empty array)
- **FR-015**: System MUST populate cross_references.chembl with ChEMBL IDs for compounds where available
- **FR-016**: System MUST populate cross_references.uniprot with UniProtKB CURIEs for known drug targets
- **FR-017**: System MUST populate cross_references.kegg with KEGG drug/compound IDs where available
- **FR-018**: System MUST populate cross_references.pubchem_compound with PubChem CIDs where available

**Canonical Envelopes (ADR-001 §8)**

- **FR-019**: System MUST return all list results (search_drugs) wrapped in PaginationEnvelope with fields: items, pagination.cursor, pagination.total_count, pagination.page_size
- **FR-020**: System MUST implement cursor-based pagination for search results with default page_size=50 (configurable 1-100)
- **FR-021**: System MUST return all errors wrapped in ErrorEnvelope with fields: success=false, error.code, error.message, error.recovery_hint, error.invalid_input
- **FR-022**: System MUST use standard error codes: UNRESOLVED_ENTITY (invalid CURIE), ENTITY_NOT_FOUND (valid CURIE but no record), AMBIGUOUS_QUERY (query too short/broad), RATE_LIMITED (throttled), UPSTREAM_ERROR (DrugBank API failure)

**Token Budgeting (ADR-001 §5)**

- **FR-023**: System MUST support slim mode (slim=True parameter) that returns minimal fields suitable for list display (~20 tokens per drug)
- **FR-024**: System MUST include in slim mode: id, name, drug_type, categories only
- **FR-025**: System MUST include in full mode (slim=False): id, name, drug_type, categories, description, indication, mechanism_of_action, cross_references (~115-300 tokens per drug)

**Testing & Quality**

- **FR-026**: System MUST include pytest-asyncio integration tests covering the complete Fuzzy-to-Fact workflow (search → extract CURIE → get full record)
- **FR-027**: System MUST include unit tests for all error conditions (invalid CURIE, short query, rate limit, upstream failure)
- **FR-028**: System MUST include performance tests validating Success Criterion SC-001 (95% of queries <2s)
- **FR-029**: System MUST include cross-database integration tests verifying cross_references contain valid identifiers that can resolve in target databases (ChEMBL, UniProt, KEGG, PubChem)

### Key Entities *(include if feature involves data)*

- **DrugSearchCandidate**: Lightweight search result containing id (DrugBank CURIE), name (preferred drug name), drug_type (small molecule, biotech), categories, and score (relevance ranking 0.0-1.0). Used in fuzzy search results for fast browsing.

- **Drug**: Complete drug record containing id, name, drug_type, categories, description (therapeutic description), indication (approved uses), mechanism_of_action, pharmacodynamics, absorption, metabolism, half_life, cross_references (object mapping to 22-key registry). Represents the authoritative DrugBank drug data.

- **CrossReferences**: Object mapping DrugBank drugs to external databases using standardized keys from the 22-key registry. Core mappings: chembl (compound data), uniprot (protein targets), kegg (pathway databases), pubchem_compound (chemical databases), drugbank (self-reference). Keys are omitted if no cross-reference exists.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 95% of drug search queries return results in under 2 seconds (measured from query submission to first result display)
- **SC-002**: 100% of valid DrugBank CURIEs resolve to complete drug records within 1 second
- **SC-003**: Search results achieve >90% relevance accuracy for well-known drug names (e.g., "aspirin", "metformin", "atorvastatin") measured by target drug appearing in top 5 results
- **SC-004**: System successfully handles 100 concurrent search requests without degradation in response time or error rate increase
- **SC-005**: All error conditions return actionable recovery hints that enable users to self-resolve issues without external documentation in >80% of cases
- **SC-006**: Cross-reference mappings achieve >95% accuracy for well-studied drugs (validated against manual curation of 100 FDA-approved drugs)
- **SC-007**: System gracefully handles DrugBank public API limitations, providing clear messaging when commercial-tier features are unavailable
