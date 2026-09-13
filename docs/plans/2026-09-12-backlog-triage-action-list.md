# Program Director — action list, 2026-09-12 evidence pass

Three triage tracks (platform-defects, governance, unfiled) over 16 open issues across three projects and five 2026-09-12 synthesis documents. Every "unverified" / "[inference]" / "read-not-executed" flag from the triage is preserved below. Where tracks disagreed, the adjudication is marked **Adjudicated:** inline.

Effort is given twice where it differs: **PD** = your time (backlog operations), **Eng** = the work the operation commits someone to.

---

## 1. Do now — ordered

### 1. Commit the five synthesis docs to `biosciences-program` main
**Issue:** file nothing — this is a git operation.
**PD effort:** ~5 min.

All five 2026-09-12 documents are **untracked** in `biosciences-program` (`git status`: five `??` entries under `docs/plans/`). Section 5 of this document maps research to issues; none of that mapping is discoverable if the research exists only in one working tree. This ranks first because every other item below cites these paths, and a citation to an uncommitted file is a dangling pointer.

```
docs/plans/2026-09-12-ddd-cqrs-pattern-governance-research.md
docs/plans/2026-09-12-interface-version-tradeoff-analysis.md
docs/plans/2026-09-12-prior-art-v1.6.0-proposed-patch.md
docs/plans/2026-09-12-regulated-event-driven-agent-architectures.md
docs/plans/2026-09-12-three-planes-claude-code-agents.md
```

