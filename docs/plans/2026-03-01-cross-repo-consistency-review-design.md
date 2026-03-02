# Cross-Repo Consistency Review & Diagram Standardization

**Date:** 2026-03-01
**Author:** Program Director (Agent 1)
**Status:** Design approved, pending implementation

---

## Problem Statement

The Open Biosciences platform's 13-repo workspace has accumulated documentation drift:

1. **9 of 11 READMEs** reference `biosciences-architecture` for governance artifacts (ADRs, specs, SpecKit commands) — but AGE-184 consolidated all governance into `biosciences-program`.
2. **2 READMEs** have wrong migration wave status (`biosciences-deepagents` says "Not Started" for completed Wave 3; `biosciences-research` says "Not Started" for in-progress Wave 4).
3. **2 READMEs** are missing standard sections (`marketplace`: no agent owner or status; `platform-skills`: no status, deps, or agent ownership section).
4. **10 diagram files** in `docs/diagrams/` are solid (8/10 quality) but drifting: marketplace repo missing from all diagrams, Wave 4 status stale in Excalidraw files, CURIE format inconsistencies, no SpecKit SDLC or gateway architecture diagrams.

## Goals

- Fix all README inconsistencies across 11 repos
- Update 5 existing diagrams to reflect current state
- Create 3 new architectural diagrams to fill gaps
- Write findings to Graphiti working namespace for project memory
- Track all work in Linear under a parent issue
- Produce a cross-repo review report validating consistency

## Non-Goals

- Graph-builder Cytoscape.js templates (workspace-template/output/ — separate effort)
- Code changes in any repo
- Migration tracker updates (those follow from README/diagram fixes)
- CLAUDE.md updates (already accurate post-AGE-184)

---

## Architecture: Hub-and-Spoke

```
Main Session (Orchestrator)
├── Reads/writes Graphiti working namespace (open-biosciences-migration-2026)
├── Creates/updates Linear issues (AGE project)
├── Dispatches work to 3 specialist agents
│
├── Agent 1: README Surgeon (worktree)
│   ├── Fix governance refs across 9 repos
│   ├── Fix wave statuses (deepagents, research, program)
│   ├── Add missing sections (marketplace, platform-skills)
│   └── Fix formatting (program README)
│
├── Agent 2: Diagram Specialist (worktree)
│   ├── Fix 5 existing diagrams
│   └── Create 3 new Mermaid diagrams
│
├── Agent 3: Cross-Repo Reviewer (after 1+2)
│   ├── Validate all changes
│   └── Produce review report
│
└── Orchestrator post-processing
    ├── Write findings to Graphiti
    ├── Create/update Linear issues
    └── Merge worktree branches
```

The orchestrator centralizes all Graphiti and Linear writes to avoid race conditions. Agents 1 and 2 run in parallel in isolated worktrees. Agent 3 runs after both complete.

---

## Agent 1: README Surgeon

### Governance Reference Fixes (9 repos)

All references to `biosciences-architecture` for ADRs/governance/specs must point to `biosciences-program`. Affected repos and locations:

| Repo | Affected Lines | Change |
|------|---------------|--------|
| biosciences-program | README.md L7, L17, L44, L116, L119, L129 | Point governance refs to self (biosciences-program) |
| biosciences-mcp | README.md L172 (deps table), L179 (related repos) | `biosciences-architecture` → `biosciences-program` |
| biosciences-deepagents | README.md L67 (deps table) | Same |
| biosciences-temporal | README.md L106 (deps table) | Same |
| biosciences-memory | README.md L94 (deps table) | Same |
| biosciences-skills | README.md L53 (deps), L58 (related repos) | Same |
| biosciences-architecture | README.md L5 (opening paragraph) | Remove "root provider" claim; clarify role is Repository Analyzer Framework only |
| biosciences-research | README.md L51 ("What Is NOT Here"), L66 (related repos) | Same |
| biosciences-evaluation | README.md L67 (related repos) + add biosciences-program | Same + add missing link |
| platform-skills | README.md L40 (related repos) | Same |

### Wave Status Fixes

