# Making Research Runs Compound — Versioned Evidence Artifacts

**Date:** 2026-09-13
**Status:** Draft for review — nothing shipped into any plugin yet
**Scope:** `open-biosciences-plugins/psychology-research` (public)
**Companion:** [`docs/diagrams/ob-research-graph-build.md`](../diagrams/ob-research-graph-build.md)

> Authored by a Claude Code session on the maintainer's behalf. This is a
> proposal from someone with less context than the reader, not a ruling.

---

## Why

The bio pipeline took three tries to become incremental, and the public run
outputs in `biosciences-workspace-template/output/` record all three stages:

| Stage | Runs | Mechanism | Result |
|---|---|---|---|
| 1 | `qw` → `qw3` → `qw4` | none | same competency question, six different JSON schemas, node counts 7 → 12 → 5. Every run re-derived from scratch. |
| 2 | `cq14` → `cq14edgefix` | `WORKSPACE_MEMORY.md` + a chronology, in markdown | learnings survive, but a *person* has to apply them |
| 3 | `sl-system-approach` | `extends` + review-as-provenance, **inside the JSON** | v1 (31 nodes / 32 edges) → v2 (41 / 51), and the fix is machine-traceable |

Stage 2 costs almost nothing and catches most of the repeated pain. Stage 3 is
the structural fix. Do them in that order.

## What `psychology-research` already has

The plugin is not missing gate machinery — it has more of it than the bio
pipeline does:

- A **JSON evidence-packet schema** already specified in
  `references/fuzzy-to-evidence.md` (`question`, `mode`, `entities`, `claims`,
  `sources`, `evidence_levels`, `limitations`, `retrieved_at`).
- **Nine validators** in `scripts/validators/`: `bibliography_integrity`,
  `citation`, `crisis_trigger`, `evidence_label_coverage`, `forbidden_language`,
  `graph_memory_fragment`, `no_citations_found`, `source_tier_minimum`,
  `template_conformance`.
- A **load-bearing contract** in `references/graph-memory-contract.md` — a
  graph-sourced fact is `local_context`, never `VERIFIED`, and a claim joining a
  graph fragment to literature is labeled at the *weaker* of the two.
- **Five classification labels** (`VERIFIED` / `SELF_REPORTED` / `SUPPORTED` /
  `UNRESOLVED` / `CONFLICTING`) — a richer analogue of bio's three-way verdict.

The schema is also stable in practice: packets produced by downstream consumers
conform to it exactly, which is already better than the `qw` chain ever managed.

So this is a narrow job. Three things are missing, and one of them is a
prerequisite for the other two.

### Gap 1 — claims have no stable identity

A claim is `{claim, status, source_ids, source_tier, limitations}`. There is no
`id`. Nothing outside the packet can refer to a specific claim, which means no
later run, review, or diff can say *which* claim it is talking about. Every
other proposal here depends on fixing this first.

### Gap 2 — persistence is a consumer convention, not a contract

`commands/psy-research.md` Step 4 is "Present Results"; `commands/psy-review.md`
accepts "a report path and evidence packet path, **or** `use conversation
context`". Consumers that do write the packet to disk get no help from the
schema in relating one file to the next — there is no version, no lineage field.

### Gap 3 — nothing carries forward

No `extends`. No provenance path by which a `psy-review` finding re-enters the
next run. The review is read by a human and then it is gone.

Note what is *not* on this list: the validators. They exist and they work — they
just guard the report rather than the packet.

---

## Proposal 1 — claim IDs (prerequisite)

```json
{ "id": "C7", "claim": "…", "status": "SUPPORTED", "source_ids": ["S3"] }
```

Stable within a packet lineage: once `C7` is assigned, a later packet reusing
that id means *the same claim*. `bibliography_integrity` already does the
equivalent bookkeeping for sources, so the pattern is established.

**Cost:** one field, plus an id-uniqueness check in the existing validator suite.

## Proposal 2 — a `WORKSPACE_MEMORY.md` convention

The cheapest win, taken directly from `output/cq14/WORKSPACE_MEMORY.md`. One
file per workspace, appended to by any run that hit friction. Four section
types, all of which the cq14 file demonstrates:

```markdown
## RULE: <short imperative>
Observation, the rule, and a worked example.

## OPEN ISSUE: <what is broken>
Problem, likely root cause, workarounds considered, current workaround, impact.
Carried forward until closed.

## GAP ANALYSIS: <run id>
Per-gap root cause — was this a *protocol* gap (the run never made the call) or
a *presentation* gap (it did, but the output failed to show it)?

