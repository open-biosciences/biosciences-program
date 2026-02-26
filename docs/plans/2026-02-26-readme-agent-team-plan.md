# README Agent Team Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Update all 10 remaining open-biosciences repo READMEs to be factually accurate, grounded in current migration state, and free of stale placeholder text.

**Architecture:** One `general-purpose` agent per repo, all dispatched in parallel in a single message. Each agent reads a shared platform facts briefing plus its repo-specific sources, writes a grounded README, commits, and records the update in Graphiti working memory.

**Tech Stack:** Claude Code general-purpose agents, Graphiti Docker MCP (`open-biosciences-migration-2026`), Git

---

## Pre-flight Checklist

Before dispatching agents, verify these files exist:
- `biosciences-program/docs/plans/platform-facts-briefing.md` ✅
- `biosciences-program/migration-tracker.md` ✅

Briefing path (absolute): `/home/donbr/open-biosciences/biosciences-program/docs/plans/platform-facts-briefing.md`

---

## Task 1: Dispatch all 10 README agents in parallel

**Dispatch as a single message with 10 simultaneous Task tool calls.**

All agents share this base instruction set, with repo-specific additions per agent:

> **Base instructions for every agent:**
> 1. Read the platform facts briefing at `/home/donbr/open-biosciences/biosciences-program/docs/plans/platform-facts-briefing.md` — this is your authoritative source of truth
> 2. Search Graphiti working memory for your repo: `mcp__graphiti-docker__search_nodes(query="<REPO_NAME>", group_ids=["open-biosciences-migration-2026"])`
> 3. Read the repo's existing README.md and CLAUDE.md
> 4. Read any repo-specific source files listed below
> 5. Write the updated README.md — grounded in facts, no internal paths, all links to `https://github.com/open-biosciences/<repo-name>`
> 6. Commit: `docs: update README with current platform facts` + `Refs: AGE-<N>`
> 7. Add a Graphiti episode: `mcp__graphiti-docker__add_memory(name="README updated: <repo>", episode_body="README.md updated on 2026-02-26. Key facts captured: <summary>", group_id="open-biosciences-migration-2026")`

---

### Agent 1 — biosciences-memory

**Repo:** `/home/donbr/open-biosciences/biosciences-memory`

**Key correction:** README currently says "Pending Wave 2 migration" — Wave 2 is **complete**. Config files are already in place. Full graphiti-fastmcp code migration is Wave 4, not Wave 2.

**Extra files to read:**
- `/home/donbr/open-biosciences/biosciences-memory/.mcp.json`
- `/home/donbr/open-biosciences/biosciences-memory/.env.example`

**README must accurately state:**
- Wave 2 complete (config in place)
- 5 MCP server connections via `.mcp.json`: graphiti-aura, neo4j-aura-management, neo4j-aura-cypher (cloud, write-frozen), graphiti-docker, neo4j-docker-cypher (local Docker, active)
- Aura write-freeze policy: reads OK, no new writes
- Wave 4 will add: graphiti-fastmcp server code → `biosciences_memory` Python package
- Owner: Memory Engineer (Agent 4)

**Commit refs:** `Refs: AGE-150`

---

### Agent 2 — biosciences-mcp

**Repo:** `/home/donbr/open-biosciences/biosciences-mcp`

**Key correction:** README carries stale content from pre-migration source repo. Needs to reflect actual migrated state.

**Extra files to read:**
- `/home/donbr/open-biosciences/biosciences-mcp/pyproject.toml`
- List contents of `/home/donbr/open-biosciences/biosciences-mcp/src/biosciences_mcp/servers/`

**README must accurately state:**
- Wave 2 complete
- Package name: `biosciences_mcp` (renamed from `lifesciences_mcp`)
- 12 servers, 13 clients, Pydantic v2 models, gateway — all migrated
- 697+ tests: 399 unit + 294 integration + 4 e2e
- FastMCP Cloud endpoint: `https://biosciences-mcp.fastmcp.app/mcp`
- Tier table (0–4) with all 12 APIs named
- Fuzzy-to-Fact protocol (ADR-001 §3)
- Owner: MCP Platform Engineer (Agent 3)

**Commit refs:** `Refs: AGE-150`

---

### Agent 3 — biosciences-architecture

**Repo:** `/home/donbr/open-biosciences/biosciences-architecture`

**Key correction:** README should accurately describe SpecKit artifacts (specs/, .specify/, 3 process docs) migrated in Wave 1-ext (AGE-183).

