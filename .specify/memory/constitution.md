<!--
SYNC IMPACT REPORT
==================
Version Change: 1.0.0 → 1.1.0 (Rate Limiting Amendment)

v1.1.0 Changes (2025-12-22):
- Added "Unbounded Concurrency" to Forbidden Patterns (API survival)
- Added "Client-side Rate Limiting" to Required Patterns (10 req/s + backoff)
- Rationale: Close gap between operational reality (code) and governance (docs)

v1.0.0 (2025-12-21):
- Initial Constitution with 6 Core Principles
- Forbidden Patterns, Required Patterns, Governance sections

Templates Status:
- ✅ plan-template.md - Constitution Check section compatible
- ✅ spec-template.md - Requirements format compatible
- ✅ tasks-template.md - Phase structure compatible

Follow-up TODOs: None
-->

# Life Sciences MCP Constitution

## Core Principles

### I. Async-First Architecture (NON-NEGOTIABLE)

All network I/O MUST use async patterns. Synchronous blocking calls are forbidden
in async contexts.

- Modern APIs (HGNC, UniProt, Open Targets): MUST use native `httpx` async clients
  with connection pooling
- ChEMBL ONLY: MAY use `run_in_executor` to wrap the synchronous
  `chembl_webresource_client` SDK
- Legacy wrappers MUST expose batch tools (e.g., `chembl_get_drugs_batch`) to
  prevent thread pool exhaustion

**Rationale:** Agentic concurrency requires non-blocking I/O. Synchronous SDKs
bottleneck the entire agent when one call stalls.

### II. Fuzzy-to-Fact Resolution Protocol

All entity lookups MUST follow a bi-modal workflow to prevent hallucinated mappings.

- **Phase 1 (Fuzzy Discovery):** Tools accept natural language and return ranked
  candidates or functional groups
- **Phase 2 (Strict Execution):** Tools accept ONLY resolved CURIEs
  (e.g., `HGNC:1101`, `CHEMBL25`)
- **Failure Mode:** Passing a raw string to a Strict Tool MUST return
  `UNRESOLVED_ENTITY` error with `recovery_hint` pointing to the resolve tool

**Rationale:** Agents hallucinate identifier mappings. The two-phase protocol
forces explicit resolution before high-stakes operations.

### III. Schema Determinism (NON-NEGOTIABLE)

All tool outputs MUST use the Canonical Envelopes defined in ADR-001 v1.2 §8.

**Pagination Envelope** (all list tools):
```json
{
  "items": [...],
  "pagination": {
    "cursor": "opaque_string",
    "total_count": 1540,
    "page_size": 50
  }
}
```

**Error Envelope** (all errors):
```json
{
  "success": false,
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "...",
    "recovery_hint": "...",
    "invalid_input": "..."
  }
}
```

**Cross-References:** Every entity MUST include a `cross_references` object using
the 22-key registry (ADR-001 Appendix A). Omit keys entirely if no reference
exists—never use `null` or empty string.

**Rationale:** Inconsistent schemas (e.g., `next_token` vs `cursor`) prevent
agents from reliably traversing datasets.

### IV. Token Budgeting

All batch tools MUST implement token-conscious output modes.

- `slim=True` parameter MUST be supported on all batch/list tools
  - Default (False): Full Agentic Biolink record (~115-300 tokens/entity)
  - Slim (True): Only `id`, `name`, `score` (~20 tokens/entity)
- Page size MUST default to 50; never exceed without explicit justification
- Slim mode is MANDATORY for list operations returning >10 entities

**Rationale:** Well-connected entities (e.g., TP53) flood context windows.
Token budgeting prevents context exhaustion during multi-hop reasoning.

### V. Specification-Before-Code

Non-trivial features MUST flow through the SpecKit workflow before implementation.

- `/speckit.specify` → Create structured specification
- `/speckit.plan` → Create architecture plan
- `/speckit.tasks` → Generate bounded task list
- **HUMAN GATE:** Implementation MUST NOT begin without plan approval

