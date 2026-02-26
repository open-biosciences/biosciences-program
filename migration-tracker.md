# Migration Tracker

Migration of predecessor repos (`lifesciences-research`, `lifesciences-deepagents`, `lifesciences-temporal`) into the `open-biosciences` org structure.

## Source Repositories

| Predecessor | Path | Key Content |
|-------------|------|-------------|
| lifesciences-research | `/home/donbr/graphiti-org/lifesciences-research` | 12 MCP servers, 697 tests, 6 ADRs, 6 skills, 15 SpecKit commands, gateway, models |
| lifesciences-deepagents | `/home/donbr/ai2026/lifesciences-deepagents-worktrees/deepagents-0312-upgrade-spike` | LangGraph supervisor, 7 specialists, React UI, MCP wrappers |
| lifesciences-temporal | `/home/donbr/graphiti-org/lifesciences-temporal` | PydanticAI agents, Temporal workflows, CQ14 pipeline |
| graphiti-fastmcp | `/home/donbr/graphiti-fastmcp` | Graphiti FastMCP server (memory persistence), 9 MCP tools, async queue, Neo4j pooling |

---

## Wave 1: Foundation

**Status:** ✅ Complete (2026-02-25)
**Target Repos:** `biosciences-architecture`, `biosciences-skills`, `biosciences-program`
**Source:** `lifesciences-research` ADRs + skills + commands

### What Moves

| Item | Source Path | Target Repo | Target Path | Status |
|------|------------|-------------|-------------|--------|
| ADR-001 v1.4 (Agentic-First Architecture) | `docs/adr/accepted/adr-001-v1.4.md` | biosciences-architecture | `docs/adr/accepted/` | ✅ Complete |
| ADR-002 v1.0 (Project Skills) | `docs/adr/accepted/adr-002-v1.0.md` | biosciences-architecture | `docs/adr/accepted/` | ✅ Complete |
| ADR-003 v1.0 (SpecKit SDLC) | `docs/adr/accepted/adr-003-v1.0.md` | biosciences-architecture | `docs/adr/accepted/` | ✅ Complete |
| ADR-004 v1.0 (FastMCP Lifecycle) | `docs/adr/accepted/adr-004-v1.0.md` | biosciences-architecture | `docs/adr/accepted/` | ✅ Complete |
| ADR-005 v1.0 (Git Worktrees) | `docs/adr/accepted/adr-005-v1.0.md` | biosciences-architecture | `docs/adr/accepted/` | ✅ Complete |
| ADR-006 v1.0 (Single Writer) | `docs/adr/accepted/adr-006-v1.0.md` | biosciences-architecture | `docs/adr/accepted/` | ✅ Complete |
| Platform Engineering Rationale | `docs/platform-engineering-rationale.md` | biosciences-architecture | `docs/` | ✅ Complete |
| 6 Domain Skills | `.claude/skills/lifesciences-*` | biosciences-skills | `.claude/skills/` | ✅ Complete |
| 15 SpecKit Commands | `.claude/commands/speckit.*` | biosciences-skills | `.claude/commands/` | ✅ Complete |
| Scaffold Skills | `.claude/commands/scaffold-*` | biosciences-skills | `.claude/commands/` | ✅ Complete |
| Graphiti Skills | `.claude/commands/graphiti-*` | biosciences-skills | `.claude/commands/` | ✅ Complete |
| specs/ (13 MCP server specs) | `lifesciences-research/specs/` | biosciences-architecture | `specs/` | ✅ Complete |
| .specify/ (SpecKit config) | `lifesciences-research/.specify/` | biosciences-architecture | `.specify/` | ✅ Complete |
| speckit-standard-prompt-v2.md | `lifesciences-research/docs/` | biosciences-architecture | `docs/` | ✅ Complete |
| speckit-standard-prompt.md (v1 legacy) | `lifesciences-research/docs/` | biosciences-architecture | `docs/` | ✅ Complete |
| speckit-scaffold-process-timeline-v2.md | `lifesciences-research/docs/` | biosciences-architecture | `docs/` | ✅ Complete |

### Acceptance Criteria
- [x] All 6 ADRs present in `biosciences-architecture/docs/adr/accepted/`
- [x] All 6 domain skills present in `biosciences-skills/.claude/skills/`
- [x] All SpecKit commands present in `biosciences-skills/.claude/commands/`
- [x] Platform engineering rationale doc migrated
- [x] No broken cross-references between migrated docs

---

## Wave 2: Platform

**Status:** ✅ Complete (2026-02-25)
**Target Repos:** `biosciences-mcp`, `biosciences-memory`
**Source:** `lifesciences-research` src/ + tests/
**Depends On:** Wave 1 (ADRs must be in place for schema references)
**Commit:** `e2c2737` — biosciences-mcp
**Linear:** AGE-150 (parent), AGE-159 through AGE-167 (sub-issues) — all Done

