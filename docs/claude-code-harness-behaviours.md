# Claude Code harness behaviours — open questions and what is established

**Status:** living reference. Update in place as answers land; do not create a dated successor.
**Created:** 2026-09-12 · **Last verified:** 2026-09-12

## Why this file exists

Three harness questions blocked `biosciences-mcp` PR #14 (`feat(tooling): pr-merge-order`)
from leaving draft for nine days. The PR was closed 2026-09-12; the questions outlived it and
will block the next piece of workflow-based tooling unless they are recorded and answered.

They share a shape: **each is cheaply testable and nobody has run the test.** Each has sat as
an open question through at least one adversarial review, one research pass, and one PR
closure. That is the cost of leaving an empirical question in prose.

> **Update, 2026-09-13.** Q3 is now answered empirically — see below. Q1 and Q2 remain open.
>
> **Correction, 2026-09-12.** The closing comment on PR #14 claimed two of the three were
> answered by `docs/plans/2026-09-12-three-planes-claude-code-agents.md`. That was wrong.
> **None of the three are answered.** That document establishes *subagent* mechanics in
> detail and says nothing about *workflow agents* — the `agent()` calls inside a Workflow
> script — which is a different execution path. The adjacent findings below are useful
> context, not answers.

---

## Q1 — Does `agentType` apply a custom agent's frontmatter hooks and `skills:` in workflow agents?

**Status: OPEN.**

### What is established

- The Workflow tool's own documentation says `opts.agentType` is *"resolved from the same
  registry as the Agent tool"* and that it *"composes with schema (the custom agent's system
  prompt gets a StructuredOutput instruction appended)."* So the **system prompt** demonstrably
  applies. Hooks and `skills:` are not mentioned either way.
- Subagent frontmatter hooks are **YAML**; `settings.json` hooks are **JSON**; and subagent
  frontmatter hooks run **only while that subagent is running**.
  [`2026-09-12-three-planes-claude-code-agents.md:314`]
- Nested-subagent tool inheritance is explicitly **unspecified** in the first-party docs.
  [same doc, `:602`]

### What is not established

Whether a workflow `agent(prompt, {agentType: 'x'})` call loads agent `x`'s frontmatter hooks,
its `skills:`, or neither. The system prompt applying does not imply the rest do.

### The experiment (~15 min)

1. Define a custom agent type with (a) a `PreToolUse` frontmatter hook that appends a marker
   line to a scratch file, and (b) a `skills:` entry for a skill whose body contains a
   distinctive token.
2. Invoke it two ways against the same prompt: once via the `Agent` tool, once via a Workflow
   script's `agent(..., {agentType: 'x'})`.
3. Compare: does the marker file gain a line in each case? Does the response contain the
   skill's token in each case?

Four outcomes, all informative. Record which of the two paths honours which mechanism.

---

## Q2 — Do `settings.json` hooks fire in workflow agents?

**Status: OPEN.** This is the load-bearing one for any governance design.

### What is established

- `PreToolUse` hooks run **before** permission rules and block **even under
  `bypassPermissions`** — exit code `2`, or JSON `hookSpecificOutput.permissionDecision:
  "deny"`. This is the strongest control in the stack.
  [`2026-09-12-three-planes-claude-code-agents.md`, §1.3]
- Hooks run **in parallel** and cannot see each other's output, so labelling hooks cannot be
  chained. State-sharing via a temp file works only across *sequential* events
  (`PreToolUse` → `PostToolUse`).
- `type: "command"` hooks are subprocesses and deterministic; `type: "prompt"` hooks are an LLM
  and are **not** an enforcement plane.

### What is not established

Whether any of that reaches an agent spawned by a Workflow script. If `settings.json` hooks do
**not** fire in workflow agents, then every enforcement guarantee written against them has a
hole exactly where fan-out happens — which is where the least supervision is.

### Why it matters concretely here

`biosciences-mcp/.claude/settings.json` now registers `validate-curie.py` as a `PreToolUse`
hook on `.*_get_.*`. If a workflow agent calls `mcp__biosciences-mcp__hgnc_get_gene`, does the
guard fire? Unknown. The same question applies to `pr-review-guard.py`.

### The experiment (~15 min)

1. Register a trivial `PreToolUse` command hook in `settings.json` matching a harmless tool,
   which appends a marker line and exits `0`.
2. Trigger that tool three ways: from the main loop, from an `Agent` subagent, and from a
   Workflow script's `agent()`.
3. Count marker lines. Then repeat with the hook exiting `2` to confirm whether *blocking*
   also propagates, which is the property that actually matters.

---

## Q3 — What is the harness behaviour inside a worktree session?

**Status: ANSWERED EMPIRICALLY at claude 2.1.270, 2026-09-13. Not a documented contract.**

### Measured

