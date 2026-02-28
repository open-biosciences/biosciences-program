# Handoff: graphiti-fastmcp Migration

**Date:** 2026-02-27
**Status:** Plan complete, ready for implementation
**Next session:** Execute the 14-task implementation plan

---

## What to do

Open a new Claude Code session in `biosciences-memory`:

```bash
cd /home/donbr/open-biosciences/biosciences-memory
claude
```

Then say:

> Implement the plan at `/home/donbr/open-biosciences/biosciences-program/docs/plans/2026-02-27-graphiti-fastmcp-migration-plan.md`

The plan is self-contained with exact file paths, code, commands, and expected output for all 14 tasks.

---

## Context summary

**What:** Migrate `/home/donbr/graphiti-fastmcp` (v1.0.1, 2,932 LOC) into `/home/donbr/open-biosciences/biosciences-memory` as a curated package.

**Approach:** Curated Migration (Approach A)
- Copy + strip to OpenAI + Neo4j only
- Add 5 biosciences entity types (Gene, Protein, Drug, Disease, Pathway)
- Add 5 edge types + edge_type_map
- Rewrite tests using FastMCP official `Client(server)` in-memory pattern
- Both entry points: factory (`server.py`) + CLI (`cli.py`)

**Constraints:**
- No PII in commits — `.env.example` with placeholders only
- Pin `fastmcp>=2.13.3,<3` (v3 is RC)
- Pin `graphiti-core==0.24.3` (strict match)
- Python >=3.11, hatchling, ruff, pyright, pytest markers

**Estimated result:** ~1,200 LOC source + ~400 LOC tests

---

## Key files

| File | Purpose |
|------|---------|
| `docs/plans/2026-02-27-graphiti-fastmcp-migration-plan.md` | 14-task implementation plan |
| `docs/plans/2026-02-27-graphiti-fastmcp-migration-design.md` | Design document with rationale |
| `/home/donbr/graphiti-fastmcp/src/` | Source to migrate FROM |
| `/home/donbr/open-biosciences/biosciences-memory/` | Target repo (bare skeleton, ready) |

---

## Task summary

| # | Task | Key output |
|---|------|------------|
| 1 | pyproject.toml | Build config, deps, pytest markers |
| 2 | Package skeleton + response models + formatting | Directory structure, verbatim copies |
| 3 | Entity types | 9 generic + 5 biosciences + edges + map |
| 4 | Config schema | Pydantic-Settings, YAML, OpenAI+Neo4j only |
| 5 | Factories | LLM + Embedder + Database (stripped) |
| 6 | Queue service | Verbatim copy (battle-tested) |
| 7 | server.py | Factory entry + 9 MCP tools + Neo4jDriverWithPoolConfig |
| 8 | cli.py | argparse + YAML config CLI |
| 9 | config.yaml | OpenAI + Neo4j with env expansion |
| 10 | Unit tests | 5 test modules (config, factories, queue, formatting, entities) |
| 11 | Integration tests | In-memory MCP tool tests |
| 12 | Lint + type check | ruff + pyright cleanup |
| 13 | .env.example + README | Docs update |
| 14 | Migration tracker + Linear + Graphiti | Cross-repo tracking |

---

## What was completed this session

1. **Wave 3 runtime verification** — All 4 acceptance criteria passed and checked off
2. **Wave 4 research docs migration** — 34 files across 4 directories, commit `13c64b0`
3. **graphiti-fastmcp design** — Approach A selected, design doc committed
4. **graphiti-fastmcp implementation plan** — 14 tasks, grounded in FastMCP + Graphiti + pytest best practices

## Commits pushed this session

| Repo | Commit | Description |
|------|--------|-------------|
| biosciences-research | `13c64b0` | Research docs migration (34 files) |
| biosciences-program | `123ba50` | Migration tracker — Wave 3 verified + Wave 4 progress |
| biosciences-program | `64c7d04` | graphiti-fastmcp migration design doc |
| biosciences-program | `7353536` | graphiti-fastmcp migration implementation plan |
