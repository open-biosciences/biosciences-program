# Platform Health Review & Strategy

**Date:** 2026-02-28
**Author:** Program Director (Agent 1)
**Method:** 3-agent parallel audit (Codebase Health, Project Tracking, Architecture & Integration)

---

## 1. Executive Summary

The Open Biosciences platform is at **80% migration completion** (41 of 51 Linear issues Done) with Waves 1-3 fully complete and verified. The platform spans 13 documented repos plus one undocumented repo (`biosciences-rag-pipeline`), with `biosciences-mcp` serving as the dominant codebase (104 Python files, 701 tests). Architecture integrity is strong: all `.mcp.json` files point to the correct endpoint, ADR governance is fully compliant, and naming conventions are consistent across the 3 core platform repos.

The primary risks are test coverage gaps in orchestration repos (`biosciences-temporal` has 23 Python files and zero tests, `biosciences-deepagents` has 5 files and zero tests), broken Python environments in two repos due to an `opentelemetry` dependency conflict, and a latent concurrency bug (AGE-196) in the JSON-RPC transport. Wave 4 has significant work already done (competency questions, research docs, graphiti-fastmcp) but Linear tracking has drifted — 3 sub-issues appear complete but remain in Backlog status.

This strategy document defines 5 prioritized phases to complete the migration, harden test coverage, resolve integration issues, and position the platform for production readiness.

---

## 2. Migration Status

### Wave Completion

| Wave | Repos | Status | Completed |
|------|-------|--------|-----------|
| 1 (Foundation) | architecture, skills, program | ✅ Complete | 2026-02-25 |
| 2 (Platform) | mcp, memory | ✅ Complete | 2026-02-25 |
| 3 (Orchestration) | deepagents, temporal | ✅ Complete | 2026-02-26 |
| 4 (Validation) | evaluation, research, memory | 🟡 In Progress | — |

### Linear Burndown

| Status | Count | Percentage |
|--------|-------|------------|
| Done | 41 | 80% |
| Backlog | 10 | 20% |
| In Progress | 0 | 0% |
| **Total** | **51** | |

### Remaining Backlog Items

| Issue | Title | Actual State | Strategy Phase |
|-------|-------|-------------|----------------|
| AGE-152 | [Wave 4] Validation — Evaluation + Research | Superseded by AGE-195 | Housekeeping |
| AGE-176 | Migrate competency questions catalog | Done (commit `3ce399e`) | Housekeeping |
| AGE-177 | Migrate research outputs | Done (commit `13c64b0`) | Housekeeping |
| AGE-178 | Migrate reference materials | N/A (no source dir) | Housekeeping |
| AGE-179 | Create evaluation rubrics | Not Started | Phase 1 |
| AGE-180 | Create quality metrics definitions | Not Started | Phase 1 |
| AGE-181 | End-to-end validation: CQ14 through new org | Not Started | Phase 1 |
| AGE-195 | Wave 4 parent (detailed tracker) | In Progress (partially complete) | Phase 1 |
| AGE-196 | HTTPMCPClient concurrency bug | Not Started | Phase 3 |
| AGE-198 | Skill deduplication policy | Not Started | Phase 4 |

---

## 3. Codebase Health Matrix

### Per-Repo Scorecard

| Repo | Agent | .py Files | Tests | Ruff Issues | Commits (since 02-25) | Last Commit | Health |
|------|-------|-----------|-------|-------------|----------------------|-------------|--------|
| biosciences-mcp | 3 | 104 | 701 (399 unit) | 11 | 13 | 2026-02-26 | Strong |
| biosciences-memory | 4 | 23 | 26 (24 unit) | 0 | 20 | 2026-02-27 | Strong |
| biosciences-temporal | 7 | 23 | 0 | 17 | 10 | 2026-02-26 | At Risk |
| biosciences-architecture | 2 | 13 | 0 (env broken) | 5 | 9 | 2026-02-27 | At Risk |
| biosciences-deepagents | 5 | 5 | 0 (env broken) | 1 | 14 | 2026-02-26 | At Risk |
| biosciences-program | 1 | 0 | — | — | 28 | 2026-02-27 | Healthy |
| biosciences-research | 6 | 0 | — | — | 5 | 2026-02-27 | Healthy |
| biosciences-evaluation | 8 | 0 | — | — | 3 | 2026-02-26 | Scaffold |
| biosciences-skills | 8 | 0 | — | — | 13 | 2026-02-28 | Healthy |
| biosciences-education | 9 | 0 | — | — | 3 | 2026-02-26 | Healthy |
| biosciences-workspace-template | 9 | 0 | — | — | 7 | 2026-02-28 | Healthy |
| platform-skills | 8 | 0 | — | — | 2 | 2026-02-27 | Healthy |
| marketplace | 8 | 0 | — | — | 13 | 2026-02-28 | Healthy |

