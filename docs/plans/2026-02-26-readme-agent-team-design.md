# Design: README Agent Team — Open Biosciences Repository Audit

**Date:** 2026-02-26
**Author:** Program Director
**Status:** Approved

---

## Problem

Eleven repositories in the `open-biosciences` org have READMEs in varying states of accuracy. Key issues:

- `biosciences-memory` still says "Pending Wave 2" — Wave 2 is complete
- `biosciences-mcp` README carries stale content from the pre-migration source repo
- `biosciences-research` may reference SpecKit artifacts that were reassigned to `biosciences-architecture` (AGE-183)
- Wave 3 and 4 repos have generic placeholders that don't reflect what the predecessor repos actually contain

The root cause: READMEs were scaffolded at repo creation time, then migration decisions evolved them out of sync.

---

## Approach: Parallel Repo Agents with Shared Briefing (Approach B)

**Phase 1 — Briefing (this session):**
Write `docs/plans/platform-facts-briefing.md` — a single authoritative document encoding all current platform facts. This is the shared ground truth all agents receive, ensuring cross-repo claims are consistent without each agent re-deriving migration history.

**Phase 2 — Parallel agents:**
Dispatch 10 independent `general-purpose` agents (one per repo, excluding `biosciences-program` which was just updated). Each agent:

1. Reads the briefing doc
2. Searches `open-biosciences-migration-2026` Graphiti working memory for its repo
3. Reads its repo's actual files (README.md, CLAUDE.md, key source files where applicable)
4. Writes a factually grounded README.md
5. Commits the change
6. Adds a Graphiti memory episode recording the update

---

## Per-Agent Scope

| Agent | Repo | Key Issue | Extra Sources |
|-------|------|-----------|---------------|
| 1 | biosciences-memory | "Pending Wave 2" stale | `.mcp.json`, `.env.example` |
| 2 | biosciences-mcp | Post-migration staleness | `src/` structure, `pyproject.toml` |
| 3 | biosciences-architecture | Confirm SpecKit artifacts described | `docs/` listing |
| 4 | biosciences-skills | Verify 6 skills + 15 commands | `.claude/` listing |
| 5 | biosciences-deepagents | Generic → grounded Wave 3 | predecessor README |
| 6 | biosciences-temporal | Generic → grounded Wave 3 | predecessor README |
| 7 | biosciences-evaluation | Quality gate layer description | migration-tracker Wave 4 |
| 8 | biosciences-research | Remove SpecKit refs; CQ catalog scope | migration-tracker Wave 4 |
| 9 | biosciences-education | Training/tutorials scope | migration-tracker Wave 4 |
| 10 | biosciences-workspace-template | Bootstrap scope | migration-tracker Wave 4 |

---

## Graphiti + Linear Integration

**Per-agent Graphiti workflow:**
- Before writing: search `open-biosciences-migration-2026` for repo-specific decisions
- After committing: add one episode to `open-biosciences-migration-2026` recording what was updated

**Linear references:**
- Wave 1 repos → AGE-149
- Wave 2 repos → AGE-150
- Wave 3 repos → AGE-151
- Wave 4 repos → AGE-152

---

## Commit Convention

```
docs: update README with current platform facts

Reflects Wave N migration status, agent ownership, and cross-repo
dependencies. Grounded against platform-facts-briefing.md.

Refs: AGE-XXX
```

---

## README Design Principles (Shared Across All Repos)

- Each README is the **front door** to its repo — not a technical manual
- Depth lives inside the repo; cross-cutting narrative lives in `biosciences-program`
- State wave status accurately: ✅ Complete or ⬜ Not Started — no ambiguity
- No internal paths (no `/home/donbr/` references)
- All cross-repo links → `https://github.com/open-biosciences/<repo-name>`
- Agent ownership clearly stated in each README
