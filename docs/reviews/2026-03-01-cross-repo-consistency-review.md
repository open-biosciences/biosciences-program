# Cross-Repo Consistency Review (AGE-219)

**Date:** 2026-03-01
**Reviewer:** Agent 3 (Cross-Repo Reviewer)
**Scope:** Agent 1 (README Surgeon) and Agent 2 (Diagram Specialist) worktree changes

---

## Criterion 1: Governance References

**Verdict: PASS**

The Agent 1 worktree `README.md` correctly removes all governance references to `biosciences-architecture`. The two remaining references to `biosciences-architecture` are both appropriate:

1. Line 7: `Repository Analyzer Framework: [biosciences-architecture](...)`
2. Line 44: Table row describing it as "Repository Analyzer Framework"

Both correctly describe the architecture repo's current purpose (Repository Analyzer Framework only). Governance is correctly pointed to `biosciences-program` in the header callout on line 7.

---

## Criterion 2: Wave Statuses

**Verdict: PASS**

The README table (lines 41-55) matches `migration-tracker.md`:

| Repo | README Status | migration-tracker.md | Match? |
|------|---------------|---------------------|--------|
| Wave 1 repos (program, architecture, skills, platform-skills) | All show checkmark Wave 1 | Wave 1 Complete (2026-02-25) | Yes |
| Wave 2 repos (mcp, memory) | All show checkmark Wave 2 | Wave 2 Complete (2026-02-25) | Yes |
| Wave 3 repos (deepagents, temporal) | All show checkmark Wave 3 | Wave 3 Complete (2026-02-26) | Yes |
| biosciences-evaluation | Shows empty-square Wave 4 | Not Started | Yes |
| biosciences-research | Shows yellow-circle Wave 4 | In Progress | Yes |
| biosciences-education | Shows empty-square Wave 4 | Not explicitly tracked but no work done | Yes |
| biosciences-workspace-template | Shows empty-square Wave 4 | Not explicitly tracked but no work done | Yes |

The workspace tree section (lines 66-93) correctly shows "Wave 4 yellow-circle in progress" for the overall wave status.

---

## Criterion 3: Marketplace to knowledge-work-plugins

**Verdict: PASS**

Agent 1 correctly replaced `marketplace` with `knowledge-work-plugins` in the README table:

- Line 55: `[knowledge-work-plugins](https://github.com/anthropics/knowledge-work-plugins)` -- correct URL pointing to Anthropic's repo (not `open-biosciences`)
- Line 108: Reference in "Get Involved" section also correctly uses `knowledge-work-plugins` with the `anthropics` org URL
- Zero occurrences of "marketplace" remain in the README (confirmed by grep)

---

## Criterion 4: Diagram -- knowledge-work-plugins

**Verdict: PASS**

**`agent-ownership-map.md`:**
- Line 31: Uses `r_kwp["knowledge-work-plugins"]` (correct, not "marketplace")
- Line 47: Agent 8 connects to `r_kwp` (correct ownership)
- Line 71: `style r_kwp fill:#f0f0f0,stroke:#666666` -- uses neutral gray styling (correct for external repo)
- Line 87: Responsibility table lists "knowledge-work-plugins" correctly

**`repo-dependency-graph.md`:**
- Line 16: External subgraph contains `kwp["knowledge-work-plugins<br/>(Anthropic bio-research plugin)"]`
- Line 45: Correct dependency edge: `mcp -->|wraps 34-tool gateway| kwp`
- Line 56: `style external fill:#e0e0e0,stroke:#666666` -- neutral gray for external subgraph
- Line 62: `style kwp fill:#f0f0f0,stroke:#666666` -- neutral gray for the node itself

---

## Criterion 5: Diagram -- Wave 4 Status

**Verdict: PASS**

In `platform-architecture.excalidraw`, line 340: `"text": "Wave 4: Validation  (In Progress)"` -- correctly updated from "Not Started" to "In Progress". The lane also uses dashed stroke style (line 326: `"strokeStyle": "dashed"`) to visually distinguish it from completed waves, which is a good design choice.

---

## Criterion 6: Diagram -- CURIE Formats

**Verdict: PARTIAL PASS (2 issues found)**

Checking each CURIE in `mcp-server-tiers.md` against the source code in `biosciences-mcp/src/`:

| Server | Diagram CURIE | Source Code Pattern | Match? |
|--------|---------------|---------------------|--------|
| ChEMBL | `CHEMBL:25` | `^CHEMBL:[0-9]+$` (clients/chembl.py) | **Yes** |
| Entrez | `NCBIGene:90` | `^NCBIGene:\d+$` | **Yes** |
| PubChem | `PubChem:CID5280453` | `^PubChem:CID\d+$` (models/pubchem_compound.py) | **Yes** |
| IUPHAR | `IUPHAR:1810` | `^IUPHAR:\d+$` (models/pharmacology.py) | **Yes** |
| STRING | `STRING:9606.ENSP00000263640` | `^STRING:\d+\.ENSP\d+$` (models/interaction.py) | **Yes** |
| WikiPathways | `WP4806` | `^WP:WP\d+$` (clients/wikipathways.py:38) | **NO -- should be `WP:WP4806`** |
| ClinicalTrials | `NCT03312634` | `^NCT:\d{8}$` (clients/clinicaltrials.py:44) | **NO -- should be `NCT:03312634`** |

The 5 CURIEs specified in the review criteria (ChEMBL, Entrez, PubChem, IUPHAR, STRING) are all correct. However, two other CURIEs in the same diagram are inconsistent with the Fuzzy-to-Fact protocol:

- **WikiPathways** shows `WP4806` but the validate pattern is `^WP:WP\d+$`, so it should be `WP:WP4806`
- **ClinicalTrials** shows `NCT03312634` but the validate pattern is `^NCT:\d{8}$`, so it should be `NCT:03312634`

**Note:** These two errors were pre-existing in the diagram before Agent 2's changes. The main branch `mcp-server-tiers.md` has the same bare identifiers. Agent 2 correctly normalized the 5 CURIEs specified in the task scope but did not catch the remaining two. This is an **Important** issue that should be fixed but does not block the current review.

---

## Criterion 7: New Diagrams Quality

**Verdict: PASS**

### `speckit-sdlc-workflow.md`
- Shows all 7 SpecKit steps: constitution, specify, clarify, plan, tasks, analyze, implement
- Four phase subgraphs: Governance, Specification, Planning, Implementation
- Optional steps (clarify, analyze) correctly shown with dashed borders
- Human approval gate between analyze and implement
- Artifact outputs (constitution.md, spec.md, etc.) shown as rounded nodes
- Uses correct wave color palette (green/blue/purple/amber for the 4 phases)
- References link to ADR-003 and ADR-002

### `gateway-architecture.md`
- Shows 3 clients (Claude Code, deepagents, temporal) connecting through `BIOSCIENCES_API_KEY`
- Gateway at `biosciences-mcp.fastmcp.app` with 34 tools
- Correctly splits 10 public servers from 2 keyed servers (BioGRID with `BIOGRID_API_KEY`, Entrez with `NCBI_API_KEY`)
- Two-tier auth model table is accurate
- Client configuration example with `.mcp.json` pattern
- References link to ADR-001, ADR-004, and mcp-server-tiers.md

### `graphiti-topology.md`
- Shows Docker Neo4j with priming namespace (READ-ONLY) and working namespace (MUTABLE)
- Shows Neo4j Aura (Cloud) as WRITE FROZEN
- Four MCP connections: graphiti-docker (R/W), graphiti-aura (R only), neo4j-docker-cypher (R/W), neo4j-aura-cypher (R only)
- Consumer list: Claude Code Session Priming, biosciences-memory, biosciences-research, biosciences-deepagents PERSIST phase
- Namespace policy table with correct group IDs
- Write-freeze policy section with accurate guidance
- Session priming protocol code example
- Uses appropriate gray styling for Aura (frozen) and teal for Docker (active)

All three diagrams are technically accurate, use the wave color palette, and include reference links.

---

## Criterion 8: Diagram Index

**Verdict: PASS**

The Agent 2 worktree `docs/diagrams/README.md` lists all 3 new diagrams in the Layer 2 section:

- Line 34: `speckit-sdlc-workflow.md` -- with accurate description
- Line 35: `gateway-architecture.md` -- with accurate description
- Line 36: `graphiti-topology.md` -- with accurate description

Additionally, the README header (line 8) now references "12 repos + knowledge-work-plugins" instead of just "12 repos", and the color scheme table (line 69) adds an "External/Partner" row for knowledge-work-plugins with gray styling.