**Extra files to read:**
- List contents of `/home/donbr/open-biosciences/biosciences-architecture/docs/`
- List contents of `/home/donbr/open-biosciences/biosciences-architecture/specs/` (just top-level count)

**README must accurately state:**
- Wave 1 + Wave 1-ext complete
- 6 ADRs in `docs/adr/accepted/`
- `platform-engineering-rationale.md` in `docs/`
- SpecKit artifacts: `specs/` (13 MCP server spec dirs, 143 files), `.specify/` (11 files), 3 SpecKit process docs in `docs/`
- No upstream dependencies — pure provider
- All 10 other repos read ADRs and schemas from here
- Owner: Platform Architect (Agent 2)

**Commit refs:** `Refs: AGE-149, AGE-183`

---

### Agent 4 — biosciences-skills

**Repo:** `/home/donbr/open-biosciences/biosciences-skills`

**Key correction:** Verify counts and names are accurate.

**Extra files to read:**
- List `/home/donbr/open-biosciences/biosciences-skills/.claude/skills/`
- List `/home/donbr/open-biosciences/biosciences-skills/.claude/commands/`

**README must accurately state:**
- Wave 1 complete
- Exact 6 skill names
- Exact 15 command names grouped: 9 SpecKit + 4 Graphiti + 2 scaffold
- Usage: consumed by all repos via Claude Code
- Owner: Quality & Skills Engineer (Agent 8)

**Commit refs:** `Refs: AGE-149`

---

### Agent 5 — biosciences-deepagents

**Repo:** `/home/donbr/open-biosciences/biosciences-deepagents`

**Key correction:** Generic "What's Coming" placeholders → grounded description drawn from predecessor.

**Extra files to read:**
- `/home/donbr/ai2026/lifesciences-deepagents-worktrees/deepagents-0312-upgrade-spike/README.md`

**README must accurately state:**
- Wave 3 not started (honest about status)
- What the repo will contain when migrated:
  - LangGraph supervisor + 7 named specialist subagents
  - Think-Act-Observe reasoning via `think_tool`
  - Streaming React chat UI (Next.js 16, React 19, TypeScript)
  - Tool approval interrupts (human-in-the-loop)
  - MCP tool wrappers calling biosciences-mcp gateway
- 7-phase Fuzzy-to-Fact protocol (ANCHOR → ENRICH → EXPAND → TRAVERSE → VALIDATE → PERSIST)
- Upstream: biosciences-mcp, biosciences-memory
- Owner: Deep Agents Engineer (Agent 5)

**Commit refs:** `Refs: AGE-151`

---

### Agent 6 — biosciences-temporal

**Repo:** `/home/donbr/open-biosciences/biosciences-temporal`

**Key correction:** Generic placeholders → grounded description from predecessor.

**Extra files to read:**
- `/home/donbr/graphiti-org/lifesciences-temporal/README.md` (if it exists, else use briefing)

**README must accurately state:**
- Wave 3 not started
- What the repo will contain:
  - PydanticAI standalone agents (testable without Temporal)
  - Temporal.io workflow + activity definitions
  - CQ14 pipeline: 5 phases (Anchor, Enrich, Expand, Traverse, Validate)
  - Docker Compose for Temporal server + Neo4j
  - Worker configuration with retry policies
- Design principle: "PydanticAI First, Temporal Second"
- Upstream: biosciences-mcp
- Owner: Temporal Engineer (Agent 7)

**Commit refs:** `Refs: AGE-151`

---

### Agent 7 — biosciences-evaluation

**Repo:** `/home/donbr/open-biosciences/biosciences-evaluation`

**Key correction:** Placeholder needs to be grounded in what this repo's role actually is.

**Extra files to read:** None beyond briefing + existing README + CLAUDE.md

**README must accurately state:**
- Wave 4 not started
- Role: quality gate layer — reads from all repos, no repo depends on it
- Will contain (new content, not migrated):
  - Evaluation rubrics for MCP server quality, agent performance, research output accuracy
  - Quality metrics definitions (test coverage thresholds, response accuracy, graph completeness)
  - Cross-repo quality standards
- Owner: Quality & Skills Engineer (Agent 8) — same agent owns biosciences-skills
- Wave 4 prerequisite: Waves 2 + 3

**Commit refs:** `Refs: AGE-152`

---

### Agent 8 — biosciences-research