| Repo | Current Text | Correct Text | Evidence |
|------|-------------|-------------|----------|
| biosciences-deepagents | "Wave 3 — Not Started" | "Wave 3 — Complete" | MEMORY.md: runtime verified 2026-02-27 |
| biosciences-deepagents | "What This Repo Will Contain" | "What This Repo Contains" | Migration commits exist |
| biosciences-research | "Wave 4 — Not Started" | "Wave 4 — In Progress" | MEMORY.md: 34 files, commit 13c64b0 |
| biosciences-research | "What Will Be Here" | "What's Here" (partial) | 34 files committed |
| biosciences-program | Wave 4 tree "⬜ not started" | "🟡 in progress" | migration-tracker.md |
| biosciences-program | `biosciences-research` row: `⬜ Wave 4` | `🟡 Wave 4` | 34 files committed |

### Missing Sections

**marketplace/README.md:**
- Add "Agent Ownership" section: Agent 8 (Quality & Skills Engineer), link to AGENTS.md
- Add "Status" line: Migration complete (AGE-185 Done)

**platform-skills/README.md:**
- Add "Status" section: Wave 1 — Complete
- Add "Dependencies" table: biosciences-program (ADR-002, ADR-003)
- Expand Agent Ownership to dedicated section with AGENTS.md link

### Formatting Fixes (biosciences-program/README.md)

| Line | Issue | Fix |
|------|-------|-----|
| L54 | Broken markdown link: `biosciences-workspace-template ]` | Remove stray `]` |
| L55 | `[marketplace (descoped)]` | Remove "(descoped)" — marketplace is complete |
| L59 | Bare URL with no context | Remove or wrap in descriptive text |
| L44 | `biosciences-architecture` described as "ADRs, schemas, SpecKit governance" | Update to "Repository Analyzer Framework" |

---

## Agent 2: Diagram Specialist

### Fix Existing Diagrams (5 changes)

1. **`agent-ownership-map.md`**
   - Add marketplace repo under Agent 8 subgraph
   - Add row to Agent 8 responsibility table
   - Update repo count from 12 to 13

2. **`repo-dependency-graph.md`**
   - Add marketplace node (likely in a new "marketplace" subgraph or under Wave 1)
   - Add dependency edges: marketplace → biosciences-mcp (server plugins), marketplace → biosciences-skills (domain skills)

3. **`platform-architecture.excalidraw`**
   - Change Wave 4 lane title from "Not Started" to "In Progress"
   - Consider adding marketplace box (Agent 8, alongside platform-skills)

4. **`mcp-server-tiers.md`**
   - Normalize "Strict CURIE Format" column to match `validate_id` patterns:
     - ChEMBL: `CHEMBL25` → `CHEMBL:25`
     - Entrez: `90` → `NCBIGene:90`
     - PubChem: `CID 5280453` → `PubChem:CID5280453`
     - IUPHAR: `Target ID 1810` → `IUPHAR:1810`
     - STRING: `9606.ENSP...` → `STRING:9606.ENSP...`

5. **`data-flow.md` + `research-workflow-sequence.md`**
   - Add cross-reference note to each: "See also: [sibling file] for [abstract/detailed] version"

### Create New Diagrams (3 files)

All new diagrams placed in `docs/diagrams/` and follow the existing color palette from `docs/diagrams/README.md`.

#### 1. `speckit-sdlc-workflow.md`

Mermaid flowchart showing the 7-step SpecKit SDLC process from ADR-003:

```
constitution → specify → clarify (optional) → plan → tasks → analyze (optional) → implement
```

Include: governance gates, artifact outputs at each step, and the spec → plan → tasks dependency chain. Reference ADR-003 and the 9 SpecKit commands in `.claude/commands/`.

#### 2. `gateway-architecture.md`

Mermaid diagram showing the FastMCP Cloud unified gateway:

```
Client → biosciences-mcp.fastmcp.app (HTTPS) → 12 FastMCP servers
```

Include the two-tier auth model:
- Client-side: `BIOSCIENCES_API_KEY` (in `.mcp.json` headers)
- Server-side: `BIOGRID_API_KEY`, `NCBI_API_KEY` (in FastMCP Cloud env)
- 10 of 12 upstream APIs are fully public (no keys)

#### 3. `graphiti-topology.md`

Mermaid diagram showing the dual-environment knowledge graph:

```
graphiti-docker (local Docker Neo4j)
  ├── priming namespace (read-only): open-biosciences-migration-2026-priming
  └── working namespace (mutable): open-biosciences-migration-2026

graphiti-aura (Neo4j Aura cloud) — WRITE FROZEN
  └── Production data (~5k nodes, 1GB free tier)
```