---

## Criterion 9: Cross-References

**Verdict: PASS**

**`data-flow.md`:**
- Line 7: `> **See also:** [research-workflow-sequence.md](research-workflow-sequence.md) for a detailed 7-phase sequence through all specialist subagents (uses ACVR1/FOP example).`

**`research-workflow-sequence.md`:**
- Line 8: `> **See also:** [data-flow.md](data-flow.md) for an abstract 4-participant overview of the same protocol (uses BRCA1 example).`

Both cross-references are present, accurate, and use relative links.

---

## Criterion 10: biosciences-architecture README

**Verdict: PASS**

The file at `/home/donbr/open-biosciences/biosciences-architecture/README.md`:

- Line 1: Title is "biosciences-architecture" (no "root provider" claim)
- Line 3: Opening line reads "Repository Analyzer Framework for the Open Biosciences platform"
- Line 5: Clear statement that governance artifacts have been consolidated into biosciences-program per AGE-184
- Lines 17-21: "Current Contents" section lists only Repository Analyzer Framework directories (`ra_agents/`, `ra_orchestrators/`, `ra_tools/`)
- Lines 23-34: "Governance Artifacts (migrated)" section clearly redirects to biosciences-program locations
- Line 36: Rationale explains the consolidation decision
- Line 49: Dependencies section says "None. This repo has no upstream dependencies."
- No "root provider" claim remains anywhere in the file

---

## Summary

| # | Criterion | Verdict |
|---|-----------|---------|
| 1 | Governance References | **PASS** |
| 2 | Wave Statuses | **PASS** |
| 3 | Marketplace to knowledge-work-plugins | **PASS** |
| 4 | Diagram -- knowledge-work-plugins | **PASS** |
| 5 | Diagram -- Wave 4 Status | **PASS** |
| 6 | Diagram -- CURIE Formats | **PARTIAL PASS** |
| 7 | New Diagrams Quality | **PASS** |
| 8 | Diagram Index | **PASS** |
| 9 | Cross-References | **PASS** |
| 10 | biosciences-architecture README | **PASS** |

### Overall Verdict: PASS with 1 advisory issue

**9 of 10 criteria fully pass. 1 partial pass.**

The partial pass on Criterion 6 is due to two pre-existing CURIE format issues (WikiPathways `WP4806` should be `WP:WP4806`, ClinicalTrials `NCT03312634` should be `NCT:03312634`) that were not in the Agent 2 task scope. The 5 CURIEs that were specified for normalization are all correct. The pre-existing issues should be tracked as a follow-up item.

### Issues Requiring Follow-Up

| Priority | Issue | File | Action |
|----------|-------|------|--------|
| Important | WikiPathways CURIE uses bare `WP4806` instead of `WP:WP4806` | `docs/diagrams/mcp-server-tiers.md` (lines 32, 80) | Update to `WP:WP4806` |
| Important | ClinicalTrials CURIE uses bare `NCT03312634` instead of `NCT:03312634` | `docs/diagrams/mcp-server-tiers.md` (lines 33, 81) | Update to `NCT:03312634` |
| Suggestion | `ecosystem-positioning.md` still uses "marketplace" terminology (line 4) | `docs/diagrams/ecosystem-positioning.md` | Consider updating to "knowledge-work-plugins" for consistency |
| Suggestion | `ecosystem-map.excalidraw` has element IDs referencing "marketplace" | `docs/diagrams/ecosystem-map.excalidraw` | Low priority -- internal IDs, not visible to users |

### What Was Done Well

- Agent 1 executed a clean, focused README surgery with no regressions. The knowledge-work-plugins migration is thorough with correct Anthropic org URLs. The `biosciences-architecture/README.md` rewrite is excellent -- clear, redirects governance consumers correctly, and maintains the Repository Analyzer Framework focus.
- Agent 2 produced three high-quality new diagrams (speckit-sdlc-workflow, gateway-architecture, graphiti-topology) that are technically accurate, use consistent styling, and include proper cross-references. The knowledge-work-plugins styling with neutral gray correctly communicates external ownership.
- Both agents maintained consistency with each other -- the README table and the diagram dependency graph tell the same story with the same terminology.

---

*Reviewed by Agent 3 (Cross-Repo Reviewer) on 2026-03-01 as part of AGE-219.*