### Config File Presence

| Repo | CLAUDE.md | .mcp.json | .env.example | pyproject.toml |
|------|-----------|-----------|--------------|----------------|
| biosciences-mcp | Y | N | Y | Y |
| biosciences-memory | Y | Y | Y | Y |
| biosciences-temporal | Y | Y | Y | Y |
| biosciences-architecture | Y | N | N | Y |
| biosciences-deepagents | Y | Y | Y | Y |
| biosciences-program | Y | Y | Y | N |
| biosciences-research | Y | N | N | N |
| biosciences-evaluation | Y | N | N | N |
| biosciences-skills | Y | Y | Y | N |
| biosciences-education | Y | N | N | N |
| biosciences-workspace-template | Y | N | N | N |
| platform-skills | Y | Y | N | N |
| marketplace | Y | N | N | N |

### Dependency Versions

| Repo | fastmcp | graphiti-core | pydantic | openai |
|------|---------|---------------|----------|--------|
| biosciences-mcp | >=2.14.1,<3.0 | — | >=2.0 | — |
| biosciences-memory | >=2.13.3,<3 | ==0.24.3 | >=2.0.0 | >=1.91.0 |
| biosciences-temporal | — | — | >=2.0 | via pydantic-ai |
| biosciences-architecture | — | — | — | via langchain-openai |
| biosciences-deepagents | — | — | — | via langchain-openai |

---

## 4. Architecture & Integration Health

### Stale Reference Audit

| Search Term | Status | Details |
|-------------|--------|---------|
| `lifesciences-research.fastmcp.app` | **2 runtime bugs** | `.env` files in biosciences-memory and biosciences-program still contain old endpoint on line 42. `.env.example` files are correct. |
| `lifesciences_mcp` | Clean | Zero matches in any source file |
| `cq14_temporal` | Clean | Zero matches in any source file |
| `LIFESCIENCES_RESEARCH_PATH` | Clean | Zero matches in any source file |
| `lifesciences-research` (in docs) | Expected | Historical references in migration-tracker.md and archive docs only |

### .mcp.json Alignment

All 8 `.mcp.json` files across the workspace correctly point to `biosciences-mcp.fastmcp.app`. Port assignments are consistent: graphiti-docker=8002, neo4j-aura-cypher=8003, neo4j-aura-management=8004, neo4j-docker-cypher=8005. One minor naming inconsistency: `biosciences-rag-pipeline` uses server key `biosciences-gateway` instead of `biosciences-mcp`.

### ADR Compliance

| Check | Expected | Found | Status |
|-------|----------|-------|--------|
| ADR count in biosciences-program | 6 | 6 | PASS |
| ADR-001 version | v1.4 | v1.4 | PASS |
| Spec directories | 13 | 13 | PASS |
| .specify/constitution.md | exists | exists (at memory/ subdir) | PASS |
| SpecKit commands | 9 | 9 | PASS |

### Dependency Graph Integrity

| Check | Status |
|-------|--------|
| biosciences-program: no Python code | PASS |
| biosciences-architecture: no governance artifacts | PASS |
| biosciences-architecture: has RA framework (ra_agents/, ra_orchestrators/, ra_tools/) | PASS |
| biosciences-skills: no upstream code dependencies | PASS |

### Naming Consistency

| Repo | pyproject.toml name | Python package | Status |
|------|---------------------|----------------|--------|
| biosciences-mcp | `biosciences-mcp` | `biosciences_mcp` | Match |
| biosciences-memory | `biosciences-memory` | `biosciences_memory` | Match |
| biosciences-temporal | `biosciences-temporal` | `biosciences_temporal` | Match |
| biosciences-architecture | `biosciences-architecture` | `ra_*` (RA framework) | Explained deviation |
| biosciences-deepagents | `biosciences-deepagents` | `apps/api/` (LangGraph) | Explained deviation |

---

## 5. Knowledge Graph Health

### Docker Neo4j (Active Development)

