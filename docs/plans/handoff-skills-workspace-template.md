# Handoff: biosciences-skills + biosciences-workspace-template Review

**Date:** 2026-02-26
**Status:** biosciences-skills is Wave 1 Complete with two actionable gaps; biosciences-workspace-template is a correctly-scoped Wave 4 stub with a private-path issue in its CLAUDE.md.

---

## biosciences-skills

### Actual State

Repository contains only `.claude/` content — no Python code, no pyproject.toml. This is correct; the repo is a skills/commands provider.

**6 domain skills in `.claude/skills/` (security-review moved to platform-skills):**

| Skill | Has SKILL.md | Extra files |
|-------|-------------|-------------|
| lifesciences-clinical | Yes | — |
| lifesciences-crispr | Yes | `references/biogrid-orcs-validation.md` |
| lifesciences-genomics | Yes | — |
| lifesciences-graph-builder | Yes | — |
| lifesciences-pharmacology | Yes | — |
| lifesciences-proteomics | Yes | — |
| security-review | Yes | — |

**6 commands in `.claude/commands/`:**
- `graphiti-health.md`, `graphiti-verify.md`, `graphiti-aura-stats.md`, `graphiti-docker-stats.md`
- `scaffold-fastmcp.md`, `scaffold-fastmcp-v2.md`

Additional files:
- `.mcp.json` — contains only `biosciences-mcp` (FastMCP Cloud, Bearer auth via `${BIOSCIENCES_API_KEY}`)
- `CLAUDE.md` — detailed skill catalog, tool name inventory for gateway, scaffold update notes
- `docs/plans/2026-02-25-security-review-team.md` — implementation plan (task artifact, complete)
- `.gitignore` — standard Python gitignore, `.env` excluded (line 139)
- `LICENSE`, `README.md`

The `security-review` skill is a 4-agent orchestration skill (Coordinator + Secrets Scanner + Path Sanitizer + Config Validator). It uses the Task tool to dispatch agents in parallel, applies a tiered severity verdict (CRITICAL/HIGH block; MEDIUM/LOW warn), and is also integrated as Step 8 of the `migration-team` workflow.

### README Accuracy

The README is accurate and up to date. Confirmed:
- Correctly lists 7 skills including `security-review`
- Correctly lists 6 commands (4 Graphiti + 2 scaffold)
- Correctly notes SpecKit commands live in `biosciences-architecture`, not here
- Lists all 9 SpecKit command names accurately
- No false claims about content that does not exist

No gaps found in README accuracy.

### Unpushed Commits

**1 unpushed commit on main:**

```
d6386ec docs: update README with current platform facts
```

This commit is ahead of `origin/main`. The README update (platform-facts alignment) has not been pushed.

### Issues / Gaps Found

**Gap 1: Both scaffold commands still reference `lifesciences_mcp` import paths (stale after Wave 2)**

Both `scaffold-fastmcp.md` and `scaffold-fastmcp-v2.md` contain multiple references to the old package:
- `src/lifesciences_mcp/` path prefix throughout
- `from lifesciences_mcp.clients import ...`
- `from lifesciences_mcp.models import ...`
- `src/lifesciences_mcp/servers/` as the target scaffold location

The CLAUDE.md explicitly flags this under "scaffold-fastmcp Updates" and documents the correct new pattern (`biosciences_mcp`, `fastmcp deploy`). The commands have not yet been updated to match.

**Gap 2: Private paths in CLAUDE.md (LOW risk, but violates README design principle)**

`CLAUDE.md` lines 80–81 contain:
```
- Skills: `/home/donbr/graphiti-org/lifesciences-research/.claude/skills/`
- Commands: `/home/donbr/graphiti-org/lifesciences-research/.claude/commands/`
```

These are under a "Pre-Migration Source" section labeled "Until Wave 1 migration." Since Wave 1 is complete, this section is now stale and the private paths are no longer relevant.

The platform-facts-briefing.md states: "No internal paths in committed files (`/home/donbr/` must never appear)." This section should be removed.

**Gap 3: `security-review` SKILL.md contains hardcoded private paths**

The `SKILL.md` contains repeated hardcoded references to `/home/donbr/open-biosciences/` in all three agent prompts. These are functional shell commands required by the skill's runtime behavior, so they are by design — the skill is intentionally workspace-scoped. However, they trigger MEDIUM findings from the skill's own Path Sanitizer when applied recursively. The SKILL.md explicitly accounts for this ("workspace reference" context). This is a known, accepted tradeoff, not a bug.

**Gap 4: migration-tracker.md lists "15 SpecKit Commands" but only 9 exist**

The migration-tracker source table (line 9) references "15 SpecKit commands" from `lifesciences-research`. The actual deployed set in `biosciences-architecture` is 9 commands. This discrepancy lives in biosciences-program's migration-tracker, not in biosciences-skills, but is noted here as context.

### Recommended Next Actions

1. **Update `scaffold-fastmcp.md` and `scaffold-fastmcp-v2.md`** to replace all `lifesciences_mcp` import paths and directory references with `biosciences_mcp`. The CLAUDE.md already documents the required changes precisely. This is a Wave 2 cleanup item that was identified but not yet executed.

2. **Push the unpushed commit** (`d6386ec`) to `origin/main`. The README update is complete and clean.

3. **Remove the "Pre-Migration Source" section from CLAUDE.md** (lines 77–81). Wave 1 is complete; these predecessor paths are stale and violate the no-private-paths convention.

