# ADR-PRG-001: Git workflow and worktree discipline for agent sessions

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-09-13 |
| **Scope** | Program — binds every repository in the `open-biosciences` org |
| **Owner** | Program Director |
| **Extends** | ADR-005 (Git Worktrees for Parallel MCP Server Development) |
| **Grounded in** | `docs/official-claude-code-git-ci-practices.md` — what upstream actually prescribes, retrieved 2026-09-13 |
| **Placement** | Per `docs/adr/README.md`, cross-repo process decisions live here as `ADR-PRG-NNN`. This is the first. |

> **Read §2 before §3.** Most of what follows is *invented*, and says so. Upstream documentation
> prescribes far less about git workflow than one would expect, and an earlier draft of this ADR
> cited upstream backing it does not have.

---

## 1. Context

Multiple Claude Code sessions work across this org concurrently — on 2026-09-12 four repositories
had open PRs from at least three sessions. ADR-005 established worktrees for one problem: several
agents implementing different servers at once, writing to the same files.

### The incident

From the `biosciences-mcp` reflog, 2026-09-03 → 2026-09-12: **four checkouts across two branches in
nineteen seconds**; the root checkout then **left on a feature branch for nine days**; then
`git pull origin main` run **while standing on it**, silently fast-forwarding the branch onto
`origin/main`. Result: a dead-alias branch, local `main` six commits stale, three files untracked
and claimed by no branch. At least two other sessions hit the same shape independently.

### What caused it — three wrong answers before the right one

A session described the absent worktree as having **"no good reason."** That phrase is why this ADR
exists: it closes an investigation where it should open. What followed is recorded because the
errors are the evidence.

1. **"The convention was undiscoverable" vs "it was ignored."** Both rejected. A reflog records
   what git did, not what a session knew. Neither is inferable from it.
2. **"It gitignored the upstream-docs path while this repo uses `.worktrees/`."** False.
   `.gitignore:209-213` carries both entries, correctly labelled for two mechanisms.
3. **"ADR-005 didn't reach the case, so it's a gap not a violation."** True, and the reason this
   ADR is warranted — but incomplete, because it assumed the *upstream* convention was clear and
   only the local one was missing.

**The complete answer, established 2026-09-13 by scouting official documentation:** there is no
upstream rule to violate. Official Claude Code documentation contains exactly **one** conditional
for using a worktree — *"Do the tasks touch the same files? Isolate the work with worktrees"* —
and says nothing about worktrees for feature work, which branch the primary checkout should sit on,
or post-merge cleanup. `.worktrees/` as a location **appears in no official page at all.**

So the 2026-09-03 session violated no documented practice, local or upstream. The rule being
enforced had never been written **by anyone.** That is what this ADR fixes, and it is why every
clause below is labelled by provenance.

---

## 2. Provenance of every clause

| Clause | Status | Basis |
|---|---|---|
| §3.1 Primary checkout on the default branch | **INVENTION** | Upstream unanswered. Partially supported by the prescribed *"Launch from the main checkout instead."* |
| §3.2 Worktree triggers | **PART PRESCRIBED, PART INVENTION** | *"Do the tasks touch the same files?"* is upstream. The session-boundary trigger is ours. |
| §3.3 Worktrees live in `.worktrees/` | **INVENTION** | `.worktrees/` appears in no official page. Compatible with the docs, not derived from them. |
| §3.4 Cleanup at merge | **INVENTION** | Upstream unanswered for manual worktrees; the automatic sweep explicitly leaves them alone. |
| §3.5 Hooks read `cwd` from stdin | **PRESCRIBED** | *"`${CLAUDE_PROJECT_DIR}` stays put… `cwd` follows Claude… Read it when a hook needs the worktree path."* |
| §3.6 Launch from the main checkout | **PRESCRIBED** | *"you launched Claude Code from inside the worktree. Launch from the main checkout instead."* |
| §3.7 Worktree-facing settings live in the repo root | **PRESCRIBED** | *"Put any other settings you need inside worktrees… in the repository root's `.claude/settings.json`."* |
| §3.8 Enforcement carrier | **PRESCRIBED** | *"If a rule must hold every time, make it a hook rather than a prompt instruction."* |
| §3.9 The gate itself | **INVENTION** | Branch-shaped gating appears nowhere upstream; every documented git control is command- or path-shaped. |
| §4 Stash protocol | **INVENTION** | Upstream enumerates what worktrees share but gives no protocol. |

**A clause marked INVENTION is not thereby wrong.** Upstream explicitly delegates: *"Repository
etiquette (branch naming, PR conventions)"* is listed as CLAUDE.md content **you supply**. The
label exists so no one cites this ADR as upstream practice.

---

## 3. Decision

### 3.1 The primary checkout stands on the default branch — *invention*

