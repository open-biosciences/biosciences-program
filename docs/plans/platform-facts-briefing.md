# Platform Facts Briefing

**Purpose:** Shared ground truth for all README-writing agents. Do not derive migration facts independently — use this document.

**Date:** 2026-02-26
**Source of truth:** `biosciences-program/migration-tracker.md`, Linear project AGE-148 through AGE-183, Graphiti namespace `open-biosciences-migration-2026`

---

## Migration Wave Status

| Wave | Repos | Status | Completed |
|------|-------|--------|-----------|
| 1 — Foundation | biosciences-architecture, biosciences-skills, biosciences-program | ✅ Complete | 2026-02-25 |
| 1-ext | biosciences-architecture (SpecKit artifacts) | ✅ Complete | 2026-02-26 (AGE-183) |
| 2 — Platform | biosciences-mcp, biosciences-memory | ✅ Complete | 2026-02-25 |
| 3 — Orchestration | biosciences-deepagents, biosciences-temporal | ✅ Complete | 2026-02-26 |
| 4 — Validation | biosciences-evaluation, biosciences-research, biosciences-education, biosciences-workspace-template | ⬜ Not Started | — |

---

## Key Migration Decisions (Evergreen Facts)

These decisions were made during migration and must be reflected accurately in all READMEs:

1. **SpecKit artifacts live in `biosciences-program`, NOT `biosciences-research` or `biosciences-architecture`** (AGE-183, then moved to program)
   - `specs/` (13 MCP server spec dirs, 143 files)
   - `.specify/` (SpecKit config: constitution, templates, scripts — 11 files)
   - `docs/speckit-standard-prompt-v2.md`, `docs/speckit-standard-prompt.md`, `docs/speckit-scaffold-process-timeline-v2.md`
   - These are architectural governance artifacts owned by the Platform Architect

2. **`biosciences-memory` Wave 2 is complete**
   - `.mcp.json` (5 MCP server connections) already in place
   - `.env.example` already in place
   - Full graphiti-fastmcp migration is Wave 4, not Wave 2
   - The repo is functional for MCP connections; code migration is Wave 4

3. **`biosciences-mcp` package rename complete**
   - `lifesciences_mcp` → `biosciences_mcp` everywhere
   - 12 servers, 13 clients, Pydantic v2 models, gateway — all migrated
   - 697+ tests: 399 unit + 294 integration + 4 e2e
   - FastMCP 3.0.2 gateway double-prefix bug fixed (AGE-182)

4. **Platform Engineering Rationale doc lives in `biosciences-architecture/docs/`**
   - Migrated from `lifesciences-research/docs/platform-engineering-rationale.md`
   - Not in `biosciences-program`

---

## Per-Repository Authoritative Facts

### biosciences-program
- **Wave:** 1 ✅ Complete
- **Owner:** Program Director (Agent 1)
- **Content:** Coordination docs only — no code
- **Key files:** AGENTS.md, migration-tracker.md
- **README:** Already updated 2026-02-26 — do not modify

### biosciences-architecture
- **Wave:** 1 + 1-ext ✅ Complete
- **Owner:** Platform Architect (Agent 2)
- **Content:** 6 ADRs, platform-engineering-rationale.md, specs/ (13 dirs), .specify/ (11 files), 3 SpecKit process docs
- **No upstream dependencies** — pure provider
- **Downstream:** All 10 other repos read ADRs and schemas from here

### biosciences-skills
- **Wave:** 1 ✅ Complete
- **Owner:** Quality & Skills Engineer (Agent 8)
- **Content:**
  - **6 domain skills**: lifesciences-clinical, lifesciences-crispr, lifesciences-genomics, lifesciences-graph-builder, lifesciences-pharmacology, lifesciences-proteomics
  - **Note:** Scaffold commands and security-review skill moved to `platform-skills` (platform-aligned, developer-facing)
  - **Note:** The 9 SpecKit commands (speckit.*) live in `biosciences-program/.claude/commands/` — they are governance artifacts owned by Program Director
  - **Note:** The 4 Graphiti commands are installed globally in `~/.claude/skills/`
- **Upstream:** biosciences-architecture (ADR-002, ADR-003)

### platform-skills
- **Wave:** 1 ✅ Complete
- **Owner:** Quality & Skills Engineer (Agent 8)
- **Content:**
  - **2 scaffold commands**: scaffold-fastmcp, scaffold-fastmcp-v2 (in `.claude/commands/`)
  - **1 platform skill**: security-review (in `.claude/skills/`)
- **Rationale:** Platform-aligned developer skills split from domain-aligned research skills per Team Topologies
- **Upstream:** biosciences-architecture (ADR-001, ADR-002, ADR-004)