Include the write-freeze policy and the MCP connection model.

### Update Diagram Index

After all changes, update `docs/diagrams/README.md`:
- Add entries for 3 new diagrams
- Update the Layer 2 section to include gateway and topology diagrams
- Add SpecKit workflow to Layer 1 or Layer 2 as appropriate

---

## Agent 3: Cross-Repo Reviewer

Runs after Agents 1 and 2 complete. Validates:

1. **Governance references**: All 11 READMEs now reference `biosciences-program` for ADRs/governance
2. **Wave statuses**: All statuses match migration-tracker.md
3. **Agent ownership**: All repos state correct agent owner per AGENTS.md
4. **Diagram completeness**: All 13 repos appear in agent-ownership-map, repo-dependency-graph, and platform-architecture
5. **CURIE consistency**: Diagram CURIE formats match MEMORY.md validate_id patterns
6. **Cross-references**: No broken links between repos, diagrams index updated
7. **New diagram quality**: SpecKit workflow, gateway architecture, and Graphiti topology are accurate and follow the color palette

Output: `docs/reviews/2026-03-01-cross-repo-consistency-review.md`

---

## Graphiti Integration

The orchestrator (main session) writes to the working namespace (`open-biosciences-migration-2026`) after each phase:

1. **After Agent 1**: Episode summarizing README fixes — repos touched, governance refs updated, wave statuses corrected, sections added
2. **After Agent 2**: Episode summarizing diagram changes — files updated, new diagrams created, CURIE formats normalized
3. **After Agent 3**: Episode with review findings — pass/fail per validation criterion, any remaining issues

No writes to the priming namespace (read-only policy).

## Linear Integration

The orchestrator creates:

- **Parent issue**: "Cross-repo consistency review and diagram standardization"
- **Sub-issues**:
  1. Fix governance references across 9 READMEs
  2. Fix wave statuses in deepagents, research, and program READMEs
  3. Add missing sections to marketplace and platform-skills READMEs
  4. Fix formatting in biosciences-program README
  5. Update 5 existing diagrams (marketplace, Wave 4 status, CURIEs, cross-refs)
  6. Create SpecKit SDLC workflow diagram
  7. Create gateway architecture diagram
  8. Create Graphiti topology diagram
  9. Cross-repo review report

---

## Execution Sequence

```
1. Orchestrator reads current state from Graphiti + Linear
2. Orchestrator dispatches Agent 1 (README) + Agent 2 (Diagrams) in parallel
   - Both run in worktrees for isolation
3. Orchestrator waits for both to complete
4. Orchestrator writes Phase 1 findings to Graphiti
5. Orchestrator dispatches Agent 3 (Reviewer) with outputs from 1+2
6. Agent 3 produces review report
7. Orchestrator writes Phase 2 findings to Graphiti
8. Orchestrator creates Linear parent + sub-issues
9. User reviews and approves merges
```

## Success Criteria

- All 11 READMEs reference `biosciences-program` for governance (zero refs to biosciences-architecture for ADRs/governance)
- All wave statuses match migration-tracker.md
- marketplace and platform-skills READMEs have standard sections
- All diagrams include 13 repos (including marketplace)
- 3 new diagrams created and indexed
- CURIE formats in diagrams match validate_id patterns
- Review report confirms consistency
- All findings recorded in Graphiti working namespace
- Linear parent issue with sub-issues created and linked

## Risks

| Risk | Mitigation |
|------|-----------|
| Worktree merge conflicts | Agents touch different repos (README vs diagrams in same repo possible — program README may overlap) |
| Excalidraw JSON edits | Only change text content in known element IDs; validate JSON after edit |
| Graphiti Docker unavailable | Degrade gracefully — write findings to markdown only |
| Linear API rate limits | Batch issue creation, use sequential calls |

---

## References

- AGE-184: Platform-skills split + governance artifact consolidation (Done)
- ADR-003: SpecKit Specification-Driven Development
- ADR-001 §3: Fuzzy-to-Fact Protocol
- `docs/diagrams/README.md`: Diagram format guide and color palette
- MEMORY.md: validate_id CURIE format table
- migration-tracker.md: Authoritative wave status
