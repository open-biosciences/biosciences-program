# Feature Specification: IUPHAR/GtoPdb MCP Server

**Feature Branch**: `011-iuphar-mcp-server`
**Created**: 2026-01-01
**Status**: Draft
**Input**: Build the IUPHAR/GtoPdb MCP Server with Fuzzy-to-Fact workflow, Agentic Biolink schema, and Canonical Envelopes.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Fuzzy Ligand Search (Priority: P1)

An LLM agent needs to find pharmacological ligands (drugs, chemicals) in the Guide to PHARMACOLOGY database using natural language queries. The agent receives ranked candidates with IUPHAR ligand IDs that can be used for strict lookups.

**Why this priority**: Ligand search is the primary entry point for drug discovery workflows. Without fuzzy search, agents cannot discover IUPHAR ligand identifiers.

**Independent Test**: Can be fully tested by searching for "ibuprofen" and verifying ranked candidates are returned with valid IUPHAR ligand IDs.

**Acceptance Scenarios**:

1. **Given** an agent searching for ligands, **When** they call `search_ligands("ibuprofen")`, **Then** they receive a PaginationEnvelope with ranked LigandSearchCandidate items containing IUPHAR IDs.
2. **Given** an agent searching by drug class, **When** they call `search_ligands("NSAID")`, **Then** they receive multiple anti-inflammatory compounds.
3. **Given** an agent searching by mechanism, **When** they call `search_ligands("COX inhibitor")`, **Then** they receive relevant enzyme inhibitors.

---

### User Story 2 - Strict Ligand Lookup (Priority: P2)

An LLM agent needs to retrieve complete ligand information using a resolved IUPHAR ligand ID. The agent receives full ligand details including approved name, type, synonyms, and cross-references to other databases.

**Why this priority**: After fuzzy discovery, strict lookup provides the detailed entity data needed for reasoning. This completes the Fuzzy-to-Fact protocol for ligands.

**Independent Test**: Can be fully tested by calling `get_ligand("IUPHAR:4139")` (ibuprofen) and verifying all required fields are populated.

**Acceptance Scenarios**:

1. **Given** an agent with a valid IUPHAR ligand ID, **When** they call `get_ligand("IUPHAR:4139")`, **Then** they receive a complete Ligand entity with approved_name, type, synonyms, and cross_references.
2. **Given** an agent with an invalid ID format, **When** they call `get_ligand("ibuprofen")` (raw string, not CURIE), **Then** they receive an UNRESOLVED_ENTITY error with recovery_hint pointing to search_ligands.
3. **Given** an agent with a non-existent ID, **When** they call `get_ligand("IUPHAR:999999")`, **Then** they receive an ENTITY_NOT_FOUND error.

---

### User Story 3 - Fuzzy Target Search (Priority: P3)

An LLM agent needs to find pharmacological targets (receptors, ion channels, enzymes, transporters) in GtoPdb using natural language queries. The agent receives ranked candidates with IUPHAR target IDs.

**Why this priority**: Target search enables receptor/enzyme discovery for mechanism-of-action reasoning.

**Independent Test**: Can be fully tested by searching for "dopamine receptor" and verifying ranked candidates are returned.

**Acceptance Scenarios**:

1. **Given** an agent searching for targets, **When** they call `search_targets("dopamine receptor")`, **Then** they receive a PaginationEnvelope with ranked TargetSearchCandidate items.
2. **Given** an agent searching by family, **When** they call `search_targets("GPCR")`, **Then** they receive G protein-coupled receptors.
3. **Given** an agent searching by gene symbol, **When** they call `search_targets("DRD2")`, **Then** they receive the dopamine D2 receptor.

---

### User Story 4 - Strict Target Lookup (Priority: P4)

An LLM agent needs to retrieve complete target information using a resolved IUPHAR target ID. The agent receives full target details including name, family, species, and cross-references.

**Why this priority**: After fuzzy discovery, strict lookup provides detailed target data for drug-target interaction reasoning.

**Independent Test**: Can be fully tested by calling `get_target("IUPHAR:214")` (Dopamine D2 receptor) and verifying all required fields are populated.

**Acceptance Scenarios**:

1. **Given** an agent with a valid IUPHAR target ID, **When** they call `get_target("IUPHAR:214")`, **Then** they receive a complete Target entity with name, family, species, and cross_references.
2. **Given** an agent with an invalid ID format, **When** they call `get_target("DRD2")` (gene symbol, not CURIE), **Then** they receive an UNRESOLVED_ENTITY error with recovery_hint pointing to search_targets.

---

### User Story 5 - Error Recovery (Priority: P5)

An LLM agent encounters errors and needs actionable guidance to self-correct. All errors include recovery hints that guide the agent to the correct tool or action.

**Why this priority**: Autonomous agents must recover from errors without human intervention.

**Independent Test**: Can be fully tested by triggering each error type and verifying recovery hints are actionable.

**Acceptance Scenarios**:

1. **Given** an agent receives UNRESOLVED_ENTITY error for ligand, **When** they follow the recovery_hint, **Then** they can call search_ligands to resolve the entity.
2. **Given** an agent receives UNRESOLVED_ENTITY error for target, **When** they follow the recovery_hint, **Then** they can call search_targets to resolve the entity.
3. **Given** an agent receives RATE_LIMITED error, **When** they follow the recovery_hint, **Then** they know to retry with backoff.