### What Moves

| Item | Source Path | Target Repo | Target Path | Status |
|------|------------|-------------|-------------|--------|
| 12 MCP Server implementations | `src/lifesciences_mcp/servers/` | biosciences-mcp | `src/biosciences_mcp/servers/` | ✅ Complete |
| 13 Client libraries | `src/lifesciences_mcp/clients/` | biosciences-mcp | `src/biosciences_mcp/clients/` | ✅ Complete |
| Pydantic models (envelopes, entities) | `src/lifesciences_mcp/models/` | biosciences-mcp | `src/biosciences_mcp/models/` | ✅ Complete |
| Gateway server | `src/lifesciences_mcp/servers/gateway.py` | biosciences-mcp | `src/biosciences_mcp/servers/gateway.py` | ✅ Complete |
| Unit tests (~399) | `tests/unit/` | biosciences-mcp | `tests/unit/` | ✅ Complete |
| Integration tests (~294) | `tests/integration/` | biosciences-mcp | `tests/integration/` | ✅ Complete |
| E2E tests (~4) | `tests/e2e/` | biosciences-mcp | `tests/e2e/` | ✅ Complete |
| pyproject.toml | `pyproject.toml` | biosciences-mcp | `pyproject.toml` | ✅ Complete |
| Graphiti MCP config | `biosciences-memory/.mcp.json` | biosciences-memory | `.mcp.json` | ✅ Already in place |
| Environment config | `biosciences-memory/.env.example` | biosciences-memory | `.env.example` | ✅ Already in place |

### Package Rename
- `lifesciences_mcp` → `biosciences_mcp` (all imports, pyproject.toml, test references)
- `lifesciences-research` → `biosciences-mcp` (package name in pyproject.toml)

### Ruff Fixes Applied During Migration
Four lint errors resolved beyond the pure rename (tracked on AGE-160 and AGE-161):
- **B904** `clients/chembl.py:134` — Added `from None` to `raise TimeoutError(...)` in except block
- **F841** `clients/hgnc.py:156` — Removed unused `alias_hgnc_ids` variable
- **UP042** `models/envelopes.py:8,16` — `ErrorCode(str, Enum)` → `ErrorCode(StrEnum)`
- **E402** `models/gene.py:15` — Moved cross_references import above module-level constant

### Pyright Notes
45 pre-existing type errors carried from source repo (not introduced by migration). Documented and accepted. See AGE-150 comment for details.

### Acceptance Criteria
- [x] All 12 servers importable from `biosciences_mcp.servers`
- [x] `uv run pytest -m unit` passes (399+ tests)
- [x] `uv run pytest -m integration` passes (294+ tests)
- [x] Gateway mounts all 12 servers
- [x] Package rename complete (`lifesciences_mcp` → `biosciences_mcp`)
- [x] Memory repo MCP connections verified

---

## Wave 3: Orchestration

**Status:** Not Started
**Target Repos:** `biosciences-deepagents`, `biosciences-temporal`
**Source:** `lifesciences-deepagents` + `lifesciences-temporal`
**Depends On:** Wave 2 (MCP servers must be importable)

### What Moves

| Item | Source Path | Target Repo | Target Path | Status |
|------|------------|-------------|-------------|--------|
| LangGraph supervisor | `apps/api/graphs/lifesciences.py` | biosciences-deepagents | `src/` | ⬜ Not Started |
| MCP tool wrappers | `apps/api/shared/mcp.py` | biosciences-deepagents | `src/` | ⬜ Not Started |
| System prompts | `apps/api/shared/prompts.py` | biosciences-deepagents | `src/` | ⬜ Not Started |
| React chat UI | `apps/web/` | biosciences-deepagents | `apps/web/` | ⬜ Not Started |
| PydanticAI agents | `src/cq14_temporal/agents/` | biosciences-temporal | `src/` | ⬜ Not Started |
| Temporal workflows | `src/cq14_temporal/temporal/` | biosciences-temporal | `src/` | ⬜ Not Started |
| Temporal config | `src/cq14_temporal/config/` | biosciences-temporal | `src/` | ⬜ Not Started |
| Entry scripts | `src/cq14_temporal/scripts/` | biosciences-temporal | `src/` | ⬜ Not Started |
| Docker compose (Temporal + Neo4j) | `docker-compose.yml` | biosciences-temporal | `docker-compose.yml` | ⬜ Not Started |

### Key Refactoring
- Update MCP tool wrapper endpoints to point to `biosciences-mcp` gateway
- Rename `lifesciences` agent references to `biosciences`
- Update `langgraph.json` entry points