---

## biosciences-workspace-template

### Actual State

The repository is a **documentation-only stub**. It contains exactly 4 files beyond `.git`:

| File | Contents |
|------|----------|
| `README.md` | Wave 4 placeholder — describes what will be here |
| `CLAUDE.md` | Detailed operational notes: canonical `.mcp.json` template, bootstrap process spec, MCP server inventory, multi-agent CoWork patterns, Aura write-freeze notes |
| `.gitignore` | Standard Python gitignore, `.env` excluded (line 139) |
| `LICENSE` | MIT |

No `scripts/`, `templates/`, or `configs/` directories exist. No `bootstrap.sh`, `verify-env.sh`, or `init-repo.sh` files exist. The repo is entirely content-free from a functional standpoint.

The CLAUDE.md is notably more detailed than a typical placeholder — it contains the canonical `.mcp.json` configuration for all 6 MCP servers, a full server reference table, duplication audit notes (dated 2026-02-26), and a bootstrap script specification. This is planning/reference content embedded in CLAUDE.md pending Wave 4 implementation.

### README Accuracy

The README is accurate and correctly states its own status: "Wave 4 — Not Started. This repo contains no scripts or templates yet."

The README describes content that does not yet exist but frames it as future intent, not current fact. This is the intended pattern per platform-facts-briefing.md for Wave 4 repos ("describe what the repo will contain").

No accuracy gaps found. The README is honest about being a stub.

### Unpushed Commits

**1 unpushed commit on main:**

```
cc220d9 docs: update README with current platform facts
```

This commit is ahead of `origin/main`. The README update has not been pushed.

### Issues / Gaps Found

**Gap 1: Private paths in CLAUDE.md (multiple occurrences — violates README design principle)**

`CLAUDE.md` contains several `/home/donbr/` references:

- Line 79: `"command": "/home/donbr/graphiti-fastmcp/scripts/run_mcp_server.sh"` (in the canonical `.mcp.json` template — a config artifact that will be copied to other repos)
- Line 107: Same path in the server reference table
- Lines 119–121: Absolute paths in the duplication audit table
- Line 185: `/home/donbr/open-biosciences/.mcp.json` in the CoWork pattern notes

The most significant issue is the `graphiti-aura` stdio entry in the canonical `.mcp.json` template. This template is intended to be distributed as a bootstrap artifact via `scripts/bootstrap.sh`. If the template contains a hardcoded `/home/donbr/` path, it cannot be used on any other machine and will silently break for any new contributor.

The `graphiti-aura` stdio transport path needs to either be made configurable (via an env variable like `${GRAPHITI_FASTMCP_PATH}`) or the template needs to document that this entry must be customized per machine.

**Gap 2: CLAUDE.md duplication audit is stale as of Wave 3 completion**

The "Current State: Duplication Audit" section (around line 113) states that 3 repos have `.mcp.json` files and 8 repos have none. Per the migration-tracker, Wave 3 is now complete (2026-02-26), which added `.mcp.json` entries to `biosciences-deepagents` and `biosciences-temporal`. Additionally, `biosciences-skills` now has its own `.mcp.json` (with the `biosciences-mcp` entry). The audit table is now out of date.

This section was intended as a point-in-time snapshot; it should either be dated more explicitly or removed/updated.

**Gap 3: The repo is a Wave 4 stub — no functional content**

This is by design and correctly documented, but means the repo provides zero value to contributors until Wave 4 work begins. Any developer trying to bootstrap the workspace today has no automation to help them. This is an accepted state until Waves 1–3 are fully settled.

### Recommended Next Actions

1. **Fix the `graphiti-aura` stdio path in the canonical `.mcp.json` template** in CLAUDE.md. Replace the hardcoded `/home/donbr/graphiti-fastmcp/scripts/run_mcp_server.sh` with a configurable reference (e.g., `${GRAPHITI_FASTMCP_PATH}/scripts/run_mcp_server.sh` or a comment instructing the user to set it). This is the highest-priority fix because it affects any bootstrapped CLAUDE.md that another agent copies.

2. **Push the unpushed commit** (`cc220d9`) to `origin/main`. The README update is complete.

3. **Update or date-stamp the duplication audit section** in CLAUDE.md to reflect the current `.mcp.json` state after Wave 3 completion, or mark it as historical.

---

## Priority Summary

**Top 3 actions across both repos, ordered by importance:**

1. **Update `scaffold-fastmcp.md` and `scaffold-fastmcp-v2.md` in biosciences-skills** to replace all `lifesciences_mcp` references with `biosciences_mcp`. These are the only two actionable files that remain stale post-Wave 2. Agents invoking the scaffold commands will receive incorrect templates pointing to a non-existent package name. The CLAUDE.md already describes exactly what needs to change.

2. **Fix the hardcoded `/home/donbr/` path in biosciences-workspace-template CLAUDE.md** (the `graphiti-aura` stdio command in the canonical `.mcp.json` template). This path will be propagated to other machines if the bootstrap script copies this config, making the `graphiti-aura` server non-functional for any contributor who is not `donbr` on this specific machine.

3. **Push the unpushed commits in both repos** (`d6386ec` in biosciences-skills, `cc220d9` in biosciences-workspace-template). Both are ahead of `origin/main` by exactly 1 commit containing README platform-facts updates. These are clean, ready commits that simply have not been pushed.
