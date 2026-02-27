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
| 9 SpecKit Commands | `.claude/commands/speckit.*` | biosciences-program | `.claude/commands/` | ✅ Complete |
| Scaffold Skills | `.claude/commands/scaffold-*` | platform-skills | `.claude/commands/` | ✅ Complete |
| Graphiti Skills | `.claude/commands/graphiti-*` | `~/.claude/skills/` (global) | Global install | ✅ Complete |
| Security Review Skill | `.claude/skills/security-review/` | platform-skills | `.claude/skills/security-review/` | ✅ Complete |
| specs/ (13 MCP server specs) | `lifesciences-research/specs/` | biosciences-program | `specs/` | ✅ Complete |
| .specify/ (SpecKit config) | `lifesciences-research/.specify/` | biosciences-program | `.specify/` | ✅ Complete |
| speckit-standard-prompt-v2.md | `lifesciences-research/docs/` | biosciences-program | `docs/` | ✅ Complete |
| speckit-standard-prompt.md (v1 legacy) | `lifesciences-research/docs/` | biosciences-program | `docs/` | ✅ Complete |
| speckit-scaffold-process-timeline-v2.md | `lifesciences-research/docs/` | biosciences-program | `docs/` | ✅ Complete |

### Acceptance Criteria
- [x] All 6 ADRs present in `biosciences-architecture/docs/adr/accepted/`
- [x] All 6 domain skills present in `biosciences-skills/.claude/skills/`
- [x] All SpecKit commands present in `biosciences-program/.claude/commands/`
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

**Status:** ✅ Complete (2026-02-26)
**Target Repos:** `biosciences-deepagents`, `biosciences-temporal`
**Source:** `lifesciences-deepagents` + `lifesciences-temporal`
**Depends On:** Wave 2 (MCP servers must be importable)
**Commits:** `058fc16`, `090c6ac` — biosciences-deepagents | `73c6ebe`, `d7ef863` — biosciences-temporal

### What Moves

| Item | Source Path | Target Repo | Target Path | Status |
|------|------------|-------------|-------------|--------|
| LangGraph supervisor | `apps/api/lifesciences.py` | biosciences-deepagents | `apps/api/biosciences.py` | ✅ Complete |
| MCP tool wrappers | `apps/api/shared/mcp.py` | biosciences-deepagents | `apps/api/shared/mcp.py` | ✅ Complete |
| System prompts | `apps/api/shared/prompts.py` | biosciences-deepagents | `apps/api/shared/prompts.py` | ✅ Complete |
| React chat UI | `apps/web/` | biosciences-deepagents | `apps/web/` | ✅ Complete |
| 9 Agent skill folders | `.deepagents/skills/lifesciences-*` | biosciences-deepagents | `.deepagents/skills/biosciences-*` | ✅ Complete |
| PydanticAI agents | `src/cq14_temporal/agents/` | biosciences-temporal | `src/biosciences_temporal/agents/` | ✅ Complete |
| Temporal workflows | `src/cq14_temporal/temporal/` | biosciences-temporal | `src/biosciences_temporal/temporal/` | ✅ Complete |
| Temporal config | `src/cq14_temporal/config/` | biosciences-temporal | `src/biosciences_temporal/config/` | ✅ Complete |
| Entry scripts | `src/cq14_temporal/scripts/` | biosciences-temporal | `src/biosciences_temporal/scripts/` | ✅ Complete |
| Docker compose | `docker-compose.yml` | biosciences-temporal | `docker-compose.yml` | ✅ Complete |

### .mcp.json Propagation (Workstream 4)

| Repo | Action | Status |
|------|--------|--------|
| biosciences-deepagents | Replace `lifesciences-research` → `biosciences-mcp` | ✅ Complete |
| biosciences-temporal | Create new `.mcp.json` with `biosciences-mcp` + graphiti entries | ✅ Complete |
| biosciences-memory | Add `biosciences-mcp` HTTP entry | ✅ Complete |
| biosciences-program | Add `biosciences-mcp` HTTP entry | ✅ Complete |
| open-biosciences/ (workspace root) | Add `biosciences-mcp` HTTP entry | ✅ Complete |

