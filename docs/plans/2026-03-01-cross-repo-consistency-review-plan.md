# Cross-Repo Consistency Review — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Fix documentation drift across 11 READMEs and 10 diagrams, then create 3 new architectural diagrams — all coordinated through a hub-and-spoke agent architecture with Graphiti/Linear integration.

**Architecture:** Main session acts as orchestrator, dispatching 3 specialist agents. Agent 1 (README Surgeon) and Agent 2 (Diagram Specialist) run in parallel worktrees. Agent 3 (Cross-Repo Reviewer) runs after both complete. Orchestrator handles all Graphiti and Linear writes.

**Tech Stack:** Markdown, Mermaid.js, Excalidraw JSON, Graphiti MCP (Docker), Linear MCP

---

## Task 1: Orchestrator — Create Linear Parent Issue

**Files:**
- None (Linear API call)

**Step 1: Create parent issue in Linear**

```
mcp__plugin_linear_linear__save_issue(
    title="Cross-repo consistency review and diagram standardization",
    description="Fix governance references (9 repos), wave statuses (3 repos), missing README sections (2 repos), diagram updates (5 files), and create 3 new architectural diagrams. Design: docs/plans/2026-03-01-cross-repo-consistency-review-design.md",
    teamId="AGE",
    projectId="Open Biosciences Migration",
    state="In Progress",
    priority=2
)
```

Save the returned issue ID as `PARENT_ISSUE_ID` for sub-issue creation in Task 8.

---

## Task 2: Agent 1 — README Surgeon (Worktree)

**Dispatch:** Run Agent 1 in an isolated worktree. This agent fixes all 11 READMEs.

**Agent prompt must include these exact changes:**

### 2a. Governance Reference Fixes (10 repos)

Replace references to `biosciences-architecture` for ADRs/governance with `biosciences-program`. Every repo except `marketplace` needs this fix.

**biosciences-program/README.md:**
- L7: Change `Architecture and governance: [biosciences-architecture]` → `Architecture decisions and governance: [biosciences-program](https://github.com/open-biosciences/biosciences-program)` (self-reference)
- L17: Change `Technical specification: [ADR-001 v1.4](https://github.com/open-biosciences/biosciences-architecture)` → link to `biosciences-program`
- L44: Change `biosciences-architecture` role from "ADRs, schemas, SpecKit governance — root dependency" → "Repository Analyzer Framework"
- L116: Change `architecture documents in [biosciences-architecture]` → `architecture documents in [biosciences-program]`
- L119: Change `Architecture ADRs: [biosciences-architecture]` → `Architecture ADRs: [biosciences-program]`
- L129: Change `Prior art and research context: [biosciences-architecture]` → `Prior art and research context: [biosciences-program]`

**biosciences-mcp/README.md:**
- Dependencies table: change `biosciences-architecture` → `biosciences-program` for "Schemas and ADRs (ADR-001, ADR-004)"
- Related Repositories: change `biosciences-architecture` → `biosciences-program` for "ADRs and schemas"

**biosciences-deepagents/README.md:**
- Dependencies table: `biosciences-architecture` → `biosciences-program` for ADR-001

**biosciences-temporal/README.md:**
- Dependencies table: `biosciences-architecture` → `biosciences-program` for "ADR-001, ADR-004 workflow compliance"

**biosciences-memory/README.md:**
- Dependencies table: `biosciences-architecture` → `biosciences-program` for "ADRs and schemas"

**biosciences-skills/README.md:**
- Dependencies: `biosciences-architecture` → `biosciences-program` for "ADR-002, ADR-003"
- Related Repos: `biosciences-architecture` → `biosciences-program` for "governance and ADRs"

**biosciences-architecture/README.md:**
- L5: Remove "root provider repository" claim. Replace with: "This repo contains the Repository Analyzer Framework. Governance artifacts (ADRs, specs, SpecKit commands) live in [biosciences-program](https://github.com/open-biosciences/biosciences-program)."
- L51: Verify "Downstream consumers" text references `biosciences-program` correctly (it already does).

