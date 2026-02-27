# Feature Specification: PubChem MCP Server

**Feature Branch**: `010-pubchem-mcp-server`
**Created**: 2026-01-01
**Status**: Draft
**Input**: Build the PubChem MCP Server with Fuzzy-to-Fact workflow, Agentic Biolink schema, and Canonical Envelopes.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Fuzzy Compound Search (Priority: P1)

An LLM agent needs to find chemical compounds in PubChem using natural language queries (compound names, synonyms, or chemical identifiers). The agent receives ranked candidates with PubChem CIDs that can be used for strict lookups.

**Why this priority**: This is the entry point for all PubChem interactions. Without fuzzy search, agents cannot discover compound identifiers, making all downstream operations impossible.

**Independent Test**: Can be fully tested by searching for "aspirin" and verifying ranked candidates are returned with valid PubChem CIDs.

**Acceptance Scenarios**:

1. **Given** an agent searching for compounds, **When** they call `search_compounds("aspirin")`, **Then** they receive a PaginationEnvelope with ranked CompoundSearchCandidate items containing PubChem CIDs.
2. **Given** an agent searching with a chemical name, **When** they call `search_compounds("acetylsalicylic acid")`, **Then** they receive candidates including aspirin.
3. **Given** an agent searching with partial name, **When** they call `search_compounds("ibuprofen")`, **Then** they receive ibuprofen and related compounds.

---

### User Story 2 - Strict Compound Lookup (Priority: P2)

An LLM agent needs to retrieve complete compound information using a resolved PubChem CID. The agent receives full compound details including SMILES, InChI, molecular formula, weight, IUPAC name, and cross-references to other databases.

**Why this priority**: After fuzzy discovery, strict lookup provides the detailed entity data needed for reasoning. This completes the Fuzzy-to-Fact protocol.

**Independent Test**: Can be fully tested by calling `get_compound("PubChem:CID2244")` (aspirin) and verifying all required fields are populated.

**Acceptance Scenarios**:

1. **Given** an agent with a valid PubChem CID, **When** they call `get_compound("PubChem:CID2244")`, **Then** they receive a complete Compound entity with SMILES, InChI, formula, weight, and cross_references.
2. **Given** an agent with an invalid ID format, **When** they call `get_compound("aspirin")` (raw string, not CURIE), **Then** they receive an UNRESOLVED_ENTITY error with recovery_hint pointing to search_compounds.
3. **Given** an agent with a non-existent ID, **When** they call `get_compound("PubChem:CID999999999999")`, **Then** they receive an ENTITY_NOT_FOUND error.

---

### User Story 3 - Cross-Database Integration (Priority: P3)

An LLM agent needs to link PubChem compounds to other databases (ChEMBL, DrugBank, UniProt targets). The agent can use cross-references to triangulate information across sources.

**Why this priority**: Cross-references enable multi-hop reasoning and evidence triangulation, a core capability for drug discovery workflows.

**Independent Test**: Can be fully tested by calling `get_compound("PubChem:CID2244")` and verifying cross-references to ChEMBL and DrugBank are populated.

**Acceptance Scenarios**:

1. **Given** an agent with a compound entity, **When** they examine cross_references, **Then** they find ChEMBL ID (if available).
2. **Given** an agent with a drug compound, **When** they examine cross_references, **Then** they find DrugBank ID (if available).

---

### User Story 4 - Error Recovery (Priority: P4)

An LLM agent encounters errors and needs actionable guidance to self-correct. All errors include recovery hints that guide the agent to the correct tool or action.

**Why this priority**: Autonomous agents must recover from errors without human intervention. This enables reliable multi-hop reasoning.

**Independent Test**: Can be fully tested by triggering each error type and verifying recovery hints are actionable.

**Acceptance Scenarios**:

1. **Given** an agent receives UNRESOLVED_ENTITY error, **When** they follow the recovery_hint, **Then** they can call search_compounds to resolve the entity.
2. **Given** an agent receives RATE_LIMITED error, **When** they follow the recovery_hint, **Then** they know to retry with backoff.
3. **Given** an agent receives UPSTREAM_ERROR, **When** they examine the error, **Then** they understand the PubChem API is temporarily unavailable.

---

### Edge Cases