**Repo:** `/home/donbr/open-biosciences/biosciences-research`

**Key correction:** Must NOT reference SpecKit artifacts (moved to biosciences-architecture per AGE-183). Scope is competency questions and research workflows.

**Extra files to read:**
- `/home/donbr/graphiti-org/lifesciences-research/docs/competency-questions/competency-questions-catalog.md` (first 30 lines to understand scope)

**README must accurately state:**
- Wave 4 not started
- Will contain (migrated from `lifesciences-research/docs/`):
  - Competency questions catalog (CQ1–CQ14+) with re-run instructions
  - Research outputs and analysis artifacts
  - Reference materials and graph-builder outputs
- **No SpecKit content** — that lives in biosciences-architecture
- Research scenarios: ARID1A synthetic lethality, drug repurposing, FOP (ACVR1 pathway), etc.
- Upstream: biosciences-mcp (MCP tools), biosciences-memory (persists results)
- Owner: Research Workflows Engineer (Agent 6)

**Commit refs:** `Refs: AGE-152`

---

### Agent 9 — biosciences-education

**Repo:** `/home/donbr/open-biosciences/biosciences-education`

**Key correction:** Thin placeholder — needs grounded description of purpose and scope.

**Extra files to read:** None beyond briefing + existing README + CLAUDE.md

**README must accurately state:**
- Wave 4 not started
- Will contain:
  - Training materials for researchers new to the platform
  - Step-by-step tutorials for using MCP servers and agent workflows
  - Onboarding guides for new contributors
  - Platform usage examples and walkthroughs
- Owner: Education & Workspace Engineer (Agent 9)
- Related: biosciences-workspace-template (same owner, complementary scope)

**Commit refs:** `Refs: AGE-152`

---

### Agent 10 — biosciences-workspace-template

**Repo:** `/home/donbr/open-biosciences/biosciences-workspace-template`

**Key correction:** Thin placeholder — needs grounded description of bootstrap/template scope.

**Extra files to read:** List `/home/donbr/open-biosciences/biosciences-workspace-template/` (top level)

**README must accurately state:**
- Wave 4 not started
- Will contain:
  - Bootstrap scripts for initializing the full Open Biosciences workspace
  - VS Code workspace configuration templates
  - Environment setup templates (`.env.example` patterns)
  - Workspace onboarding automation
- Owner: Education & Workspace Engineer (Agent 9) — same agent owns biosciences-education
- Related: biosciences-education (same owner, training materials)

**Commit refs:** `Refs: AGE-152`

---

## Task 2: Verify all 10 READMEs were committed

After all agents complete, run in `open-biosciences/` parent directory:

```bash
for repo in biosciences-memory biosciences-mcp biosciences-architecture biosciences-skills \
            biosciences-deepagents biosciences-temporal biosciences-evaluation \
            biosciences-research biosciences-education biosciences-workspace-template; do
  echo "=== $repo ==="
  cd /home/donbr/open-biosciences/$repo && git log --oneline -1
done
```

Expected: Each repo shows a recent "docs: update README" commit.

**Also verify no internal paths leaked:**

```bash
for repo in biosciences-memory biosciences-mcp biosciences-architecture biosciences-skills \
            biosciences-deepagents biosciences-temporal biosciences-evaluation \
            biosciences-research biosciences-education biosciences-workspace-template; do
  grep -l "/home/donbr" /home/donbr/open-biosciences/$repo/README.md && echo "FAIL: $repo has internal path" || echo "OK: $repo"
done
```

Expected: All 10 print `OK`.

---

## Task 3: Record completion in Graphiti working memory

After verification passes, add one summary episode:

```python
mcp__graphiti-docker__add_memory(
  name="README audit complete — all 10 repos updated 2026-02-26",
  episode_body="All 10 open-biosciences repo READMEs updated on 2026-02-26 to reflect current migration state. Key corrections: biosciences-memory no longer says 'Pending Wave 2'; biosciences-research SpecKit refs removed (AGE-183); biosciences-mcp post-migration staleness resolved; Wave 3/4 repos grounded with predecessor content. Briefing doc at biosciences-program/docs/plans/platform-facts-briefing.md.",
  group_id="open-biosciences-migration-2026"
)
```

---

## Execution Note

This plan uses `superpowers:dispatching-parallel-agents` for Task 1. All 10 agents run simultaneously. There are no dependencies between agents — each operates in its own repo with its own git context.