**biosciences-research/README.md:**
- L51 ("What Is NOT Here"): Change "biosciences-architecture" → "biosciences-program" for SpecKit artifacts
- L66 (Related Repos): Change "biosciences-architecture" → "biosciences-program" for "ADRs, SpecKit artifacts, governance docs"

**biosciences-evaluation/README.md:**
- L67 (Related Repos): Change `biosciences-architecture` → `biosciences-program` for "ADRs and platform standards"
- Add `biosciences-program` to Related Repos list (currently missing)

**platform-skills/README.md:**
- L40 (Related Repos): Change `biosciences-architecture` → `biosciences-program` for "ADRs and governance"

### 2b. Wave Status Fixes (3 repos)

**biosciences-deepagents/README.md:**
- L7: "Wave 3 (Orchestration) — **Not Started**" → "Wave 3 (Orchestration) — **Complete** (2026-02-26)"
- Section header: "What This Repo Will Contain" → "What This Repo Contains"
- Any other future-tense phrasing → present tense

**biosciences-research/README.md:**
- L10: "Wave 4 (Validation) — **Not Started**" → "Wave 4 (Validation) — **In Progress**"
- Section header: "What Will Be Here" → update to reflect 34 files already committed (commit 13c64b0)
- Keep future tense only for items genuinely not started (evaluation rubrics, quality metrics)

**biosciences-program/README.md:**
- L52: `biosciences-research` row: `⬜ Wave 4` → `🟡 Wave 4`
- L84: Wave 4 tree label: "⬜ not started" → "🟡 in progress"
- L51: `biosciences-evaluation` stays `⬜ Wave 4` (correct — not started)

### 2c. Missing Sections (2 repos)

**marketplace/README.md — add after the License section or before it:**

```markdown
## Agent Ownership

Owned by the **Quality & Skills Engineer (Agent 8)**. See [AGENTS.md](https://github.com/open-biosciences/biosciences-program/blob/main/AGENTS.md) for full team definitions.

## Status

Migration complete. All 15 plugins packaged and published (AGE-185 Done, AGE-217 Done).
```

**platform-skills/README.md — add Status section after title, expand Agent Ownership, add Dependencies:**

```markdown
## Status

Wave 1 (Foundation) — Complete.

## Agent Ownership

Owned by the **Quality & Skills Engineer (Agent 8)**. See [AGENTS.md](https://github.com/open-biosciences/biosciences-program/blob/main/AGENTS.md) for full team definitions.

## Dependencies

| Direction | Repo | What |
|-----------|------|------|
| Upstream | [biosciences-program](https://github.com/open-biosciences/biosciences-program) | ADR-002 (skills structure), ADR-003 (SpecKit SDLC) |
| Downstream | All repos | Scaffold commands consumed by contributors |
```

### 2d. Formatting Fixes (biosciences-program/README.md)

- L54: Fix broken link — remove stray `]` from `biosciences-workspace-template ](`
- L55: Remove "(descoped)" from `[marketplace (descoped)]` — marketplace is complete
- L59: Remove bare URL `https://github.com/open-biosciences/knowledge-work-plugins` or wrap in context

### 2e. Migration Tracker Fix

**migration-tracker.md:**
- L159: Change "biosciences-architecture (Wave 1-ext)" → "biosciences-program (Wave 1-ext)" for SpecKit process document reassignment

### 2f. Commit