### Acceptance Criteria
- [ ] LangGraph supervisor starts on `:2024`
- [ ] React UI connects and streams responses
- [ ] Temporal worker starts and connects
- [ ] Standalone agent mode works (`run_agent.py`)
- [ ] MCP tool wrappers connect to `biosciences-mcp` servers

---

## Wave 4: Validation

> **Note:** SpecKit process documents (specs/, .specify/, speckit-*.md) were originally scoped to biosciences-research (Wave 4). They have been reassigned to biosciences-architecture (Wave 1-ext) because they are architectural governance artifacts owned by the Platform Architect, not research outputs.

**Status:** Not Started
**Target Repos:** `biosciences-evaluation`, `biosciences-research`, `biosciences-memory`
**Source:** `lifesciences-research` docs/ + `graphiti-fastmcp`
**Depends On:** Wave 2 + Wave 3 (graphiti-fastmcp usage patterns confirmed through orchestration layer)

### What Moves

| Item | Source Path | Target Repo | Target Path | Status |
|------|------------|-------------|-------------|--------|
| Competency questions catalog | `docs/competency-questions/` | biosciences-research | `docs/competency-questions/` | ⬜ Not Started |
| Research outputs | `docs/research/` | biosciences-research | `docs/research/` | ⬜ Not Started |
| Evaluation rubrics | (new) | biosciences-evaluation | `rubrics/` | ⬜ Not Started |
| Quality metrics definitions | (new) | biosciences-evaluation | `metrics/` | ⬜ Not Started |
| Reference materials | `reference/` | biosciences-research | `reference/` | ⬜ Not Started |

### graphiti-fastmcp Migration

| Item | Source | Target Repo | Target Path | Status |
|------|--------|-------------|-------------|--------|
| FastMCP server factory (`server.py`) | `graphiti-fastmcp/src/server.py` | biosciences-memory | `src/biosciences_memory/server.py` | ⬜ Not Started |
| Queue service (`queue_service.py`) | `graphiti-fastmcp/src/services/` | biosciences-memory | `src/biosciences_memory/services/` | ⬜ Not Started |
| Config schema (`schema.py`) | `graphiti-fastmcp/src/config/` | biosciences-memory | `src/biosciences_memory/config/` | ⬜ Not Started |
| Provider factories (`factories.py`) | `graphiti-fastmcp/src/services/` | biosciences-memory | `src/biosciences_memory/services/` | ⬜ Not Started |
| Response models | `graphiti-fastmcp/src/models/` | biosciences-memory | `src/biosciences_memory/models/` | ⬜ Not Started |
| Test suite (unit + integration) | `graphiti-fastmcp/tests/` | biosciences-memory | `tests/` | ⬜ Not Started |
| pyproject.toml | (new) | biosciences-memory | `pyproject.toml` | ⬜ Not Started |

**Curation steps:**
- Align to hatchling build backend
- Apply project ruff config
- Apply pytest markers (`unit`/`integration`/`e2e`)
- Pin graphiti-core version consistent with rest of platform
- Rename `graphiti_fastmcp` → `biosciences_memory` package internals
- Add biosciences-specific entity schemas (Gene, Protein, Drug, Disease, Pathway)

### Acceptance Criteria
- [ ] Competency questions catalog migrated and indexed
- [ ] Evaluation rubrics defined for each research workflow
- [ ] Quality metrics baseline established
- [ ] End-to-end validation: CQ14 runs through new org structure
- [ ] `graphiti-fastmcp` source migrated and curated into `biosciences-memory/src/`
- [ ] `biosciences-memory` has `pyproject.toml` with hatchling + ruff + pytest markers
- [ ] `uv run pytest -m unit` passes in `biosciences-memory`
- [ ] Memory layer validates end-to-end: deepagents PERSIST phase writes to Docker Neo4j
- [ ] CQ14 Temporal workflow persists research output to knowledge graph

---

## Migration Rules

1. **Copy, don't move** — Predecessor repos remain functional during migration
2. **Rename on copy** — `lifesciences_*` → `biosciences_*` in all package names and imports
3. **Test after each wave** — No wave starts until previous wave's acceptance criteria pass
4. **Track in issues** — Each migration item gets a GitHub issue in the target repo
5. **ADRs travel first** — Architectural specs must be in place before code that references them

## Timeline

| Wave | Dependencies | Estimated Effort |
|------|-------------|-----------------|
| Wave 1 (Foundation) | None | Low — mostly file copying + path updates |
| Wave 2 (Platform) | Wave 1 | High — package rename, 697 test fixes |
| Wave 3 (Orchestration) | Wave 2 | Medium — endpoint updates, agent rewiring |
| Wave 4 (Validation) | Wave 2, 3 (graphiti-fastmcp patterns confirmed) | Medium — evaluation content + graphiti-fastmcp curation + validation runs |