A resting-state invariant, not a prohibition on commands. `git checkout -b` is not forbidden;
leaving the primary checkout parked elsewhere is what caused harm.

### 3.2 When a worktree is required

| Situation | Worktree | Basis |
|---|---|---|
| Tasks touch the same files | **Required** | **prescribed** upstream |
| Parallel agents on one repo | **Required** | ADR-005 |
| Work that will outlive the current session | **Required** | *invention* — the trigger the incident needed |
| A throwaway inspection, committed and merged inside one session | Optional | — |

### 3.3 Manual worktrees live in `.worktrees/` — *invention*

Already gitignored across the org. Upstream uses `.claude/worktrees/` for Claude-created worktrees
and sibling paths (`../project-feature-a`) in examples. **Both entries belong in `.gitignore`;** they
are different mechanisms and the 2026-09-03 session was right to add the second.

### 3.4 Cleanup is part of merging — *invention*

On merge, in the same session: remove the worktree, delete the local branch, return the primary
checkout to an up-to-date default branch. Upstream's automatic sweep does not touch manual
worktrees, so nothing does this for us.

### 3.5 Hooks read `cwd` from the stdin payload — *prescribed*

**This supersedes an earlier draft of this ADR**, which invented a `git rev-parse --git-dir` vs
`--git-common-dir` worktree detector. That works, but it is unnecessary: upstream already prescribes
the answer. `${CLAUDE_PROJECT_DIR}` **stays at the launch root**; `cwd` follows Claude. A hook that
needs the worktree path reads `cwd`.

### 3.6 Launch Claude from the main checkout, not from inside a worktree — *prescribed*

Upstream troubleshooting names launching from inside a worktree as the fault condition.
**Consequences for §5.2, which measured behaviour in exactly that configuration.**

### 3.7 Settings meant for worktrees live in the repository root's `.claude/settings.json` — *prescribed*

A worktree loads *the checked-out copy* of the repo-root file. Note also that `settings.local.json`,
saved permission approvals, and project-scope plugins are **shared with the main checkout**, not
per-worktree.

### 3.8 A rule that must hold every time is a hook, not a document — *prescribed*

And the consequence this ADR must state about itself:

> Upstream never names a decision record as an enforcement mechanism. **An ADR is a durable
> rationale artifact; it is not loaded at the moment of decision.**

So **this document cannot make anything stick.** Every clause above must compile down to a carrier,
and §6 records which. Upstream's escalation ladder is the map:

- convention got wrong twice → **CLAUDE.md**
- same prompt retyped → **a skill**
- must happen every time without asking → **a hook**
- a second repository needs it → **a plugin**

### 3.9 Enforcement: one gate, two advisories — *invention, with prescribed constraints*

**Gate.** A `PreToolUse` hook returning `permissionDecision: "ask"` on a pull or merge of
`origin main` while `cwd` is the primary checkout and `HEAD != main`. This is precisely the 23:19
event; legitimate cases are near-zero, so the prompt stays rare enough to be read.

Four prescribed constraints bound the design:

- **`exit 2` blocks; `exit 1` does not.** Return JSON or exit 2 — never exit 1.
- **The `if` pre-filter is best-effort.** Upstream: *"use the permission system rather than a hook
  to enforce a hard allow or deny."* Our gate is an `ask`, not a hard deny, which is why a hook is
  the right carrier.
- **Bash rules match command text and are not a security boundary.** `Bash(git push *)` misses
  `git -C . push`. Upstream states this; we are not the ones who discovered it.
- **A timed-out hook does not block.** Don't count on a stalled hook as a gate.

**Advisories.** `SessionStart` reports the primary checkout's branch, divergence, and whether its
PR merged. `SessionEnd` reports merged branches and worktrees still present. Non-blocking, no
false-positive cost.

**Unresolved, and quoted rather than reconciled:** upstream endorses command-gating hooks (the
flagship walkthrough blocks `rm -rf`; the canonical `if` example calls `check-git-policy.sh`) while
also saying to prefer the permission system for hard allow/deny. Both are official.

---

## 4. Rejected: a gate on branch creation

Three grounds, one of which is now upstream-confirmed:

1. **Wrong event.** Branch creation is harmless and reversible. Neither real defect — the 19-second
   thrash, the nine-day park — is prevented by it.
2. **Command-text matching is bypassable, and upstream says so.** `Bash(git checkout -b*)` misses
   `git switch -c`, `checkout -B`, `bash -c '…'`, heredocs. This was independently found in
   `biosciences-mcp/docs/adr/critique/pr-review-tooling-decision-record-2026-09-03.md`; upstream
   documents the same limitation directly.
3. **It trains click-through.** An `ask` on a cheap, frequent, legitimate operation is answered
   *proceed* by reflex, stops being read, and then looks like coverage while being none.

