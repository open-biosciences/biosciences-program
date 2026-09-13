# DDD/CQRS and Pattern Governance — Decision Synthesis for the Program Director

**Date:** 2026-09-12 · **Scope:** all Open Biosciences repos · **Recommended vehicle:** ADR-PRG-001 (first program-scoped ADR) + ADR-MEM-002 (project-scoped)

---

## 1. Verdict on DDD/CQRS

**Selectively adopt — two terms, in one repo, bound to one artifact each. Everything else: decline, permanently and in writing.**

The originating question's answer stands: no DDD/CQRS vocabulary exists anywhere in the platform (verified zero hits across `.md` and `.py` in all platform repos). That makes this a greenfield naming decision with no cleanup benefit — adoption is pure new cost unless each term is attached to a defect it demonstrably names better than existing language.

### IN — adopt (biosciences-memory only)

| Concept | Bound to | Why it earns its keep |
|---|---|---|
| **Aggregate** = `group_id` | `biosciences-memory`, enforced by a new contract test | `group_id` is currently called a "namespace" — a naming-collision device that carries no consistency obligation. That is why 526 edges with `invalid_at < created_at` exist with no owner. The concurrency discipline is *already implemented and unnamed* (one `asyncio.Queue` + one worker per `group_id`). The word supplies the missing obligation: every write leaves the boundary consistent or fails. |
| **Anti-Corruption Layer** = the biosciences-mcp → biosciences-memory boundary | An owned translation module + a named write-model owner in `AGENTS.md` | Two incompatible definitions of "Gene" already coexist unnamed: a validated `^HGNC:\d+$` model on HGNC's authority, and three unvalidated `str \| None` fields populated by an LLM. ADR-MEM-001 already makes this declaration in prose. Naming it reclassifies deepagents' `TARGETS` edge type (the ontology defines `TargetOf`) from typo to contract violation with an owner. |

### OUT — decline explicitly, so nobody re-proposes them

| Concept | Why out |
|---|---|
| **CQRS** | No command side exists. All 36 MCP tools are `search_*`/`get_*`; the only HTTP write verb in the 13-client layer is a GraphQL query POST. Fuzzy-to-Fact is two GETs on one client against one base URL, and `SearchCandidate` is produced by a method on `Gene` itself for token budget. Naming it CQRS sends engineers hunting a sync/staleness problem that does not exist. |
| **Event sourcing / domain events** | A Graphiti episode is raw text re-interpreted by an LLM at ingestion. Replay is nondeterministic; there is no append-only log of *decided* facts. The word imports a replay guarantee the system cannot keep, and would reframe a write-path defect as a projection bug. |
| **Bounded context per MCP server** | The gateway mounts twelve thin pass-throughs over twelve foreign authorities with name prefixes. Calling them bounded contexts *licenses* twelve divergent "Gene" definitions exactly where drift is worst (five `CrossReferences` classes in one package, three ChEMBL spellings). If the phrase is used at all, the only defensible claim is that there are exactly **two** — read federation and memory graph — and the number is the entire content of the word. |
| **Repository / Unit of Work** | No referent. Nothing is persisted or change-tracked in the read layer; there is no transaction. (The only current hits are SpecKit boilerplate and an illustrative block in ADR-003 — which is itself `Status: Draft` while sitting in `accepted/`.) |
| **Command/event vocabulary in biosciences-temporal** | A straight-line DAG of nine read activities, no signals/queries/updates, result returned as a JSON string, no Graphiti reference in the tree. Verb-shaped names invite it; nothing mutates. Adopting here would name nothing and dilute the one place a write-side word has teeth. |
| **"Single writer" (DDD sense)** | **Hard prohibition.** ADR-006 already owns that exact phrase for git file ownership, cited as a Key Standard in `AGENTS.md`. Use **write-model owner** / **aggregate owner**, and amend ADR-006's gloss to say Single Writer means files, not the write model. One line of inoculation against the most likely failure of the whole proposal. |
| **"Ubiquitous language" as a program initiative** | The CURIE drift it targets already has an *executable* remedy that is simply not wired up (`tests/contract/registry.py`, 104 passing tests, no CI). Adding a word here substitutes vocabulary for a CI job. |

