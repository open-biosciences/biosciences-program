# Version, Protocol and Identifier-Authority Tradeoffs — biosciences-mcp

**For:** Program Director, Open Biosciences
**Date:** 2026-09-12
**Scope:** what to add to the six-item queue, and what to refuse

---

## 1. Bottom line

Add three items. Decline the rest.

1. **Pin the tool-name contract today** (~1h, on the current 2.14.5): a test asserting the exact 34 gateway tool names. It is the guard for the highest-risk change ahead and costs nothing now.
2. **FastMCP 2 → 3** (~half a day; effort is [inference]). One file — `servers/gateway.py`. No SDK move, no httpx2, no snake_case. Buys seven months of security hardening and retires a pin that is no longer defensible. Stage behind a preview deployment: the pin **is** the deploy.
3. **Consolidate identifier patterns into one internal module** (~2–3 days; [inference]). This is not new scope — it absorbs the queued CURIE-validation-hook item, closes the unowned "wire `CROSS_REF_PATTERNS` as a validator" gap, and is the prerequisite that makes AGE-700 answerable rather than guesswork.

Explicitly decline: FastMCP 4 this quarter; MRTR, the tasks extension, `subscriptions/listen`, `x-mcp-header`; a Resources migration; TRAPI 2.0 alignment; Bioregistry / Identifiers.org / Node Normalizer as a *runtime* authority; and any response to the competitive landscape beyond one registry submission.

Nearly free, worth doing in passing: `readOnlyHint`/`openWorldHint` on the 34 tools; one `meta.yaml` PR to the BioContextAI Registry.

---

## 2. Version currency: fastmcp and mcp SDK

### Three corrections to the brief before the table

- **173 contract tests, not 104** (933 repo-wide). The parametrized surface grew since that figure was recorded. Any plan costed on 104 is costed on the wrong number.
- **12 mounts, not 13.** DrugBank is excluded from the gateway by a hardcoded comment (commercial key). The gateway is 12 servers / 34 tools.
- The FastMCP API surface in this repo is **13 × `FastMCP(name)`, 36 `@mcp.tool`, 12 `mount()`, `mcp.run()`, `Client(server)` in tests** — and nothing else. Zero Context, lifespan, resources, prompts, middleware, auth, sampling, `mcp.types` imports. [inference, from grep over `src/`, `tests/`, `scripts/`] That single fact collapses the migration from "audit 13 servers" to "edit one file," and it is the load-bearing premise of everything below.

### Decision table

| | **Stay on 2.14.x** | **Go to 3.4.7** | **Go to 4.0.3** |
|---|---|---|---|
| **What breaks** | Nothing today. | `gateway.py` only: `mount(prefix=)` → `mount(namespace=)`; `as_proxy=` removed; **`tool_names=` composition order inverts**. Server modules: zero edits. | Same gateway change, plus mcp SDK 1.26.0 → 2.x, `httpx2>=2.5.0` enters the closure, MCP model fields go snake_case (Python attributes only — the wire stays camelCase). Repo's 17 `httpx` call sites are all in `clients/` and untouched. |
| **What is gained** | — | Seven months of security hardening (v3.4.3 SSRF, v3.4.5 JWKS, diskcache CVE-2025-69872 dropped in 3.0.0). Component versioning. Tag-based enable/disable. Tool Search transform. | Protocol 2026-07-28 with per-connection version negotiation. `cache_ttl`/`cache_scope` (two constructor kwargs). Sessionless scaling behind a plain LB. `Mcp-Method`/`Mcp-Name` edge routing. |
| **Effort (13 servers + gateway + 173 contract tests)** | 0 | **~half a day** [inference]. Dominated by one contract run, not authorship. In-memory `Client(server)` pattern and snake_case `CallToolResult` fields survive unchanged. | **~1–2 days** [inference]. Dominated by re-validating 173 integration-marked tests against 12 live public APIs. Prerequisite: audit downstream consumers (deepagents `shared/mcp.py`, temporal activities, bio-research `.mcp.json`) — **not yet done**. |
| **Risk of staying** | No security backports on a predictable schedule for a public-facing gateway. Growing distance from the SDK. | Low. `mcp>=1.24.0,<2.0` applies to *all* of 3.x, so 3.4.7 is a genuine resting point, not a waypoint. | n/a |

### Is the `<3.0` pin still defensible?

**No, as written.** 3.0.0 went GA 2026-02-18; 4.0.3 is stable as of 2026-09-05 (PyPI upload timestamps, [upstream-changelog]). The pin was set against 3.0.0b1 and no longer describes reality.

