# Phase 0 Research: HGNC MCP Server

**Feature**: 001-hgnc-mcp-server
**Date**: 2025-12-21
**Status**: Complete

## Research Questions

### Q1: HGNC REST API Endpoints and Response Format

**Decision**: Use the HGNC REST API at `https://rest.genenames.org/` with JSON responses.

**Rationale**:
- Official, well-documented API maintained by HGNC (genenames.org)
- Provides both Search (fuzzy) and Fetch (strict) endpoints
- JSON format is compact and directly parseable
- No API key required for basic usage

**Alternatives Considered**:
- BioMart: More complex query builder, not suited for simple lookups
- NCBI Entrez: Would require additional mapping from HGNC IDs
- Local database: Adds operational complexity, data freshness concerns

### Q2: Search vs Fetch Endpoints for Fuzzy-to-Fact

**Decision**: Map the Fuzzy-to-Fact protocol directly to HGNC endpoints:
- `search_genes` → HGNC `/search/` endpoint (wildcards, Boolean operators)
- `get_gene` → HGNC `/fetch/hgnc_id/` endpoint (exact CURIE lookup)

**Rationale**:
- `/search/` returns `hgnc_id`, `symbol`, `score` — perfect for ranked candidates
- `/fetch/` returns complete gene record with all cross-references
- Natural alignment with FR-001 (fuzzy) and FR-002 (strict)

**Alternatives Considered**:
- Use `/fetch/symbol/` for search: Less flexible, no ranking
- Custom fuzzy matching: Unnecessary, HGNC search handles wildcards

### Q3: Rate Limiting Strategy

**Decision**: Implement polite client behavior with 10 req/s limit.

**Rationale**:
- HGNC documentation explicitly requests "10 requests per second" maximum
- Exceeding may result in 403 errors and IP blocking
- For batch operations, implement request queuing with backoff

**Implementation**:
- Use `httpx` with connection pooling (single client instance)
- Add exponential backoff on 429/403 errors
- Return RATE_LIMITED error envelope when throttled

### Q4: Cross-Reference Mapping

**Decision**: Map HGNC response fields to ADR-001 cross-reference keys.

**Rationale**:
- HGNC provides direct links to Ensembl, UniProt, Entrez, RefSeq
- Fields are well-documented and stable
- Supports triangulation requirement (SC-006)

**Mapping Table**:

| HGNC Field | ADR-001 Key | Notes |
|------------|-------------|-------|
| hgnc_id | hgnc | Format: "HGNC:1100" |
| ensembl_gene_id | ensembl_gene | Format: "ENSG00000012048" |
| uniprot_ids | uniprot | List of accessions |
| entrez_id | entrez | Numeric string |
| refseq_accession | refseq | List of accessions |
| omim_id | omim | Numeric string |

### Q5: Async HTTP Client

**Decision**: Use `httpx.AsyncClient` with connection pooling.

**Rationale**:
- Constitution Principle I mandates async-first architecture
- `httpx` is modern, async-native, well-maintained
- Connection pooling reduces latency for repeated requests
- Supports HTTP/2 if needed

**Alternatives Considered**:
- `aiohttp`: Also valid, but `httpx` has cleaner API
- `requests` with `run_in_executor`: Violates Constitution (sync in async)

### Q6: FastMCP Integration Pattern

**Decision**: Use FastMCP `@mcp.tool()` decorators for tool registration.

**Rationale**:
- FastMCP is the standard MCP framework for Python
- Decorator pattern is clean and declarative
- Built-in validation and error handling
- Supports both stdio and SSE transports

**Implementation Pattern**:
```python
mcp = FastMCP("HGNC Gene Server")

@mcp.tool()
async def search_genes(query: str, slim: bool = False) -> dict:
    """Fuzzy search for genes by name, symbol, or description."""
    ...

@mcp.tool()
async def get_gene(hgnc_id: str) -> dict:
    """Get complete gene record by HGNC CURIE."""
    ...
```

### Q7: Pagination Implementation

**Decision**: Implement client-side pagination over HGNC results.

**Rationale**:
- HGNC `/search/` returns all matches (no server-side pagination)
- We cap at 50 items per page per FR-010
- Use index-based cursor encoding (base64 of offset)

**Implementation**:
- Fetch all results from HGNC (up to reasonable limit)
- Slice into pages of 50
- Return opaque cursor = base64(offset)
- Handle cursor in subsequent requests

### Q8: Error Handling Strategy

**Decision**: Map HGNC errors to Canonical Error Envelope.

**Rationale**:
- FR-004 requires structured error responses
- Recovery hints enable agent self-correction
- Consistent error codes across all life sciences servers

**Error Mapping**:

| Condition | Error Code | Recovery Hint |
|-----------|------------|---------------|
| Raw string to get_gene | UNRESOLVED_ENTITY | "Call search_genes first" |
| HGNC ID not found | ENTITY_NOT_FOUND | "Verify HGNC ID format" |
| Empty/short query | AMBIGUOUS_QUERY | "Provide at least 2 characters" |
| HGNC 429/403 | RATE_LIMITED | "Retry after {n} seconds" |
| HGNC 500/502 | UPSTREAM_ERROR | "HGNC API unavailable" |

## Resolved Clarifications

All technical decisions are resolved. No NEEDS CLARIFICATION markers remain.

## Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| fastmcp | ^2.0 | MCP server framework |
| httpx | ^0.27 | Async HTTP client |
| pydantic | ^2.0 | Data validation |
| pytest-asyncio | ^0.23 | Async test support |

## References

- [HGNC REST API Documentation](https://www.genenames.org/help/rest/)
- [FastMCP Documentation](https://gofastmcp.com/)
- [ADR-001 v1.2](../../docs/adr/accepted/adr-001-v1.2.md)