### Where I split the advocate and skeptic, and on what evidence

The advocate's third proposed term, **Command** ("a command must carry every field its invariant depends on"), rests on the strongest single piece of evidence in the dossier, which I verified first-hand this session:

- `biosciences-memory/src/biosciences_memory/server.py:325-331` — `add_memory(name, episode_body, group_id, source, source_description, uuid)`. **No `reference_time` parameter.**
- `biosciences-memory/src/biosciences_memory/services/queue_service.py:~145` — `reference_time=datetime.now(UTC)` is evaluated *inside* the worker closure, at dequeue time.

So ADR-MEM-001's mandated batch reference-time discipline is literally unexpressible through the tool's signature, and because the queue serializes per group, an N-episode batch gets N strictly increasing timestamps spaced by LLM latency — manufacturing exactly the artificial `valid_at` ordering the ADR blames.

**I adopt the rule and decline the word.** The rule changes a signature and gains a test; the word changes nothing here and materially raises the risk of "command" leaking into Temporal, where the advocate itself agrees it has no referent. Call it the **write-tool completeness rule**: *a write tool MUST accept every field its invariant depends on; a value an invariant depends on MUST NOT be synthesized at execution time.* That is one sentence, testable, and carries no priors about transactions or handlers.

I also endorse the skeptic's strongest structural point over the advocate's framing: **the same session's verification shows `server.py:197-216` silently leaves `custom_edge_types` and `custom_edge_type_map` as `None` whenever config supplies `entity_types`** — making ADR-MEM-001's *required* Lever 1 optional configuration. No noun fixes that. A guard clause and a test do. Adoption is therefore conditional: **if the three artifacts in §5's Clause 6 do not land, the vocabulary does not land either.**

---

## 2. The real problem

**Yes — unambiguously. The pattern-governance gap is the larger problem, and the DDD question is downstream of it.** Adopting *any* vocabulary into the current governance stack would produce dead text within a quarter.

The evidence is not speculative:

- **No CI.** `biosciences-mcp/.github/workflows/` contains only the Claude review bot. No pytest, no ruff, no pyright, no pre-commit. The 104 contract tests that encode ADR-001 run when a human or agent remembers. `psychology-mcp` — the smaller, newer repo — has a real blocking `ci.yml`.
- **The constitution is stale against its own cited authority** on three specific counts: cites ADR-001 v1.2 against accepted v1.4; says "22-key registry" against 23 keys in `registry.py`; mandates a fixed 10 req/s that ADR-007 superseded. Last touched by a file move (2026-03-03) while `src/` took 20 commits.
- **Principle VI is unsatisfiable**: it mandates using `deploy-cloud`, a skill that appears in exactly one file in the entire workspace — that constitution line itself.
- **The workspace CLAUDE.md points at four directories that do not exist** (`biosciences-program/.specify`, `/specs`, `/.claude/commands`, `/docs/adr/accepted`). ADRs moved twice in five weeks; two skills repos still name `biosciences-architecture` as governance upstream, two relocations out of date.
- **ADR-003 — the ADR mandating the entire SpecKit workflow — sits in `accepted/` marked `Status: Draft`,** and the review agent explicitly refuses to block on it for that reason.
- **ADR-007's status is asserted three different ways in three places** simultaneously (Accepted / Proposed / absent).
- **A governance statement in a project CLAUDE.md was simply wrong for months and nothing caught it**: `biosciences-program/CLAUDE.md:22` says biosciences-mcp is on the January 2026 dotted layout and is the upgrade target. The reverse is true — biosciences-mcp is Spec Kit **1.0.4** (2026-09-03, hyphenated skills incl. `speckit-converge`); psychology-mcp is the older **0.16.4**.

The pattern that makes this decision-ready: **the only governance layer in the workspace that actually binds is executable** — `tests/contract/registry.py` declaring itself "the contract the wire tests assert against," and the `pr-review` hook that makes reviewers provably read-only. **The only prose layer that is actually maintained** is `pr-review-standard/SKILL.md` §2 — a seven-row drift register, dated "verified 2026-09-03," mapping each stale statement to its current authority with named Linear expiries.