Worse, it never did what it looks like it does. FastMCP's own releases policy states that "minor versions may include breaking changes when necessary for the ecosystem's evolution" [vendor-docs]. A major-version ceiling is therefore not a compatibility guarantee under this project's versioning contract — `>=2.14.1,<3.0` held only because no 2.15 ever shipped. **Every FastMCP pin in this org should be an exact floor plus a MINOR ceiling** (`>=3.4.7,<3.5`), not a major ceiling. That is a standing convention worth writing down once.

### Support posture of the 2.x line

Backport-only life support, inferred from cadence, not declared: 2.14.5 (2026-02-03), 2.14.6 (2026-03-27), 2.14.7 (2026-04-13), then five months of silence. A `release/2.x` maintenance branch exists in the documented policy; **no EOL date is published for either 2.x or 3.x** [vendor-docs]. "Five months of silence" is an observation about release cadence, not a stated deprecation — do not put it in an ADR as one.

### The one dangerous change

`tool_names=` survives into FastMCP 4 but its composition with the prefix inverts. Under 2.14.5 it *overrides* the prefixed name (verified empirically: the gateway enumerates exactly 34 single-prefixed names). Under 4.0.3 the source comment reads *"Apply tool renames first (scoped to this provider), then namespace"* and `Namespace._transform_name` unconditionally returns `f'{prefix}_{name}'` — so a naive port yields `hgnc_hgnc_search_genes` across all 34 tools. The server starts fine; every downstream binding silently breaks.

The correct port is to **delete the `tool_names` dicts entirely** — `namespace="hgnc"` alone reproduces today's names, because those dicts only ever restated prefix + original name.

**This is read from source, not executed** [inference-from-source]. Prove it in a throwaway venv before touching `gateway.py`, and land the 34-name assertion on 2.14.5 *first* so the test exists before the change it guards.

### Sequencing verdict

Land 2 → 3 now; **schedule 4.x, do not queue it.** Three unresolved gates sit in front of 4.x, none of them code: (a) whether Prefect Horizon / FastMCP Cloud currently builds FastMCP 4 and still honours the v1 `fastmcp.json` schema — the deployment docs state no supported version range; (b) the downstream consumer audit across five repos; (c) `tests/contract/test_wire_contracts.py::test_entity_has_no_null_values` as the canary for the ADR-001 §4 omit-null guarantee, whose text-block serialization path in v4 was **not traced**. Independent corroboration that stopping at 3.x is reasonable: `biothings/smartapi-mcp` pins `fastmcp>=3.3.1,<4` with the explicit note that "fastmcp 4.x is not yet supported — it moves to the MCP 2.x SDK and httpx2."

Two governance items ride along with 2 → 3: **ADR-004 reasons from the premise that direct mounting runs no lifespan**, which v4 reverses; and the deployment target resolves its FastMCP major from `pyproject.toml`, so changing the pin changes what is deployed at `biosciences-mcp.fastmcp.app` — the endpoint five repos point at.

---

## 3. Protocol revision 2026-07-28

Installed SDK targets `LATEST_PROTOCOL_VERSION = "2025-11-25"` (measured in the venv). The repo is **one wire revision behind and two tooling majors behind**. The wire gap is smaller than the dependency gap, and essentially all of it arrives free with the FastMCP upgrade — there are no handlers to write.

### Adopt

| Item | Why | Cost |
|---|---|---|
| **`readOnlyHint: true` + `openWorldHint: true` on all 34 tools** | By omitting annotations the federation currently advertises the worst-case defaults: `readOnlyHint` **false**, `destructiveHint` **true**, `idempotentHint` **false**. A host applying confirmation policy by annotation will prompt for approval on `hgnc_get_gene`. Every tool is a GET over a public API. [normative-spec] | Trivial. Confirm 2.14.5 accepts the annotations kwarg [inference — not verified for that version]. |
| **Deterministic `tools/list` ordering** | New SHOULD, aimed at client caching and LLM prompt-cache hit rates. The 12 sequential `mount()` calls are already deterministic in source order — but incidentally, not by contract. | Trivial (a contract test). |
| **`cache_ttl` / `cache_scope="public"` on the gateway** | `FastMCP("...", cache_ttl=300, cache_scope="public")` is the whole change. Default scope is `"private"`; forgetting the second kwarg silently forfeits the win. | Trivial — **but gated on FastMCP 4.** Defer with 4.x. |
| **`server/discover` `instructions`** | An unexploited slot and the natural home for the Fuzzy-to-Fact protocol statement, currently repeated across 34 tool descriptions. Cacheable, so free per call. | Small; arrives with 4.x. |
| **A negative capability assertion in ADR-001** | "This platform declares no roots, sampling, logging or elicitation capability." Stops a future contributor adding one. | Trivial. |