Note when committing: the prior-art patch also exists as `biosciences-mcp/docs/prior-art-revisions/2026-09-12-v1.5.0-to-v1.6.0-patch.md` on branch `docs/prior-art-v1.6.0` (PR #15). Two copies of the same artifact in two repos is the drift pattern the governance research names; pick one canonical location in the commit message.

---

### 2. Correct `biosciences-program/CLAUDE.md` lines 19 and 22
**Issue:** add as a task on **AGE-702**; line 19 also noted on AGE-696.
**PD effort:** ~10 min (this repo, two lines). **Eng:** none.

Ranks second because it is the only item entirely inside the repo you own, costs minutes, and both lines are currently false in a document every agent in the workspace reads at session start. The governance research records that line 22 stood wrong for months with nothing catching it.

- **Line 22** asserts `biosciences-mcp` is still on the January 2026 dotted-command Spec Kit layout and is "the upgrade target." Verified false: `biosciences-mcp` main has `.claude/skills/speckit-*` (12 skills incl. `speckit-converge`), an empty `.claude/commands/`, and `.specify/integration.json` at `"version": "1.0.4"`, `"invoke_separator": "-"`. PR #9 merged at `f6ced95`. `psychology-mcp` is the repo still on 0.16.4.
- **Line 19** says platform ADRs are "(001–006, 007 proposed)". ADR-007 was accepted (`8f0c973`, `be2b785`, PR #6).

**Comment to attach to AGE-702:**

````
## Disposition 2026-09-12 — one task to add, otherwise unaffected

The interface-version and identifier-authority work reviewed today (FastMCP 2→3/4, MCP protocol
revision 2026-07-28, Bioregistry) has **no bearing on this issue**. No dependency, no sequencing
constraint, no change to the remaining constitution PATCH.

One finding does land, from
`biosciences-program/docs/plans/2026-09-12-ddd-cqrs-pattern-governance-research.md` §2:

**`biosciences-program/CLAUDE.md` line 22 states the inverse of this issue's completed outcome.**
It currently reads:

> `biosciences-mcp` is still on the January 2026 dotted-command Spec Kit layout; the current
> upstream skills layout (`.claude/skills/speckit-*`, hyphen invocation, `converge`) is already
> in use elsewhere in the org and is the upgrade target.

Verified against `biosciences-mcp` **main** today:

- `.claude/skills/` — 12 entries including `speckit-converge`
- `.claude/commands/` — empty (the nine legacy dotted commands were deleted, as task 2 records)
- `.specify/integration.json` — `"version": "1.0.4"`, `"invoke_separator": "-"`
- PR #9 merged at `f6ced95`

The upgrade is done and main-visible; the governing document says it has not started.
`psychology-mcp` is the repo on the older 0.16.4.

**Add to Tasks:**

- [ ] Correct `biosciences-program/CLAUDE.md` §"SpecKit and ADRs" — biosciences-mcp is on Spec Kit
      1.0.4 (skills layout, hyphen invocation, `speckit-converge`) as of PR #9 `f6ced95`; the
      "upgrade target" framing is stale. Same sweep: line 19 still says ADR-007 is *proposed*,
      which the merged ADR-007-accepted change (`8f0c973`, `be2b785`) contradicts.

One note for the remaining constitution PATCH task: `biosciences-mcp/.specify/memory/constitution.md`
also needs the ADR-001 version citation (v1.2 → current, lines 58 and 169) and the "22-key registry"
line 86 (registry.py asserts 23). Those belong to AGE-700, but they are the same file — sequence the
two PATCHes or land them together rather than touching the constitution twice.
````

Once both land, consider moving AGE-702 toward Done rather than leaving it In Progress on a one-line edit.

---

### 3. Comment on AGE-703 — merged finding: wider blast radius **and** it now blocks CI
**Issue:** AGE-703 (existing, High, Backlog). No re-file, no priority change.
**PD effort:** ~10 min. **Eng:** +~1h on top of the filed fix (one extra edit, one extra acceptance check). Still "small."

**Adjudicated:** platform-defects said *re-scope*; unfiled said *no-change, duplicate cleared*. Both are right about different halves and neither contradicts the other — platform-defects widens the scope (model imports are in the failure domain; the filed acceptance criteria would pass with the second lever intact), unfiled adds a downstream consequence (the proposed CI's `fastmcp inspect` step imports the package). One merged comment, one issue. The availability framing and HIGH priority hold in both tracks: **do not re-file this as a latency/performance item.** It ranks third because it is now a prerequisite for item 4.

**Comment to attach:**

````
**Import-cost measurement (2026-09-12) — confirms the availability framing, widens the blast
radius, and makes this a CI prerequisite**

Source: the 2026-09-12 version/protocol tradeoff pass
(`biosciences-program/docs/plans/2026-09-12-interface-version-tradeoff-analysis.md` session
measurements), re-verified against source.

Measured: importing `biosciences_mcp.models.cross_references` costs **1,234,245 us cumulative, of
which `biosciences_mcp.clients` accounts for 1,234,124 us** — 99.99% of the import cost is the one
blocking EBI call.

**1. Scope is wider than "the gateway, every server module, and every test process."** A *pure
model* import pays the fetch too, because Python executes `src/biosciences_mcp/__init__.py` before
any submodule. Confirmed: `src/biosciences_mcp/__init__.py:42` re-exports ten clients, and
`src/biosciences_mcp/clients/chembl.py:23` does
`from chembl_webresource_client.new_client import new_client` at module level. Any work that imports
only models — the ADR-001 v1.5 registry amendment (AGE-700), a consolidated `identifiers.py`,
`.claude/hooks/validate-curie.py` — is inside the failure domain today.

**2. The proposed fix does not cover the second lever, and the acceptance criteria would not catch
it.** Lazy-importing the SDK inside `ChEMBLClient` satisfies "`import biosciences_mcp` makes no
network call" while leaving the eager re-export at `__init__.py:42` in place, so every model import
still pulls in thirteen client modules. Suggest adding to Fix and Acceptance:

- narrow or lazily resolve the `__init__.py:42` client re-export (module-level `__getattr__`, PEP 562);
- acceptance: `uv run python -X importtime -c "import biosciences_mcp.models.cross_references"`
  shows no `biosciences_mcp.clients` entry.

**3. New downstream consequence: this now blocks CI.** A blocking CI workflow is being filed for
this repo (`.github/workflows/` currently holds only `claude.yml` and `claude-code-review.yml` — no
test has ever gated a PR). That workflow's server-load gate runs `uv run fastmcp inspect
fastmcp.json`, which imports the package. **While EBI is down, that step fails on every PR in every
repo state**, including PRs that touch nothing near ChEMBL. Two ways to sequence: land this first,
or ship the step `continue-on-error: true` and remove the escape hatch in this issue's PR. Either is
fine; discovering it during an outage is not.

Confirmed unchanged on main — `from chembl_webresource_client.new_client import new_client` is still
at `clients/chembl.py:23`, and it is the only `chembl_webresource` reference in `src/` outside
docstrings.

**Framing and priority hold.** The availability argument (the gateway will not start during an EBI
outage) is the right one; the 1.23 s is a symptom of the same blocking call. Do not re-file this as
a latency/performance item.
````

---

### 4. File the CI issue
**Issue:** file new — see §3, issue **N2**. Priority High.
**PD effort:** ~10 min to file. **Eng:** ~half a day, blocked-by AGE-703 for the `fastmcp inspect` step only.

Ranks above every remaining engineering item because it is the multiplier: `.github/workflows/` holds only `claude.yml` and `claude-code-review.yml`, and **933 tests have never gated a PR**. Every disposition in this pass that ends "make it a test" — the 34-name pin, the two-layer invariant, the drift oracle, AGE-688's re-bounded timing test — is inert without it. Duplicate-cleared: AGE-579 is psychology-mcp/plugin scoped and already implemented in that repo's `ci.yml`; AGE-584 is Canceled.

---

### 5. File the IUPHAR `type_filter` no-op
**Issue:** file new — see §3, issue **N1**. Priority High.
**PD effort:** ~10 min. **Eng:** ~1–2h including tests; land in one PR with AGE-704 against `clients/iuphar.py`.

Ranks above the remaining new issues because it is the only **live, verified-against-the-live-API, silent wrong-answer** defect surfaced in this pass, and it is free to land: same file as AGE-704, which is already queued. Verified live 2026-09-12, not read from a doc — four requests to `https://www.guidetopharmacology.org/services/targets?name=adenosine` with `type=GPCR`, `type=gpcr`, `type=Enzyme` and no `type` all returned the same 13 records spanning `{enzyme, gpcr, transporter}`. Not covered by AGE-688 (that issue's IUPHAR item is response casing), AGE-704, or AGE-698.

---

### 6. File the 34-tool-name pin (re-scoped: tighten and re-mark an existing test)
**Issue:** file new — see §3, issue **N3**. Priority High.
**PD effort:** ~10 min. **Eng:** ~1h.

**Adjudicated:** platform-defects proposed one issue combining the name pin with the FastMCP 2→3 upgrade; unfiled split them and re-scoped the pin after reading the code. Unfiled wins on both counts — the test **already exists** (`tests/integration/test_gateway.py::test_gateway_tools_exposed`) but asserts a 20-name subset plus `len(tool_names) >= 30` and is **unmarked** (1 of only 4 unmarked tests in 933; `-m unit`, `-m integration`, `-m contract` and `-m e2e` all deselect it). The missing work was the marker, not the test. Splitting is right because the pin must land *alone and first* as the guard for the upgrade, and item 4's CI step 6 needs the expected set to exist.

Ranks here, above the identifiers work, because it costs ~1h, it is a prerequisite for two other filed items, and it fixes a written contract that is already wrong: `servers/gateway.py` line 15 advertises `pubchem_get_compound_synonyms`, which is not among the 34 tools enumerated at runtime.

---

### 7. Attach the AGE-688 correction; record "do not split"
**Issue:** AGE-688 (existing). Scope unchanged, priority unchanged, no split.
**PD effort:** ~5 min. **Eng:** unchanged.

The whole point of the pass, recorded on the issue that proves it: **AGE-688 already carried today's IUPHAR and WikiPathways root causes in more detail than the re-derivation produced**, including live probe results the re-run did not have. Only one item needed correction, and it moved toward *less* severe.

**Comment to attach:**

````
**Root causes re-verified 2026-09-12; one correction to the Problem statement**

Two of the three reproduced deterministically and match this issue exactly — treat this issue as the
record, not the re-derivation:

- **IUPHAR.** Upstream returns `"gpcr"` where the tests assert `"GPCR"`, passed through verbatim at
  `src/biosciences_mcp/clients/iuphar.py:701` (`target_family=target_data.get("type", "")`).
  Confirms Root cause (B) and the `TARGET_TYPE_CANONICAL` map as written. (The live probe already
  recorded here — `other_protein` / `catalytic_receptor` returning 404 as `?type=` filters — is the
  detail a naive `.title()`/`.upper()` fix would miss; it is not in any newer note.)
- **WikiPathways.** BRCA1 yields 20 and TP53 85 against `total_count > 100`. Confirms Root cause (A).
  Worth stating plainly in the issue text: these assertions are pinned to live upstream data *volume*
  and were always going to rot, which is why fix step 1's `>= 10` / `>= 40` floors plus a
  named-pathway check is the right shape rather than a re-tuned threshold.

**Correction.** The Problem statement says the six tests fail "with stable values".
`test_concurrent_search_performance` is not stable: the 2026-09-12 run measured **5.98 s** — outside
the recorded 5.0–5.3 s range — and **did not reproduce** on repeat. It is a genuine flake sitting on
a knife-edge, not a fixed regression. The fix is unaffected: `total_elapsed < len(queries) * 2.0`
(20 s) clears the observed spread with margin, which is the argument for re-bounding rather than
re-tuning. Suggest amending the Problem line to "five tests fail with stable values; the sixth
(`test_concurrent_search_performance`) is an intermittent knife-edge failure, 5.0–6.0 s observed
against a hard 5.0 s bound."

**On splitting (asked 2026-09-12): no.** The three causes are unrelated, but "three separable commits
on one branch" is already the right shape — one acceptance run, one PR, and the IUPHAR commit is the
only one touching `src/`, so it is what carries the AGE-694 / AGE-698 ordering this issue already
records. Three issues would triplicate that constraint and buy nothing.
````

---

### 8. Attach the AGE-701 sequencing comment; force the DepMap cross-reference-class decision onto its checklist
**Issue:** AGE-701 (existing). Migration scope and HIGH priority unchanged; the recorded dependency graph changes.
**PD effort:** ~10 min. **Eng:** migration unchanged; adds a gate (see decision **D4**).

Ranks here because it is the item that makes the *recorded* dependency graph match the actual one before anyone starts the pilot. Recorded today: AGE-701 blocks AGE-700. Unrecorded: identifiers consolidation also gates AGE-700, and the pilot adds a site to it.

**Comment to attach:**

````
**Unrecorded sequencing: an identifier-consolidation step sits between this pilot and AGE-700**

Source: `biosciences-program/docs/plans/2026-09-12-interface-version-tradeoff-analysis.md`
§1 item 3 and §5.

This issue records that it blocks AGE-700. Two upstream constraints are recorded nowhere.

**1. `src/biosciences_mcp/identifiers.py` is the prerequisite that makes AGE-700 answerable, and it
had no issue until today.** The 2026-09-12 analysis measured **~20 pattern declaration sites in
`src/`**; a direct grep count the same day put the total regex surface higher (~70 occurrences: 40
`re.compile` constants, 18 `pattern=` declarations across 7 model modules, 12 inline
`re.match/search/fullmatch` with literals) — including a bare `re.match(r'^[NX][MR]_\d+$')` inline
at `clients/entrez.py:435`. `models/target.py::DISEASE_ID_PATTERN`
(`^(EFO|MONDO|Orphanet|HP|DOID|OTAR)_\d+$`, duplicated byte-identically in `clients/opentargets.py`)
is a *third* spelling of the disease identifiers, distinct from both the registry's colon form and
Bioregistry's bare form. Amending Appendix A across that many hand-maintained sites is the guesswork
the consolidation removes. See the new consolidation issue.

**2. This pilot adds a site to that count.** The checklist here keeps `DepMapCrossReferences`
(`ccle`, `cosmic`) as "a sixth feature-local cross-reference class". Decide it explicitly and record
it here rather than discovering it during AGE-700: either the consolidation's Phase 1 lands first and
DepMap imports `XREF_PATTERNS` / `CURIE_PATTERNS` from it, or DepMap ships its class and the
consolidation absorbs one more site.

**3. Related ordering worth noting on the pilot.** If `cross_references` later moves to bare local
IDs, ~7 keys change on the wire across deepagents, temporal and the bio-research plugin. The analysis
(§5, failure mode (f)) identifies FastMCP 3's component versioning as the mechanism for shipping that
as `get_pathway` v2 rather than a flag-day break — making the FastMCP 2 → 3 upgrade an argument for
landing *before* AGE-700 resolves, not after.

**Scope and priority of the migration itself are unchanged.**
````

---

### 9. File the identifiers.py consolidation (two phases, re-estimated)
**Issue:** file new — see §3, issue **N4**. Priority Medium.
**PD effort:** ~15 min. **Eng:** 5–8 days total; **Phase 1 alone is ~1.5–2 days** and is what unblocks AGE-700.

**Adjudicated three ways.**
- *Shape:* platform-defects proposed one issue including the Bioregistry snapshot; governance proposed two issues; unfiled proposed one issue in two phases with the oracle separate. Two issues (N4 + N5), N4 phased — governance and unfiled agree the oracle is separable, and unfiled's phasing is the only proposal that survives review (a single ~70-site PR is unreviewable and collides with AGE-698 and AGE-704 in `clients/`).
- *Effort:* 2–3 days [inference, from the tradeoff analysis §1 item 3] vs 5–8 days [measured inventory]. **5–8 days**, split. The measured count wins over the inferred one. It is still an estimate — nobody has done it — and the source doc warns its estimates skew low.
- *Priority:* platform-defects said High; unfiled said Medium. **Medium.** Under the refseq adjudication (decision **D1**) nothing is wrong on the wire today, and filing a refactor above the CI gate that protects it would invert the ranking that makes the refactor safe.

Also mark **AGE-700 blocked on N4 Phase 1**.

---

### 10. Attach the AGE-700 re-scope comment
**Issue:** AGE-700 (existing). Re-scope with two named blockers.
**PD effort:** ~15 min. **Eng:** ADR-authoring roughly unchanged; the critical path lengthens by N4 Phase 1.

Ranks after N4 is filed so the comment can reference it. All four claims in the brief were verified against source; three checklist items change and two blockers are added.

**Comment to attach:**

````
## Disposition 2026-09-12 — re-scope, with two blockers named

Evidence: `biosciences-program/docs/plans/2026-09-12-interface-version-tradeoff-analysis.md` §5 and
`biosciences-mcp/docs/prior-art-revisions/2026-09-12-v1.5.0-to-v1.6.0-patch.md` (branch
`docs/prior-art-v1.6.0`, PR #15 — **not on main**; cite the branch, not
`docs/prior-art-api-patterns.md`). Every code claim below was read from source today.

### Blocker 1 — prior-art v1.6.0 must merge first

The §4 reconciliation item cites `prior-art-api-patterns.md` §7.5. v1.5.0 asserts industry-precedent
grounding for two things the MCP spec now mandates outright, and claims novelty for `slim=` and
recovery hints that Anthropic's tool-design guidance predates by four months. The patch record's own
urgency note: bump the revision **before this document is cited in any new ADR**. PR #15 is the gate.
ADR-001 v1.5 must cite v1.6.0 §7.3 and the new §10, not v1.5.0 §7.5.

### Blocker 2 — pattern consolidation is a prerequisite, not a parallel track

The Appendix A amendments assume the registry has one declaration site. It does not. Until the
patterns have one owner, amending Appendix A amends one of several copies. Mark this issue blocked on
Phase 1 of the identifier-consolidation issue.

### Item-by-item

**`refseq_protein` — REWORD, do not delete, do not scope as written.**

The stated premise is stale. `clients/uniprot.py:151-159` now takes `NucleotideSequenceId` (an
`NM_`/`XM_` accession), not the entry id (`NP_`); `clients/entrez.py:435` filters on an inline
`^[NX][MR]_\d+$`. **No client emits `NP_` into the `refseq` key today** — so this is not a live wire
defect.

The real defect is different. Three sites declare the refseq pattern, two spellings, all citing
Appendix A:

| Site | Pattern |
| -- | -- |
| `tests/contract/registry.py:54` | `^[NX][MR]_\d+$` |
| `models/cross_references.py:21` | `^[NX][MR]_\d+$` |
| `models/ensembl.py:28` | `^[NX][MPRG]_\d+(\.\d+)?$` |

The divergent copy is the one closer to upstream truth: RefSeq genuinely issues `NP_`, `NG_`, `NC_`,
`XP_`, and upstream accessions are versioned.

**The open question nobody has stated: is a registry key defined as the identifier space it names,
or as the subset this platform emits?** The repo has answered it both ways simultaneously. Reword the
item to put that question once, registry-wide, with refseq as the instance that forces it — not as a
refseq-only regex widening. This is escalated as a human decision; do not settle it in the ADR draft
before it is answered.

**`ensembl_transcript` unversioned — MARK ZERO-CODE, and widen to three keys.**

Already true in code. `models/cross_references.py` `normalize_xref` ends with:

```python
if key in ("ensembl_gene", "ensembl_transcript", "refseq"):
    return value.split(".", 1)[0]
```

Documentation catching up, not work. Annotate the item **[doc-only, no code change]**. Two
amendments: the ADR sentence must cover `ensembl_gene` and `refseq` too, and version-stripping should
be stated as an Appendix A rule **naming `normalize_xref` as the mechanism** — the same shape as the
omit-null / `OmitNoneModel` item already on this checklist.

**Key count 23 — CONFIRMED, and wider than "everywhere it is cited".**

`tests/contract/registry.py:78` carries `assert len(REGISTRY) == 23`. Twelve prose sites still say
"22-key":

- `models/target.py:119`, `models/drug.py:138`, `models/protein.py:66` — **Pydantic `description=`
  strings**, i.e. the wrong count ships inside the tool schema agents read
- `models/pharmacology.py:10,103,189`, `models/ensembl.py:38`, `models/entrez.py:178`,
  `clients/uniprot.py:114`, `clients/chembl.py:68`
- `.specify/memory/constitution.md:86`
- `docs/speckit-standard-prompt-v2.md:71`

Stronger recommendation than correcting all twelve: **delete the count from prose rather than fix
it.** `registry.py` owns the number and cannot drift silently; every prose restatement is a second
source that can. If a count is wanted somewhere, generate it. This is the exact failure mode
`docs/plans/2026-09-12-ddd-cqrs-pattern-governance-research.md` §4 names as the thing to eliminate.

Separately: the constitution also cites **ADR-001 v1.2** (lines 58, 169) against accepted v1.4. The
acceptance criteria already carry the v1.2→v1.5 constitution update; note that the same PATCH must
carry the key-count line, and that AGE-702's remaining constitution PATCH task touches the same file —
sequence them.

### New, not on the checklist

**Bioregistry as a CI oracle, explicitly not a runtime authority.** §5 measured all 23 keys against
`bioregistry 0.14.6`: 8 drop-in, 7 silently widen, 7 break, 1 (`ucsc`) has no pattern. The
disqualifying result: **Bioregistry's `ensembl` pattern accepts the literal string `BRCA1`** (and
`hello`), so adopting it as the runtime validator would disarm the ADR-001 §3 rejection the
Fuzzy-to-Fact guarantee rests on. It is also unusable at the tool-input layer — `is_valid_curie()`
returns False for all 10 project CURIEs, and `normalize_prefix()` returns `None` for `IUPHAR` and
`WP`. **ADR-001 v1.5 should record this as a decision with its reason**, not leave it unstated, or it
will be re-proposed. The oracle design is a separate issue.

**`DISEASE_ID_PATTERN` is a third spelling of disease identifiers**, duplicated byte-identically at
`models/target.py:23` and `clients/opentargets.py:49`: `^(EFO|MONDO|Orphanet|HP|DOID|OTAR)_\d+$` —
underscore-separated, distinct from both the registry's colon form (`^EFO:\d{7}$`, `^MONDO:\d{7}$`,
`^ORPHA:\d+$`) and Bioregistry's bare form. The Appendix A disease-key cardinality item cannot be
settled without deciding whether this third spelling is a fourth registry layer or a bug.

**`CROSS_REF_PATTERNS` (`models/cross_references.py:16`) is defined and imported nowhere** — a grep
for the symbol across `src/` and `tests/` returns exactly its definition line. AGE-687 listed wiring
it as a validator as out of scope and filed no successor. It is also the direct cause of the
recurring CRITICAL theme in AGE-702's baseline converge, which hit all 13 specs: *"cross-reference
regex validation declared but never applied (`INVALID_CROSS_REFERENCE` never raised)."* Now owned by
the consolidation issue.

### Two small additions §4 should carry

Both from the tradeoff analysis §3, both trivial and both preventing a future re-derivation:

- **A negative capability assertion**: "This platform declares no roots, sampling, logging or
  elicitation capability."
- **The error-code placement as a decision, not an accident**: `-32020`–`-32099` is now reserved to
  the spec and `-32000`–`-32019` is closed to new allocation; returning `UNRESOLVED_ENTITY` inside a
  tool result is the correct side of that line. State it.

One verification item rides along, since it decides whether the ErrorEnvelope works at all: **does
the federation set `isError: true` on the wire?** If it returns a success result containing an error
object, the spec's self-correction path never engages and every `recovery_hint` is inert. One
contract assertion settles it. Flagged as unverified in the v1.6.0 patch (Patch 4) and unresolved.

### Correction to carry into the ADR draft

DCR is deprecated by **PR #2858**, not SEP-2577. The brief that fed this work had it wrong; do not
let that reach an ADR.
````

---

### 11. Get prior-art PR #15 merged
**Issue:** no Linear issue — PR **#15** on `biosciences-mcp`, branch `docs/prior-art-v1.6.0` (worktree at `biosciences-mcp/.worktrees/prior-art-v1.6`).
**PD effort:** review + merge. **Eng:** none (the revision is written).

Ranks here because it is Blocker 1 on AGE-700 and it is a review, not authorship. As v1.5.0 stands, the document asserts industry-precedent grounding for things the MCP spec now mandates outright and claims novelty a reviewer can disprove in one search. Until it merges, **any issue or ADR citing it must cite the branch, not `docs/prior-art-api-patterns.md` on main.**

---

### 12. Attach the AGE-696 re-scope; close one of its open tasks as decided
**Issue:** AGE-696 (existing). The rule is unaffected; two of three remaining tasks change.
**PD effort:** ~10 min. **Eng:** the grep task folds into the CI work (item 4 / N2).

**Adjudicated:** the visibility rule itself is untouched by the evidence base, so this is not a re-open — it is closing one task as answered and re-homing another into work that now exists.

**Comment to attach:**

````
## Disposition 2026-09-12 — re-scope the two remaining tasks

The rule is unaffected. Its two open tasks are answered by
`biosciences-program/docs/plans/2026-09-12-ddd-cqrs-pattern-governance-research.md`.

**"Consider a CI grep in the public repos that fails on `open-biosciences/psychology-mcp` links"** —
§2 records that **`biosciences-mcp` has no CI**: `.github/workflows/` holds only the Claude review
bot; no pytest, no ruff, no pyright, no pre-commit. So this is not a task that can be done as written
— there is no workflow to add a step to. §4 designs the missing job: copy
`psychology-mcp/.github/workflows/ci.yml`, and add `biosciences-program/scripts/check_ids.py`, which
walks every `.md` in the workspace, extracts `PRIN-*` / `ADR-*` citations, and fails on any that does
not resolve in `governance/index.json`.

That scanner already walks the same corpus this task wants scanned. **Re-scope: make the
public→private link check a rule inside `check_ids.py`, not a second grep**, and mark this task
blocked on the CI job existing (see the new biosciences-mcp CI issue). Note also that psychology-mcp
went public on 2026-09-03, so the literal `open-biosciences/psychology-mcp` string is no longer the
target — the check should be generic (resolve each org repo reference against a visibility field in
`index.json`) so it works for the next private repo, which is the stated reason this issue stays open
at all.

**"Decide whether the rule belongs in AGENTS.md or the workspace CLAUDE.md as well"** — §3 answers
this **no**, with evidence. CLAUDE.md is ranked last of six precedence levels by
`pr-review-standard/SKILL.md`: *"context only; never as a merge requirement on their own."* It is
also, empirically, the workspace's highest-drift layer — 19–20 files, three currently teaching
`/speckit.*` commands that no longer exist, four dead paths, and one inverted version claim that
stood for months. A visibility rule placed there is unenforceable and will rot. Close the task as
decided: the rule stays in `docs/adr/README.md` rule 4 as the citable statement, and its teeth are
the CI check. If CLAUDE.md mentions it at all, it should be a pointer to rule 4, never a restatement.

**"Confirm the rule wording in the placement README (biosciences-program PR #4)"** — unaffected,
still open.

One adjacent stale line found while checking: `biosciences-program/CLAUDE.md` line 19 says platform
ADRs are "(001–006, 007 proposed)", but ADR-007 was accepted (`8f0c973`, `be2b785`, PR #6). The
research doc names ADR-007's status being asserted three different ways in three places as a live
example; this is one of the three. Being corrected under AGE-702.
````

---

### 13. Attach the AGE-698 clarification (one line into Fix step 2)
**Issue:** AGE-698 (existing). **Scope, MED priority and the AGE-688 → AGE-694/695 → AGE-698 ordering all stand as written.**
**PD effort:** ~5 min. **Eng:** unchanged.

This is here rather than in §4 only because the comment records a *deliberate behaviour removal* that AGE-704's HIGH title currently contradicts. Nothing in the evidence base touches rate limiting or retry.

**Comment to attach:**

````
**2026-09-12 evidence pass: nothing here changes.** Reviewed against the version/protocol tradeoff
analysis, the prior-art v1.5.0 → v1.6.0 revision and the pattern-governance research — no finding
bears on rate limiting or retry. Scope, priority and the AGE-688 → AGE-694/695 → this ordering stand
as written.

One clarification worth recording, on the AGE-704 relationship. AGE-704 is a strict subset of this
issue — step 1 (backoff sleep outside the interval lock) plus step 2 (delete `iuphar`'s
`_rate_limited_get`) removes the deadlock, and the test list here already carries AGE-704's
acceptance case ("429-then-200 without hanging"). But the deadlock has **two arms**:
`src/biosciences_mcp/clients/iuphar.py:69-72` (429/503) and `:74-81` (`httpx.TimeoutException` /
`ConnectError`), both recursing into `_rate_limited_get` under the same non-reentrant lock. Out of
scope names "IUPHAR network-error retries", and the base client retries statuses (`RETRY_STATUSES`),
not network exceptions — so deleting the client copy closes the timeout arm by **removing the retry
behaviour**, not by fixing it.

That is probably the right call, but AGE-704 is filed HIGH and its title covers the timeout case, so
it should be a recorded decision rather than a side effect. Suggest one line in Fix step 2: "IUPHAR
loses network-error retry; timeouts surface as `UPSTREAM_ERROR` on the first failure (closes the
timeout arm of AGE-704 by removal, deliberately)."
````

---

### 14. Collapse the Synapse strand: close AGE-240 and AGE-238, re-scope AGE-242 and AGE-239, link AGE-680 ↔ AGE-239
**Issues:** AGE-240 (close), AGE-238 (close + reparent), AGE-242 (re-scope + drop due date), AGE-239 (re-scope + reparent + repriority), AGE-680 (add relates-to link). **AGE-241's close is gated on decision D2 — do not close it in this batch.**
**PD effort:** ~30 min for the batch. **Eng:** AGE-242 drops to ~1h of release hygiene, all inside `sprime-lung-repro`, none in biosciences-program. AGE-239 becomes ~half a day (disposition table) + ~half a day (deterministic slice).

Ranks last of the substantive items because nothing else depends on it and nothing in the evidence base bears on it — but it is the largest single reduction in open-issue count in this pass, and it removes a five-month Urgent that devalues Urgent for every other reader of the queue.

**AGE-240 — comment then close:**

````
Audit complete. Answers below; closing.

**Scope correction:** the issue named four examples; there are now **seven**
`*-synapse-grounding.md` artifacts under `biosciences-workspace-template/output/` —
`fabry-disease-gb3-accumulation/` (2026-02-28), `cc-tyrosine-kinase/` (2026-03-02),
`cq14/.ob-cq/cq14/` (2026-03-02), `cq14edgefix/` (2026-03-03), `qw4/` (2026-03-06),
`methods-pharm/` (2026-03-12), `sl-system-approach/` (2026-03-15) — plus three more in
`biosciences-program/docs/samples/claude-code/doxorubicin-nf1-toxicity{,-v4,-v5}/` and a duplicate of
the cq14edgefix one in `graphiti/sample_data/`.

**Q1 — Real Synapse entity IDs, or grounding checks?** Checks, unambiguously, in all seven. Every
`syn` ID in every artifact belongs to a **third-party public dataset** the pipeline *searched for* —
e.g. `syn274449` (TCGA Lung Adenocarcinoma), `syn232866` (GEO GSE31852), `syn2384331` (Broad-DREAM
Gene Essentiality), `syn5889324` (CCLE), `syn21641955` (NF1 SL Investigation), `syn52229845` (the
Fabry project, which turns out to hold two images and a link). **Not one artifact contains a syn ID
for open-biosciences-derived data.** Nothing has ever been deposited. `methods-pharm/` records the
opposite state outright: "The Synapse.org data repository MCP tool was not available during this
publication pipeline execution … all `synapse_grounding` arrays in the KG JSON remain empty (`[]`)."

**Q2 — Annotations / vocabularies present?** None that a deposit would use. The artifacts carry a
*review* vocabulary, not a metadata vocabulary: a grounding-strength ordinal (Strong / Moderate /
Analogous / Weak), a coverage table (nodes grounded / edges grounded, with MEMBER_OF edges excluded
from the denominator), the CURIEs of the KG nodes each dataset grounds, and a
`[Source: search_synapse("…")]` citation. There is no genotype, tissue, compound-library,
source-DB-version, analysis-method or pipeline-version annotation anywhere — i.e. none of the
annotation axes AGE-241 proposed to standardise appear in these files at all.

**Q3 — Provenance structure?** Only the query trail: which search strings were issued and which tool
call produced each row. No `used`/`executed` relationships, no run identifier, no pipeline version.

**Q4 — Consistent convention, or drift?** Drift in presentation, not in substance. Coverage-metric
tables differ (`Total nodes / Grounded nodes / %` vs `Total KG nodes / Node coverage / Groundable
edges`); headers differ; the tools-used line is sometimes present, sometimes absent; `methods-pharm`
has no grounded-dataset section at all because the MCP was down. Node coverage ranges 18%–46% (qw4
18%, cq14edgefix 21%, fabry 27%, doxorubicin-v5 46%). Substantively, every artifact does the same one
thing, stable across seven runs and three weeks.

**Q5 — Gaps the ADR must address?** This is the **wrong artifact class** for the ADR's question.
`*-synapse-grounding.md` is a read-only corroboration report produced at Stage 1b of `/ob-publish`; a
deposit manifest is a write-side artifact for data the pipeline *produced*. They share a filename
stem and nothing else. The ADR's open question — "is this a grounding check or an entity manifest?" —
resolves to **check**, and a deposit convention would need an entirely new artifact rather than an
extension of this one. The Synapse MCP surface confirms it: `search_synapse`, `get_entity`,
`get_entity_annotations`, `get_entity_children` — four read tools, no writes
(`cq14edgefix/synapse-integration-analysis.md` §1).

**Acceptance:** ✅ each example has an inspection summary (seven, not four) · ✅ baseline documented —
checks, not entities · ✅ gaps listed · ✅ referenced by name from AGE-241.
````

**AGE-238 — comment then close:**

````
Closing and handing the umbrella role to AGE-680.

**Why:** AGE-680 (2026-09-02) — "Open-Biosciences plugin roadmap: skills & connectors (from S′
lung-paper learnings)" — is already a second umbrella for this same paper, built from a September
session that rebuilt Supplement 7. It has a live subtree (AGE-681 In Progress; related AGE-701) and
it does not link back here. Two umbrellas for one dogfooding run is how AGE-680 came to re-derive
manuscript-QC lessons (its items 5 and 6) without consulting the eight candidates AGE-239 had already
catalogued in April.

**Acceptance criteria, honestly assessed:**
- *Paper submitted* — not by the 4/26–29 target; still in active work as of 2026-09-02. Tracked as
  COMPBIO-* in the mocomakers/CBWG workspace, which this Linear connection cannot read, so it cannot
  be closed from here either way.
- *At least one platform-improvement issue spawned from this run has landed* — **met**, via AGE-680 →
  AGE-681 / AGE-701, with the AGE-687 cross-reference series Done.
- *Venn-STRING run adopts the same umbrella pattern* — did not happen; AGE-680 supplied a different,
  working pattern instead.

**Re-homing before close:**
- AGE-239 → reparent under AGE-680 (its candidates 5/7/8 overlap AGE-680 items 5 and 6 directly)
- AGE-242 → re-scoped to the Zenodo DOI; park under AGE-680 or leave standalone
- AGE-240 → closed (audit delivered)
- AGE-241 → pending a decision on whether a live Synapse.org relationship exists outside this
  workspace; see that issue

**One process note worth carrying to AGE-680:** the reason a September session re-derived April's
findings is that no workflow read the April backlog first. Worth a line in AGE-680 pointing at
AGE-239 so the next dogfooding pass starts from the catalogue instead of rebuilding it.
````

**AGE-242 — comment, retitle, unblock, drop the 2026-04-29 due date:**

````
Re-scoping. The deposit happened — as a public GitHub repo with a Zenodo-DOI-on-release plan, which
is acceptance criterion (b) of this issue, not (a). Dropping the 2026-04-29 due date.

**Proposed title:** Mint the release DOI for `sprime-lung-repro` so the lung paper's Data
Availability Statement can cite the derived S′/ΔS′ tables

**Already satisfied by `github.com/donbr/sprime-lung-repro`:**
- Derived tables per genotype — `results/sprime_lung_pairs.csv` (117,025 rows),
  `results/lung_genotypes.csv` (PTEN / CDKN2A / RB1 / TP53 calls), `results/demeter_validation.csv`,
  `results/bootstrap_ci_gate.csv`, `concordance/results/concordance_report.csv`
- Provenance to PRISM + DepMap — figshare version + md5 pins per input (`DOWNLOAD_CHECKLIST.md`),
  seeded RNG, canonical-order CSV writes, CI drift gate asserting every figure quoted in docs still
  matches the committed CSVs
- Column/units/interpretation README, plus `CITATION.cff` with the Corsello 2020 (PRISM) and
  McFarland 2018 (DEMETER2) source citations

**Actually remaining (~1 hour, all inside `sprime-lung-repro`, none in biosciences-program):**
1. Resolve the `CITATION.cff` maintainer TODO — replace the entity author ("The sprime-lung-repro
   authors") with named people + ORCIDs. The file explicitly warns this propagates into the minted
   DOI, so it must precede step 2.
2. Tag a release and mint the Zenodo DOI (README L93 already commits to this).
3. Paste the resulting DOI into the paper's Data Availability Statement, citing the underlying
   PRISM/DepMap dataset DOIs separately as `CITATION.cff` instructs.

**Dropped from scope:** the Synapse deposit and the AGE-241 convention dependency. The Synapse MCP
has no write tool (`search_synapse` / `get_entity` / `get_entity_annotations` /
`get_entity_children` only) and the connector is unauthenticated, so this was never executable
through the platform. A Synapse mirror stays available later as a migration off a minted DOI, exactly
as this issue's fallback clause anticipated.

**Caveat I could not resolve:** COMPBIO-7 is in the mocomakers/CBWG workspace, which this Linear
connection cannot read. The 4/29 target is five months past and AGE-680 shows the paper still in
active work in September, so please re-anchor the deadline to the current submission target.
````

**AGE-239 — comment, reparent under AGE-680, drop the ADR framing, set Medium (see D7):**

````
Keeping this — it is the survivor of the April strand — but re-scoping it and reparenting.

**1. Not an ADR.** Under the placement rule decided 2026-09-02
(`biosciences-program/docs/adr/README.md`), ADRs are placed by scope: platform decisions in
`biosciences-mcp` binding every connector; project decisions in the repo they govern; program
decisions as `ADR-PRG-NNN` for cross-repo process, ownership, release or migration. A rubric change
to `bio-research:biosciences-reporting-quality-review` is a scoped change to one plugin's skill.
Proposed deliverable instead: **a disposition table in the bio-research plugin repo alongside the
quality-review skill**, one row per candidate with `adopt-as-rubric | adopt-as-skill | reject +
rationale`, adopted rows filed as implementation issues linking back here. That satisfies all three
acceptance boxes — including "rejected items explicitly listed so they don't resurface" — without
reserving the first `ADR-PRG` number for a checklist.

**2. Reparent under AGE-680 and reconcile the overlap.** AGE-680 (2026-09-02) re-derived
manuscript-QC lessons from the same paper without seeing this issue:
- AGE-680 item 6, *manuscript reference-audit skill* (batch PubMed verification; caught a fabricated
  Jansen 2017 citation and a mis-attributed Poulikakos citation) **subsumes candidate 5**
  (uncited-reference check) and overlaps **candidate 8** (reviewer-feedback ingestion).
- AGE-680 item 5, *reference-set curation skill + linter* (PMID/DOI + search_terms + freeze-timestamp
  discipline), is adjacent to candidates 5 and 8.
- AGE-680's CONSIDER list already names the `anthropic-skills:publication-metadata-audit` skill for
  exactly this — evaluate before building.
Candidates **1, 2, 3, 4, 6, 7** have no counterpart in AGE-680 and remain wholly this issue's:
placeholder-token detector, figure-number contiguity, number-agreement parser, backmatter
completeness gate, caption↔section topic cross-check, docx-highlight extractor.

**3. Cheapest first slice.** Candidates 1, 2, 3 and 4 are deterministic string/structure checks over
the manuscript with near-zero false-positive risk and no external calls — a single `/ob-publish` exit
gate covering all four is maybe half a day and would have caught four of the five P1s on the lung
paper (COMPBIO-4, -5, -6, -7). Candidates 5–8 need judgement or a network call and belong in the
second slice.

**4. Priority.** Nothing has moved in five months and this does not block the paper. Medium is the
honest rating unless manuscript QC is genuinely next after the MCP-defect stream.

The QC evidence is intact: `cowork-ws/lung-paper/QC_REPORT_2026-04-21.md` — 32 issues against the
10-dimension rubric. **Caveat: `cowork-ws/lung-paper/` is not present on this machine**, so the QC
report was not re-read in this pass; the citation is carried forward from the issue body. The
10-dimension rubric itself is exercised in every `*-quality-review.md` under
`biosciences-workspace-template/output/`, so there is a working baseline to extend rather than a
design from scratch.
````

**AGE-680 — add a relates-to link to AGE-239** and one line: "Start from AGE-239's eight catalogued candidates before deriving new manuscript-QC lessons."

---

### 15. AGE-225 — attach the unblocked comment; schedule the sweep
**Issue:** AGE-225 (existing). Keep open, keep Low.
**PD effort:** ~5 min. **Eng:** ~10 min, three files.

Last of the substantive items only because it is Low. It moves out of "maybe blocked on AGE-700" into "provably unblocked", which is the reason it stopped moving.

**Comment to attach:**

````
## Disposition 2026-09-12 — keep open, and it is unblocked

**Re-verified at HEAD.** Still valid, unchanged in five months:

| File | Occurrences |
| -- | -- |
| `docs/diagrams/mcp-server-tiers.md` | `WP4806` at lines 32 (Mermaid node) and 80 (table); `NCT03312634` at lines 33 and 81 |
| `docs/diagrams/research-workflow-sequence.md` | `NCT03312634` at 9 sites (lines 62, 63, 71, 81, 86, 97, 110), plus `NCT03706625` |
| `docs/diagrams/fuzzy-to-fact-journey.excalidraw` | `NCT03312634` at lines 1907, 1927 |

Code side unchanged and still contradicting them: `clients/wikipathways.py:36`
`PATHWAY_ID_PATTERN = ^WP:WP\d+$`; `clients/clinicaltrials.py:45` `NCT_ID_PATTERN = ^NCT:\d{8}$`.
Correct forms remain `WP:WP4806` and `NCT:03312634`.

**Why it should not wait on AGE-700.**
`docs/plans/2026-09-12-interface-version-tradeoff-analysis.md` §5 draws a line this workspace has been
blurring: **`CURIE_PATTERNS`** (prefixed, tool inputs, what `validate_id` enforces) and
**`XREF_PATTERNS`** (bare local IDs, `cross_references` values). All the format churn the analysis
forecasts is in the second layer — failure mode (f) contemplates moving `cross_references` to bare
LUIs, changing ~7 keys on the wire.

`WP:WP4806` and `NCT:03312634` are tool-input CURIEs. They live in the layer the analysis says is
**not** moving. Nothing in AGE-700, the pattern consolidation, or the Bioregistry oracle work will
change them. Fix them now; there is no format decision pending.

**Do not close it.** Three diagrams teach agents and readers a CURIE form the strict tools reject by
design, which is the one guarantee the tradeoff analysis §4 names as this platform's actual
differentiator against BioMCP and ToolUniverse. A published diagram contradicting the enforced
contract is worse than a missing diagram. Keep at Low; it is one sweep across three files.

While in there: check `docs/diagrams/README.md`, which the AGE-219 review also flagged as carrying
these strings.
````

---

### 16. File the tail — FastMCP 2→3, Bioregistry oracle, pin convention, BioContextAI
**Issues:** file new — §3 issues **N5** (Low, blocked), **N6** (Medium, gated), **N7** (Low), **N8** (Low).
**PD effort:** ~20 min for all four.

These are filed last and ranked last deliberately. N6 is gated on N3 landing and on a preview deployment answering an unresolved question; N5 is blocked on N4 Phase 1; N7 and N8 are ~1h filler each. Filing them now is what stops the research being lost — none of them is urgent, and none should displace items 1–15.

---

## 2. Decisions needed from a human

### D1. Is a registry key defined as the identifier space it names, or as the subset this platform emits?
The forcing instance is `refseq`. Three declaration sites, two spellings, all citing ADR-001 Appendix A: `tests/contract/registry.py:54` and `models/cross_references.py:21` say `^[NX][MR]_\d+$`; `models/ensembl.py:28` says `^[NX][MPRG]_\d+(\.\d+)?$`. RefSeq genuinely issues `NP_`/`XP_`/`NG_`/`NC_` and versioned accessions.

**Adjudicated first, because the tracks disagree on the facts.** platform-defects and unfiled both call this "a live defect — the declared contract is wrong." Governance read the clients and found no client emits `NP_` into the `refseq` key today (`clients/uniprot.py:151-159` takes `NucleotideSequenceId`, an `NM_`/`XM_` accession; `clients/entrez.py:435` filters on an inline `^[NX][MR]_\d+$`). **Governance is right: nothing is wrong on the wire.** This is a definitional question, not a bug, and it is why it is escalated rather than filed.

- **Answer "the identifier space it names"** → widen to `^[NX][MPRG]_\d+(\.\d+)?$`, and every other key gets re-examined against the same standard. Larger Appendix A amendment; the contract tier stops constraining actual output.
- **Answer "the subset this platform emits"** (the triage's recommendation, offered as a recommendation not a finding) → keep `^[NX][MR]_\d+$`, correct `models/ensembl.py:28` to match, and state the rule in Appendix A's Cardinality/Format Rules with `NP_`/`XP_`/`NG_`/`NC_` recorded as deliberately out of scope **with the reason** (no tool emits protein or genomic RefSeq accessions), so the next reader does not re-file it as a bug.

Either way, the three sites must agree — that is N4's job. **This blocks the Appendix A half of AGE-700 and shapes N4 Phase 1.**

### D2. Is there a live Synapse.org relationship or commitment outside this Linear workspace?
The AGE-241 close rests on four expired premises (downstream unblocked itself via GitHub + Zenodo; no write path exists in the MCP; the artifact question is answered "check"; the stated deliverable location is stale). It does **not** rest on the mocomakers/CBWG side, which this Linear connection cannot read (`list_teams` returns only Agentic-wisdom; `get_issue COMPBIO-7` returns not-found).

- **Answer "no live relationship"** → close AGE-241 with the superseded comment; the strand collapses to AGE-239 (infrastructure) + AGE-242 (~1h of release hygiene), and Synapse retains no priority anywhere.
- **Answer "yes, with a named counterpart"** → do not reopen this scope. File a much narrower successor: "Annotation + provenance profile for one deposited artifact," with `sprime_lung_pairs.csv` as the worked example, and only after a write-capable Synapse path exists. The four March analysis docs remain the design input either way.

Also unresolved and dependent on the same workspace: **the paper's actual submission status and the final DAS wording.** AGE-242's deadline must be re-anchored to the current submission target; 4/29 is five months past and AGE-680 shows the paper in active work in September.

### D3. STRING: keep, defer to `mcp.string-db.org`, or both?
`mcp.string-db.org` is live and vendor-operated by the Mering lab. Both the tradeoff analysis (§4) and prior-art v1.6.0 (§10) flag it and both explicitly decline to decide it. **Filed here as a decision, not as an engineering ticket**, because deferring is not cheap: 3 of the 34 gateway tools are STRING (`string_search_proteins`, `string_get_interactions`, `string_get_network_image_url`), the `STRING:9606.ENSP00000269305` CURIE form is baked into `models/interaction.py:21,24` and ADR-001 Appendix A, `string_get_network_image_url` has no obvious vendor counterpart, and deepagents / temporal / bio-research all bind tools by name.

- **A — keep and record why** (the triage's recommendation): one paragraph in prior-art §10 or the ADR saying this was assessed 2026-09-12 and kept deliberately. ~30 min. No issue needed.
- **B — defer**: remove 3 tools, amend Appendix A, update 3+ downstream repos. Multi-day, multi-repo breaking change. File an issue only if you pick this.
- **C — both**: doubles the surface for the same data; worst on agent-facing clarity. Not recommended.

**One unrun probe would move this:** does `mcp.string-db.org` enforce a strict phase or accept bare symbols (as BioMCP does)? **Not investigated.** If it accepts bare symbols, A is straightforwardly right.

Context to record whichever way: no other upstream in this federation ships an official MCP server; per EMBL (2025-12-02) MCP is "something we are exploring." There is no wave about to obsolete the other 11 sources.

### D4. Does identifiers.py Phase 1 land before the DepMap pilot (AGE-701)?
- **Phase 1 first** → AGE-701 slips ~1.5–2 days; `DepMapCrossReferences` (`ccle`, `cosmic`) imports `XREF_PATTERNS`/`CURIE_PATTERNS` and adds no site; AGE-700 becomes answerable sooner.
- **DepMap first** → the pilot ships a sixth feature-local cross-reference class, the consolidation absorbs one more site, and the pilot validates against a registry that is about to move.

Either is defensible. What is not defensible is leaving it undecided until it is discovered mid-AGE-700 — record the answer on AGE-701's checklist.

### D5. Does AGE-700's doc-only work proceed while blocked on N4?
AGE-700 splits cleanly. The `ensembl_transcript` item is **[doc-only, zero-code]**; the 23-key sweep is mechanical (12 prose sites, or a deletion); the negative-capability and error-code-placement additions are new prose. The Appendix A amendments are what N4 Phase 1 gates.

- **Ship the doc-only half now** → v1.5 lands in two passes, and the constitution PATCH can be sequenced with AGE-702's remaining task (same file — do not touch it twice).
- **Hold the whole ADR** → one clean v1.5, but it waits 1.5–2 days on N4 Phase 1 *and* on PR #15 merging.

### D6. Python 3.11 or 3.13?
`biosciences-mcp/.python-version` **exists, is untracked, created 2026-09-12, and says `3.13`**, while `pyproject.toml` declares `requires-python = ">=3.11"` and `[tool.pyright] pythonVersion = "3.11"`. `biosciences-mcp-edge` also has one; every other repo has none. An untracked interpreter pin that disagrees with the repo's own type-check target is worse than no file — type errors and runtime behaviour diverge between the machine that has it and every machine that does not.

Pick one, then make all four declarations agree: `requires-python`, `[tool.pyright] pythonVersion`, `.python-version` (committed), and the CI `uv python install` step. **This must be answered before N2's CI lands**, or CI encodes the disagreement.

### D7. AGE-239 priority — Medium or High?
Medium is the honest rating: nothing has moved in five months and it does not block the paper. High is defensible **only if manuscript QC is genuinely the next stream after the MCP defects**. This is a queue-ordering call, not an evidence call.

### D8. Does the project get renamed?
"Open-Biosciences Platform v1.1 — dogfooding + Synapse alignment" holds 18 issues: the 5 April dogfooding/Synapse issues (all Backlog, untouched) and 13 September MCP-defect issues (AGE-686–AGE-705; 6 Done, 2 Todo, 4 Backlog, actively worked). If items 14 and 16 land, the project contains only MCP-defect work. Part of why AGE-241 read as "the only Urgent in the project" is that it sits in a project that has since become something else.

---

## 3. File these

All new issues go to the project holding AGE-686–AGE-705 — **"Open-Biosciences Platform v1.1 — dogfooding + Synapse alignment"**, team Agentic-wisdom (AGE) — *[assumed from the triage's project description; verify at filing, and see decision D8 on the name]*.

---

### N1 — `[iuphar] search_targets(type_filter=...) is a silent no-op — GtoPdb ignores the type parameter`
**Priority:** High · **Labels:** platform, mcp-server, bug · **Related:** AGE-704 (same file), AGE-688 (different IUPHAR defect)

````
## Problem

`search_targets(type_filter=...)` advertises a filter that is never applied. GtoPdb ignores the
`type` query parameter entirely, and the client does no client-side filtering to compensate.

`clients/iuphar.py:507-509` forwards the value:

```python
if type_filter:
    params["type"] = type_filter
```

`clients/iuphar.py:559-596` then paginates the response unchanged — no filter, and `family`/`type` on
each candidate are copied straight from the API's own `type` field.

`servers/iuphar.py:217-223` declares the parameter to the model ("Optional target type filter (e.g.,
'GPCR', 'Enzyme', 'Ion channel')") and `servers/iuphar.py:249` gives a worked example:
`search_targets("receptor", type_filter="GPCR")`.

## Evidence (live, 2026-09-12)

Four requests to `https://www.guidetopharmacology.org/services/targets?name=adenosine`:

| `type` param | records | distinct `type` values in response |
|---|---|---|
| `GPCR` | 13 | `enzyme`, `gpcr`, `transporter` |
| `gpcr` | 13 | `enzyme`, `gpcr`, `transporter` |
| `Enzyme` | 13 | `enzyme`, `gpcr`, `transporter` |
| *(omitted)* | 13 | `enzyme`, `gpcr`, `transporter` |

Identical result sets. Casing is not the issue — the parameter is inert.

## Why this is worse than its size

The failure is silent and model-facing. An agent asking for GPCRs receives enzymes and transporters
with no signal that the filter did not apply, and the returned `family`/`type` fields look
authoritative because they are real upstream values. A caller cannot distinguish "these 13 are the
GPCRs" from "the filter was dropped." This is a wrong-answer bug in a research pipeline, not a
usability defect.

## Fix (pick one, both are ~1h)

**A — filter client-side (preserves the advertised contract).** Drop `params["type"]`, keep the
parameter, and filter `targets` on `target.get("type", "").casefold() == type_filter.casefold()`
before pagination. `total_count` must be computed after the filter or the pagination envelope lies.
Casefold matters: the API emits lowercase `gpcr` while the tool description says `GPCR`.

**B — remove the parameter.** Delete it from `servers/iuphar.py` and `clients/iuphar.py`, and remove
the `type_filter="GPCR"` example from the tool docstring. Cheapest and honest, but it removes a
capability agents currently try to use.

Recommend **A**. The 13-record result sets show upstream page sizes here are small enough that
client-side filtering costs nothing.

## Acceptance

* Unit test with a mocked 13-record `adenosine` response: `type_filter="GPCR"` returns only `gpcr`
  candidates; `type_filter="gpcr"` returns the same set (casefold); `total_count` equals the filtered
  length, not 13.
* Integration test marked `integration` asserting the live `adenosine` + `GPCR` call returns no
  `enzyme` or `transporter` candidate.
* If option B: no `type_filter` remains anywhere in `src/` or in the tool description.

## Relation

Same file as AGE-704 (`_rate_limited_get` deadlock). Both are small; landing them in one PR against
`clients/iuphar.py` avoids a rebase. Not covered by AGE-688 — that issue's IUPHAR item is response
casing, a different defect.
````

---

### N2 — `[biosciences-mcp] No blocking CI — 933 tests have never gated a PR; port psychology-mcp's ci.yml`
**Priority:** High · **Labels:** platform, ci, infrastructure · **Blocked by:** AGE-703 (for the `fastmcp inspect` step only) · **Related:** N3, AGE-688, AGE-696

````
## Problem

`biosciences-mcp/.github/workflows/` contains exactly two files — `claude.yml` and
`claude-code-review.yml`. Both are Claude-review workflows. **Neither runs a test.** 933 tests have
never blocked a merge.

`psychology-mcp/.github/workflows/ci.yml` is a working blocking gate in the same org and can be
ported almost verbatim.

## Measured staging constraint (2026-09-12)

```
pytest --collect-only -q             → 933 tests
pytest --collect-only -q -m unit     → 562/933 (371 deselected)
pytest --collect-only -q -m contract → 173/933 (760 deselected)
```

Green and blockable today: **unit (562)**, **contract (173)**, `ruff check`, `ruff format --check`.

Cannot block yet: **integration** (6 known failures, AGE-688) and **pyright** (20 errors). Both
belong in a separate non-blocking workflow until those are cleared, so CI never spends a public API's
rate budget on every push.

## Unexpected finding — a marker-scoped CI would skip the gateway guard

`tests/integration/test_gateway.py::test_gateway_tools_exposed` is **unmarked**. It is one of only 4
unmarked tests in the repo:

```
pytest --collect-only -q -m "not unit and not integration and not contract and not e2e" → 4/933
```

`-m unit`, `-m integration`, `-m contract` and `-m e2e` all deselect it. It runs only on a bare
`pytest` with no `-m`. So the repo's only assertion about the gateway's tool surface would be
invisible to the CI proposed here. Fixed in the companion 34-tool-name issue; that issue should land
inside or immediately before this one.

## Proposed workflow

Port `psychology-mcp/.github/workflows/ci.yml` with these deltas:

1. Keep: checkout, `astral-sh/setup-uv@v5` with cache, `uv python install <pinned>`,
   `uv sync --extra dev`. (Interpreter version is pending a decision — `.python-version` currently
   says 3.13 while pyright targets 3.11; see the pin-convention issue.)
2. `uv run ruff check .`
3. `uv run ruff format --check .`
4. `uv run pytest -m unit -q`
5. **Add** `uv run pytest -m contract -q` — 173 tests, offline (in-memory `Client(server)`), and they
   are the ADR-001 wire contract. psychology-mcp has no equivalent tier.
6. `uv run fastmcp inspect fastmcp.json` then the tool-surface assertion, with the expected set being
   the **34 gateway names** (see companion issue), not a 2-name smoke check. Keep psychology-mcp's
   `-o inspect.json` workaround comment verbatim — piping `--format fastmcp` to a parser fails on
   injected newlines.
7. **Omit** `uv run pyright` with a comment naming the 20-error backlog, and **omit** all
   integration/e2e markers with a comment naming AGE-688. Both get a separate `nightly.yml` on a
   schedule, non-blocking.

The `uv sync --extra dev` comment in the psychology-mcp file is load-bearing — without the extra,
`uv run pytest` falls through to a pytest on PATH under a different interpreter and every test errors
on import, making the repo look broken when it is only unsynced. Copy the comment.

## Caveat that will bite step 6

`import biosciences_mcp` currently performs a live EBI fetch (AGE-703): `clients/chembl.py:23`
imports `chembl_webresource_client.new_client` at module level, which GETs
`https://www.ebi.ac.uk/chembl/api/data/spore` on import. **`fastmcp inspect` will therefore fail CI
during any EBI outage, on every PR, in every repo state.** Either land AGE-703 first, or mark step 6
`continue-on-error: true` until it lands and remove the escape hatch in the AGE-703 PR.

## Also worth hosting here

AGE-696's public→private link check has no CI job to attach to. The pattern-governance research §4
designs `biosciences-program/scripts/check_ids.py` (walks every `.md` in the workspace, resolves
`PRIN-*` / `ADR-*` citations against `governance/index.json`, fails on unresolvable ones). The link
check is the same shape of check over the same corpus and should be a rule inside it, generic over a
visibility field rather than hardcoding `open-biosciences/psychology-mcp` (which went public
2026-09-03).

## Acceptance

* A PR touching `src/` cannot merge with a failing unit or contract test, a lint error, or a format
  diff.
* The workflow is `on: [push (main), pull_request]` with `concurrency` + `cancel-in-progress`,
  `timeout-minutes: 15`.
* Every omitted tier carries an inline comment naming the issue that unblocks it.

## Rank

Above AGE-701 (DepMap pilot) and AGE-702 (Spec Kit upgrade): both add surface to a repo with no gate,
and this makes every subsequent fix in the backlog durable. Below AGE-703 and AGE-704 only because
those are live outages, and because AGE-703 is a hard prerequisite for step 6.

## Duplicate check

AGE-579 ("[CI] Add fastmcp inspect validation gate") is parented to AGE-552 and is
psychology-mcp/plugin scoped — already implemented in that repo's `ci.yml`, whose inline comments cite
AGE-579 and AGE-582 by number. AGE-584 (weekly live integration) is Canceled. Nothing covers
biosciences-mcp CI.
````

---

### N3 — `[biosciences-mcp] Pin the exact 34 gateway tool names — tighten and re-mark the existing test_gateway.py`
**Priority:** High · **Labels:** platform, mcp-server, test · **Blocks:** N6 (FastMCP 2→3) · **Related:** N2

````
## What exists today

`tests/integration/test_gateway.py::test_gateway_tools_exposed` already imports the gateway and calls
`get_tools()`. Three problems:

1. **It asserts a 20-name subset**, not the full surface. A tool silently vanishing is caught only if
   it is one of the 20.
2. **`assert len(tool_names) >= 30`** — a lower bound. Adding a tool, or a rename that produces a
   duplicate under a new spelling, passes.
3. **It is unmarked.** `pytest --collect-only -q -m "not unit and not integration and not contract
   and not e2e"` returns 4/933 tests, and this is one of them. `-m unit`, `-m integration`,
   `-m contract` and `-m e2e` all deselect it. Any marker-scoped CI skips it entirely.

The test needs no network — it imports `biosciences_mcp.servers.gateway` and calls `get_tools()`
in-process. It is misfiled under `tests/integration/`.

## The actual surface (enumerated at runtime on 2.14.5, 2026-09-12)

Exactly **34** tools:

```
biogrid_get_interactions, biogrid_search_genes,
chembl_get_compound, chembl_get_compounds_batch, chembl_search_compounds,
clinicaltrials_get_trial, clinicaltrials_get_trial_locations, clinicaltrials_search_trials,
ensembl_get_gene, ensembl_get_transcript, ensembl_search_genes,
entrez_get_gene, entrez_get_pubmed_links, entrez_search_genes,
hgnc_get_gene, hgnc_search_genes,
iuphar_get_ligand, iuphar_get_target, iuphar_search_ligands, iuphar_search_targets,
opentargets_get_associations, opentargets_get_target, opentargets_search_targets,
pubchem_get_compound, pubchem_search_compounds,
string_get_interactions, string_get_network_image_url, string_search_proteins,
uniprot_get_protein, uniprot_search_proteins,
wikipathways_get_pathway, wikipathways_get_pathway_components,
wikipathways_get_pathways_for_gene, wikipathways_search_pathways
```

**The gateway docstring is already wrong.** `servers/gateway.py` line 15 advertises
`pubchem_get_compound_synonyms`; it is not in the 34. Fix the docstring in the same PR — it is
currently the only written statement of the tool contract, and it does not match the code.

## Change

1. Move the file to `tests/contract/test_gateway_tool_names.py` (or leave it in place and mark it) and
   add `@pytest.mark.contract` — it is a wire-contract assertion and belongs in the 173-test offline
   tier, not `integration`.
2. Replace the subset check and the `>= 30` bound with:

```python
EXPECTED_GATEWAY_TOOLS = frozenset({ ... all 34 ... })

assert frozenset(tool_names) == EXPECTED_GATEWAY_TOOLS, (
    f"added: {sorted(set(tool_names) - EXPECTED_GATEWAY_TOOLS)}  "
    f"removed: {sorted(EXPECTED_GATEWAY_TOOLS - set(tool_names))}"
)
assert len(tool_names) == 34
```

3. Keep the existing negative assertion (no unprefixed `search_genes` etc.) — it is good and it is the
   half that catches a mount losing its prefix.
4. Correct the `servers/gateway.py` docstring to the 34 real names.

Set equality, not a subset: the failure mode being guarded produces *renamed* tools, so the
removed-set is the signal.

## Why now

Open question 1 of the version tradeoff analysis: under FastMCP 4, `Namespace._transform_name`
unconditionally returns `f'{prefix}_{name}'` while `tool_names` is applied first, so a naive port of
`servers/gateway.py` yields `hgnc_hgnc_search_genes` across all 34 tools. **The server starts fine and
every downstream binding silently breaks** — deepagents `apps/api/shared/mcp.py`, temporal activities,
and the `bio-research` plugin all address tools by name.

That inversion is **read from v4 source, not executed** — the v4 docstring still says "override" while
the code prepends. Prove it in a throwaway venv before editing `gateway.py`. But this test costs
~30 minutes and is correct regardless of whether the inversion claim holds.

Note the current 20-name subset check *would* have caught the double-prefix case (`hgnc_search_genes`
would go missing). It would **not** catch a partial rename, a dropped mount above the `>= 30` floor,
or anything at all under a marker-scoped CI.

## Acceptance

* The test is collected by `pytest -m contract`.
* It fails with a readable added/removed diff when a mount prefix or a `tool_names` entry changes.
* `servers/gateway.py`'s docstring lists exactly the 34 names the server exposes.

## Effort

~30 min to tighten the assertions (the test exists) + ~15 min for the marker move + ~10 min for the
docstring. ~1h total — the missing piece was the marker, not the test.
````

---

### N4 — `[biosciences-mcp] One owner for identifier patterns: src/biosciences_mcp/identifiers.py (two layers, two phases)`
**Priority:** Medium · **Labels:** platform, mcp-server, refactor · **Blocks:** AGE-700 · **Related:** AGE-701 (pilot), AGE-687 (orphaned out-of-scope item), AGE-698 (same `clients/` files)

````
## Problem

There is no owner for identifier patterns. Measured in `src/` on 2026-09-12:

* **40** `re.compile(...)` constants
* **18** `pattern=` declarations across 7 model modules (`gene`, `pathway`, `ensembl`, `drug`,
  `biogrid`, `provenance`, `pharmacology`)
* **12** inline `re.match` / `re.search` / `re.fullmatch` calls with literal patterns
* plus `tests/contract/registry.py` (50 keyed entries) and `.claude/hooks/validate-curie.py`, whose
  own docstring states the patterns are *transcribed* from `src/` and that the file is stale whenever
  a server's pattern changes

(The tradeoff analysis §5 reported "~20 pattern declaration sites" — that was an inference over a
narrower definition. The counts above are a direct grep and supersede it.)

### Four byte-identical duplications

| Pattern | Site A | Site B |
|---|---|---|
| `^(EFO\|MONDO\|Orphanet\|HP\|DOID\|OTAR)_\d+$` | `models/target.py:23` | `clients/opentargets.py:49` |
| `^ENSG\d{11}$` | `models/target.py:21` | `clients/opentargets.py:47` |
| `^DrugBank:DB\d{5}$` | `models/drug.py:15` | `clients/drugbank.py:43` |
| `^ENSG\d{11}$` *(third spelling, different constant name)* | `models/ensembl.py:21` | — |

`DISEASE_ID_PATTERN` is a **third spelling** of the disease identifiers — underscore-separated,
distinct from both the registry's colon forms (`^EFO:\d{7}$`, `^MONDO:\d{7}$`, `^ORPHA:\d+$`) and
Bioregistry's bare form. Consolidation is larger than 23 keys.

### A dead dict that looks like a validator

`CROSS_REF_PATTERNS` at `models/cross_references.py:16` is a 23-key `{key: re.compile(...)}` mapping.
`grep -rn CROSS_REF_PATTERNS src/ tests/` returns **exactly one hit — the definition.** Nothing
imports it. It has never validated anything. AGE-687 listed wiring it as a validator as out of scope
and filed no successor. It is also the direct cause of the top recurring CRITICAL in AGE-702's
baseline converge, which hit all 13 specs: *"cross-reference regex validation declared but never
applied (`INVALID_CROSS_REFERENCE` never raised)."* Two issues had independently recorded the same
defect from opposite ends without either citing the other.

### The refseq disagreement this surfaces

* `tests/contract/registry.py:54` → `^[NX][MR]_\d+$`
* `models/cross_references.py:21` → `^[NX][MR]_\d+$`
* `models/ensembl.py:28` → `^[NX][MPRG]_\d+(\.\d+)?$`

Three sites, two spellings, all citing "ADR-001 Appendix A". **Not a live wire defect** — no client
currently emits `NP_` into the `refseq` key (`clients/uniprot.py:151-159` takes
`NucleotideSequenceId`; `clients/entrez.py:435` filters on `^[NX][MR]_\d+$`). It is a definitional
question escalated to the Program Director: *is a registry key the identifier space it names, or the
subset this platform emits?* **Do not resolve it silently in this refactor** — take the answer from
AGE-700 and make the three sites agree with it.

## Phase 1 (~1.5–2 days) — create the owner, no migration. This is what unblocks AGE-700.

* `src/biosciences_mcp/identifiers.py` with two explicitly named layers — the distinction the repo has
  been conflating:
  * **`XREF_PATTERNS`** — bare local IDs, for `cross_references` values (ADR-001 Appendix A)
  * **`CURIE_PATTERNS`** — prefixed forms, for Fuzzy-to-Fact tool inputs (`validate_id`)
* Absorb the dead `CROSS_REF_PATTERNS` into `XREF_PATTERNS` and delete it. Decide explicitly whether
  it becomes a real validator raising `INVALID_CROSS_REFERENCE` or is removed — do not leave it
  dangling a second time.
* Collapse the four duplications to imports.
* Record (do not decide) the refseq divergence as an AGE-700 registry amendment.
* Resolve `DISEASE_ID_PATTERN`: either a documented fourth layer with a stated reason, or reconciled
  to the registry form.
* Contract tests for the two invariants the analysis flags as new-and-rottable — make them tests, not
  comments:
  * both layers cover the same concept set (failure mode (d))
  * adding a key without adding it to both layers fails (failure mode (b))

## Phase 2 (~3–5 days) — migrate the call sites

* Repoint the remaining `re.compile` constants and `Field(pattern=)` declarations at the module, one
  client/model family per PR.
* Make `.claude/hooks/validate-curie.py` **generate** its table by importing the module instead of
  transcribing it — the step that permanently kills the staleness its docstring already admits to.
* Point `tests/contract/registry.py` at the same source.

## Effort

**5–8 days total, split.** The tradeoff analysis's 2–3 day figure was `[inference]` over a ~20-site
inventory; the measured inventory is ~70 occurrences across 20+ files with four duplications and one
substantive disagreement requiring a decision. This re-estimate is itself an estimate — nobody has
done it — and the source doc warns its estimates skew low. A single 70-site PR is unreviewable and
will collide with AGE-698 and AGE-704 in `clients/`.

Net new effort is less than the headline: this absorbs the unfiled CURIE-validation-hook item and the
unowned "wire `CROSS_REF_PATTERNS` as a validator" gap from AGE-687.

## Acceptance (Phase 1)

* `identifiers.py` exists with both layers and a docstring stating which layer serves which ADR-001
  section.
* `grep -rn CROSS_REF_PATTERNS src/` returns nothing.
* No pattern literal appears in two files.
* The refseq change is recorded against AGE-700 with the upstream justification.
* `pytest -m contract` green.

## Not in this issue

The Bioregistry snapshot/oracle — separate issue, blocked on Phase 1.

## Sequencing

**Blocks AGE-700**: the Appendix A amendments assume one declaration site; amending before this lands
amends one of several copies. §5 calls this *"the prerequisite that makes AGE-700 answerable rather
than guesswork"* and *"unambiguously correct regardless of the registry decision"* — it does not wait
on the Bioregistry question. Coordinate with AGE-701 (see that issue's checklist decision). Sequence
Phase 2 after AGE-698 lands or expect a rebase.
````

---

### N5 — `[biosciences-mcp] Vendor a pinned Bioregistry snapshot as a CI drift oracle (explicitly not a runtime authority)`
**Priority:** Low · **Labels:** platform, mcp-server, test · **Blocked by:** N4 Phase 1 · **Related:** AGE-700

> **Filing note for the PD:** the *refusal* half of this is not blocked and should go into ADR-001 v1.5 (AGE-700) immediately as a recorded decision with its reason, or it will be re-proposed. This issue is only the oracle half.

````
## Why

Two things need recording, and they point opposite ways.

**The refusal.** `docs/plans/2026-09-12-interface-version-tradeoff-analysis.md` §5 measured all 23
registry keys against `bioregistry 0.14.6` on 2026-09-12: **8 drop-in, 7 silently widen, 7 break, 1
(`ucsc`) has no pattern at all.** The disqualifying finding is the widening, not the breakage:
**Bioregistry's `ensembl` pattern accepts the literal string `BRCA1`** (and `hello`). Adopting it as
the runtime validator would make `BRCA1` a syntactically valid Ensembl gene ID and **disarm the
ADR-001 §3 rejection that the enforced strict phase rests on** — which the v1.6.0 prior-art revision
names as this platform's one remaining differentiator against BioMCP and ToolUniverse. The same
pattern is served by Identifiers.org, so it is inherited from MIRIAM, not a curation lapse.

At the tool-input layer it is unusable anyway: `is_valid_curie()` returns **False for all 10** project
CURIEs, and `normalize_prefix()` returns `None` for `IUPHAR` and `WP`. Bridging would need a
hand-maintained project→Bioregistry prefix map — the artifact you were trying to delete. Node
Normalizer answers a different question entirely (`HGNC:NOTANID`, `FOO:bar` and `HGNC:99999999` all
return identical `null`, so it cannot distinguish malformed from nonexistent — the exact distinction
the ErrorEnvelope encodes).

**The oracle.** Used as a *check* rather than an authority, it earns its place: the refseq divergence
AGE-700 is arguing about — three declaration sites, two spellings — is precisely what a drift test
would have surfaced.

## What

1. `data/bioregistry-patterns.json`, ~2 KB: prefix, pattern, bioregistry version, Zenodo DOI pin
   (`10.5281/zenodo.22681298`). `bioregistry` goes in the **dev/test extra only** — it is a 7.3 MB
   wheel that would otherwise add `pydantic[email]`, `curies`, `sssom-pydantic` and `urllib3` to every
   server's runtime closure for static offline data.
2. `tests/contract/test_bioregistry_drift.py`, marked `integration` so unit runs stay offline. Per
   key, assert a **recorded relationship with a written justification** —
   `{equivalent, project_is_narrower, prefix_layer_differs, concept_mismatch}` — **never equality**.
3. Assert `set(XREF_PATTERNS) == set(snapshot)` explicitly, or a new key added without a snapshot
   entry is invisible (failure mode (b)).
4. A max-age assertion or scheduled refresh on the recorded version — a snapshot nobody refreshes
   stops being an oracle while still looking like one (failure mode (c)).

## Stated failure modes, carried forward verbatim from the source

- **This does not eliminate hand-maintained regexes.** It makes 15 of 23 divergences deliberate rather
  than accidental. If the goal was "stop hand-maintaining patterns," this does not achieve it, and the
  evidence says nothing can — 7 break and 7 widen.
- `ucsc`, `string`, `stitch` get **no oracle coverage** and stay purely hand-owned. File two upstream
  issues on `biopragmatics/bioregistry` for the STRING and STITCH resources; cheap, and turns two
  `concept_mismatch` rows into resolvable ones.
- **The "8 drop-in" verdict is measured on hand-authored accept/reject vectors**, not proven
  equivalence. Before acting, replay real `cross_references` values captured from the e2e suite
  through both patterns.
- **Pattern churn rate for these 23 prefixes specifically was not measured** — only the registry-wide
  rate. A `git log -L` over those pattern lines would set the drift-test cadence empirically instead
  of by projection.

## Effort

Small once N4 Phase 1 lands — a ~2 KB data file and one parametrized test. No estimate is given in the
source; do not inherit one. The prerequisite is where the cost sits, not here.
````

---

### N6 — `[biosciences-mcp] Upgrade fastmcp 2.14.5 → 3.4.7 and replace the major-version pin with a minor ceiling`
**Priority:** Medium · **Labels:** platform, mcp-server, dependencies · **Blocked by:** N3 · **Related:** AGE-700, AGE-701, N7

````
## Problem

`pyproject.toml:11` pins `fastmcp>=2.14.1,<3.0`. That ceiling was set when 3.x was an RC (3.0.0b1).
3.0.0 went GA 2026-02-18 and 4.0.3 is stable as of 2026-09-05. The 2.x line has had no release since
2.14.7 (2026-04-13) — five months, which reads as backport-only life support **[inference from
cadence; no EOL is published for either line — do not state one as fact in an ADR]**. Staying on 2.x
forgoes seven months of security hardening, including the v3.4.3 SSRF fix, the v3.4.5 JWKS fix, and
the diskcache CVE-2025-69872 dependency dropped in 3.0.0.

## Why this is one file, not thirteen

The repo's entire FastMCP API surface is `13 × FastMCP(name)`, `36 × @mcp.tool`, `12 × mount()`,
`mcp.run()`, and `Client(server)` in tests. No `Context`, no lifespan, no resources, prompts,
middleware, auth, sampling, or `mcp.types` imports **[inference, from grep over `src/`, `tests/`,
`scripts/`]**. Every documented 3.x break is therefore a no-op except in `servers/gateway.py`.

## The change

`servers/gateway.py`, 12 identical edits:

* `mount(prefix="hgnc", ...)` → `mount(namespace="hgnc", ...)`
* `as_proxy=False` → removed (the parameter is gone; direct mounting is the behaviour)
* **`tool_names={...}` → deleted entirely.** Those dicts only ever restated `prefix + original name`;
  `namespace="hgnc"` alone reproduces today's 34 names. Keeping them is what produces
  `hgnc_hgnc_search_genes` in v4 and is a latent hazard in 3.x.

Server modules: zero edits. The in-memory `Client(server)` test pattern and snake_case
`CallToolResult` fields survive unchanged.

## Pin convention

Replace `>=2.14.1,<3.0` with **`>=3.4.7,<3.5`**. FastMCP's own releases policy states minor versions
may include breaking changes, so a major ceiling is not a compatibility guarantee under this
project's versioning contract — `<3.0` held only because no 2.15 ever shipped. All of 3.x pins
`mcp>=1.24.0,<2.0`, so the installed SDK 1.26.0 carries through untouched and 3.4.7 is a genuine
resting point, not a waypoint. See the companion pin-convention issue.

## Preconditions

1. **The 34-name contract test must land first** (companion issue). The pin change *is* the deploy:
   `biosciences-mcp.fastmcp.app` resolves its FastMCP major from `pyproject.toml`, and five repos
   point at that endpoint.
2. **Stage behind a preview deployment.** Open question 3 in the tradeoff analysis is unresolved: the
   Prefect Horizon / FastMCP Cloud deployment docs state no supported FastMCP version range. A preview
   deploy answers it; nothing else does.
3. Run `pytest -m contract` (173 tests, offline) before and after.
   `tests/contract/test_wire_contracts.py::test_entity_has_no_null_values` is the canary for the
   ADR-001 §4 omit-null guarantee.

## Nearly free, worth doing in the same PR

`readOnlyHint: true` + `openWorldHint: true` on all 34 tools. By omitting annotations the federation
currently advertises worst-case defaults (`readOnlyHint` false, `destructiveHint` true), so a host
applying confirmation policy by annotation will prompt for approval on `hgnc_get_gene`. Every tool is
a GET over a public API. **Confirm 2.14.5 accepts the kwarg — not verified for that version.** Note
the spec makes annotations explicitly untrusted, so this is a UI-hint improvement and never a policy
boundary.

## Governance rider

**ADR-004 reasons from the premise that direct mounting runs no lifespan.** v4 reverses that. 3.x does
not, so this upgrade does not invalidate ADR-004 — but the reasoning is now version-scoped and the ADR
should say so before anyone reads it as timeless. Record it; do not act on it yet.

## Explicitly not in scope

**FastMCP 4.** Three unresolved gates sit in front of it, none of them code: (a) whether Horizon
builds v4 and still honours the v1 `fastmcp.json` schema; (b) a downstream consumer audit across
deepagents `apps/api/shared/mcp.py`, temporal activities, and the bio-research plugin — **not done**;
(c) the v4 text-block serialization path versus the omit-null guarantee — **not traced**.
`biothings/smartapi-mcp` independently pins `fastmcp>=3.3.1,<4` with the note that 4.x moves to the
MCP 2.x SDK and httpx2. Corroboration that stopping at 3.x is reasonable, not proof.

## Acceptance

* `pyproject.toml` reads `fastmcp>=3.4.7,<3.5`; `uv.lock` regenerated.
* `pytest -m unit` and `pytest -m contract` green.
* The 34-name contract test passes unchanged — same 34 names, no double prefixes.
* A preview deployment serves `tools/list` with the same 34 names before the pin merges to main.
* ADR-004's lifespan premise is annotated as scoped to FastMCP < 4.

## Effort and rank

~half a day of edits and one contract run **[inference, dominated by one contract run over 173 tests,
not by authorship]**, plus a preview deployment cycle that is wall-clock, not effort. Below AGE-703,
AGE-704, the IUPHAR filter bug, the CI gate, AGE-701 and AGE-702: nothing is broken today. The
dividend is security posture and the component-versioning mechanism that would let AGE-700's registry
amendment ship as `get_pathway` v2 instead of a flag-day break across five repos — which is the
strongest argument for landing it *before* AGE-700 resolves.
````

---

### N7 — `[governance] Pin convention: exact floor + MINOR ceiling for fastmcp, and a tracked .python-version per repo`
**Priority:** Low · **Labels:** governance, dependencies · **Related:** N2 (CI must install the agreed interpreter), N6

````
## Two conventions, one small issue

### 1. FastMCP pins: exact floor + MINOR ceiling

FastMCP's releases policy states that **minor versions may include breaking changes when necessary for
the ecosystem's evolution**. A major-version ceiling is therefore not a compatibility guarantee under
FastMCP's own versioning contract. `>=2.14.1,<3.0` held only because no 2.15 ever shipped — that is
luck, not a constraint.

Current state across the org, measured 2026-09-12:

| Repo | Pin |
|---|---|
| `biosciences-mcp` | `fastmcp>=2.14.1,<3.0` |
| `psychology-mcp` | `fastmcp>=2.14.1,<3.0` |
| `biosciences-memory` | `fastmcp>=2.13.3,<3` |
| `biosciences-mcp-edge` | `fastmcp>=2.0,<3.4.3` |

Four repos, four different pins, and `biosciences-mcp-edge` spans two majors with a floor of 2.0 and a
patch-precision ceiling of 3.4.3 — a shape that permits both FastMCP 3.0.0 and FastMCP 2.0. That one
is a bug, not a convention choice.

**Convention to write down:** `fastmcp>=X.Y.Z,<X.(Y+1)` — exact known-good floor, minor ceiling.
Bumping the minor becomes a deliberate act with a contract run behind it, which is the behaviour a
major ceiling only pretended to give.

Apply as each repo is next touched; do not open four PRs at once. `biosciences-mcp`'s bump rides along
with its 2→3 upgrade issue.

### 2. `.python-version`: track it, or delete it

An earlier framing said "no repo has one." That is no longer true and the current state is worse than
absence:

* `biosciences-mcp/.python-version` **exists, is untracked, created 2026-09-12, and says `3.13`** —
  while `pyproject.toml` declares `requires-python = ">=3.11"` and `[tool.pyright] pythonVersion =
  "3.11"`.
* `biosciences-mcp-edge` also has one.
* Every other repo in the workspace has none.

An untracked `.python-version` is the worst of the three states: it silently pins one developer's `uv`
interpreter to 3.13 while CI, pyright and every other clone resolve 3.11. Type errors and runtime
behaviour diverge between the machine that has the file and every machine that does not, with nothing
in the repo to explain it.

**Convention:** every Python repo commits a `.python-version` matching the version its CI installs and
its pyright targets. Pick 3.11 or 3.13 deliberately and make `requires-python`,
`[tool.pyright] pythonVersion`, `.python-version` and the CI `uv python install` step agree.

## Acceptance

* A short convention note in `biosciences-program` (or the ADR that covers dependency management)
  stating both rules and the FastMCP policy quote that motivates the first.
* `biosciences-mcp/.python-version` either committed with all four declarations agreeing, or deleted.
* `biosciences-mcp-edge`'s `fastmcp>=2.0,<3.4.3` corrected — that one is a live inconsistency, not a
  style preference.

## Rank

Low. Nothing is broken *because* of these today. Worth filing so the fix rides along with work already
touching each `pyproject.toml`, and so the `.python-version` drift is resolved rather than
accumulating. **Note: the interpreter choice must be settled before the CI issue lands, or CI encodes
the disagreement.**
````

---

### N8 — `[biosciences-mcp] Submit a meta.yaml to the BioContextAI Registry`
**Priority:** Low · **Labels:** platform, docs, discovery · **Sequence after:** N2 (CI)

````
## What

Submit a `meta.yaml` PR to the **BioContextAI Registry** (Nat Biotechnol 2025, `s41587-025-02900-9`) —
71 schema-validated biomedical MCP servers, CI-enforced schema, published as the discovery layer for
this field.

`biosciences-mcp` is not listed. Peers and direct competitors already are: `genomoncology-biomcp`,
`meringlab-string-mcp`, `mims-harvard-ToolUniverse`, and EBI-adjacent servers.

## Does it clear the bar?

Yes, on the criteria that were checkable locally:

* **MIT licensed** — yes, org-wide.
* **Typed tools** — 34 tools, Pydantic v2 models, `inputSchema` generated by FastMCP.
* **Tests** — 933, of which 173 are wire-level ADR-001 contract tests.
* **Free academic access to every exposed service** — **yes.** This was the flagged risk and it is
  clean. `servers/gateway.py` mounts 12 servers and excludes DrugBank by explicit comment ("DrugBank
  excluded (requires commercial API key)"). A runtime enumeration of the gateway on 2026-09-12 returns
  exactly 34 tools, none of them `drugbank_*`. 10 of the 12 remaining upstreams need no key at all;
  BioGRID and NCBI keys are held server-side by the platform operator, not required of a consumer.

**One disclosure to make honestly in the PR:** `src/biosciences_mcp/servers/drugbank.py` and
`clients/drugbank.py` still ship in the repository and DrugBank contract tests are collected — the
code exists, it is simply not mounted or deployed. Describe the *deployed* surface (12 sources, 34
tools) and say so plainly if asked. Do not let a reviewer discover it.

## Content for the meta.yaml

* Endpoint: `https://biosciences-mcp.fastmcp.app/mcp`
* 34 tools across 12 sources: HGNC, UniProt, ChEMBL, Open Targets, STRING, BioGRID, Ensembl, Entrez,
  PubChem, IUPHAR/GtoPdb, WikiPathways, ClinicalTrials.gov
* The differentiator, stated in the project's own words: an **enforced** strict phase (raw string →
  `UNRESOLVED_ENTITY`, contract-tested) and the 23-key cross-reference registry. Not "progressive
  disclosure" or "two-phase search" — a 630-star competitor (BioMCP) already has both, and claiming
  them reads as a small ToolUniverse.

## Unverified before filing

**The BioContextAI `meta.yaml` schema was not checked** (no web search in this pass). If it requires a
maintainer contact address, AGE-697's single open item (contact alias for the polite-pool header)
becomes a blocker on this rather than a loose end. Confirm the schema first.

## Sequencing

After CI lands, so the listing points at a repo with a visible green gate rather than two
Claude-review workflows and no test gate. Not blocked by it — the registry does not require CI — but
the listing is a shop window.

## Rank

Lowest of everything filed in this pass. Nothing depends on it, it is reversible, and it competes with
no defect. But it is one hour and it is the cheapest visibility the project will ever buy.
````

---

## 4. Leave alone

These were examined and need no edit. Listed so nobody re-triages them next week.

| Issue | Why nothing changes |
|---|---|
| **AGE-704** — IUPHAR rate limiter deadlock | Read from source rather than re-derived: `clients/iuphar.py:47-85` recurses into `_rate_limited_get` on both the 429/503 arm (`:69-72`) and the network-error arm (`:74-81`), both inside `async with self._lock`, and `asyncio.Lock` is not reentrant. The issue is exactly right as filed, including the 52-minute hang and the AGE-698 relationship. HIGH is correct. Nothing in the evidence base bears on rate limiting or retry. The only new fact is *which PR it rides in* (with N1, same file) — recorded on N1, not here. The adjacent finding about the timeout arm is recorded on AGE-698. |
| **AGE-705** — UniProt cursor pagination reads the cursor from the JSON body | Unaffected by all five documents. The nearest adjacency is prior-art §4.6's note that `resources/read` supports caching but not pagination — an argument against a Resources migration, not a change to how `PaginationEnvelope` is populated from a REST response. Scope is well specified (parse `Link: rel="next"` and `X-Total-Results`, correct `specs/002-uniprot-mcp-server/research.md` R3, replace the vacuous test) and MED is right: page 2 is unreachable, a functional gap, not an availability one. |
| **AGE-697** — Personal data in public org repos | Nothing in the evidence base touches author email, participant names, home paths, or private run identifiers. The single unchecked item (contact alias for the polite-pool header) is a maintainer action, unchanged in substance and priority. One adjacency deliberately not acted on: *if* the BioContextAI `meta.yaml` schema requires a maintainer contact, this item becomes N8's blocker. **The schema was not checked** — that is a check to run when N8 is picked up, not a finding, and it does not justify an edit here now. |
| **AGE-687** — cross-reference series (Done) | Its out-of-scope item ("wire `CROSS_REF_PATTERNS` as a validator") is now owned by N4. Do not reopen. |
| **AGE-694 / AGE-695** | The AGE-688 → AGE-694/695 → AGE-698 ordering stands exactly as already written on those issues. No evidence touches them. |
| **AGE-681** — DepMap genotype-selective MCP (In Progress) | Live work under AGE-680. Untouched by the evidence base. |
| **AGE-680** — plugin roadmap umbrella | Stays as the live umbrella for the lung-paper strand. The only touch is a relates-to link to AGE-239 plus one line pointing at the April catalogue — recorded under Do-now item 14. No scope, priority or acceptance change. |
| **AGE-579** — `[CI] Add fastmcp inspect validation gate` | Parented to AGE-552, psychology-mcp/plugin scoped, and **already implemented** in that repo's `ci.yml` (whose inline comments cite AGE-579 and AGE-582 by number). Duplicate-cleared for N2. No action. |
| **AGE-584** — weekly live integration | Canceled. N2 deliberately does not revive it: integration and pyright go to a separate non-blocking nightly, not a weekly live run. |
| **AGE-552** | Parent of AGE-579, psychology-mcp scope. Untouched. |
| The remaining **AGE-686–AGE-705** issues not named above | The Synapse track counted 13 September MCP-defect issues in that band (6 Done, 2 Todo, 4 Backlog). Six were dispositioned in this pass; the rest were not examined. No evidence attaches to them *that anyone looked for* — see §6. |
| Everything outside the AGE-686–AGE-705 band and the five-issue Synapse strand | Not dispositioned in this pass at all. No claim is made about them either way. |

**Declined — do not file:**

| Candidate | Why declined |
|---|---|
| ChEMBL lazy import as its own issue | Fully covered by **AGE-703**, verbatim including acceptance criteria. Confirmed unchanged on main. Filing it would be exactly the re-derivation this pass exists to prevent. The one new fact (the CI dependency) is attached as a comment. |
| `readOnlyHint` / `openWorldHint` on the 34 tools | Rides in the N6 PR. Filing separately fragments the backlog for no gain. Note the spec makes annotations explicitly untrusted — a UI-hint improvement, never a policy boundary. |
| Negative capability assertion in ADR-001 ("no roots, sampling, logging or elicitation") | Rides in AGE-700's §4 additions. |
| STRING build-vs-defer as an engineering ticket | It is a Program Director decision, not a ticket — see **D3**. If the answer is A (recommended), the output is one paragraph and no issue exists. Filing it as an engineering ticket invites someone to start option B, which is a multi-repo breaking change. |

---

## 5. Where the research attaches

The point of the exercise: after this pass, every finding is reachable from the backlog, not only from `docs/plans/`. **All five documents are currently untracked in git — Do-now item 1 must land or none of these citations resolve.**

| Synthesis document | Location / status | Issues it informs | What it settles there |
|---|---|---|---|
| **`2026-09-12-interface-version-tradeoff-analysis.md`** | `biosciences-program/docs/plans/` — **untracked** | AGE-703, AGE-701, AGE-700, AGE-225, **N3**, **N4**, **N5**, **N6**, **N7**, **N8**, decision **D3** | §1 item 3 + §5: the identifier-consolidation prerequisite and the ~20-site inventory (superseded by N4's measured ~70). §2: the FastMCP pin/upgrade case and the `>=3.4.7,<3.5` convention. §3: negative capability assertion, error-code placement. §4: BioContextAI, STRING build-vs-defer. §5: the two-layer `XREF_PATTERNS` / `CURIE_PATTERNS` split (which is what proves AGE-225 unblocked), the Bioregistry 8/7/7/1 measurement and the refusal, failure modes (b)(c)(d)(f). Session measurements: the 1,234,245 us import cost behind AGE-703. |
| **`2026-09-12-prior-art-v1.6.0-proposed-patch.md`** | `biosciences-program/docs/plans/` — **untracked**; also as `biosciences-mcp/docs/prior-art-revisions/2026-09-12-v1.5.0-to-v1.6.0-patch.md` on branch `docs/prior-art-v1.6.0`, **PR #15, not on main** | AGE-700 (Blocker 1), **N5**, **N8**, decision **D3** | Patches 5 and 8 feed AGE-700's item-by-item. Patch 4 carries the unresolved `isError: true` question. §7.3 and the new §10 are what ADR-001 v1.5 must cite instead of v1.5.0 §7.5. §10 is the STRING assessment. The differentiator wording (enforced strict phase vs BioMCP/ToolUniverse) is what N5's refusal and N8's positioning both rest on. **Any issue citing this must cite the branch, not `docs/prior-art-api-patterns.md` on main.** |
| **`2026-09-12-ddd-cqrs-pattern-governance-research.md`** | `biosciences-program/docs/plans/` — **untracked** | AGE-702, AGE-696, AGE-700, **N2** | §2: `biosciences-program/CLAUDE.md` line 22 states the inverse of AGE-702's outcome; and `biosciences-mcp` has no CI at all — the finding N2 exists for. §3: CLAUDE.md ranks last of six precedence levels ("context only; never as a merge requirement on their own") — which answers AGE-696's open AGENTS.md-vs-CLAUDE.md question *no*. §4: the `scripts/check_ids.py` design that AGE-696's grep task folds into, and the "delete the count from prose" recommendation feeding AGE-700's 23-key item. |
| **`2026-09-12-three-planes-claude-code-agents.md`** | `biosciences-program/docs/plans/` — **untracked** | **None.** | No track produced a single disposition citing it. Either it is not backlog-relevant or nobody triaged it — see §6. |
| **`2026-09-12-regulated-event-driven-agent-architectures.md`** | `biosciences-program/docs/plans/` — **untracked** | **None.** | Same. The governance track counted "five evidence documents" but its citations only ever reached the three above. |

---

## 6. What this pass did not settle

**Facts that would change dispositions, still unverified:**

1. **The refseq key definition** (decision **D1**). Everything downstream — N4 Phase 1's refseq handling, AGE-700's Appendix A amendment, whether `models/ensembl.py:28` is corrected or promoted — is provisional until it is answered.
2. **Does the federation set `isError: true` on the wire when returning an `ErrorEnvelope`?** If it returns a success result containing an error object, the spec's self-correction path never engages and every `recovery_hint` is inert — which would undercut the §7.2 claim in the prior-art revision. One contract assertion settles it. Flagged unverified in the v1.6.0 patch (Patch 4). Riding along with AGE-700; **not independently owned.**
3. **Does FastMCP 2.14.5 emit a usable `outputSchema` for the 34 union return types?** Cheap, unfiled, and **unowned** — it is the one open verification item from the evidence base with no home anywhere in this action list.
4. **The FastMCP 4 double-prefix inversion.** `Namespace._transform_name` appears to unconditionally return `f'{prefix}_{name}'` while `tool_names` is applied first. **Read from v4 source, not executed**, and the v4 docstring still says "override" while the code prepends. Does not gate 3.x; prove it in a throwaway venv before anyone edits `gateway.py`.
5. **Does 2.14.5 accept the `annotations=` kwarg?** Not verified for that version. Gates the free `readOnlyHint` rider in N6.
6. **Does the Prefect Horizon / FastMCP Cloud deployment platform support FastMCP 3.x, and does it still honour the v1 `fastmcp.json` schema?** No supported version range is documented. Only a preview deployment answers it; nothing in this pass did.
7. **The downstream consumer audit is not done** — deepagents `apps/api/shared/mcp.py`, temporal activities, the bio-research plugin's `.mcp.json`. Every "5 repos point at that endpoint" statement in this document rests on it.
8. **The v4 text-block serialization path versus the ADR-001 §4 omit-null guarantee** — not traced.
9. **Does `mcp.string-db.org` enforce a strict phase or accept bare symbols?** One `tools/list` plus one deliberately-malformed `get` would settle decision **D3** and possibly close it in half an hour. Not probed.
10. **The BioContextAI `meta.yaml` schema was not checked** (no web search this pass). If it requires a maintainer contact address, AGE-697's alias item becomes N8's blocker.
11. **Bioregistry's "8 drop-in" verdict was measured on hand-authored accept/reject vectors**, not on real data, and **pattern churn rate for these 23 prefixes specifically was never measured** — only the registry-wide rate. Both are prerequisites for acting on N5, recorded in the issue body.

**Things not visible from here:**

12. **The mocomakers / Computational Biology Working Group Linear workspace is unreadable from this connection** (`list_teams` returns only Agentic-wisdom; `get_issue COMPBIO-7` returns not-found). The lung paper's actual submission status and the final DAS wording are unconfirmed. This gates decision **D2** partly and AGE-242's deadline entirely.
13. **`cowork-ws/lung-paper/` is not present on this machine.** The `QC_REPORT_2026-04-21.md` and the process docs cited by all five April Synapse issues could not be read; AGE-239's eight candidates are carried forward from the issue body, not re-verified against the manuscript.

**Coverage gaps in the pass itself:**

14. **Two of the five synthesis documents — `three-planes-claude-code-agents.md` and `regulated-event-driven-agent-architectures.md` — produced zero dispositions across all three tracks.** Nobody stated whether that is because they are not backlog-relevant or because nobody read them against the backlog. Until someone says which, the "research is discoverable from the backlog" claim in §5 covers three documents, not five.
15. **Roughly half the AGE-686–AGE-705 band was never examined.** The Synapse track counted 13 September MCP-defect issues there; six were dispositioned. The rest are in §4 as "no action" only in the sense that nobody looked — that is not the same as "the evidence does not touch them."
16. **The `~20` vs `~70` identifier-site disagreement is adjudicated but not reconciled.** The two counts measure different things (declaration *sites* per the tradeoff analysis vs. all regex occurrences including duplicates and `Field(pattern=)` per the grep). The 5–8 day estimate that follows from the larger count is an estimate over a measured inventory, not experience — and the source doc warns its estimates skew toward *more* time chasing live-API flakes.
17. **The recurring process failure is diagnosed but not fixed.** Three independent instances this pass: AGE-688 already held today's IUPHAR and WikiPathways root causes in more detail than the re-derivation produced; AGE-680 re-derived April's manuscript-QC lessons without consulting AGE-239's catalogue; and AGE-702's converge and AGE-687 had independently recorded the same `CROSS_REF_PATTERNS` defect from opposite ends without either citing the other. The mitigation proposed (link AGE-680 ↔ AGE-239; make "list open issues with the relevant label" step one of any future pass) is a habit, not a mechanism, and nothing in this action list enforces it.