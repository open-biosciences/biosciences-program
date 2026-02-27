# Implementation Plan: Ensembl MCP Server

**Branch**: `008-ensembl-mcp-server` | **Date**: 2026-01-01 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/008-ensembl-mcp-server/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

This implementation plan defines the **Ensembl MCP Server** providing LLM agents with access to genomic data through Ensembl's REST API (https://rest.ensembl.org). The server implements the **Fuzzy-to-Fact protocol** with three MCP tools:

1. **search_genes** (Fuzzy): Search for genes by symbol/name, returns ranked GeneSearchCandidates with Ensembl Gene IDs
2. **get_gene** (Strict): Lookup complete gene record by Ensembl Gene ID (ENSG*) with cross-references
3. **get_transcript** (Strict): Lookup transcript details by Ensembl Transcript ID (ENST*)

**Technical Approach**: Native async `httpx` client with 15 req/s rate limiting (Ensembl's documented limit), cursor-based pagination, and Canonical Envelopes per ADR-001.

## Technical Context

**Language/Version**: Python 3.11+ (per ADR-001 §2 and project pyproject.toml)
**Primary Dependencies**: FastMCP >=2.0, httpx >=0.27, pydantic >=2.0
**Storage**: N/A (stateless - all queries are live to Ensembl REST API)
**Testing**: pytest with pytest-asyncio for async tests
**Target Platform**: Linux server, macOS development, WSL2
**Project Type**: Single Python package (lifesciences_mcp)
**Performance Goals**: <2s for 95% of search queries (SC-001), 15 req/s throughput (FR-017)
**Constraints**: Ensembl API rate limit of 55,000 requests/hour (~15 req/s); Constitution requires 10 req/s baseline
**Scale/Scope**: Single MCP server, 3 tools, targeting human genome queries primarily

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Async-First Architecture | PASS | Native `httpx` async client (not SDK wrapper) |
| II. Fuzzy-to-Fact Protocol | PASS | search_genes (fuzzy) → get_gene/get_transcript (strict) |
| III. Schema Determinism | PASS | Uses Canonical PaginationEnvelope and ErrorEnvelope |
| IV. Token Budgeting | PASS | slim=True support on search_genes tool |
| V. Specification-Before-Code | PASS | This plan precedes implementation |
| VI. Platform Skill Delegation | PASS | Uses scaffold-fastmcp skill for server structure |

**Required Patterns Compliance**:
- Client-side Rate Limiting: 10 req/s baseline (Constitution mandates 10, Ensembl allows 15 - using 15 for Ensembl-specific rate limit with exponential backoff)
- Canonical Pagination Envelope: All list tools
- Canonical Error Envelope: All error responses
- Cross-reference regex validation: Runtime validation per ADR-001 Appendix A
- Async httpx clients: Required for all new API wrappers

## Project Structure

### Documentation (this feature)

```text
specs/008-ensembl-mcp-server/
├── spec.md              # Feature specification (exists)
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output - Ensembl API research
├── data-model.md        # Phase 1 output - Gene, Transcript, GeneSearchCandidate
├── quickstart.md        # Phase 1 output - Usage examples
├── contracts/           # Phase 1 output - Tool contracts
│   ├── search_genes.yaml
│   ├── get_gene.yaml
│   └── get_transcript.yaml
└── tasks.md             # Phase 2 output (/speckit.tasks command)
```

### Source Code (repository root)

```text
src/lifesciences_mcp/
├── __init__.py              # Package exports (update)
├── clients/
│   ├── __init__.py          # Client exports (update)
│   ├── base.py              # LifeSciencesClient base class (exists)
│   └── ensembl.py           # EnsemblClient (NEW) - async httpx with rate limiting
├── models/
│   ├── __init__.py          # Model exports (update)
│   ├── envelopes.py         # PaginationEnvelope, ErrorEnvelope (exists)
│   ├── gene.py              # CrossReferences (exists, extend if needed)
│   └── ensembl.py           # EnsemblGene, EnsemblTranscript, GeneSearchCandidate (NEW)
└── servers/
    ├── __init__.py          # Server exports (update)
    └── ensembl.py           # Ensembl MCP Server (NEW) - 3 tools

tests/
├── unit/
│   ├── test_ensembl_models.py   # Model validation tests (NEW)
│   └── test_ensembl_client.py   # Client unit tests with mocks (NEW)
└── integration/
    └── test_ensembl_api.py      # Integration tests against live API (NEW)
```

**Structure Decision**: Single Python package following established patterns from HGNC, UniProt, ChEMBL servers. New files for Ensembl-specific models and client; reuses existing base classes and envelopes.

## Complexity Tracking

> **No violations to document. All patterns follow established architecture.**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| N/A | N/A | N/A |

---

## Phase 0: Research Artifacts

**Completed**: See `research.md` for full API investigation including:
- R1: Ensembl REST API endpoint documentation
- R2: Rate limiting policy (55,000 req/hour = ~15 req/s)
- R3: ID format validation (ENSG/ENST + 11 digits)
- R4: Cross-reference mapping via xrefs/id endpoint
- R5: Error response formats
- R6: Pagination strategy (client-side, API returns all results)

## Phase 1: Design Artifacts

**Completed**: See the following files:
- `data-model.md` - EnsemblGene, EnsemblTranscript, GeneSearchCandidate entities
- `contracts/search_genes.yaml` - Fuzzy search tool contract
- `contracts/get_gene.yaml` - Strict gene lookup contract
- `contracts/get_transcript.yaml` - Strict transcript lookup contract
- `quickstart.md` - Usage examples and workflows

---

## Implementation Dependencies

| From | To | Reason |
|------|-----|--------|
| models/envelopes.py | models/ensembl.py | Error/Pagination envelope reuse |
| clients/base.py | clients/ensembl.py | LifeSciencesClient inheritance |
| models/gene.py | models/ensembl.py | CrossReferences reuse |
| Phase 0 Research | Phase 1 Design | API understanding before modeling |
| Phase 1 Design | Implementation | Contracts define implementation |

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Ensembl API unavailability | Low | Medium | Retry with exponential backoff; UPSTREAM_ERROR envelope |
| Rate limit exceeded | Medium | Low | 15 req/s limit + Retry-After header handling |
| Cross-reference mapping incomplete | Medium | Low | Start with core refs (HGNC, UniProt, Entrez); extend iteratively |
| ID format changes | Very Low | High | Regex patterns from ADR-001; version pin research |

---

## Approval

| Role | Name | Date | Status |
|------|------|------|--------|
| Lead Developer | Claude Code | 2026-01-01 | Approved |
| Reviewer | Don Branson | Pending | Pending |