### Decline

**MRTR** — normative, not a proposal, and inert here: no tool needs mid-call user input. Adopting it would import a signing-key lifecycle, principal binding and a replay window into a federation whose only server-side secrets are upstream API keys (the spec imposes a **MUST** on integrity-protecting `requestState`). Additionally, `anthropics/claude-ai-mcp` issue #1027 (open, unassigned) reports that claude.ai fails the state-only `InputRequiredResult` case outright — so an MRTR tool would work in Claude Code and SDK clients and break in claude.ai.

**`subscriptions/listen`, the tasks extension, `resources/subscribe`** — all for stateful or long-running servers. The tool set is fixed at 34; every call is a sub-second upstream GET. The caching page explicitly blesses TTL-only: a server MAY provide `ttlMs` without advertising `listChanged`.

**`x-mcp-header`** — no routing dimension worth a header, and a malformed annotation causes clients to **MUST** exclude the tool from `tools/list`. High blast radius, zero payoff. The `$ref` exclusion would bite the union-typed schemas anyway.

**SEP-2577 deprecations** cost nothing: Roots, Sampling and Logging are unused; earliest removal is "first revision released on or after 2027-07-28." **Correction to the brief: DCR is deprecated by PR #2858, not SEP-2577.** Do not let that error reach an ADR — it would misattribute the CIMD migration.

### Resources vs Tools — the specific answer

The technical case is narrower than it looks, and it rests on one asymmetry: **protocol-level caching is available only on `server/discover`, `tools/list`, `prompts/list`, `resources/*`. `tools/call` is not on the list.** A tool result can never be client-cached, no matter how static `HGNC:1100` is. That asymmetry — not any Resources aesthetic — is the entire argument.

Against a wholesale migration, four hard constraints [normative-spec]:
- `resources/read` is **application-driven**; tools are **model-controlled**. A registry exposed only as a resource is invisible to the agent in most hosts.
- `resources/read` takes only `uri`. `slim=` has nowhere to live, and the fuzzy half of Fuzzy-to-Fact cannot be a resource at all.
- `resources/read` supports caching but **not** pagination. `PaginationEnvelope` does not map.
- Declaring the `resources` capability adds a listChanged/subscribe surface to reason about.

The defensible middle — keep all 34 tools, add ~3–6 resources for genuinely static artifacts (the 23-key `cross_references` registry, the CURIE→`validate_id` table, the server inventory) with long public TTLs, referenced from tool results via `resource_link` — is real but **not worth queuing now**, for two reasons: it depends on hosts rendering `resource_link` usefully, which is **not investigated**; and `tools/list` request volume against the gateway is **unmeasured**, so the benefit is directionally right and quantitatively unknown. Revisit after 4.x lands and after measuring cold-start frequency.

### Route around

**Structured output.** SEP-2106 loosened `inputSchema`/`outputSchema` to any JSON Schema 2020-12 and `structuredContent` to any JSON value — which is exactly this repo's `X | ErrorEnvelope` anyOf-at-root shape. But FastMCP 4.0.3 still defaults `wrap_non_object_output_schema=True` and still requires dict `structured_content`, so **nothing changes today**. The action item is defensive: add a contract test asserting the `{"result": ...}` wrapper, so a future FastMCP minor legally dropping it cannot land silently.