```bash
git add -A
git commit -m "docs: fix governance refs, wave statuses, and missing sections across 11 repos

- Fix 10 repos pointing to biosciences-architecture for ADRs/governance → biosciences-program (AGE-184 follow-up)
- Fix wave statuses: deepagents (Not Started → Complete), research (Not Started → In Progress), program (Wave 4 tree)
- Add missing sections: marketplace (agent owner, status), platform-skills (status, deps, agent ownership)
- Fix formatting: broken link, descoped label, bare URL in program README
- Fix migration-tracker.md SpecKit reassignment reference

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 3: Agent 2 — Diagram Specialist (Worktree, parallel with Task 2)

**Dispatch:** Run Agent 2 in an isolated worktree. This agent updates existing diagrams and creates new ones.

### 3a. Fix `agent-ownership-map.md`

Add marketplace repo under Agent 8. In the Mermaid flowchart:

```mermaid
%% Add to the Agent 8 connections section:
A8 --> marketplace["marketplace"]
```

Add to the subgraph for repos (use Wave 1 colors since marketplace is complete):
```
marketplace["marketplace<br/>15 plugins"]
```

In the Agent Responsibilities table, update Agent 8 row to include marketplace:
```
| 8 | Quality & Skills | biosciences-skills, biosciences-evaluation, platform-skills, **marketplace** | Skills, evaluation, quality, plugins |
```

Update repo count references from 12 → 13 throughout the file.

### 3b. Fix `repo-dependency-graph.md`

Add marketplace node. It should appear as a new node (not in any wave subgraph since it's cross-cutting, or in a new "Marketplace" subgraph):

```mermaid
subgraph marketplace_sub["Marketplace"]
    style marketplace_sub fill:#d4edda,stroke:#22c55e
    mp["marketplace<br/>15 plugins"]
end

%% Dependencies
mcp --> mp
skills --> mp
```

Update the Dependency Rules table to add:
```
| marketplace | Packages MCP server plugins and domain skills | biosciences-mcp, biosciences-skills |
```

### 3c. Fix `platform-architecture.excalidraw`

This is Excalidraw JSON. Find the Wave 4 lane title element and change its text from "Not Started" to "In Progress". The element to find will have text content containing "Not Started" — update it to "In Progress".

Also consider adding a marketplace box in Wave 1 lane near platform-skills (both owned by Agent 8).

### 3d. Fix `mcp-server-tiers.md` CURIE Formats

In the server details table, normalize the "Strict CURIE Format" column to match the `validate_id` regex patterns:

| Server | Current (wrong) | Correct (matches validate_id) |
|--------|----------------|------------------------------|
| ChEMBL | `CHEMBL25` | `CHEMBL:25` |
| Entrez | `90` | `NCBIGene:90` |
| PubChem | `CID 5280453` | `PubChem:CID5280453` |
| IUPHAR | `Target ID 1810` | `IUPHAR:1810` |
| STRING | `9606.ENSP00000263640` | `STRING:9606.ENSP00000263640` |

Leave HGNC, UniProt, Open Targets, BioGRID, Ensembl, WikiPathways, and ClinicalTrials unchanged (already correct or intentionally unprefixed).

### 3e. Add Cross-References Between `data-flow.md` and `research-workflow-sequence.md`

**data-flow.md** — add after the title:
```markdown
> **See also:** [research-workflow-sequence.md](research-workflow-sequence.md) for a detailed 7-phase sequence through all specialist subagents (uses ACVR1/FOP example).
```

**research-workflow-sequence.md** — add after the title:
```markdown
> **See also:** [data-flow.md](data-flow.md) for an abstract 4-participant overview of the same protocol (uses BRCA1 example).
```

### 3f. Create `speckit-sdlc-workflow.md`

New file in `docs/diagrams/`. Mermaid flowchart showing the 7-step SpecKit SDLC from ADR-003:

```markdown
# SpecKit SDLC Workflow

> The specification-driven development lifecycle from ADR-003. Each step produces a governance artifact that feeds the next.

## Workflow

