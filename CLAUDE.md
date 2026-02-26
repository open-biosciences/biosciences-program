# CLAUDE.md — biosciences-program

## Session Priming (REQUIRED)

Session priming is required before any work in this repo. See the workspace-level
[`../CLAUDE.md`](../CLAUDE.md) §Session Priming for the mandatory Graphiti queries.

## Purpose

Program management and cross-repo coordination for the Open Biosciences platform. This repo is owned by the **Program Director** agent.

## Key Files

| File | Purpose |
|------|---------|
| `AGENTS.md` | Agent team definitions (9 agents, repo ownership, responsibilities) |
| `migration-tracker.md` | Migration wave status from predecessor repos |

## Responsibilities

- Track migration progress across 4 waves
- Manage cross-repo dependency graph
- Coordinate release milestones
- Triage cross-repo issues to the correct agent owner
- Maintain the agent team definition as repos evolve

## Migration Context

Three predecessor repos are being migrated into 11 new repos:
- `lifesciences-research` → architecture, mcp, skills, evaluation, research
- `lifesciences-deepagents` → deepagents
- `lifesciences-temporal` → temporal

See `migration-tracker.md` for wave-by-wave status.

## Conventions

- This repo contains **no code** — only coordination documents
- Markdown files use tables for structured tracking
- Status uses emoji indicators: ⬜ Not Started, 🟡 In Progress, ✅ Complete, ⛔ Blocked
- Cross-references use relative paths to sibling repos where possible