**Exceptions:** Trivial changes (<3 lines, single-file, obvious fix) may skip
the full workflow but MUST document the change rationale in commit message.

**Rationale:** Agents that skip specification generate code faster than humans
can review. Edge cases surface in production instead of in specs.

### VI. Platform Skill Delegation

When a Platform Skill exists for a task, it MUST be used instead of manual
code generation.

- Creating a new MCP server → Use `scaffold-fastmcp` skill
- Deploying to cloud → Use `deploy-cloud` skill with pre-flight checks
- Adding new endpoint → Use relevant scaffolding skill

**Constraint:** Direct code generation that bypasses Platform Skills creates
architectural drift. `/implement` MUST check for applicable skills before
writing files.

**Rationale:** Platform Skills encode organizational standards. Bypassing them
reinvents (incorrectly) what the organization has already standardized.

## Forbidden Patterns

The following patterns are PROHIBITED in this codebase:

| Pattern | Violation | Why Forbidden |
|---------|-----------|---------------|
| Synchronous blocking in async | `requests.get()` in async function | Blocks event loop |
| Hardcoded credentials | API keys in source | Security risk |
| Raw strings to strict tools | `get_gene("BRCA1")` without resolve | Hallucination risk |
| Null cross-references | `"uniprot": null` | Token waste; omit key instead |
| Skip specification | Code before spec for non-trivial features | Review debt |
| Bypass Platform Skills | Manual scaffolding when skill exists | Architectural drift |
| Deep JSON nesting | TRAPI-style nested responses | Agent parsing difficulty |
| **Unbounded Concurrency** | `asyncio.gather(*[client.get(id) for id in ids])` without rate limiting | HTTP 429 loops, IP bans, API access revocation |

## Required Patterns

The following patterns are MANDATORY:

| Pattern | Applies To | Enforcement |
|---------|------------|-------------|
| Canonical Pagination Envelope | All list tools | `/analyze` validation |
| Canonical Error Envelope | All error responses | `/analyze` validation |
| Cross-reference regex validation | All `cross_references` values | Runtime validation |
| Pre-flight checks | All deployments | `deploy-cloud` skill |
| Human approval gate | All non-trivial implementations | `/plan` → approval → `/implement` |
| Async httpx clients | All new API wrappers | Code review |
| slim=True support | All batch tools | Contract tests |
| **Client-side Rate Limiting** | All API clients | `_rate_limited_get()` with 10 req/s, exponential backoff, thundering herd prevention |

## Governance

This Constitution codifies the binding decisions from:
- **ADR-001 v1.2:** Agentic-First Architecture
- **ADR-002 v1.0:** Project Skills as Platform Engineering
- **ADR-003 v1.0:** SpecKit for Specification-Driven Development
- **ADR-006 v1.0:** Single Writer Package Architecture (rate limiting pattern)

### Amendment Process

1. Amendments MUST be proposed via Architecture Decision Record (ADR)
2. ADR MUST include rationale, consequences, and migration plan
3. ADR MUST be reviewed and approved before Constitution update
4. Version increment follows semantic versioning:
   - **MAJOR:** Principle removed or redefined incompatibly
   - **MINOR:** New principle or section added
   - **PATCH:** Clarification, typo, non-semantic refinement

### Compliance

- All PRs MUST pass `/analyze` validation against this Constitution
- Violations MUST be justified in Complexity Tracking section of plan.md
- Unjustified violations block merge

### Guidance Documents

For runtime development guidance, consult:
- `docs/platform-engineering-rationale.md` — Strategic context
- `docs/adr/accepted/adr-001-v1.2.md` — Technical specification
- `CLAUDE.md` — Quick reference for Claude Code

**Version**: 1.1.0 | **Ratified**: 2025-12-21 | **Last Amended**: 2025-12-22