| Metric | Value |
|--------|-------|
| Total Nodes | 7,180 |
| Total Edges | 24,094 |
| Top Node Labels | Entity (6,717), Episodic (463) |
| Top Edge Types | MENTIONS (13,863), RELATES_TO (10,231) |
| Entity Subtypes | Document (1,146), Requirement (1,064), Topic (752), Organization (493), Event (424), Procedure (330), Object (246), Preference (226), Location (220), Bare Entity (1,816) |

#### Namespace Health

| Namespace | Nodes | Edges | Episodes | Status |
|-----------|-------|-------|----------|--------|
| Priming (`open-biosciences-migration-2026-priming`) | 104 | 577 | 8 | Healthy (read-only) |
| Working (`open-biosciences-migration-2026`) | 306 | 1,398 | 38 | Healthy (active) |

#### Entity Type Registry vs Graph

| Registry Type | In Graph? | Count |
|---------------|-----------|-------|
| Document | Yes | 1,146 |
| Requirement | Yes | 1,064 |
| Topic | Yes | 752 |
| Organization | Yes | 493 |
| Event | Yes | 424 |
| Procedure | Yes | 330 |
| Object | Yes | 246 |
| Preference | Yes | 226 |
| Location | Yes | 220 |
| Gene | No | — |
| Protein | No | — |
| Drug | No | — |
| Disease | No | — |
| Pathway | No | — |

The 9 generic Graphiti entity types are fully populated. The 5 biosciences-specific domain types are registered in code but await data ingestion through the biosciences-memory FastMCP server.

### Aura Neo4j (Production — Write-Frozen)

| Metric | Value |
|--------|-------|
| Total Nodes | 5,058 |
| Total Edges | 19,102 |
| Top Labels | Entity (4,625), Episodic (433) |
| Top Edge Types | MENTIONS (10,917), RELATES_TO (8,185) |
| Capacity | ~1GB free-tier limit — write-frozen |

No Open Biosciences migration data in Aura — all new data correctly routes to Docker per the write-freeze policy.

---

## 6. Risk Register

| # | Risk | Severity | Owner | Source | Mitigation |
|---|------|----------|-------|--------|------------|
| R1 | biosciences-temporal: 23 .py files, 0 tests | **Critical** | Agent 7 (Temporal) | Agent A | Write unit tests for agents, activities, and worker; add integration tests for MCP tool calls |
| R2 | biosciences-deepagents: broken Python environment (opentelemetry conflict) | **High** | Agent 5 (Deep Agents) | Agent A | Resolve opentelemetry-sdk version conflict; pin compatible versions |
| R3 | biosciences-architecture: broken Python environment (same conflict) | **High** | Agent 2 (Platform Architect) | Agent A | Same fix as R2; repos share identical pyproject.toml (likely copied) |
| R4 | biosciences-deepagents: 5 .py files, 0 tests | **High** | Agent 5 (Deep Agents) | Agent A | Write tests for LangGraph supervisor and MCP client wrapper |
| R5 | AGE-196: HTTPMCPClient `id:1` concurrency bug | **High** | Agent 7 (Temporal) + Agent 5 (Deep Agents) | Agent B | Investigate shared transport; implement atomic ID generation or SDK-native transport |
| R6 | Stale `.env` files with old endpoint | **Medium** | Agent 4 (Memory) + Agent 1 (Program) | Agent C | Update `FASTMCP_CLOUD_ENDPOINT` in both `.env` files to `biosciences-mcp.fastmcp.app` |
| R7 | Linear tracking drift: 3 sub-issues done but still Backlog | **Medium** | Agent 1 (Program) | Agent B | Review AGE-176, AGE-177, AGE-178; mark Done where appropriate; close AGE-152 in favor of AGE-195 |
| R8 | biosciences-temporal: 17 ruff lint issues | **Medium** | Agent 7 (Temporal) | Agent A | Run `ruff check --fix . && ruff format .` |
| R9 | biosciences-architecture: 13 .py files, 0 tests | **Medium** | Agent 2 (Platform Architect) | Agent A | Write tests for Repository Analyzer Framework (ra_agents, ra_orchestrators, ra_tools) |
| R10 | Undocumented repo: biosciences-rag-pipeline (14th repo) | **Medium** | Agent 1 (Program) | Agent C | Document in workspace CLAUDE.md, assign agent owner, or mark experimental |
| R11 | Wave 3 sub-issues never created in Linear | **Low** | Agent 1 (Program) | Agent B | Retroactively create and close, or document as accepted tracking gap |
| R12 | 5 biosciences domain entity types not yet in graph | **Low** | Agent 4 (Memory) | Agent C | Expected — will populate when biosciences-memory FastMCP server processes research data |
| R13 | biosciences-mcp: 11 ruff lint issues | **Low** | Agent 3 (MCP Platform) | Agent A | Run `ruff check --fix .` to resolve E402/F841 issues |
| R14 | AGE-198: Skill deduplication policy undefined | **Low** | Agent 8 (Quality & Skills) | Agent B | Document canonical skill locations; establish update propagation policy |
| R15 | fastmcp lower-bound drift (2.14.1 vs 2.13.3) | **Low** | Agent 3 (MCP Platform) + Agent 4 (Memory) | Agent A | Align lower bounds when next updating either pyproject.toml |
| R16 | Duplicate pyproject.toml in architecture + deepagents | **Low** | Agent 2 + Agent 5 | Agent C | Tailor dependency lists to each repo's actual needs |