Probe hook invoked by absolute path (so it fired regardless of the variable), registered via
`--settings` from a scratchpad, session started with cwd = a worktree. No repo file or config
touched.

```
CLAUDE_PROJECT_DIR = …/biosciences-mcp/.worktrees/curie-hook
PWD                = …/biosciences-mcp/.worktrees/curie-hook
$CLAUDE_PROJECT_DIR/.claude/hooks/validate-curie.py exists?  YES
```

**`$CLAUDE_PROJECT_DIR` resolves to the worktree, not the common root.**

From the same payload, `transcript_path` landed under a project namespace keyed to the worktree
path — `…-biosciences-mcp--worktrees-curie-hook` — distinct from the root's. **Claude Code treats
a worktree as its own project**, so a worktree session loads the worktree's own
`.claude/settings.json`. A branch that modifies a hook therefore tests its own version of it.

Also measured: **`$CLAUDE_PROJECT_DIR` is unset in ordinary Bash tool environments.** It exists
only during hook execution and cannot be read by an agent orienting itself mid-task.

### What remains open

The worktree's own `settings.json` hook was **not observed firing** — its matcher (`.*_get_.*`)
had no triggerable tool in that environment, and triggering it would have required editing repo
config or a live API key. The conclusion rests on the namespace assignment plus the variable
resolution, not on a fired hook. The one-call test: a session started in the worktree makes any
`mcp__biosciences-mcp__*_get_*` call with a deliberately invalid CURIE; a `deny` carrying the
`validate-curie:` prefix confirms it.

### Why this does not close the design question

This is an observation at one version, not a guarantee. If upstream changes project-root
derivation, every hook depending on it changes behaviour silently. **The durable answer is that
hooks should self-locate** — the `PreToolUse` stdin payload carries `cwd`, and

```sh
[ "$(git rev-parse --git-dir)" != "$(git rev-parse --git-common-dir)" ]   # true in a worktree
git rev-parse --show-toplevel                                            # correct root in both
```

detects a worktree with no harness facility at all. Recorded as a decision in ADR-PRG-001 §2.4.
A self-locating hook is also testable outside Claude Code, which the org's current hooks are not.

### Superseded prior text



### What is established

- Subagent `isolation: 'worktree'` puts the agent in its own branched checkout and is a **real
  filesystem boundary** — unlike MCP Roots, which never enforced anything and is now deprecated
  under SEP-2577. [`2026-09-12-three-planes-claude-code-agents.md:433`, `:523`]
- Worktree-scoped `settings.local.json` **precedence is undocumented**. [same doc, `:602`]
- Empirically, from this session: entering a worktree changes the reported primary working
  directory, and the git stash stack is **shared** across the main checkout and all worktrees.

### What is not established

Which `settings.json` applies inside an `EnterWorktree` session — the parent repo's, the
worktree's, or a merge; whether hooks registered in the parent fire; and whether
`$CLAUDE_PROJECT_DIR` resolves to the worktree or the original checkout. That last one is not
academic: `biosciences-mcp/.claude/settings.json` invokes its hook as
`$CLAUDE_PROJECT_DIR/.claude/hooks/validate-curie.py`, so if the variable points at the main
checkout while work happens in a worktree, the hook runs the *wrong copy* of itself.

### The experiment (~10 min)

1. In a worktree, run a command that echoes `$CLAUDE_PROJECT_DIR`.
2. Place a distinguishable `settings.local.json` in both the main checkout and the worktree;
   observe which permission rules apply.
3. Register a marker hook in the parent's `settings.json` only; act inside the worktree;
   check whether it fires.

---

## Standing rule this suggests

All three questions were raised by an adversarial review on 2026-09-03, restated as PR
blockers, carried through a research pass on 2026-09-12, and are still open on 2026-09-12 —
while each is a fifteen-minute experiment.

**An empirical question about harness behaviour should be answered by running it, not by
researching it.** Research establishes what the documentation *claims*; only execution
establishes what the harness *does*, and the two have diverged at least twice in this
workspace (subagent `permissionMode` frontmatter being silently ignored under a permissive
parent; MCP Resources being model-pulled in Claude Code despite the spec's application-driven
framing).

When the answers land, record them here with the date and the command that produced them, and
delete the corresponding experiment section.

---

## Related

- `docs/plans/2026-09-12-three-planes-claude-code-agents.md` — full subagent, hook and
  permission mechanics; the source for every "established" item above
- `biosciences-mcp` PR #14 (closed 2026-09-12) — where these blocked, with the adversarial
  review that first raised them at
  `biosciences-mcp/docs/adr/critique/pr-review-tooling-2026-09-03/adversary-round2.md`
- `biosciences-mcp/.claude/hooks/validate-curie.py` and `pr-review-guard.py` — the two hooks
  whose guarantees depend on Q2 and Q3