That is the whole answer in miniature: *executable contracts bind; dated drift registers survive; undated prose rots.* Adding DDD prose to a stack that cannot keep its own version numbers straight would teach, in psychology-mcp's own words, that every obligation is optional.

---

## 3. Mechanism comparison

| Mechanism | What it binds | When it is loaded | Enforceability | Drift risk |
|---|---|---|---|---|
| **Spec Kit constitution** (`.specify/memory/constitution.md`) | Normative MUST/SHOULD principles, project-wide | Read fresh at runtime by each command; **unconditional in `plan`, `**IF EXISTS**` in `specify`/`clarify`/`tasks`/`checklist`/`implement`**. No hook, no script loads it. | **Adjudication only.** `/speckit-analyze` treats a MUST conflict as automatically CRITICAL; `/speckit-converge` is the one place a MUST violation mechanically produces a tracked CRITICAL task. No validator, no runtime check. Complexity Tracking is a table the judged model fills in itself — zero mechanical validation in CLI, scripts, or tests. | **High, and silent.** One unconditional file, so it cannot scale — a 14th principle dilutes the other 13. No inheritance across repos (upstream says so explicitly). Our copy is 6+ months stale and has been wrong on three counts the whole time. |
| **Kiro steering** (`.kiro/steering/*.md`) | Project standards shaping generation; can embed live files via `#[[file:path]]` | **Routed**: `always` / `fileMatch`+glob / `manual` (`#name`) / `auto` (model-matched). Corpus can grow without proportional context cost. | **Generation is prompt-level**, same as Spec Kit. Real enforcement is adjacent: `PreToolUse` hooks whose non-zero exit *blocks* the write. Post-file events cannot block. **No command audits a spec against steering** — Analyze Requirements reasons only within one requirement set. | **Medium-low if `#[[file:]]` is used** (embeds the live file, not a transcription). Higher if prose-transcribed. Known defect: `fileMatch` reportedly does not work for *global* steering — keep conditional files workspace-scoped. *(Kiro rows rest partly on `[medium]`-confidence scout findings; see §6.)* |
| **CLAUDE.md** | Nothing, formally | Hierarchically, always, by directory scope | **None.** `pr-review-standard/SKILL.md` ranks it **last** of six precedence levels: "context only; never as a merge requirement on their own." | **Highest in this workspace, empirically.** 19–20 files, three currently teaching dotted `/speckit.*` commands that no longer exist, four dead paths, one inverted version claim that stood for months. |
| **ADRs** | The decision itself — clauses citable as authority | Never automatically. Read when an agent or human goes looking. | **Indirect but real**: cited by the PR template ("name the clauses you relied on"), by the `pr-review` agent's decision matrix, and encoded executably in `tests/contract/registry.py`. Enforcement is the test, not the ADR. | **Medium on content, high on location.** Content is versioned and reviewed. Location has moved twice in five weeks and every path reference to it is currently wrong somewhere. |
| **Skills** (`.claude/skills/*/SKILL.md`) | Procedure and reviewer conduct | Progressive disclosure — name+description always, body on match | **Strongest non-test layer here.** `pr-review` attaches a deterministic `PreToolUse` hook (`pr-review-guard.py`, 90-case self-test) that denies git/gh mutation. But it enforces *reviewer conduct*, not architectural clauses. | **Low where dated and owned** (the drift register). High where duplicated — byte-identical `scaffold-fastmcp` copies in two repos; 12 stale Graphiti copies deleted in one sweep. |
| *(baseline — not in the brief, but the actual floor)* **Contract tests** (`tests/contract/registry.py`) | Regexes, enums, key sets — anything machine-checkable | On every `pytest -m contract` | **Absolute.** A mismatch fails a test. Cannot drift silently. | **Zero on content; total on invocation** — no CI runs it. |
| *(baseline)* **CI workflow** | Whatever it runs | Every PR | **The only blocking gate that exists.** | N/A — and it does not exist in biosciences-mcp. |