---

## 7. Strategy: Prioritized Phases

### Phase 1: Complete Wave 4 (Target: Week of 2026-03-03)

**Goal:** Close out the migration. All 51 Linear issues Done.

| Task | Owner | Linear | Depends On |
|------|-------|--------|------------|
| Create evaluation rubrics in biosciences-evaluation | Agent 8 (Quality & Skills) | AGE-179 | — |
| Create quality metrics definitions in biosciences-evaluation | Agent 8 (Quality & Skills) | AGE-180 | — |
| Execute graphiti-fastmcp 14-task implementation plan | Agent 4 (Memory) | AGE-195 | — |
| End-to-end validation: CQ14 through new org structure | Agent 7 (Temporal) + Agent 5 (Deep Agents) | AGE-181 | AGE-179, AGE-180 |
| Memory layer E2E: deepagents PERSIST → Docker Neo4j | Agent 4 (Memory) + Agent 5 (Deep Agents) | AGE-195 | graphiti-fastmcp |
| CQ14 Temporal workflow → knowledge graph persistence | Agent 7 (Temporal) + Agent 4 (Memory) | AGE-195 | graphiti-fastmcp |
| Linear housekeeping: close AGE-176, AGE-177, AGE-178; close AGE-152 | Agent 1 (Program) | multiple | — |

### Phase 2: Test Coverage Hardening (Target: Week of 2026-03-10)

**Goal:** Every repo with Python code has meaningful test coverage. All environments buildable.

| Task | Owner | Priority | Depends On |
|------|-------|----------|------------|
| Fix opentelemetry dependency conflict in biosciences-deepagents | Agent 5 (Deep Agents) | High | — |
| Fix opentelemetry dependency conflict in biosciences-architecture | Agent 2 (Platform Architect) | High | — |
| Write unit tests for biosciences-temporal (agents, activities, worker) | Agent 7 (Temporal) | Critical | — |
| Write unit tests for biosciences-deepagents (supervisor, MCP client) | Agent 5 (Deep Agents) | High | opentelemetry fix |
| Write unit tests for biosciences-architecture (RA framework) | Agent 2 (Platform Architect) | Medium | opentelemetry fix |
| Resolve ruff lint issues in biosciences-temporal (17 issues) | Agent 7 (Temporal) | Medium | — |
| Resolve ruff lint issues in biosciences-mcp (11 issues) | Agent 3 (MCP Platform) | Low | — |
| Resolve ruff lint issues in biosciences-architecture (5 issues) | Agent 2 (Platform Architect) | Low | opentelemetry fix |

### Phase 3: Integration & Bug Fixes (Target: Week of 2026-03-17)

**Goal:** Cross-repo integrations verified, known bugs resolved.

| Task | Owner | Linear | Depends On |
|------|-------|--------|------------|
| Investigate & fix AGE-196 HTTPMCPClient concurrency bug | Agent 7 (Temporal) + Agent 5 (Deep Agents) | AGE-196 | Phase 2 test coverage |
| Fix stale `.env` files (lifesciences-research.fastmcp.app) | Agent 4 (Memory) + Agent 1 (Program) | — | — |
| Cross-repo integration smoke tests (MCP → deepagents → temporal → memory) | Agent 3 (MCP Platform) | — | Phase 2 |
| Document biosciences-rag-pipeline or mark experimental | Agent 1 (Program) | — | — |
| Tailor pyproject.toml dependencies for architecture + deepagents | Agent 2 + Agent 5 | — | opentelemetry fix |