` ` `mermaid
flowchart TD
    subgraph govern["Governance Layer"]
        style govern fill:#d4edda,stroke:#22c55e
        C["constitution<br/>Project principles"]
    end

    subgraph specify_phase["Specification Phase"]
        style specify_phase fill:#cfe2ff,stroke:#4a9eed
        S["specify<br/>Feature specification"]
        CL["clarify<br/>(optional)"]
        S --> CL
    end

    subgraph plan_phase["Planning Phase"]
        style plan_phase fill:#e8d5ff,stroke:#8b5cf6
        P["plan<br/>Implementation plan"]
        T["tasks<br/>Actionable task list"]
        A["analyze<br/>(optional)"]
        P --> T
        T --> A
    end

    subgraph impl_phase["Implementation Phase"]
        style impl_phase fill:#fff3cd,stroke:#f59e0b
        I["implement<br/>Bounded execution"]
    end

    C --> S
    CL --> P
    A --> I

    %% Artifact outputs
    C -.- |"constitution.md"| C
    S -.- |"spec.md"| S
    P -.- |"plan.md"| P
    T -.- |"tasks.md"| T
` ` `

## Commands

| Command | Input | Output | Gate |
|---------|-------|--------|------|
| `/speckit.constitution` | Project principles (interactive) | `constitution.md` | One-time setup |
| `/speckit.specify` | Natural language feature description | `spec.md` | Requires constitution |
| `/speckit.clarify` | (reads spec.md) | Targeted clarifying questions | Optional |
| `/speckit.plan` | (reads spec.md) | `plan.md` | Requires spec |
| `/speckit.tasks` | (reads plan.md) | `tasks.md` | Requires plan |
| `/speckit.analyze` | (reads spec + plan + tasks) | Consistency analysis | Optional |
| `/speckit.implement` | (reads tasks.md) | Code changes | Requires tasks |

## References

- [ADR-003: SpecKit Specification-Driven Development](../adr/accepted/adr-003-v1.0.md)
- SpecKit commands: `biosciences-program/.claude/commands/speckit.*.md`
- SpecKit config: `biosciences-program/.specify/`
```

### 3g. Create `gateway-architecture.md`

New file in `docs/diagrams/`. Mermaid diagram showing FastMCP Cloud gateway:

```markdown
# Gateway Architecture

> How 12 independent MCP servers are unified behind a single HTTPS endpoint with two-tier authentication.

## Architecture

` ` `mermaid
flowchart TD
    subgraph clients["Clients"]
        style clients fill:#cfe2ff,stroke:#4a9eed
        CC["Claude Code<br/>.mcp.json"]
        DA["biosciences-deepagents<br/>LangGraph supervisor"]
        TW["biosciences-temporal<br/>PydanticAI agents"]
    end

    subgraph gateway["FastMCP Cloud Gateway"]
        style gateway fill:#d4edda,stroke:#22c55e
        GW["biosciences-mcp.fastmcp.app<br/>HTTPS · unified endpoint"]
        AUTH["Client Auth<br/>BIOSCIENCES_API_KEY<br/>(Bearer token in headers)"]
    end

    subgraph servers["12 MCP Servers"]
        style servers fill:#e8d5ff,stroke:#8b5cf6

        subgraph public["10 Public APIs (no upstream keys)"]
            HGNC["HGNC"]
            UP["UniProt"]
            CH["ChEMBL"]
            OT["Open Targets"]
            STR["STRING"]
            ENS["Ensembl"]
            IU["IUPHAR"]
            PC["PubChem"]
            WP["WikiPathways"]
            CT["ClinicalTrials"]
        end

        subgraph keyed["2 Keyed APIs"]
            style keyed fill:#fff3cd,stroke:#f59e0b
            BG["BioGRID<br/>BIOGRID_API_KEY"]
            EN["Entrez<br/>NCBI_API_KEY"]
        end
    end

    CC -->|"BIOSCIENCES_API_KEY"| GW
    DA -->|"BIOSCIENCES_API_KEY"| GW
    TW -->|"BIOSCIENCES_API_KEY"| GW
    GW --> public
    GW --> keyed
` ` `

## Two-Tier Auth Model