### biosciences-mcp
- **Wave:** 2 ✅ Complete
- **Owner:** MCP Platform Engineer (Agent 3)
- **Content:**
  - 12 FastMCP servers: HGNC, UniProt, ChEMBL, Open Targets, STRING, BioGRID, Ensembl, Entrez, PubChem, IUPHAR/GtoPdb, WikiPathways, ClinicalTrials.gov
  - 13 async client libraries (httpx-based)
  - Pydantic v2 models (envelopes, cross-references, domain entities)
  - Unified gateway server
  - 697+ tests (399 unit + 294 integration + 4 e2e)
  - FastMCP Cloud deployment via Prefect Horizon at `https://biosciences-mcp.fastmcp.app/mcp`
- **Server tiers:**
  - Tier 0 (Drug Discovery Core): ChEMBL, Open Targets
  - Tier 1 (Gene/Protein Foundation): HGNC, UniProt, STRING, BioGRID
  - Tier 2 (Pharmacology): IUPHAR/GtoPdb, PubChem
  - Tier 3 (Pathways & Trials): WikiPathways, ClinicalTrials.gov
  - Tier 4 (Genomics & Identifiers): Ensembl, Entrez
- **Upstream:** biosciences-architecture (schemas, ADR-001, ADR-004)
- **Downstream:** biosciences-deepagents, biosciences-temporal, biosciences-research

### biosciences-memory
- **Wave:** 2 ✅ Complete (config); Wave 4 (graphiti-fastmcp code migration)
- **Owner:** Memory Engineer (Agent 4)
- **Current state:** `.mcp.json` and `.env.example` in place; no Python code yet
- **`.mcp.json` provides 5 MCP server connections:**
  - graphiti-aura, neo4j-aura-management, neo4j-aura-cypher (cloud — write-frozen, reads OK)
  - graphiti-docker, neo4j-docker-cypher (local Docker — active)
- **Aura write-freeze policy:** Neo4j Aura is at capacity (~5k nodes). Reads are fine; no new writes.
- **Wave 4 will add:** graphiti-fastmcp server code, queue service, config schema, entity models (`biosciences_memory` package)
- **Upstream:** biosciences-architecture
- **Downstream:** biosciences-research (persists research results), biosciences-deepagents (PERSIST phase)

### biosciences-deepagents
- **Wave:** 3 ✅ Complete (commit `058fc16`)
- **Owner:** Deep Agents Engineer (Agent 5)
- **Will contain (from predecessor `lifesciences-deepagents`):**
  - LangGraph supervisor orchestrating 7 specialist subagents via `create_deep_agent()`
  - 7 specialists: anchor_specialist, enrichment_specialist, expansion_specialist, traversal_drugs_specialist, traversal_trials_specialist, validation_specialist, persistence_specialist
  - Think-Act-Observe reasoning loops with `think_tool`
  - React chat UI (Next.js 16, React 19, TypeScript, Tailwind, Radix UI)
  - Streaming subagent visualization
  - Tool approval interrupts (human-in-the-loop checkpointing)
  - MCP tool wrappers calling biosciences-mcp gateway
- **Wave 3 prerequisite:** Wave 2 (MCP servers must be importable)
- **Upstream:** biosciences-mcp, biosciences-memory

### biosciences-temporal
- **Wave:** 3 ✅ Complete (commit `73c6ebe`)
- **Owner:** Temporal Engineer (Agent 7)
- **Will contain (from predecessor `lifesciences-temporal`):**
  - PydanticAI standalone agents (testable without Temporal infrastructure)
  - Temporal.io workflow definitions wrapping agents into durable pipelines
  - Temporal activity definitions for each MCP tool call
  - Worker configuration with retry policies and timeouts
  - Docker Compose for Temporal server + Neo4j local environment
  - CQ14 pipeline — 5 phases: Anchor, Enrich, Expand, Traverse, Validate
- **Design principle:** "PydanticAI First, Temporal Second" — agents are fully testable standalone
- **Wave 3 prerequisite:** Wave 2
- **Upstream:** biosciences-mcp

### biosciences-evaluation
- **Wave:** 4 ⬜ Not Started
- **Owner:** Quality & Skills Engineer (Agent 8) — same owner as biosciences-skills
- **Will contain (new content, not migrated from predecessor):**
  - Evaluation rubrics for MCP server quality, agent performance, research output accuracy
  - Quality metrics definitions (test coverage thresholds, API response accuracy, knowledge graph completeness)
  - Cross-repo test quality standards
- **Role:** Quality gate layer — reads from all repos, no repo depends on it
- **Wave 4 prerequisite:** Waves 2 + 3