### Key Refactoring Applied
- MCP URL updated: `lifesciences-research.fastmcp.app` → `biosciences-mcp.fastmcp.app`
- Package rename: `cq14_temporal` → `biosciences_temporal`
- Env var rename: `LIFESCIENCES_RESEARCH_PATH` → `BIOSCIENCES_MCP_PATH`
- Env var rename: `LIFESCIENCES_MCP_URL` → `BIOSCIENCES_MCP_URL`
- Task queue rename: `cq14-task-queue` → `biosciences-task-queue`
- `langgraph.json` graph key: `"lifesciences"` → `"biosciences"`, entrypoint → `biosciences.py`
- Skill folder renames: all 9 `lifesciences-*` → `biosciences-*` in `.deepagents/skills/`
- Architecture preserved: UnsandboxedWorkflowRunner, one-shot MCP client pattern

### Acceptance Criteria
- [x] `langgraph.json` entrypoint resolves to `apps/api/biosciences.py:graph`
- [x] No references to `lifesciences-research.fastmcp.app` remain in source files
- [x] All 9 skill folders renamed to `biosciences-*` in `.deepagents/skills/`
- [x] No references to `cq14_temporal` or `lifesciences-research` remain in `src/`
- [x] `biosciences-mcp` HTTP entry present in all 5 `.mcp.json` locations
- [x] `uv run langgraph dev` starts on port `:2024` (verified 2026-02-27)
- [x] `uv run python -m src.biosciences_temporal.scripts.run_agent --gene-a BRCA1 --gene-b TP53` runs without import errors (verified 2026-02-27)
- [x] `docker compose up -d` starts Temporal on ports 7233/8233 (verified 2026-02-27)
- [x] `yarn` installs + dev server runs on `:3000` in `apps/web/` (verified 2026-02-27)

---

## Wave 4: Validation

> **Note:** SpecKit process documents (specs/, .specify/, speckit-*.md) were originally scoped to biosciences-research (Wave 4). They have been reassigned to biosciences-architecture (Wave 1-ext) because they are architectural governance artifacts owned by the Platform Architect, not research outputs.

**Status:** 🟡 In Progress (2026-02-27)
**Target Repos:** `biosciences-evaluation`, `biosciences-research`, `biosciences-memory`
**Source:** `lifesciences-research` docs/ + `graphiti-fastmcp`
**Depends On:** Wave 2 + Wave 3 (graphiti-fastmcp usage patterns confirmed through orchestration layer)
**Linear:** AGE-195 (parent)
**Commits:** `3ce399e` — biosciences-research (competency questions, 72 files) | `13c64b0` — biosciences-research (research docs, 34 files)

### What Moves

| Item | Source Path | Target Repo | Target Path | Status |
|------|------------|-------------|-------------|--------|
| Competency questions catalog | `docs/competency-questions/` | biosciences-research | `docs/competency-questions/` | ✅ Complete |
| Research outputs | `docs/research/` + `docs/research-reports/` | biosciences-research | `docs/research/` + `docs/research-reports/` | ✅ Complete |
| Evaluation docs | `docs/evaluation/` | biosciences-research | `docs/evaluation/` | ✅ Complete |
| Research scenarios | `docs/scenarios/` | biosciences-research | `docs/scenarios/` | ✅ Complete |
| Evaluation rubrics | (new) | biosciences-evaluation | `rubrics/` | ⬜ Not Started |
| Quality metrics definitions | (new) | biosciences-evaluation | `metrics/` | ⬜ Not Started |
| Reference materials | `reference/` | biosciences-research | `reference/` | ⬜ N/A (no reference/ dir in source) |

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
- [x] Competency questions catalog migrated and indexed
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