- What happens when search returns no results? → Empty items array with total_count: 0
- What happens when PubChem API is unavailable? → UPSTREAM_ERROR with recovery_hint
- What happens when rate limit is exceeded? → RATE_LIMITED error with retry guidance
- What happens when compound has no cross-references? → Cross-reference keys are omitted (not null)
- What happens when searching by SMILES string? → Treated as text search, returns matching compounds
- What happens when CID is very large (>1 billion)? → Valid if exists in PubChem

## Requirements *(mandatory)*

### Functional Requirements

#### Search Tool (Fuzzy Discovery)

- **FR-001**: System MUST provide `search_compounds(query, page_size?, cursor?)` tool accepting natural language queries
- **FR-002**: System MUST return CompoundSearchCandidate objects with id (PubChem:CID*), name, formula, and relevance score
- **FR-003**: System MUST support cursor-based pagination with default page_size of 50
- **FR-004**: System MUST wrap results in Canonical PaginationEnvelope
- **FR-005**: System MUST handle compound name synonyms in search

#### Lookup Tool (Strict Execution)

- **FR-006**: System MUST provide `get_compound(pubchem_id)` tool accepting ONLY valid PubChem CIDs (regex: `^PubChem:CID\d+$`)
- **FR-007**: System MUST return UNRESOLVED_ENTITY error when raw strings are passed to strict tool
- **FR-008**: System MUST return ENTITY_NOT_FOUND error when valid ID format yields no results

#### Schema Requirements

- **FR-009**: Compound entities MUST include: id, name, iupac_name, molecular_formula, molecular_weight
- **FR-010**: Compound entities MUST include structure representations: canonical_smiles, isomeric_smiles, inchi, inchikey
- **FR-011**: Compound entities MUST include cross_references object using 22-key registry (chembl, drugbank, etc.)
- **FR-012**: System MUST omit cross-reference keys entirely when no reference exists (never use null)
- **FR-013**: Molecular weight MUST be in Daltons (g/mol)

#### Error Handling

- **FR-014**: All errors MUST use Canonical ErrorEnvelope with code, message, recovery_hint, invalid_input
- **FR-015**: Error codes MUST be from standard registry: UNRESOLVED_ENTITY, ENTITY_NOT_FOUND, AMBIGUOUS_QUERY, RATE_LIMITED, UPSTREAM_ERROR
- **FR-016**: Recovery hints MUST be actionable (e.g., "Call search_compounds to resolve entity first")

#### Rate Limiting & Reliability

- **FR-017**: Client MUST implement rate limiting at 5 requests/second
- **FR-018**: Client MUST respect 400 requests/minute limit
- **FR-019**: Client MUST implement exponential backoff on 429 responses
- **FR-020**: Client MUST implement thundering herd prevention for concurrent requests

### Key Entities

- **Compound**: Chemical compound with name, IUPAC name, molecular formula, molecular weight, SMILES (canonical and isomeric), InChI, InChIKey, and cross-references to ChEMBL, DrugBank
- **CompoundSearchCandidate**: Slim representation for search results with id, name, formula, and relevance score
- **CrossReferences**: Object with optional keys from 22-key registry mapping to external database identifiers

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 95% of search queries return results in under 2 seconds
- **SC-002**: Rate limiting prevents HTTP 429 errors during normal operation
- **SC-003**: Cross-references are populated for compounds with known external database mappings (ChEMBL, DrugBank)
- **SC-004**: Agents can complete Fuzzy-to-Fact workflow (search → select → get) without human intervention
- **SC-005**: Error recovery hints enable agents to self-correct in 90% of error scenarios
- **SC-006**: All integration tests pass against live PubChem PUG REST API
- **SC-007**: SMILES, InChI, and InChIKey are correctly populated for all compounds

## Assumptions

- PubChem PUG REST API (https://pubchem.ncbi.nlm.nih.gov/rest/pug) remains publicly accessible without authentication
- Rate limits are 5 req/s and 400 req/min
- PubChem CID format is stable (integer identifier)
- Cross-reference mappings to ChEMBL and DrugBank are available via PubChem compound properties

## Out of Scope

- Structure similarity search (2D/3D fingerprints)
- Substructure search (SMARTS patterns)
- Bioassay data retrieval
- Patent information
- Batch compound retrieval (may be added in future iteration)
- 3D conformer data