### biosciences-research
- **Wave:** 4 ⬜ Not Started
- **Owner:** Research Workflows Engineer (Agent 6)
- **Will contain (migrated from `lifesciences-research/docs/`):**
  - Competency questions catalog (CQ1–CQ14+) with re-run instructions
  - Research outputs and analysis artifacts
  - Reference materials
  - Graph-builder workflow outputs
- **CRITICAL:** SpecKit artifacts (specs/, .specify/) are NOT coming here — they went to biosciences-architecture (AGE-183)
- **Wave 4 prerequisite:** Waves 2 + 3
- **Upstream:** biosciences-mcp (graph-builder workflows call MCP tools), biosciences-memory (persists results)

### biosciences-education
- **Wave:** 4 ⬜ Not Started
- **Owner:** Education & Workspace Engineer (Agent 9)
- **Will contain (new content):**
  - Training materials for researchers new to the platform
  - Tutorials for setting up and using MCP servers
  - Onboarding guides for new contributors
  - Platform usage documentation

### biosciences-workspace-template
- **Wave:** 4 ⬜ Not Started
- **Owner:** Education & Workspace Engineer (Agent 9) — same owner as biosciences-education
- **Will contain (new content):**
  - Bootstrap scripts for initializing the Open Biosciences workspace
  - VS Code workspace configuration templates
  - Environment setup templates

---

## Cross-Repo Dependency Graph

```
biosciences-architecture  ← all repos read ADRs and schemas
        │
        ├── biosciences-skills  ← all repos consume skills and commands
        │
        ├── biosciences-mcp  ← deepagents, temporal, research consume API tools
        │       │
        │       ├── biosciences-deepagents
        │       ├── biosciences-temporal
        │       └── biosciences-research
        │
        ├── biosciences-memory  ← research, deepagents persist to knowledge graph
        │
        └── biosciences-evaluation  ← reads all repos, no repo depends on it
```

**Dependency rules:**
- `biosciences-architecture` and `biosciences-skills`: no upstream dependencies — pure providers
- `biosciences-mcp`: depends only on `biosciences-architecture`
- `biosciences-deepagents` and `biosciences-temporal`: consume `biosciences-mcp` tools
- `biosciences-memory`: consumed by `biosciences-research` and `biosciences-deepagents`
- `biosciences-evaluation`: reads from all repos but no repo depends on it

---

## Agent Ownership (9-Agent Team)

| # | Agent | Primary Repo(s) |
|---|-------|-----------------|
| 1 | Program Director | biosciences-program |
| 2 | Platform Architect | biosciences-architecture |
| 3 | MCP Platform Engineer | biosciences-mcp |
| 4 | Memory Engineer | biosciences-memory |
| 5 | Deep Agents Engineer | biosciences-deepagents |
| 6 | Research Workflows Engineer | biosciences-research |
| 7 | Temporal Engineer | biosciences-temporal |
| 8 | Quality & Skills Engineer | biosciences-evaluation, biosciences-skills |
| 9 | Education & Workspace Engineer | biosciences-education, biosciences-workspace-template |

---

## Graphiti Namespace Policies

| Namespace | MCP Server | Purpose | Writable? |
|-----------|------------|---------|-----------|
| `open-biosciences-migration-2026-priming` | graphiti-docker | Static project context | **NO — read-only** |
| `open-biosciences-migration-2026` | graphiti-docker | Working memory — active decisions | Yes |

All new memory writes go to `graphiti-docker` using the `open-biosciences-migration-2026` group_id.
Never write to `graphiti-aura` (write-frozen).

---

## README Design Principles

- Each README is the **front door** to its repo — not a technical manual
- Depth lives inside the repo; the org-level narrative lives in `biosciences-program`
- State wave status accurately: ✅ Complete or ⬜ Not Started — no ambiguity, no "in progress" hedging
- No internal paths in committed files (`/home/donbr/` must never appear)
- All cross-repo links → `https://github.com/open-biosciences/<repo-name>`
- Agent ownership clearly stated
- For Wave 3/4 repos: describe what the repo **will contain** based on the predecessor, not vague placeholders
- For Wave 1/2 repos: describe what the repo **actually contains** right now

---

## Linear Project Reference

- **Project:** Open Biosciences Migration
- **URL:** https://linear.app/agentic-wisdom/project/open-biosciences-migration-f926beb444f6
- **Wave parent issues:** AGE-149 (W1), AGE-150 (W2), AGE-151 (W3), AGE-152 (W4)
- **All Wave 1+2 sub-issues:** Done
- **All Wave 3+4 sub-issues:** Backlog