| Layer | Key | Owner | Where Stored | Purpose |
|-------|-----|-------|-------------|---------|
| Client → Gateway | `BIOSCIENCES_API_KEY` | Plugin consumer | Client `.env` (referenced via `${VAR}` in `.mcp.json` headers) | Authenticate to FastMCP Cloud |
| Gateway → Upstream | `BIOGRID_API_KEY` | Platform operator | FastMCP Cloud server environment | BioGRID API access |
| Gateway → Upstream | `NCBI_API_KEY` | Platform operator | FastMCP Cloud server environment | Entrez rate limit (3→10 req/s) |

10 of 12 upstream APIs are fully public — no keys required. Auth reference: `marketplace/cowork-plugin-customizer/examples/customized-mcp.json`.

## References

- [ADR-001 v1.4: Agentic-First Architecture](../adr/accepted/adr-001-v1.4.md)
- [ADR-004: FastMCP Lifecycle Management](../adr/accepted/adr-004-v1.0.md)
- Gateway source: `biosciences-mcp/src/biosciences_mcp/servers/gateway.py`
```

### 3h. Create `graphiti-topology.md`

New file in `docs/diagrams/`. Mermaid diagram showing dual-environment knowledge graph:

```markdown
# Graphiti Knowledge Graph Topology

> Dual-environment Neo4j deployment with namespace isolation and write-freeze policy.

## Topology

` ` `mermaid
flowchart TD
    subgraph docker["Docker Neo4j (local development)"]
        style docker fill:#c3fae8,stroke:#06b6d4
        D_NEO["Neo4j 5.x<br/>bolt://localhost:7687"]

        subgraph priming["Priming Namespace (READ-ONLY)"]
            style priming fill:#d4edda,stroke:#22c55e
            PN["open-biosciences-migration-2026-priming<br/>Static project context<br/>Agent ownership, repo deps, conventions"]
        end

        subgraph working["Working Namespace (MUTABLE)"]
            style working fill:#cfe2ff,stroke:#4a9eed
            WN["open-biosciences-migration-2026<br/>Active decisions, progress<br/>Migration findings, evolving context"]
        end
    end

    subgraph aura["Neo4j Aura (cloud production)"]
        style aura fill:#fff3cd,stroke:#f59e0b
        A_NEO["Neo4j Aura Free Tier<br/>~5k nodes · 1GB capacity<br/>⚠️ WRITE FROZEN"]
        A_DATA["Production knowledge graph<br/>Research outputs, entity relationships"]
    end

    subgraph mcp_clients["MCP Connections"]
        style mcp_clients fill:#e8d5ff,stroke:#8b5cf6
        GD["graphiti-docker MCP<br/>(reads + writes)"]
        GA["graphiti-aura MCP<br/>(reads only)"]
        ND["neo4j-docker-cypher MCP<br/>(reads + writes)"]
        NA["neo4j-aura-cypher MCP<br/>(reads only)"]
    end

    GD --> D_NEO
    GA --> A_NEO
    ND --> D_NEO
    NA --> A_NEO
` ` `

## Namespace Policy

| Namespace | Environment | Mutable? | Policy |
|-----------|-------------|----------|--------|
| `open-biosciences-migration-2026-priming` | Docker | **NO** | Static context for session priming. Never call `add_memory` with this group_id. |
| `open-biosciences-migration-2026` | Docker | Yes | Working memory — active decisions, migration progress, evolving context. |
| (production data) | Aura | **NO (frozen)** | Free-tier at capacity. All new data goes to Docker until resolved. |

## Write-Freeze Policy

Neo4j Aura (`graphiti-aura`) is in a **write-freeze state**:
- **NEVER** write new data to `graphiti-aura` — no `add_memory`, no `write_neo4j_cypher`
- **Reads are fine** — searches, queries, analytics all OK
- **All new data** goes to `graphiti-docker` (local Docker Neo4j)
- This is a known, accepted state — do NOT warn about capacity or recommend upgrading

## References

