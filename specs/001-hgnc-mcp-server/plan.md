# Implementation Plan: HGNC MCP Server

**Branch**: `001-hgnc-mcp-server` | **Date**: 2025-12-21 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-hgnc-mcp-server/spec.md`

## Summary

Build the foundational HGNC MCP Server implementing the Fuzzy-to-Fact protocol for
gene resolution. The server exposes two primary tools:
- `search_genes` (Fuzzy): Natural language search returning ranked candidates
- `get_gene` (Strict): CURIE-based lookup returning full Agentic Biolink entities

Technical approach uses async httpx for HGNC REST API integration, FastMCP for the
MCP server framework, and Pydantic for schema validation.

## Technical Context

**Language/Version**: Python 3.11+
**Primary Dependencies**: FastMCP, httpx, pydantic
**Storage**: N/A (stateless; live queries to HGNC REST API)
**Testing**: pytest-asyncio
**Target Platform**: MCP server (stdio/SSE transport)
**Project Type**: Single project (MCP server library)
**Performance Goals**: <500ms p95 response time for single-gene lookups
**Constraints**: Async-only I/O; respect HGNC rate limits
**Scale/Scope**: Foundation for 15+ life sciences API wrappers

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Evidence |
|-----------|--------|----------|
| I. Async-First Architecture | PASS | FR-009 mandates async I/O; httpx client |
| II. Fuzzy-to-Fact Protocol | PASS | US1 (search_genes) + US2 (get_gene) |
| III. Schema Determinism | PASS | FR-003/FR-004 specify Canonical Envelopes |
| IV. Token Budgeting | PASS | FR-007 requires slim=True support |
| V. Specification-Before-Code | PASS | This plan follows spec.md |
| VI. Platform Skill Delegation | PASS | Will use scaffold-fastmcp skill |

**Gate Status**: PASSED — No Constitution violations detected.

## Project Structure

### Documentation (this feature)

```text
specs/001-hgnc-mcp-server/
├── plan.md              # This file
├── research.md          # Phase 0: HGNC API research
├── data-model.md        # Phase 1: Pydantic models
├── quickstart.md        # Phase 1: Usage examples
├── contracts/           # Phase 1: Tool schemas
│   └── hgnc-tools.json  # MCP tool definitions
└── tasks.md             # Phase 2: Implementation tasks
```

### Source Code (repository root)

```text
src/lifesciences_mcp/
├── __init__.py
├── client.py                 # LifeSciencesClient (async httpx)
├── envelopes.py              # PaginationEnvelope, ErrorEnvelope
├── models/
│   ├── __init__.py
│   ├── gene.py               # Gene, SearchCandidate, CrossReferences
│   └── envelopes.py          # Canonical envelope Pydantic models
└── servers/
    ├── __init__.py
    └── hgnc.py               # HGNC MCP server (FastMCP)

tests/
├── conftest.py               # pytest-asyncio fixtures
├── unit/
│   ├── test_models.py        # Gene, CrossReferences tests
│   └── test_envelopes.py     # Envelope validation tests
├── contract/
│   └── test_hgnc_tools.py    # Tool schema contract tests
└── integration/
    └── test_hgnc_api.py      # Live HGNC API tests (optional)
```

**Structure Decision**: Single project layout following ADR-001 target structure.
Source code in `src/lifesciences_mcp/`, tests in `tests/` with unit/contract/integration
separation.

## Complexity Tracking

> No Constitution violations to justify.

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| (none) | — | — |