### Phase 4: Production Readiness (Target: Week of 2026-03-24)

**Goal:** Platform operational posture suitable for real workloads.

| Task | Owner | Depends On |
|------|-------|------------|
| Define Aura capacity resolution strategy (upgrade, migrate, or archive) | Agent 4 (Memory) + Agent 1 (Program) | Phase 1 graphiti-fastmcp |
| Populate biosciences domain entity types in Docker Neo4j (Gene, Protein, Drug, Disease, Pathway) | Agent 4 (Memory) | Phase 1 graphiti-fastmcp |
| Document skill deduplication policy (AGE-198) | Agent 8 (Quality & Skills) | — |
| Align fastmcp lower bounds across mcp + memory repos | Agent 3 (MCP Platform) + Agent 4 (Memory) | — |
| Add missing .env.example files to architecture + temporal repos | Agent 2 (Platform Architect) + Agent 7 (Temporal) | — |
| Retroactively reconcile Wave 3 Linear tracking gap | Agent 1 (Program) | — |

### Phase 5: Forward Roadmap (Target: 2026-04 onward)

**Goal:** Platform evolution beyond migration.

| Task | Owner | Notes |
|------|-------|-------|
| New MCP servers (DrugBank unblocking, additional data sources) | Agent 3 (MCP Platform) | DrugBank has 33 tests but is blocked on API access |
| Skill evolution: biosciences-* skill updates, marketplace sync | Agent 8 (Quality & Skills) | Coordinate with AGE-198 deduplication policy |
| Education content: tutorials, onboarding notebooks | Agent 9 (Education & Workspace) | Depends on stable platform from Phase 4 |
| Research workflow code: graph-builder src/ in biosciences-research | Agent 6 (Research Workflows) | Scoped in AGE-195 but deferred to post-migration |
| CI/CD pipeline for automated testing across repos | Agent 1 (Program) + Agent 3 (MCP Platform) | Requires Phase 2 test coverage first |
| Monitoring & observability for MCP server fleet | Agent 3 (MCP Platform) | Production readiness capability |
| Aura write-unfreeze or migration to self-hosted production Neo4j | Agent 4 (Memory) | Depends on Phase 4 capacity strategy |

---

## 8. Agent Ownership Summary

| Agent | # | Strategic Tasks | Phases |
|-------|---|-----------------|--------|
| Program Director | 1 | Linear housekeeping, document rag-pipeline, Wave 3 tracking reconciliation, CI/CD coordination | 1, 3, 4, 5 |
| Platform Architect | 2 | Fix opentelemetry env, write RA framework tests, resolve ruff, tailor pyproject.toml, add .env.example | 2, 3, 4 |
| MCP Platform Engineer | 3 | Resolve ruff issues, cross-repo smoke tests, align fastmcp versions, DrugBank unblocking, CI/CD, monitoring | 2, 3, 4, 5 |
| Memory Engineer | 4 | graphiti-fastmcp implementation, fix stale .env, populate domain entities, Aura capacity strategy | 1, 3, 4, 5 |
| Deep Agents Engineer | 5 | Fix opentelemetry env, write supervisor tests, fix AGE-196 concurrency bug, tailor pyproject.toml | 2, 3 |
| Research Workflows Engineer | 6 | Graph-builder workflow code in biosciences-research | 5 |
| Temporal Engineer | 7 | Write temporal tests, resolve ruff, CQ14 E2E validation, fix AGE-196, add .env.example | 1, 2, 3, 4 |
| Quality & Skills Engineer | 8 | Evaluation rubrics, quality metrics, skill deduplication policy, marketplace sync | 1, 4, 5 |
| Education & Workspace Engineer | 9 | Education content, tutorials, onboarding notebooks | 5 |

---

## Appendix: Data Sources

This strategy was produced by dispatching 3 specialist agents in parallel:

- **Agent A (Codebase Health Auditor):** Scanned all 13 repos for file counts, test counts, lint status, commit history, config presence, and dependency versions.
- **Agent B (Project Tracking Analyst):** Queried Linear for all 51 issues, Graphiti Docker for priming/working namespace health, and Neo4j Aura/Docker for graph statistics.
- **Agent C (Architecture & Integration Reviewer):** Searched for stale references, verified .mcp.json alignment, ADR compliance, dependency graph integrity, naming consistency, and Neo4j schema alignment.

All data collected 2026-02-28. Agent findings were merged, cross-referenced, and synthesized into this document by the Program Director.
