# ADR-PRG-001: Git workflow and worktree discipline for agent sessions

| | |
|---|---|
| **Status** | Proposed |
| **Date** | 2026-09-13 |
| **Scope** | Program — binds every repository in the `open-biosciences` org |
| **Owner** | Program Director |
| **Extends** | ADR-005 (Git Worktrees for Parallel MCP Server Development), which it does not replace |
| **Placement** | Per `docs/adr/README.md`, cross-repo process decisions live here as `ADR-PRG-NNN`. This is the first. |

---

## 1. Context

Multiple Claude Code sessions work across this org concurrently — on 2026-09-12 four repositories had open PRs simultaneously, authored by at least three different sessions. ADR-005 established worktrees, but for one specific problem: several agents implementing different servers at once, writing to the same files. Its own framing is *"3 agents write to the same file → merge conflicts guaranteed."*

That leaves a gap, and the gap has cost us.

### The incident

Reconstructed from the reflog in `biosciences-mcp`, 2026-09-03 to 2026-09-12:

- **21:14:55 → 21:15:14.** Four checkouts across two branches in **nineteen seconds**, all in the root checkout, to hold two pieces of work in one working directory.
- **The root checkout was then left on a feature branch for nine days.**
- **2026-09-12 23:19.** `git pull origin main` was run *while standing on that feature branch*, silently fast-forwarding it onto `origin/main`. The branch became a dead alias for main; local `main` sat six commits stale; three files were untracked and claimed by no branch.

A session investigating this described the absent worktree as having **"no good reason."** That phrase is the reason this ADR exists — not because it was rude, but because it closes an investigation exactly where it should open, and it names no cause anyone can act on.

### The cause, stated honestly

The first analysis proposed two candidates: the convention was not discoverable at the moment of decision, or it was discoverable and skipped. **Both were rejected, and the framing was the error.**

A reflog records what git did, not what a session knew or weighed. Neither candidate is inferable from it. The supporting evidence assembled for the first — that the session gitignored the upstream-docs path `.claude/worktrees/` while this repo uses `.worktrees/`, implying it was reading documentation instead of the repository — **is false.** `.gitignore:209-213` carries both entries, correctly labelled for two different mechanisms, and `.worktrees/` was already present.

The actual cause is simpler and more useful:

> **ADR-005 does not reach this case.** It scopes worktrees to parallel implementation. The 2026-09-03 work was one session, sequential, documentation-only. A convention that does not cover a case cannot be bypassed in it.

This is a **gap, not a violation.** The rule now being enforced had never been written, which is why nothing enforced it. That matters for what follows: *enforcement is downstream of stating the rule, never a substitute for it.*

---

## 2. Decision

### 2.1 The root checkout stands on the default branch

A repository's primary working directory stays on `main`. Branch work happens in a worktree under `.worktrees/`, already gitignored in every repo in this org.

This is stated as a resting-state invariant, not a prohibition on commands. `git checkout -b` is not forbidden — leaving the root checkout parked somewhere else is what causes harm.

### 2.2 A worktree is required when work spans a session boundary

ADR-005's trigger was parallel agents. This adds a second, and it is the one the incident needed:

| Situation | Worktree |
|---|---|
| Parallel agents on one repo | **Required** (ADR-005) |
| Work that will outlive the current session | **Required** |
| A branch expected to become a PR | **Required** |
| A throwaway inspection, committed and merged inside one session | Optional |

The second row is the operative one. The nine-day park happened because work that outlived its session had nowhere to live except the root checkout.

### 2.3 Cleanup is part of merging, not a later chore

When a PR merges, in the same working session: remove the worktree, delete the local branch, and return the root checkout to an up-to-date `main`. A merged worktree left in place is indistinguishable from active work.

### 2.4 Hooks self-locate; they do not depend on injected environment variables

Any hook this org ships derives its own location. The `PreToolUse` stdin payload carries `cwd` directly, and a worktree is detectable with no harness facility at all:

```sh
[ "$(git rev-parse --git-dir)" != "$(git rev-parse --git-common-dir)" ]   # true inside a worktree
git rev-parse --show-toplevel                                            # correct root in both cases
```

Rationale in §4.3. A hook built this way is also testable outside Claude Code, which the ones currently in the org are not.

### 2.5 Enforcement: one gate, two advisories

Deliberately minimal, and deliberately not aimed at branch creation.

**Gate — `PreToolUse`, `permissionDecision: "ask"`.** A pull or merge of `origin main` while the cwd is the root checkout and `HEAD != main`. This is precisely the 23:19 event. Legitimate cases are near-zero — it is almost always a slip for *"switch to main, then pull"* — so the prompt stays rare enough to actually be read.

**Advisory — `SessionStart`.** Report the root checkout's branch, its divergence, and whether its PR is merged. This is the only mechanism that reaches the nine-day park, for a structural reason recorded in §4.1.

**Advisory — `SessionEnd`.** Report merged branches and worktrees that still exist.

Both advisories are non-blocking and have no false-positive cost.

---

## 3. What was rejected, and why

**A gate on branch creation** (`Bash(git checkout -b*)` or similar). Rejected on three independent grounds:

1. **It guards the wrong event.** Branch creation is harmless and reversible in seconds. Neither defect the reflog supports — the 19-second thrash, the nine-day park — is prevented by a hook that fires at `checkout -b`.
2. **Prefix matching on command text is bypassable, and we have already written this down.** The surviving finding of `biosciences-mcp/docs/adr/critique/pr-review-tooling-decision-record-2026-09-03.md` is that a guard matching the outer command verb is defeated by interpreter indirection. `Bash(git checkout -b*)` misses `git switch -c`, `checkout -B`, `git branch x && git checkout x`, `bash -c '…'`, and heredocs.
3. **It trains click-through.** An `ask` on a cheap, frequent, usually-legitimate operation is answered *proceed* by reflex, stops being read, and is then worse than nothing because it looks like coverage.

The first proposal in this ADR's own drafting was exactly that rejected hook — proposed by the session that had transcribed finding (2) into a tracker comment the day before. Recorded because it is evidence for §2 rather than an anecdote: **a written finding did not survive contact with a design decision twenty-four hours later.** Prose does not bind its own author.

---

## 4. Traps, each measured

### 4.1 A resting state cannot be caught by a `PreToolUse` hook

The nine-day park is not an action. No hook on any tool call can observe "this checkout has been on the wrong branch since last Tuesday," because there is no event. This is why §2.5 pairs one gate with `SessionStart` reporting rather than reaching for a second gate — **the class of defect determines the class of control**, and a non-event needs a periodic observer.

### 4.2 `git branch -d` does not check what it appears to

When a branch has an upstream, `-d` refuses unless the branch is merged **into that upstream** — not into `main`. A branch fast-forwarded onto `origin/main` while standing on it therefore reads as *unmerged* relative to its own stale remote:

```
warning: not deleting branch 'docs/…' that is not yet merged to
         'refs/remotes/origin/docs/…', even though it is merged to HEAD.
```

Verify with `git rev-list --count <branch> ^main` before reaching for `-D`. If that returns `0`, the force is safe. *(Finding contributed by the `biosciences-mcp-wsl` session, 2026-09-13.)*

### 4.3 `$CLAUDE_PROJECT_DIR` resolves to the worktree — measured, not documented

Measured at **claude 2.1.270** with a probe hook invoked by absolute path, session cwd set to a worktree:

```
CLAUDE_PROJECT_DIR = …/biosciences-mcp/.worktrees/curie-hook
PWD                = …/biosciences-mcp/.worktrees/curie-hook
```

Claude Code also assigns a worktree **its own project namespace**, keyed to the worktree path, so a worktree session loads the worktree's `.claude/settings.json` rather than the root's. A branch that modifies a hook therefore tests its own version — which is the behaviour we want.

Two boundaries on this result, both load-bearing:

- **It is an observation, not a contract.** If upstream changes project-root derivation, every hook depending on it changes behaviour silently. That is the whole argument for §2.4.
- **Partially evidenced.** The worktree's own hook was not observed *firing* — its matcher had no triggerable tool in that environment. The conclusion rests on the namespace assignment plus the variable resolution.

Also measured: **`$CLAUDE_PROJECT_DIR` is unset in ordinary Bash tool environments.** It exists only during hook execution and cannot be read by an agent orienting itself mid-task.

Recorded in `docs/claude-code-harness-behaviours.md`, which is the living register for findings of this kind.

---

## 5. Consequences

**Positive.** The nine-day park becomes visible on day two. The root checkout is always a usable `main`. Work that outlives a session has a home that is not the root checkout. Hooks stop depending on an undocumented harness behaviour.

**Cost.** One more prompt in a rare situation; three hooks to write and maintain; a worktree per PR, which is the cost ADR-005 already accepted.

**Risk.** The §4.3 measurement is version-pinned. §2.4 is the mitigation: a self-locating hook survives the behaviour changing.

**Explicitly not decided.** Whether `.claude/settings.json` and hooks should be shared across worktrees or owned per-branch. The current behaviour — per-worktree — is what §4.3 measured and what this ADR assumes; changing it is a separate decision.

---

## 6. Compliance

| Obligation | Verified by | State |
|---|---|---|
| §2.1 root checkout on default branch | `SessionStart` advisory | hook not yet written |
| §2.3 cleanup at merge | `SessionEnd` advisory | hook not yet written |
| §2.4 hooks self-locate | review; the two existing hooks do **not** comply | outstanding |
| §2.5 gate | `PreToolUse` on pull/merge | hook not yet written |

**Every obligation here is currently advisory, and this table says so rather than implying coverage.** Per `psychology-mcp`'s constitution, obligations report as *checked*, *deferred* or *uncheckable* — never a blanket pass. All four are **deferred** until the hooks land.

`biosciences-mcp/.claude/hooks/validate-curie.py` and `pr-review-guard.py` both invoke through `$CLAUDE_PROJECT_DIR` and do not self-locate. Per §2.4 they should migrate; neither is broken today, since §4.3 shows the variable resolves correctly at 2.1.270.

---

## 7. Review

Re-verify §4.3 on each Claude Code minor version bump, or when a hook that depends on worktree resolution misbehaves. Revisit §2.5's gate if the prompt proves either too frequent to be read or too rare to matter.

## 8. References

- ADR-005 — `biosciences-mcp/docs/adr/accepted/adr-005-v1.0.md`
- `docs/claude-code-harness-behaviours.md` — the living register for measured harness behaviour
- `biosciences-mcp/docs/adr/critique/pr-review-tooling-decision-record-2026-09-03.md` — origin of the interpreter-indirection finding in §3
- `docs/adr/README.md` — the placement rule that reserves this series