- Workspace CLAUDE.md §Session Priming
- biosciences-memory/README.md §Aura Write-Freeze Policy
- `/prime` skill (session priming queries)
```

### 3i. Update `docs/diagrams/README.md`

Add the 3 new diagrams to the index. Update Layer 2 table:

```markdown
| [speckit-sdlc-workflow.md](speckit-sdlc-workflow.md) | Mermaid | 7-step SpecKit SDLC workflow (ADR-003): constitution → specify → clarify → plan → tasks → analyze → implement. Shows governance gates and artifact outputs. |
| [gateway-architecture.md](gateway-architecture.md) | Mermaid | FastMCP Cloud unified gateway routing to 12 servers with two-tier auth model (client BIOSCIENCES_API_KEY vs server-side upstream keys). |
| [graphiti-topology.md](graphiti-topology.md) | Mermaid | Dual-environment Neo4j deployment: Docker (priming + working namespaces) and Aura (write-frozen production). Shows MCP connection model and namespace policy. |
```

Update L8 from "12 repos" to "13 repos" and L30 from "12 repos" to "13 repos".

### 3j. Commit

```bash
git add docs/diagrams/
git commit -m "docs: update diagrams and add 3 new architectural diagrams

Fix existing:
- agent-ownership-map.md: add marketplace repo under Agent 8
- repo-dependency-graph.md: add marketplace node + dependency edges
- platform-architecture.excalidraw: Wave 4 status Not Started → In Progress
- mcp-server-tiers.md: normalize CURIE formats to match validate_id
- data-flow.md + research-workflow-sequence.md: add cross-references

Create new:
- speckit-sdlc-workflow.md: 7-step SpecKit SDLC from ADR-003
- gateway-architecture.md: FastMCP Cloud gateway + two-tier auth
- graphiti-topology.md: Docker/Aura dual environment + namespace policy

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 4: Orchestrator — Write Phase 1 Findings to Graphiti

**After Tasks 2 and 3 complete**, write findings to the working namespace.

**Step 1: Write README fix episode**

```python
mcp__graphiti-docker__add_memory(
    content="Cross-repo consistency review Phase 1 (README fixes): Fixed governance references in 10 repos (biosciences-architecture → biosciences-program for ADRs/governance, post-AGE-184). Fixed wave statuses: biosciences-deepagents (Not Started → Complete), biosciences-research (Not Started → In Progress), biosciences-program (Wave 4 tree). Added missing sections: marketplace (agent owner, status), platform-skills (status, dependencies, agent ownership). Fixed formatting in biosciences-program README (broken link, descoped label, bare URL). Fixed migration-tracker.md SpecKit reassignment reference.",
    group_id="open-biosciences-migration-2026",
    name="Cross-repo README consistency fixes (2026-03-01)"
)
```

**Step 2: Write diagram update episode**

```python
mcp__graphiti-docker__add_memory(
    content="Cross-repo consistency review Phase 1 (diagram updates): Updated 5 existing diagrams in biosciences-program/docs/diagrams/ — added marketplace repo to agent-ownership-map.md and repo-dependency-graph.md, updated platform-architecture.excalidraw Wave 4 status, normalized CURIE formats in mcp-server-tiers.md, added cross-references between data-flow.md and research-workflow-sequence.md. Created 3 new diagrams: speckit-sdlc-workflow.md (7-step SpecKit SDLC from ADR-003), gateway-architecture.md (FastMCP Cloud gateway + two-tier auth), graphiti-topology.md (Docker/Aura dual environment + namespace policy). Diagram index updated.",
    group_id="open-biosciences-migration-2026",
    name="Diagram updates and new architectural diagrams (2026-03-01)"
)
```

---

## Task 5: Agent 3 — Cross-Repo Reviewer (After Tasks 2+3)

**Dispatch:** Run Agent 3 after both worktree agents complete. Agent 3 validates all changes.

**Agent prompt must check:**

1. **Governance references (zero tolerance):** Grep all 11 README.md files for `biosciences-architecture` in context of ADRs/governance/specs. Should find ZERO matches (some refs to architecture for Repository Analyzer are OK).