Two structural facts to carry into §4: **neither toolchain has a cross-spec consistency checker** (`/speckit-analyze`, `/speckit-converge`, and Kiro's Analyze Requirements are each bounded to one feature), and **Spec Kit deliberately removed constitution→template propagation** (#3790, 0.14.4), demoting it to an opt-in preset whose own README argues against it at org scale. For a 13+-repo platform, cross-repo consistency is a CI job you write, not a toolchain feature you configure.

---

## 4. Recommended approach

**Design principle: the canonical thing is executable data plus stable IDs. Every toolchain file is a thin pointer with routing — never a generated copy, never a restatement.**

Reject both alternatives outright:
- **Generated projections** — upstream removed them for cause; this workspace has independently lived that failure (byte-identical skill copies, 12 stale Graphiti copies).
- **Shared-ADR-by-path referencing in its current form** — already proven to fail here; path references *are* the drift.

### Single source of truth, by artifact

| Artifact | Path | Authored / generated | Role |
|---|---|---|---|
| **Org principle set** | `biosciences-program/governance/principles.md` | Hand-authored, **6–8 principles max** | Each principle gets a permanent ID (`PRIN-01`…) that encodes no location. One sentence of obligation + one of rationale. Nothing else. This is the largest common denominator across constitution / steering / CLAUDE.md — do not try to express gates, severity, or inclusion modes here. |
| **ID → location resolver** | `biosciences-program/governance/index.json` | Hand-edited, CI-verified | Maps every `PRIN-*`, `ADR-*`, `ADR-MEM-*`, `ADR-PRG-*` ID to its current repo, path, version, and status. **The only file that knows paths.** When ADRs move a fourth time, you edit one file instead of auditing 1,138 markdown files. |
| **CURIE key registry** | `biosciences-mcp/tests/contract/registry.py` (canonical) → `biosciences-mcp/governance/registry.json` (generated) | **Generated by a test** that fails if the committed JSON differs | Already the only artifact in the workspace that cannot drift silently. The JSON export makes it consumable by non-Python toolchains. Nothing restates its key count in prose, ever — the constitution's "22-key registry" line is exactly the failure mode being eliminated. |
| **ID resolution check** | `biosciences-program/scripts/check_ids.py` + a CI job | New, ~20–40 lines | Walks every `.md` in the workspace, extracts `PRIN-\d+` / `ADR-(?:[A-Z]+-)?\d+` citations, fails if any does not resolve in `index.json`. This single check kills the bug class currently live in three CLAUDE.md files. |
| **Per-repo constitution** | `<repo>/.specify/memory/constitution.md` | Hand-authored per repo | Structure: (a) *"Adopts PRIN-01…PRIN-07 by ID; authority is `governance/index.json`, not this document"*; (b) repo-local principles; (c) a **dated verification line**; (d) a **drift register section** ported from `pr-review-standard/SKILL.md` §2. **Do not write one constitution for 13 repos** — Spec Kit has no inheritance and Kiro merges rather than overrides, so a shared body would silently fork. |
| **Constitution wording template** | `psychology-mcp/.specify/memory/constitution.md` (v1.6.1) | Existing — copy it | It has already solved by hand the two hardest sub-problems: it forbids naming a skill a session cannot actually invoke (which is precisely biosciences' dead Principle VI), and it forbids a vacuous pass ("report each obligation as **checked**, **deferred**, or **uncheckable**, never as a blanket pass — a vacuous pass is worse than a recorded deferral"). |
| **Kiro steering** *(deferred — phase 2)* | `<repo>/.kiro/steering/*.md`, **workspace-scoped only** | Hand-authored, thin | `fileMatch`-routed files that cite principle IDs and embed `#[[file:governance/registry.json]]` rather than transcribing it. One domain per file. Global `~/.kiro/steering/` gets only always-on cross-cutting text — never `fileMatch` (documented defect). |
| **CLAUDE.md** | every repo | Hand-authored | **Demoted to context by rule.** Must not restate counts, versions, or paths that `index.json` owns; must cite IDs. Add the precedence sentence already in `pr-review-standard/SKILL.md`: *a governing document without a dated verification line is context, not authority.* |
| **CI** | `biosciences-mcp/.github/workflows/ci.yml` | **Copy `psychology-mcp/.github/workflows/ci.yml`** | ruff check, ruff format --check, pyright, `pytest -m unit`, `pytest -m contract`, `fastmcp inspect`, tool-surface assertion. |

### What generates what — and what deliberately does not

- `registry.py` → `registry.json`. **Test-enforced.** The only generator in the design.
- Nothing generates prose. No constitution generator, no `constitution-sync` preset, no CLAUDE.md sync. Every prose file is hand-owned and cites IDs.
- `index.json` is hand-edited but **CI-verified**, which is the inverse of today: paths are cheap to change and expensive to get wrong.

### Sequencing (four weeks, one maintainer)

1. **Copy psychology-mcp's `ci.yml` into biosciences-mcp.** Highest leverage, lowest cost. Any governance scheme layered on a repo with no CI degrades to advice.
2. **Add the `registry.py` → `registry.json` export test.**
3. **Create `governance/index.json` + `check_ids.py` + CI job.** Fix the four dead paths and the inverted Spec Kit claim as the first red build.
4. **Rewrite the biosciences-mcp constitution** to ID'd principles citing the registry path, clear the seven drift-register rows, fix Principle VI (install the skill or amend the principle — do not leave it standing and unmet), and resolve ADR-003's Draft-in-accepted status.
5. **Only then** evaluate Kiro steering.

**Steps 1–3 are worth more than 4–5 and depend on no toolchain decision.** Do not gate them on ADR-PRG-001.

---

## 5. Draft outline for ADR-PRG-001

**Placement** (per `biosciences-program/docs/adr/README.md`, placement rule decided 2026-09-02): program scope = cross-repo process/ownership → `biosciences-program/docs/adr/`, series `ADR-PRG-NNN`. This is the first, so it also establishes the `proposed/` → `accepted/` subdirectory convention already used by the `MEM-` series.

**File:** `biosciences-program/docs/adr/proposed/adr-prg-001-v0.1.md`
**Title:** *Pattern Governance: Stable IDs, Executable Contracts, and Pointer-Not-Copy Toolchain Files*
**Companion:** `ADR-MEM-002` (project scope, `biosciences-memory`) carries the write-model changes; ADR-PRG-001 must not.

### Section-by-section

**0. Front matter** — Status: Proposed · Date: 2026-09-12 · Owner: Program Director · Supersedes: none · Linear: new AGE issue · Verified-against date.

**1. Context**
- The originating question and its verified answer (zero DDD/CQRS references across 1,138 markdown files; the only "repository pattern" hits are SpecKit placeholders and an ADR-003 illustration, and ADR-003 is `Status: Draft` inside `accepted/`).
- Four governance layers exist (ADRs, constitution, CLAUDE.md, skills); only executable ones bind.
- Evidence of rot, itemized with paths: no CI in biosciences-mcp; constitution stale on ADR version, key count, and rate constant; unsatisfiable Principle VI; four dead paths in the workspace CLAUDE.md; two skills repos naming a two-relocations-stale upstream; ADR-007 status asserted three ways; `biosciences-program/CLAUDE.md:22` inverted on Spec Kit vintage.
- Toolchain constraints: Spec Kit has no constitution inheritance and removed template propagation; neither Spec Kit nor Kiro has a cross-spec consistency checker.

**2. Decision** *(written out)*

> **Governance artifacts in this organization are addressed by stable ID, enforced by executable contract, and consumed by pointer.**
>
> 1. Every normative statement carries a permanent identifier that encodes no location — `PRIN-NN` for org principles, `ADR-NNN` for platform decisions, `ADR-<PREFIX>-NNN` for project and program decisions. Identifiers are assigned once and never reused. No document in any repository may cite a governance artifact by file path; it cites the identifier.
> 2. Exactly one machine-readable resolver, `biosciences-program/governance/index.json`, maps identifier → repository, path, version, and status. It is the sole artifact that knows where anything lives, and relocating an artifact is an edit to that one file.
> 3. Any obligation expressible as a regex, schema, enum, count, or version is canonically held as **executable data with a passing test**, and prose layers cite it as authority rather than restating it. `biosciences-mcp/tests/contract/registry.py` is the canonical cross-reference key registry; a test exports it to `governance/registry.json` and fails if the committed export differs.
> 4. Every repository that ships code runs a blocking CI workflow that executes its lint, type, unit, and contract suites on every pull request. **A governance obligation that no CI job can fail is advisory, and must be labelled as such where it is written.**
> 5. Toolchain context files — Spec Kit constitutions, Kiro steering files, and CLAUDE.md — are **pointers with routing, never generated copies and never restatements**. Each repository authors its own constitution, which adopts org principles by identifier, adds only repository-local principles, and carries (a) a dated verification line and (b) a drift register naming each statement known to be stale, its current authority, and its expiry condition. **A governing document without a dated verification line is context, not authority**, and may not be cited as a merge requirement on its own.
> 6. **The organization does not adopt Domain-Driven Design or CQRS as a platform vocabulary.** CQRS, event sourcing, domain events, repository/unit-of-work, bounded-context-per-server, and command/event vocabulary in `biosciences-temporal` are declined on the evidence that each lacks a referent in a read-only federation whose sole write model is `biosciences-memory`. Two terms — **Aggregate** (bound to `group_id`) and **Anti-Corruption Layer** (bound to the biosciences-mcp → biosciences-memory boundary) — are adopted **within `biosciences-memory` only**, under `ADR-MEM-002`, and only if each lands as a signature change, a passing test, and a named owner. **The phrase "single writer" retains its ADR-006 meaning — git file ownership — throughout the organization; the write-model concept is named "write-model owner" and never "single writer."**
> 7. A write tool MUST accept every field its invariants depend on, and MUST NOT synthesize an invariant-bearing value at execution time.

**3. Scope and non-goals**
- Binds: all repos in the org (program scope per the placement rule).
- Non-goals: does not amend ADR-001/-006/-007; does not choose between Spec Kit and Kiro; does not mandate Kiro adoption; does not itself change any code in `biosciences-memory` (→ ADR-MEM-002).

**4. Consequences**
- *Positive:* relocation churn becomes a red build; counts and versions stop being restated; a small maintainer can hold 19 repos.
- *Negative / cost:* `index.json` is a new artifact to maintain; every repo owes a constitution rewrite; `check_ids.py` will initially fail loudly across several CLAUDE.md files (this is the point).
- *Risk:* another dead obligation if Clause 4's CI does not land. **Mitigation: Clause 4 is sequenced first and is the acceptance gate for this ADR.**

**5. Alternatives considered and rejected**
- (a) Generated projections / `constitution-sync` — rejected: upstream removed propagation for cause (#3790, 0.14.4) and the preset's own README names three failure modes; this workspace has already lived the duplicate-copy failure twice.
- (b) One shared constitution across all repos — rejected: no Spec Kit inheritance; Kiro merges rather than overrides (and its own docs contradict each other on conflict resolution); a shared body forks silently.
- (c) Better discipline on path references — rejected: empirically falsified twice in five weeks.
- (d) Adopt DDD platform-wide — rejected on referent grounds (§1) and on the "Single Writer" collision.
- (e) Do nothing — rejected: the current state already produces confidently wrong agent output (`TARGETS` vs `TargetOf`).

**6. Compliance and verification**
- Acceptance gates, each with a Linear issue: CI green in biosciences-mcp; `registry.json` export test passing; `check_ids.py` green across the workspace; biosciences-mcp constitution rewritten with zero outstanding drift rows; ADR-003's Draft-in-`accepted/` resolved.
- Reporting rule, adopted verbatim from psychology-mcp: each obligation reports **checked / deferred / uncheckable**, never a blanket pass.

**7. Review and expiry**
- Re-verified quarterly or on any repository relocation, whichever comes first; verification date stamped in the document and mirrored into `index.json`.

---

## 6. What we do not know

| Gap | Confidence of what we have | What would resolve it |
|---|---|---|
| **Whether `biosciences-mcp`'s contract tests pin the CURIE spellings** — i.e. whether the `CHEMBL:` / `CHEMBL` and `DrugBank:` divergences are frozen contracts or unnoticed drift | Scout did not read `tests/contract/`; the divergence itself is file-cited and high-confidence | Read `biosciences-mcp/tests/contract/` and diff its assertions against ADR-001 Appendix A. ~30 min. **Blocks §4's registry-as-canonical claim if the tests disagree with the ADR.** |
| **Whether ADR-MEM-001's numbers still hold** (526 impossible lifetimes, 85% sole-edge) | Measured 2026-08-24; ADR still in `proposed/`; Lever 1/2 A/B validation may not have run | Re-run the Cypher against the live graph. **This is the load-bearing evidence for adopting "Aggregate" at all** — if the numbers have moved, revisit §1. |
| **Which Graphiti actually runs in production** — PyPI `graphiti-core==0.24.3` vs the org fork at `/home/donbr/open-biosciences/graphiti` | Unresolved from the checkout | Inspect the deployed environment. Determines whether ADR-MEM-001's Lever 3 (fork) is available. |
| **Whether the `pr-review` agent team has ever actually blocked a PR** | Agents, hook, and standard were read; merged PR threads (#8–#11) were not | Fetch the review threads. If it has never blocked, the "one maintained prose layer" claim weakens and §4 leans harder on CI. |
| **Whether biosciences-mcp's missing CI is deliberate** (Claude bot deemed sufficient) or an oversight | No ADR or CLAUDE.md statement explains it | Ask the MCP Platform Engineer before opening the PR. Cheap; avoids relitigating step 1. |
| **Whether the 13 specs were re-validated after ADR-001 went v1.2 → v1.4** | `compliance-analysis-*.md` files exist, unread | Read them; determine which ADR version they graded against. |
| **Graph-vs-file drift** — the Graphiti priming namespace may assert governance facts contradicting the files | Unmeasured; only filesystem artifacts were inventoried | Query `open-biosciences-migration-2026-priming` and diff against `index.json` once it exists. A fifth, invisible governance layer is a real possibility. |
| **Whether any repo besides biosciences-mcp and psychology-mcp has a `.specify/`** | Two confirmed present, one confirmed absent; 19 repos not exhaustively scanned | `find . -maxdepth 3 -name .specify`. One command. Sizes the constitution-rewrite work in §4 step 4. |
| **Kiro `fileMatchPattern` semantics** — what triggers a match (files read/written this turn? open in editor? both?) and the glob dialect | **LOW — docs are non-normative; examples only.** | Empirical test in Kiro before any `.kiro/steering/` work. §4 defers Kiro to phase 2 partly for this reason. |
| **Kiro global-`fileMatch` defect** | **LOW — two GitHub issues (#9176, #6171), unverified** | Same empirical test. §4 already routes around it by keeping conditional steering workspace-scoped. |
| **Kiro steering merge vs override across global/workspace scope** | **LOW — Kiro's own docs contradict each other** | Empirical test. Affects whether a two-tier steering design is safe. |
| **Community Spec Kit extensions** (`charter`, `arch-governance`, `blueprint-index`, `closed-vocabulary`) | **LOW — catalog descriptions only, unverified against source** | Read the repos before depending on any. **`charter` is the most relevant** (constitution inheritance from shared fragments) and would partly replace §4's hand-rolled `index.json` if it is real. |
| **Whether agents actually honor Spec Kit's prose gates** | Established: no mechanical enforcement anywhere in CLI, scripts, or tests. **Not established: model compliance rates.** | Not resolvable cheaply. §4's design assumes they do not, which is the safe direction. |
| **Whether `biosciences-mcp-edge` and `psychology-mcp` add further CURIE-form divergences** | Not examined; ADR-007 §2(f) binds them to the same base-client contract | Grep their `validate_id` regexes against `registry.py`. Feeds §4's registry export. |
| **Whether ADR-007's consolidation into `clients/base.py` has landed** | `hgnc.py:62` still defines its own `_rate_limited_get`, suggesting eleven copies remain | Read `clients/base.py`. Affects nothing in this recommendation, but it is a live drift-register row (AGE-698). |

**Confidence summary:** every claim in §1 and §2 rests on `[high]`, file-and-line-cited evidence, and the two most load-bearing (the missing `reference_time` and the silently-dropped edge ontology) were re-verified first-hand in this session. §3's Kiro rows and §6's Kiro entries rest on `[medium]` scout evidence and contradictory vendor docs — treat them as directional, not decisive, and do not commit to a Kiro rollout before the empirical tests above.