**Two things to verify with one contract assertion each, because both are load-bearing and neither is confirmed:** does the federation set `isError: true` on the wire when returning an `ErrorEnvelope`? (If it returns a success result containing an error object, the spec's self-correction path never engages.) And does FastMCP 2.14.5 emit a usable `outputSchema` for the union return types on all 34 tools?

**Error codes.** `-32020`–`-32099` is now reserved to the spec; `-32000`–`-32019` is legacy and closed to new allocation. The current design returns `UNRESOLVED_ENTITY` inside a tool result, which is the correct side of the line — worth stating in ADR-001 as a **decision**, not an accident. The spec's Tool Execution Error guidance ("actionable feedback that language models can use to self-correct") has independently converged on the ErrorEnvelope-with-recovery-hints design. Cite that as spec confirmation, not industry precedent.

---

## 4. Prior art: what held, what moved

`docs/prior-art-api-patterns.md` v1.5.0 (2026-01-24) has aged unevenly: the historical grounding is solid, the novelty claims are wrong, and the landscape survey is missing the three systems that matter most.

| Jan-2026 claim | Verdict | Evidence |
|---|---|---|
| **STRING verb taxonomy** (§1) — `/get_string_ids` is really a resolver | **HOLDS, strengthened** | No STRING 2026 release; `/api/tsv/version` → `12.0`; the 23 REST methods unchanged. The Mering lab now ships `meringlab/string-mcp` at `mcp.string-db.org` whose first tool is literally `string_resolve_proteins`. The January reading was an inference from REST naming; it is now the vendor's own MCP design. [vendor-docs, probed live 2026-09-12] |
| **TRAPI alignment** (§9) | **MOVED** | 1.5.0 → 1.6.0-beta → 2.0.0-beta (2026-04-02) → 2.0.0-beta2 (**2026-09-10, two days ago**). Breaking: constraints refactored, bindings `id`→`ids` with `attributes` removed, KL/AT promoted to required Edge properties, `null` banned. **No stable 2.0.0.** [normative-spec, ChangeLog.md] |
| **BioThings Explorer federation** (§4) | **MOVED** | Not archived, but last push 2026-03-31, 113 open issues, 15 stars — while `NameResolutionAPI`, `mygene.info`, `biothings.api` in the same org ship daily. Cite Callaghan 2023 for the pattern; stop calling BTE "the reference implementation." Compounding: NCATS' own `translator-ingests` is consolidating a Koza/KGX "Tier 1" graph, which undercuts using Translator as evidence that *federation* is where the field is heading. [upstream-changelog] |
| **Biolink** (§3/§4.5) | **HOLDS** | v4.4.4 (2026-08-10), commits this week, no v5, no change to identifier handling. New and useful: PR #1772 adds first-class STRING confidence scores, PR #1771 adds StringDB-ingest association representation — which makes the §9 evidence-channel recommendation *cheaper* than it was. [upstream-changelog] |
| **CURIE transformation pattern** (§4.5) | **HOLDS** | W3C CURIE 1.0, Bioregistry, identifiers.org unchanged; Babel pushed 2026-09-11, NameResolutionAPI 2026-09-12. No amendment needed. [vendor-docs] |
| **`slim=` as novel contribution** (§6, §7.1, §7.2) | **SUPERSEDED — and it was never true** | Anthropic's "Writing effective tools for agents" (**2025-09-11, four months before v1.5.0**) prescribes a `response_format: concise\|detailed` enum plus pagination/filtering/truncation, *and* "prompt-engineer your error responses to clearly communicate specific and actionable improvements" — which also covers the recovery-hints claim. By today, `meringlab/string-mcp`, BioMCP, ToolUniverse and `smartapi-mcp` all budget tokens. TRAPI 2.0 stripped fields for the same reason. **[blog — see §7]** |

### What is new since January that would change a decision

**BioContextAI Registry** (Nature Biotechnology 2025, `s41587-025-02900-9`) — 71 schema-validated biomedical MCP servers, CI-enforced `meta.yaml`, published as the discovery layer for this field. Peers and competitors already listed: `genomoncology-biomcp`, `meringlab-string-mcp`, `mims-harvard-ToolUniverse`, EBI-adjacent servers. **biosciences-mcp is not listed**, and it clears the bar today (MIT, public APIs, typed tools, tests). One `meta.yaml` PR is the highest-leverage, lowest-cost action available. *Caveat: the bar requires free academic access to every exposed service — DrugBank's licensing was not investigated, but DrugBank is already excluded from the gateway.*

**ToolUniverse** (Harvard/Zitnik; arXiv 2509.23426; 1,680 stars; Nature Methods feature) — the scale reference, with its own "AI-Tool Interaction Protocol." **BioMCP** (`genomoncology/biomcp`, 630 stars, pushed today) — a direct competitor that independently implements two-phase search→get, identifier abstraction, progressive disclosure and next-command hints across overlapping sources.

**Strategic consequence, and it is the substantive edit AGE-700 should carry:** three of the five headline "novel contributions" exist in a 630-star competitor. The remaining differentiator is narrow but real and defensible — **the *enforced* strict phase** (raw string → `UNRESOLVED_ENTITY`, contract-tested) **and the 23-key `cross_references` registry**. BioMCP accepts `BRAF` everywhere; ToolUniverse's standard requires only a typed schema. §6/§7 must be rewritten from "our innovation" to "our disciplined, contract-tested implementation of a now-standard pattern." Say the differentiator explicitly or the project reads as a small ToolUniverse.

**One cheap steal:** `string-mcp` returns truncation *metadata* (`notes`, `truncated`, `truncated_categories`, `omitted_categories`) alongside truncated payloads. This project's `slim` mode silently drops fields. That is a real quality gap, and it is a strictly better version of the pattern.

**MCPmed** (Briefings in Bioinformatics, 2026-02-23) — the closest thing to a *published* convention for exactly what this is (EDAM/Bioschemas-tagged tool schemas). Cite it; do not adopt it. Its reference repos have 1–12 stars and mostly stopped in July 2025.

**One decision this forces:** with `mcp.string-db.org` live and vendor-operated, the STRING server in this federation is now a third-party duplicate. Make the build-vs-defer call deliberately rather than by default. No other upstream ships an official MCP server — per EMBL (2025-12-02), MCP is "something we are exploring across the EMBL-EBI services" — so there is no wave about to obsolete the rest of the federation.

---

## 5. The CURIE authority decision

**Recommendation: no external runtime authority. Consolidate internally first, then vendor a pinned Bioregistry snapshot as a CI oracle.**

### Why a direct swap fails

Measured empirically against `bioregistry 0.14.6` on 2026-09-12, across the 23 registry keys:

- **8 drop-in**, **7 silently widen**, **7 break**, **1 has no pattern at all** (`ucsc`).
- The 7 breaks split two ways: **5 are a layer mismatch** — Bioregistry patterns describe the *bare local ID* (`^\d{1,5}$` for hgnc) while this project stores the *prefixed* form (`HGNC:1100`) — and **2 are genuine concept mismatches** (Bioregistry's `string` is a UniProt-accession resource, not `9606.ENSP...`; its `stitch` is a flat InChIKey).
- **The widening is the more dangerous half.** Bioregistry's `ensembl` pattern accepts `BRCA1` and `hello` (its fourth alternative matches essentially any alphanumeric token). Adopting it would make `BRCA1` a syntactically valid Ensembl gene ID and **disarm the exact ADR-001 §3 rejection the Fuzzy-to-Fact guarantee depends on.** The identical pattern is served by Identifiers.org, so this is inherited from MIRIAM, not a curation lapse.
- At the tool-input layer Bioregistry is unusable as-is: `is_valid_curie()` returns **False for all 10** of this project's CURIEs (it emits canonical lowercase prefixes), and `normalize_prefix()` returns `None` for **`IUPHAR`** and **`WP`**. Bridging would require a hand-maintained project→Bioregistry prefix map — a new hand-maintained artifact, i.e. the problem you were deleting.

### Two objections in the brief dissolve; one alternative is answering a different question

- **No network dependency.** The 7.3 MB wheel ships `bioregistry.json` offline. `get_pattern`/`is_valid_identifier` read bundled JSON. *But this cuts the other way:* if the data is static and offline anyway, a pinned committed snapshot gets the same benefit without adding `pydantic[email]`, `curies`, `sssom-pydantic` and `urllib3` to the runtime closure of every server.
- **Identifiers.org is not a second opinion.** Bioregistry imports and aligns MIRIAM; 16 of 23 patterns are byte-identical, and Identifiers.org is staler (12 of 20 records last modified 2019-06-11) with no record for `omim`, `orpha` or `ucsc`. Rules out any cross-check-two-registries design.
- **Node Normalizer answers a different question.** `HGNC:NOTANID`, `FOO:bar` and `HGNC:99999999` all return identical `null` — it cannot distinguish malformed from valid-but-nonexistent, which is precisely the distinction the ErrorEnvelope encodes. Self-hosting is ~200 GB of Redis. Keep it in mind for a *separate, optional* `cross_references` enrichment feature, where a 0.6 s hop and 20 equivalent IDs is the payload you actually want.

### The precedent argues against regex-as-authority entirely

STRING does **no** syntax validation: `get_string_ids?identifiers=!!NOT_A_GENE!!` returns HTTP 200 with a confident fuzzy match to ISL1. Biolink — the type system for all of Translator — declares **67 `id_prefixes` lists and exactly one regex.** BTE delegates to Node Normalizer and flags unresolvable IDs. **The federation-scale precedent is: validate the prefix (a closed vocabulary — exactly what Bioregistry is authoritative for) and let resolution handle the local part.**

### The finding that reframes the whole item

The drift that prompted this is not an authority problem. **There are ~20 pattern declaration sites in `src/`, not three or four** — `tests/contract/registry.py`, `models/cross_references.py`, `models/ensembl.py` (12 constants + 5 inline `Field(pattern=...)`), `.claude/hooks/validate-curie.py` (whose own docstring admits it is stale), plus independent regexes across a dozen `models/*.py` and six `clients/*.py`, including a bare `re.match(r'^[NX][MR]_\d+$')` inline at `clients/entrez.py:435`.

And **the "drifted" refseq copy is the one closer to upstream truth.** `registry.py:54` and `cross_references.py:21` say `^[NX][MR]_\d+$`; `models/ensembl.py:28` says `^[NX][MPRG]_\d+(\.\d+)?$`; RefSeq genuinely issues `NP_` protein accessions and versioned accessions. **The declared contract is simply wrong about RefSeq.** That is a live defect for AGE-700, not a style question — and Bioregistry, used as an oracle, would have caught it.

Scope warning: `models/target.py`'s `DISEASE_ID_PATTERN` (`^(EFO|MONDO|Orphanet|HP|DOID|OTAR)_\d+$`, duplicated verbatim in `clients/opentargets.py`) uses a **third** spelling of the same disease identifiers, distinct from both the registry's colon form and Bioregistry's bare form. The consolidation is likely larger than 23 keys.

### The recommendation, in three steps

1. **One owner.** `src/biosciences_mcp/identifiers.py` holding both layers explicitly: `XREF_PATTERNS` (bare-LUI, for `cross_references`) and `CURIE_PATTERNS` (prefixed, for tool inputs). Everything imports. The validation hook *generates* its table instead of transcribing it. **This step is unambiguously correct regardless of the registry decision**, and it is what turns the queued CURIE-validation-hook item and the unowned "wire `CROSS_REF_PATTERNS` as a validator" gap into one piece of work with one owner.
2. **Vendor a ~2 KB snapshot.** `data/bioregistry-patterns.json`: prefix, pattern, bioregistry version, Zenodo DOI pin (`10.5281/zenodo.22681298`). `bioregistry` goes in the dev/test extra only.
3. **`tests/contract/test_bioregistry_drift.py`**, marked `integration` so unit runs stay offline. Per key, assert a recorded **relationship** — `{equivalent, project_is_narrower, prefix_layer_differs, concept_mismatch}` with a written justification — **not equality**. It fails when upstream widens or drops a pattern, or when a project pattern moves relative to upstream.

Bioregistry then owns the **prefix vocabulary** and the **justification for each of the 15 deliberate divergences**. It does not own the runtime validator.

### Stated failure modes

(a) **This does not eliminate hand-maintained regexes.** It makes 15 of 23 divergences deliberate instead of accidental. If the goal was "stop hand-maintaining patterns," this does not achieve it — and the evidence says nothing can, because 7 break and 7 widen.
(b) A **new key added without a snapshot entry is invisible** unless the test also asserts `set(XREF_PATTERNS) == set(snapshot)`. Assert it explicitly.
(c) **A stale pin is a silent no-op** — a snapshot nobody refreshes stops being an oracle while still looking like one. Mitigate with a scheduled job or a max-age assertion on the recorded version.
(d) The **two-layer split is itself a new invariant that can rot**. A test asserting both layers cover the same concept set is required, not optional.
(e) `ucsc`, `string`, `stitch` — **3 of 23 keys get no oracle coverage** and stay purely hand-owned. (File two upstream issues on `biopragmatics/bioregistry` for the STRING and STITCH resources; it is cheap and turns two `concept_mismatch` rows into resolvable ones.)
(f) If the project later moves `cross_references` to bare LUIs, **~7 keys change on the wire.** That is a real ADR-001 Appendix A amendment. **This recommendation deliberately does not make that call** — and note that FastMCP 3's component versioning is precisely the mechanism for shipping it as `get_pathway` v2 while v1 keeps serving existing agents, instead of a flag-day break across deepagents, temporal and the bio-research plugin. That is the strongest non-security argument for taking 2 → 3 *before* AGE-700 resolves.

---

## 6. What not to do

| Option | Why not |
|---|---|
| **Jump straight 2.14.5 → 4.0.3** | Inherits both majors' breaks at once with three unresolved gates in front of it (Horizon's supported FastMCP range, the downstream consumer audit, the v4 text-block serialization path vs the omit-null guarantee). 3.4.7 is a genuine resting point — all of 3.x pins `mcp<2.0`, so the installed 1.26.0 carries through untouched. |
| **Migrate the reference corpus from Tools to Resources** | Chases a caching win that two constructor kwargs on `tools/list` capture most of, at the cost of breaking the agent-facing contract: resources are application-driven, take no arguments (so no `slim=`), and support no pagination. |
| **Adopt MRTR "to be current"** | Normative but inert here, imports a signing-key lifecycle and replay window into a federation with no server-side secrets, and is broken in claude.ai today for the state-only case (issue #1027, open, no maintainer response). |
| **Align the envelope with TRAPI now** | 2.0 is beta and its binding/constraint shapes changed twice in 2026, most recently two days ago. Downgrade §9 from "immediate opportunity" to "revisit at 2.0.0 GA." Borrow only the stable ideas: KL/AT as first-class edge properties, and the no-null discipline you already have. |
| **Make Bioregistry (or Identifiers.org, or Node Normalizer) the runtime validator** | Breaks 7 keys, widens 7 including the two Ensembl keys the strict-tool guarantee rests on, can't see two of your prefixes, and adds four transitive deps to every server for what is static data. |
| **Model STRING directionality against v12.5** | `version-12-5.string-db.org/api/tsv/version` returns `12.0` and `/help/api` documents no regulatory-network endpoint. v12.5 is a web-UI preview. The 7-channel evidence recommendation *is* actionable today, and Biolink v4.4.4's new STRING confidence slots make it cheaper than in January. |
| **Rewrite `prior-art-api-patterns.md` as a project** | Do the targeted correction instead — §6/§7.1/§7.2 novelty claims, the TRAPI version, the BTE framing, and one paragraph naming the real differentiator. Bump the revision **before it is cited in any new ADR**, or it will assert industry-precedent grounding for things the spec now mandates outright, and misdate the error-code guidance. |
| **Respond to ToolUniverse / BioMCP by broadening the tool surface** | Their axis is breadth; yours is depth of contract. Competing on breadth against 1,000–2,700 tools with 34 is a losing frame. One registry submission and one honest paragraph is the whole response. |
| **Adopt MCPmed's EDAM/Bioschemas tagging now** | A published call, not an adopted convention — its reference repos have 1–12 stars and mostly stopped in July 2025. Cite it in §5; revisit if uptake appears. |

---

## 7. Source quality and open questions

### Tier 1 — normative specification (highest confidence)

MCP 2026-07-28 spec pages and `schema.ts`; the deprecated-features registry; SEP-2596 feature-lifecycle policy. Everything in §3 about statelessness, `server/discover`, MRTR shape and requirements, `CacheableResult` (`ttlMs`/`cacheScope` are **non-optional** in `schema.ts:1081-1110`), `Mcp-Method`/`Mcp-Name`, the `-32020`–`-32099` reservation, annotation defaults and the untrusted-annotations clause. Caveat: SEP-2596 and SEP-2549 source documents were **not read in full** — the 12-month window, the 90-day expedited-removal exception and the per-row removal dates come from the derived registry, which itself says the per-feature notices and changelog are the normative records.

### Tier 2 — upstream changelog, release notes, and source read (high confidence)

FastMCP upgrade guides and v4.0.3 source from GitHub raw; PyPI `requires_dist` and upload timestamps; TRAPI `ChangeLog.md`; Biolink releases; MCP Python SDK release notes and `ROADMAP.md`. **All 2.14.5 claims are empirical from the local venv. No 3.x or 4.x claim was executed** — none of them was tested against an installed runtime.

### Tier 3 — vendor documentation (reliable, but vendor-framed)

`gofastmcp.com` releases policy and deployment docs; string-db.org API help; Prefect Horizon docs; the EMBL "Connecting AI to biology" piece.

### Tier 4 — peer-reviewed papers

Hoyt et al. 2022 (Bioregistry, *Scientific Data*); BioContextAI (*Nature Biotechnology* 2025); MCPmed (*Briefings in Bioinformatics* 2026); ToolUniverse (arXiv 2509.23426 — **preprint, not peer-reviewed**); Callaghan et al. 2023 (BTE).

### Load-bearing [blog] claims — named, not smoothed

**One, and it carries a governance edit.** The reversal of the `slim=True` novelty claim rests on **Anthropic's engineering blog, "Writing effective tools for agents," 2025-09-11** — a vendor engineering post, not a specification or a paper. It is load-bearing for the recommended rewrite of prior-art §6/§7.1/§7.2. What makes it usable despite the tier is that the *priority* argument turns on two published dates (2025-09-11 vs the doc's 2026-01-24), which are checkable, not on the post's authority. The corroboration is independent and stronger than the blog: four biomedical MCP servers now budget tokens, and TRAPI 2.0's changelog does the same work for the same stated reason. **Do not cite the blog as a standard.** Cite it as evidence of date.

### Load-bearing [inference] claims — named

1. **The FastMCP 4 `tool_names` double-prefix inversion.** Read from v4.0.3 source and code comments; **not observed**. The v4 docstring still says "override" while the code prepends — the docstring is stale relative to the implementation, which is exactly the situation where a source read can be wrong. This is the single highest-risk item in the whole analysis. **Prove it in a throwaway venv before anyone edits `gateway.py`.**
2. **"The repo's thin FastMCP surface makes almost every documented break a no-op."** Derived from grep over `src/`, `tests/`, `scripts/`. Solid, but it is the premise that collapses the effort estimate from days to hours — if a Context or lifespan usage was missed, the estimate is wrong.
3. **Every effort estimate here (~1h, ~half a day, ~1–2 days, ~2–3 days) is inferred**, from measured inventory (14 modules, 36 decorators, 12 mounts, 173 contract tests, 17 httpx sites) against the published guides. **None is documented anywhere and none is validated by having done it.** The cost is verification, not authorship — which means the estimates are most likely to be wrong in the direction of *more* time spent chasing live-API flakes, not less.
4. **The Bioregistry "8 drop-in" verdict** is measured, but on **hand-authored accept/reject vectors**. "Drop-in" means "equivalent on tested vectors," not "provably equivalent." Two regexes agreeing on three strings is weak evidence. Before acting, replay real `cross_references` values captured from the e2e suite through both patterns.
5. **Pattern churn rate for the 23 prefixes specifically was not measured.** The registry-wide rate (~10–12 real pattern edits in two years across ~2,000 prefixes) is measured; the projection onto a 23-prefix subset is inference. A `git log -L` over those pattern lines would replace it and set the drift-test cadence empirically.
6. **MCPmed's non-adoption** is judged from stars and push dates only — no download, citation or registry-uptake numbers.

### Open questions, in the order they block work

| # | Question | Blocks |
|---|---|---|
| 1 | Does FastMCP 4 actually double-prefix under `namespace=` + `tool_names=`? | The 2 → 3 port (and the shape of the guard test) |
| 2 | What protocol version does the live gateway at `biosciences-mcp.fastmcp.app` negotiate today? Probe it with a 2026-07-28 POST. | Any version claim in an ADR |
| 3 | Does Prefect Horizon build FastMCP 4, and does it still honour the v1 `fastmcp.json` schema? Docs state no supported range — needs a preview deployment or a direct question. | The 3 → 4 step, and staging the pin change |
| 4 | Does the federation set `isError: true` on the wire for `ErrorEnvelope`? One contract assertion settles it. | Whether the spec's self-correction path engages at all |
| 5 | Does FastMCP 2.14.5 emit a usable `outputSchema` for the 34 union return types? | Whether SEP-2106 is a real unblock or a non-issue |
| 6 | Do deepagents (`apps/api/shared/mcp.py`), temporal activities, and the bio-research plugin hardcode tool names or read camelCase model attributes? **Not audited.** | Prerequisite for 3 → 4, not for 2 → 3 |
| 7 | Does ADR-001 Appendix A's *prose* state an intent that `cross_references` hold bare local IDs? (Grep confirms it never mentions Bioregistry, Identifiers.org or Node Normalizer; the 23-key table was taken as faithfully transcribed into `registry.py`.) | Failure mode (f) — whether the two-layer split is a fix or a deferral |
| 8 | Does Bioregistry's `docs/GOVERNANCE.md` state a backward-compatibility policy for pattern changes? | Whether pinning a tag suffices or the drift test is load-bearing |
| 9 | What is actual `tools/list` volume against the gateway? Unmeasured. | Whether the caching and Resources work is worth anything |
| 10 | Do hosts render `resource_link` blocks usefully? Not investigated. The spec only notes resource links from tools are not guaranteed to appear in `resources/list`. | The dual-expose Resources design, if ever revisited |