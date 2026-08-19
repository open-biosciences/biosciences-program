# Checkout Duplication Audit — 2026-08-15

**Linear issue:** [AGE-567](https://linear.app/agentic-wisdom/issue/AGE-567/p2-checkout-sprawl-same-repos-cloned-across-3-roots-wsl-x2-windows)  
**Scope:** `/home/donbr/open-biosciences`, `/home/donbr/hci`, and `C:\Users\donbr\codex` (observed from WSL as `/mnt/c/Users/donbr/codex`)  
**Method:** normalized remotes, pre-fetch Git state, fetched remote refs for the duplicated Open Biosciences repositories, post-fetch classification, and read-only GitHub API checks for HCI repositories.

## Executive conclusion

The proposed root convention is supported by the evidence:

- `/home/donbr/open-biosciences` is the strongest authoritative root for repositories owned by `open-biosciences`. Every duplicated organization repository present there is clean and current with its upstream branch.
- `/home/donbr/hci` should remain observation-only until the active Claude Code work there is complete. It contains active dirty work, clean but stale copies, pushed worktrees, and local-only HCI feature commits.
- The Windows root contains both clean redundant clones and the highest-risk stranded work. Two files under the Windows `open-biosciences-plugins` clone remain untracked and unique by filename, and `hci-storyboard-runs` remains five commits ahead with 39 dirty entries.
- No checkout should be deleted yet.

## Safety and mutation record

The audit began with a read-only snapshot. Before the instruction to keep `/home/donbr/hci` observation-only was received, `git fetch --all --prune --tags` was run for the duplicated Open Biosciences repositories across the three roots. This included 11 checkout paths under `/home/donbr/hci`. Fetch updated remote-tracking metadata and `FETCH_HEAD`; it did not change local branches, the index, or working files. Pre-fetch and post-fetch working-tree states were captured.

After the observation-only instruction, no further Git mutation was performed under `/home/donbr/hci`. A later HCI-wide fetch attempt was interrupted; existing `FETCH_HEAD` timestamps show that it did not refresh the additional HCI repositories, and no fetch process remained running. Current GitHub state for those repositories was obtained through the read-only GitHub API instead.

The workspace-required legacy Graphiti priming tools were not exposed in this session. This report therefore relies on direct filesystem, Git, and GitHub evidence.

## Open Biosciences duplicate inventory

Thirteen organization repository identities occur in more than one checkout, producing 37 checkout paths:

| Repository | Checkout paths | Verified state |
|---|---:|---|
| `biosciences-architecture` | 3 | All clean and identical at `0f4ce3f6dd66` |
| `biosciences-deepagents` | 3 | All clean and identical at `fe58ca47ea14` |
| `biosciences-education` | 3 | All clean and identical at `19be1cf6f188` |
| `biosciences-evaluation` | 2 | Both clean and identical at `4c6cd999c086` |
| `biosciences-mcp` | 3 | All clean and identical at `5d81a8f13df9` |
| `biosciences-mcp-edge` | 3 | Open Biosciences copy is current at `de77110bbaaf`; HCI and Windows copies are clean and two commits behind at `b29d47138011` |
| `biosciences-memory` | 2 | Both clean and identical at `2248e9d0a21f` |
| `biosciences-program` | 3 | All clean and identical at `111f53b02873` |
| `biosciences-research` | 2 | Both clean and identical at `a6d5be12fe4d` |
| `biosciences-skills` | 3 | All clean and identical at `6afa94006568` |
| `biosciences-temporal` | 3 | All clean and identical at `3ad5c3f87033` |
| `biosciences-workspace-template` | 3 | All clean and identical at `f5801f881886` |
| `open-biosciences-plugins` | 4 | Main is `f368ce2c904f`; the HCI feature worktree is clean and pushed at `b2b68c5ab1bd`; the Windows main clone has two unique untracked files |

Summary after remote refresh:

- 30 paths belong to 11 wholly clean, commit-identical duplicate groups.
- The authoritative `biosciences-mcp-edge` checkout is current; the two redundant copies are clean and stale by two commits.
- The `feat/psychology-connector-research` worktree is clean, tracks `origin/feat/psychology-connector-research`, and has no unpushed commits. Its six commits and two design artifacts are durable on GitHub.
- No stashes were found in the 37 Open Biosciences duplicate paths.
- No local-only commits remain in those duplicate groups after refreshed remote refs.

## Unresolved Open Biosciences artifacts

The Windows `open-biosciences-plugins` clone is on current `main` but contains two untracked files found nowhere else by filename:

| File | Size | State |
|---|---:|---|
| `docs/2026-08-13-diagnosis-counseling-findings-and-plugin-backlog.md` | 38,191 bytes | Untracked, unique, not durable |
| `psy-gate-fixes.patch` | 12,667 bytes | Untracked, unique, not durable |

These are separate from the pushed psychology connector design specification and implementation plan. Pushing `feat/psychology-connector-research` resolved that branch's durability concern, but it did not rescue these two Windows-only files.

## HCI and adjacent duplicate groups — observation only

No local refs were refreshed for this section after the observation-only instruction. GitHub default-branch SHAs were queried without modifying local repositories.

| Repository | Observation |
|---|---|
| `donbr/hci-canon` | WSL and Windows `main` both match GitHub at `cf990c03dc5f`. WSL main has 18 dirty entries (7 tracked, 11 untracked). Windows main reports 1,211 untracked entries, primarily content under the nested `.worktrees/age-542` path. |
| `hci-canon` feature worktree | `/home/donbr/hci/hci-canon-gate-metric-normalization` is clean at pushed branch commit `6d9451837fb9`. |
| `hci-canon` local feature history | The WSL clone contains four commits absent from remote branches: one on `feat/pack-identity-only` and three on `feat/canonical-timestamp-emission`. |
| `donbr/hci-storyboards` | GitHub `main` and Windows are at `6373b48d543b`. The WSL committed HEAD is 44 commits stale at `b150badb3f08` and its working tree has two untracked files. Windows has 11 dirty entries (1 tracked, 10 untracked). |
| `donbr/story-suite` | GitHub and Windows are at `9a17845cbc42`. The clean WSL copy is four commits stale at `9438793c5acc`. |

The Windows `hci-canon` clone reports nine worktree records total: the main checkout plus eight auxiliary records. When inspected from WSL, all eight auxiliary records are marked prunable because their gitdir paths use Windows `C:/...` syntax. That WSL result is not sufficient evidence that the worktrees are actually abandoned on Windows; they must be audited from a native Windows Git session before pruning. Ten sibling `hci-canon*` directories are present: one standalone clone, five directories with Windows worktree-link files, and four directories without a `.git` entry.

## Stranded work named by AGE-567

`C:\Users\donbr\codex\hci-storyboard-runs` remains unresolved:

- `main` is five commits ahead of GitHub commit `2eb0e50b6fdf`.
- The working tree contains 39 entries: 6 tracked changes/deletions and 33 untracked files.
- GitHub does not recognize local HEAD `d1ed983` in the repository comparison endpoint, consistent with the commits being unpushed.

The two Windows-only plugin artifacts listed above are also unresolved.

## Recommended disposition

1. Keep `/home/donbr/hci` unchanged while the active Claude Code instance is working there.
2. Declare `/home/donbr/open-biosciences` authoritative for `open-biosciences/*` repositories in workspace documentation.
3. Rescue the two Windows-only plugin files into a reviewed branch or other durable store.
4. Push or bundle the five `hci-storyboard-runs` commits and preserve its 39 dirty entries before cleanup.
5. Review the four local-only `hci-canon` commits and the dirty WSL/Windows trees with the active HCI session owner.
6. Audit Windows `hci-canon` worktrees from native Windows Git before any `worktree prune` or directory removal.
7. Once all valuable work is durable, remove or explicitly mark redundant stale clones as read-only references in a separate cleanup step.

## Acceptance-criteria status

| AGE-567 criterion | Status |
|---|---|
| Authoritative root per project family decided and documented | 🟡 Evidence supports the proposed convention; formal workspace documentation remains |
| Zero unpushed commits and valuable untracked files in non-authoritative copies | ⛔ Not met |
| `hci-storyboard-runs` five commits resolved | ⛔ Not met |
| 08-13 backlog document committed durably | ⛔ Not met |
| `biosciences-mcp-edge` divergence resolved | 🟡 Diagnosed as clean staleness, not divergence; redundant copies remain two commits behind |
| Windows `hci-canon-*` worktrees audited | 🟡 WSL inventory complete; native Windows validation remains |