## RUN LOG: <run id>
Question, date, packet path, counts, what changed from the prior run.
```

The protocol-vs-presentation distinction is the most transferable idea in
`cq14edgefix/tp53-sl-pipeline-chronology.md`: of ten review dimensions scored
PARTIAL, several were reports failing to *cite* work that had actually been
done. Re-running the pipeline to fix those would have been wasted effort.

**Cost:** a documentation convention and a paragraph in `psy-research.md`. No code.

## Proposal 3 — packet lineage

Four fields added to the existing schema. Nothing is renamed, so every current
validator keeps working:

```json
{
  "packet_version": 2,
  "extends": "<prior packet filename>",
  "supersedes_claims": ["C7", "C12"],
  "pipeline_version": "fuzzy-to-evidence-v1"
}
```

and on each claim:

```json
"provenance": ["licensing_board_lookup(<jurisdiction>, 2026-09-13)",
               "quality_review_v1_recommendation"]
```

- **`extends`** — names the predecessor. Same semantics as the
  `sl-system-approach` knowledge graph.
- **`supersedes_claims`** — psychology needs something bio did not. A bio graph
  accretes: new nodes rarely invalidate old ones. Psychology claims *do* get
  overturned — a `SELF_REPORTED` credential becomes `VERIFIED`, a guideline is
  revised, a paper is retracted. Accretion alone would silently keep a stale
  claim alive, so supersession has to be explicit.
- **`provenance`** — a list, not a string, so a review recommendation can sit
  beside the retrieval call. This is the mechanism that made the bio v1→v2 fix
  traceable: `"opentargets_get_associations(…), quality_review_v1_recommendation"`.

**Cost:** four schema fields, a file-write step in `psy-research.md` Step 4, and
making the packet path required rather than optional in `psy-review.md`.

## Proposal 4 — a pre-emit gate on the packet

A tenth validator, `packet_preemit.py`, run before the packet is written. The
bio rule is "every Gene node carries symbol · name · ensembl · uniprot ·
location". The psychology equivalent, composed from rules already written down
elsewhere in the plugin:

| Check | Rule | Where the rule already lives |
|---|---|---|
| Claim identified | every claim has a unique `id` | new (Proposal 1) |
| Label present | every claim carries one of the five `status` labels | `fuzzy-to-evidence.md` |
| Source resolvable | every `source_ids` entry resolves to a `sources[]` entry | mirrors `bibliography_integrity` |
| Tier declared | every source has `source_tier` and `retrieved_at` | `fuzzy-to-evidence.md` |
| Tier floor | `VERIFIED` requires a `primary` source | Source Hierarchy table |
| Graph-memory rule | `local_context` claims are never `VERIFIED` | `graph-memory-contract.md` — already enforced for *fragments*; extend to *claims* |
| Weaker-of-two | a claim citing both a graph fragment and literature takes the weaker label | `graph-memory-contract.md` |
| Conflict not collapsed | `CONFLICTING` requires ≥2 `source_ids` | new |

Failing claims do not silently vanish: they downgrade to `UNRESOLVED` and land
in `limitations[]`, which is what the packet already uses for this.

**Cost:** one validator module in an established pattern, beside nine siblings.

---

## The loop, once these land

```
   question ──> psy-research ──> packet v1 ──> psy-report ──> psy-review
                                    ^                             │
                                    │                             │ findings
                                    │                             v
                                    └──── packet v2 ──────────────┘
                                          "extends": v1
                                          supersedes_claims: [C7]
                                          C7.provenance += review_v1
```

With `WORKSPACE_MEMORY.md` accumulating alongside, out of band, catching the
tool-level failures no schema can express.

## On Temporal

If any of this migrates to a durable-execution runtime, the mapping is direct:
each stage (LOCATE / RETRIEVE / EXTRACT / CLASSIFY / SYNTHESIZE) is an activity;
the pre-emit gate is a workflow validation step before the persist activity; and
`extends` is the prior run's output passed as workflow input. Durable execution
supplies the per-stage retry the skill currently describes in prose, and
`supersedes_claims` maps onto a workflow reading its own prior result — the
normal continue-as-new pattern.

## Open questions

1. Should `supersedes_claims` live on the packet or on each claim? Packet-level
   is simpler to validate; claim-level is easier to read.
2. Does `psy-publish` need the full packet lineage, or only the latest?
3. Is `WORKSPACE_MEMORY.md` per-workspace or per-plugin? cq14 used
   per-workspace, which meant learnings did not travel between runs in different
   directories — arguably a bug in that design.

## Boundary note

This document is deliberately scoped to the public plugin and to public run
outputs. Consumer-specific design — how a given downstream repo lays out run
folders, binds graph-memory fragments, or handles sensitive material — belongs
in that consumer's own repo, per the public/private practices adopted 2026-09.