That rejected hook was this ADR's own first proposal, designed by the session that had transcribed
ground (2) into a tracker comment the day before. Recorded as evidence for §3.8, not as anecdote:
**prose does not bind its own author.**

---

## 5. Traps

### 5.1 A resting state cannot be caught by a `PreToolUse` hook

The nine-day park is not an action; no tool-call hook can observe it. This is why §3.9 pairs one
gate with periodic reporting rather than a second gate. **The class of defect determines the class
of control.**

### 5.2 `${CLAUDE_PROJECT_DIR}` — a measurement taken in a configuration upstream tells you not to use

Measured at claude 2.1.270, session launched **inside** a worktree:

```
CLAUDE_PROJECT_DIR = …/biosciences-mcp/.worktrees/curie-hook
```

**This does not contradict the documentation, and it should not be relied on.** Upstream defines the
variable as *"the project root where the session started"* and states that it **stays put** when
Claude *enters* a worktree mid-session. Our measurement launched *in* the worktree — the exact
configuration §3.6 says to avoid — so it measured the launch root, consistently with the definition.

Two corrections to an earlier draft follow: the behaviour is **not** "resolves to the worktree" as a
general rule, and the conclusion that a worktree has *its own settings* is **half right** —
`settings.local.json`, permission approvals and project-scope plugins are shared with the main
checkout.

**Therefore §3.5, not this measurement, is the durable answer.** Read `cwd` from stdin.

### 5.3 `git branch -d` does not check what it appears to

With an upstream present, `-d` refuses unless the branch is merged **into that upstream** — not into
`main`. A branch fast-forwarded onto `origin/main` while standing on it reads as *unmerged* against
its own stale remote. Verify with `git rev-list --count <branch> ^main` before `-D`; `0` means safe.
*(Contributed by the `biosciences-mcp-wsl` session, 2026-09-13.)*

### 5.4 The shared git stash stack — *invention, zero upstream backing*

Worktrees share `.git`, so they share the stash stack; upstream never warns about this. Local rule:
never bare `git stash` / `git stash pop`. Use `git stash push -u -m "<unique-tag>"`, capture the SHA,
restore with `apply <sha>`, drop by tag. Prefer a WIP commit.

---

## 6. Which carrier each clause compiles to

Per §3.8, this document enforces nothing. Status is **DEFERRED** across the board — nothing is
written yet, and the table says so rather than implying coverage.

| Clause | Carrier | State |
|---|---|---|
| §3.1 primary checkout on default branch | `SessionStart` advisory | deferred |
| §3.2 worktree triggers | CLAUDE.md (context) | deferred |
| §3.3 `.worktrees/` location | `.gitignore` (already present) | **done** |
| §3.4 cleanup at merge | `SessionEnd` advisory | deferred |
| §3.5 hooks read `cwd` | code review; **both existing org hooks violate it** | outstanding |
| §3.6 launch from main checkout | CLAUDE.md (context) | deferred |
| §3.7 worktree settings in repo root | code review | deferred |
| §3.9 gate | `PreToolUse` hook | deferred |
| §5.4 stash protocol | CLAUDE.md (context) | deferred |

`biosciences-mcp/.claude/hooks/validate-curie.py` and `pr-review-guard.py` both resolve through
`${CLAUDE_PROJECT_DIR}` rather than stdin `cwd`, contrary to §3.5. Neither is broken today; both
should migrate.

---

## 7. Consequences

**Positive.** The nine-day park becomes visible on day two. The primary checkout is always a usable
default branch. Work outliving a session has a home. Hooks follow the prescribed path-resolution
rule instead of an invented one.

**Cost.** One prompt in a rare situation; three hooks; a worktree per PR.

**Risk.** Most clauses are inventions. If upstream later prescribes something incompatible, §2's
provenance table is what makes the conflict findable.

**Explicitly not decided.** Whether `.claude/settings.json` and hooks should be per-branch or shared
across worktrees. §3.7 records the documented mechanism; choosing a policy on top of it is separate.

---

## 8. Review

Re-verify §5.2 and the §2 provenance table on each Claude Code minor version bump — upstream
documentation changes, and a clause marked INVENTION may become PRESCRIBED. Revisit §3.9's gate if
the prompt proves too frequent to be read or too rare to matter.

## 9. References

- `docs/official-claude-code-git-ci-practices.md` — every upstream citation above, with URLs and
  retrieval date
- ADR-005 — `biosciences-mcp/docs/adr/accepted/adr-005-v1.0.md`
- `docs/claude-code-harness-behaviours.md` — the living register for measured harness behaviour
- `biosciences-mcp/docs/adr/critique/pr-review-tooling-decision-record-2026-09-03.md`
- `docs/adr/README.md` — the placement rule reserving this series