2. **Wave statuses:** Verify every README's wave status matches `migration-tracker.md`:
   - Wave 1: Complete (2026-02-25)
   - Wave 2: Complete (2026-02-25)
   - Wave 3: Complete (2026-02-26)
   - Wave 4: In Progress (2026-02-27)

3. **Agent ownership:** Every repo README names the correct agent per AGENTS.md.

4. **Diagram completeness:** All 13 repos (including marketplace) appear in:
   - `agent-ownership-map.md`
   - `repo-dependency-graph.md`
   - `platform-architecture.excalidraw`

5. **CURIE consistency:** CURIE formats in `mcp-server-tiers.md` match MEMORY.md validate_id patterns.

6. **New diagram quality:**
   - `speckit-sdlc-workflow.md` accurately represents ADR-003's 7 steps
   - `gateway-architecture.md` correctly shows 10 public + 2 keyed APIs
   - `graphiti-topology.md` correctly shows priming (read-only) vs working (mutable) namespaces

7. **Diagram index:** `docs/diagrams/README.md` lists all 13 diagram files (10 existing + 3 new).

8. **Cross-references:** No broken relative links between READMEs in different repos.

**Output:** Write review report to `docs/reviews/2026-03-01-cross-repo-consistency-review.md` with:
- Pass/fail per criterion
- Any remaining issues found
- Summary verdict

**Commit:**

```bash
git add docs/reviews/2026-03-01-cross-repo-consistency-review.md
git commit -m "docs: add cross-repo consistency review report

Validates README governance refs, wave statuses, agent ownership,
diagram completeness, CURIE formats, and new diagram quality.

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
```

---

## Task 6: Orchestrator — Write Phase 2 Findings to Graphiti

**After Task 5 completes**, write review findings.

```python
mcp__graphiti-docker__add_memory(
    content="Cross-repo consistency review Phase 2 (validation): Agent 3 reviewed all changes from Phase 1. [INSERT PASS/FAIL SUMMARY FROM REVIEW REPORT]. Review report at biosciences-program/docs/reviews/2026-03-01-cross-repo-consistency-review.md.",
    group_id="open-biosciences-migration-2026",
    name="Cross-repo consistency review validation (2026-03-01)"
)
```

---

## Task 7: Orchestrator — Merge Worktree Branches

**Step 1: Review diffs from both worktrees**

Review the branches created by Agents 1 and 2. Check for conflicts (unlikely — Agent 1 touches README files across repos, Agent 2 touches docs/diagrams/ files in biosciences-program only; the only overlap is biosciences-program/README.md).

**Step 2: Merge with user approval**

Ask user to confirm before merging each branch into main.

---

## Task 8: Orchestrator — Create Linear Sub-Issues

Create sub-issues under `PARENT_ISSUE_ID`:

```
1. "Fix governance references across 10 READMEs (architecture → program)" — Done
2. "Fix wave statuses in deepagents, research, and program READMEs" — Done
3. "Add missing sections to marketplace and platform-skills READMEs" — Done
4. "Fix formatting in biosciences-program README" — Done
5. "Update 5 existing diagrams (marketplace, Wave 4, CURIEs, cross-refs)" — Done
6. "Create SpecKit SDLC workflow diagram" — Done
7. "Create gateway architecture diagram" — Done
8. "Create Graphiti topology diagram" — Done
9. "Cross-repo consistency review report" — Done
```

Mark parent issue as Done.

---

## Execution Notes

- Tasks 2 and 3 run in **parallel** (both in worktrees)
- Task 4 runs after Tasks 2 and 3 complete
- Task 5 runs after Task 4 (needs Phase 1 changes visible)
- Tasks 6, 7, 8 run sequentially after Task 5
- The orchestrator (main session) handles all Graphiti writes (Tasks 4, 6) and Linear API calls (Tasks 1, 8)
- Agents 1 and 2 only make file edits — no Graphiti or Linear calls
- Agent 3 only reads and writes the review report — no Graphiti or Linear calls