---

### Edge Cases

- What happens when search returns no results? → Empty items array with total_count: 0
- What happens when GtoPdb API is unavailable? → UPSTREAM_ERROR with recovery_hint
- What happens when rate limit is exceeded? → RATE_LIMITED error with retry guidance
- What happens when ligand has no targets? → Empty target associations
- What happens when target has no ligands? → Empty ligand associations
- What happens when entity has no cross-references? → Cross-reference keys are omitted (not null)

## Requirements *(mandatory)*

### Functional Requirements

#### Ligand Search Tool (Fuzzy Discovery)

- **FR-001**: System MUST provide `search_ligands(query, page_size?, cursor?)` tool accepting natural language queries
- **FR-002**: System MUST return LigandSearchCandidate objects with id (IUPHAR:*), name, type, and relevance score
- **FR-003**: System MUST support cursor-based pagination with default page_size of 50
- **FR-004**: System MUST wrap results in Canonical PaginationEnvelope

#### Ligand Lookup Tool (Strict Execution)

- **FR-005**: System MUST provide `get_ligand(iuphar_id)` tool accepting ONLY valid IUPHAR ligand IDs (regex: `^IUPHAR:\d+$`)
- **FR-006**: System MUST return UNRESOLVED_ENTITY error when raw strings are passed to strict tool
- **FR-007**: System MUST return ENTITY_NOT_FOUND error when valid ID format yields no results

#### Target Search Tool (Fuzzy Discovery)

- **FR-008**: System MUST provide `search_targets(query, page_size?, cursor?)` tool accepting natural language queries
- **FR-009**: System MUST return TargetSearchCandidate objects with id (IUPHAR:*), name, family, and relevance score
- **FR-010**: System MUST support cursor-based pagination with default page_size of 50
- **FR-011**: System MUST wrap results in Canonical PaginationEnvelope

#### Target Lookup Tool (Strict Execution)

- **FR-012**: System MUST provide `get_target(iuphar_id)` tool accepting ONLY valid IUPHAR target IDs (regex: `^IUPHAR:\d+$`)
- **FR-013**: System MUST return UNRESOLVED_ENTITY error when raw strings are passed to strict tool
- **FR-014**: System MUST return ENTITY_NOT_FOUND error when valid ID format yields no results

#### Schema Requirements

- **FR-015**: Ligand entities MUST include: id, approved_name, type (synthetic organic, metabolite, peptide, etc.), synonyms
- **FR-016**: Target entities MUST include: id, name, target_family, species, gene_symbol
- **FR-017**: Both entities MUST include cross_references object using 22-key registry (chembl, drugbank, uniprot, etc.)
- **FR-018**: System MUST omit cross-reference keys entirely when no reference exists (never use null)

#### Error Handling

- **FR-019**: All errors MUST use Canonical ErrorEnvelope with code, message, recovery_hint, invalid_input
- **FR-020**: Error codes MUST be from standard registry: UNRESOLVED_ENTITY, ENTITY_NOT_FOUND, AMBIGUOUS_QUERY, RATE_LIMITED, UPSTREAM_ERROR
- **FR-021**: Recovery hints MUST be actionable and specify correct tool (search_ligands vs search_targets)

#### Rate Limiting & Reliability

- **FR-022**: Client MUST implement rate limiting at 1 request/second (conservative limit for GtoPdb)
- **FR-023**: Client MUST implement exponential backoff on 429 responses
- **FR-024**: Client MUST implement thundering herd prevention for concurrent requests

### Key Entities

- **Ligand**: Pharmacological compound with approved_name, type (synthetic organic, metabolite, peptide, antibody, etc.), synonyms, and cross-references to ChEMBL, DrugBank, PubChem
- **Target**: Pharmacological target with name, target_family (GPCR, ion channel, enzyme, transporter, etc.), species, gene_symbol, and cross-references to UniProt, Ensembl, HGNC
- **LigandSearchCandidate**: Slim representation for search results with id, name, type, and relevance score
- **TargetSearchCandidate**: Slim representation for search results with id, name, family, and relevance score
- **CrossReferences**: Object with optional keys from 22-key registry mapping to external database identifiers

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 95% of search queries return results in under 2 seconds
- **SC-002**: Rate limiting prevents HTTP 429 errors during normal operation
- **SC-003**: Cross-references are populated for entities with known external database mappings
- **SC-004**: Agents can complete Fuzzy-to-Fact workflow for both ligands and targets without human intervention
- **SC-005**: Error recovery hints enable agents to self-correct in 90% of error scenarios
- **SC-006**: All integration tests pass against live GtoPdb API
- **SC-007**: Ligand types are correctly categorized (synthetic organic, peptide, antibody, etc.)
- **SC-008**: Target families are correctly identified (GPCR, ion channel, enzyme, transporter)

## Assumptions

- GtoPdb API (https://www.guidetopharmacology.org/services) remains publicly accessible without authentication
- Conservative rate limit of 1 req/s is appropriate for this community resource
- IUPHAR IDs for both ligands and targets are stable integer identifiers
- Cross-reference mappings to ChEMBL, DrugBank, UniProt are available via GtoPdb API

## Out of Scope

- Ligand-target interaction data (affinity, potency values - may be added in future iteration)
- Disease associations
- Clinical trial information
- Selectivity profiles
- Batch retrieval for multiple ligands/targets
