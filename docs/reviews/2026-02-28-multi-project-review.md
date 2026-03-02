# Multi-Project Review: biosciences-data-sources, gdelt-knowledge-base, rag-pipeline

**Date**: 2026-02-28
**Scope**: 4 parallel agent reviews + HF dataset card extraction
**Projects**: biosciences-data-sources, gdelt-knowledge-base, biosciences-rag-pipeline, knowledge-work-plugins, biosciences-research

---

## Step 2 Completed: HF Dataset Card Extraction

Extracted 4 dataset cards from inline Python strings to external markdown files:

| File | Source | Size |
|------|--------|------|
| `data/cards/sources_card.md` | `publish_interim_datasets.py:create_sources_card()` | 2,080 chars |
| `data/cards/golden_testset_card.md` | `publish_interim_datasets.py:create_golden_testset_card()` | 2,175 chars |
| `data/cards/evaluation_inputs_card.md` | `publish_processed_datasets.py:create_evaluation_datasets_card()` | 4,371 chars |
| `data/cards/evaluation_metrics_card.md` | `publish_processed_datasets.py:create_detailed_results_card()` | 5,793 chars |

Both publish scripts refactored to use `load_card(name)` helper. Syntax verified, cards load correctly.

---

## Consolidated Findings by Priority

### CRITICAL

| # | Finding | Project | File(s) | Agent |
|---|---------|---------|---------|-------|
| 1 | **Live API keys in `.env`** — OpenAI, Cohere, HF, LangSmith, Qdrant credentials on disk | data-sources | `.env` | A |
| 2 | **Live Neo4j Aura credentials committed to git** — URI + password in tracked `.env` | gdelt | `.env` | B |
| 3 | **Cypher label injection** — `node['type']` from external data interpolated into f-string Cypher queries without validation | gdelt | `builders/neo4j.py:59-72` | B |
| 4 | **Neo4j driver never closed on connection failure** — constructor creates driver before `_verify_connection()`, no cleanup on exception | gdelt | `builders/neo4j.py:20-23` | B |
| 5 | **`evaluate_metrics.py` runs evaluation on import** — no `main()` guard, RAGAS evaluation executes at module scope | data-sources | `scripts/evaluate_metrics.py` | A |

### IMPORTANT

| # | Finding | Project | File(s) | Agent |
|---|---------|---------|---------|-------|
| 6 | **Relationship MERGE uses wrong label** — `rel['type']` used as both node label and relationship type, so no relationships are ever created | gdelt | `builders/neo4j.py:65-77`, `builders/base.py` | B |
| 7 | **`COLLECTION_NAME`, `OPENAI_MODEL_NAME` can be `None`** — no env var validation, confusing downstream errors | data-sources | `src/config.py:36-38` | A |
| 8 | **Vector dimension `1536` hardcoded** — diverges if embedding model changes | data-sources | `src/config.py:136` | A |
| 9 | **No error handling around `push_to_hub`/`upload_file`** — partial uploads leave HF Hub inconsistent | data-sources | both publish scripts | A |
| 10 | **`run_full_evaluation.py` duplicates entire `src/`** — 530 lines parallel implementation, permanent divergence risk | data-sources | `scripts/run_full_evaluation.py` | A |
| 11 | **DuckDB connection never closed on query error** — no context manager | gdelt | `utils/processor.py:17-43` | B |
| 12 | **Dependency management split across 3 files** — `pyproject.toml` (empty), `setup.py`, `requirements.txt` with version conflicts | gdelt | root config files | B |
| 13 | **`event_timeline` query calls `date()` on GDELT integer string** — returns null, empty timeline | gdelt | `queries/analytical.py:15` | B |
| 14 | **Bare `except:` catches `KeyboardInterrupt`** in `_ver()` | data-sources | `scripts/ingest_raw_pdfs.py:236` | A |
| 15 | **`==` pinning in `pyproject.toml`** — violates workspace `>=` convention | data-sources | `pyproject.toml:11-14` | A |
| 16 | **Stage 3 retrieval harness script missing** — pipeline-blocking gap, `/run-retrieval` has no backing script | rag-pipeline | `scripts/run_retrieval.py` (absent) | C |
| 17 | **No `pyproject.toml` or dependency manifest** in rag-pipeline — `uv sync` cannot work | rag-pipeline | repo root | C |
| 18 | **biosciences-gateway not wired** — `.mcp.json` configures gateway, README references 5 `~~category` connectors, but no command actually calls them | rag-pipeline | commands/, CONNECTORS.md | C |

### NICE-TO-HAVE

| # | Finding | Project | File(s) | Agent |
|---|---------|---------|---------|-------|
| 19 | `HF_USERNAME = "dwb2023"` unused in both publish scripts | data-sources | both publish scripts | A |
| 20 | `validate_and_normalize_ragas_schema()` defined but never called (60 lines) | data-sources | `run_full_evaluation.py:70` | A |
| 21 | `generated_by` in manifest is a stale hardcoded path | data-sources | `src/utils/manifest.py:153` | A |
| 22 | `visualizer.py` uses `notebook=True` in script context — `display()` NameError risk | gdelt | `utils/visualizer.py:23` | B |
| 23 | Entity deduplication absent — 100 MERGEs for same node in a batch | gdelt | `builders/base.py:43-57` | B |
| 24 | `gdelt-hf.py` re-assigns `HF_TOKEN` to itself (no-op) | gdelt scripts | `gdelt-hf.py:7-11` | B |
| 25 | `__pycache__` directories committed to git | gdelt | multiple dirs | B |
| 26 | `~~vector-store`, `~~embedding-model`, `~~reranker`, `~~llm` used in commands but not defined in CONNECTORS.md | rag-pipeline | commands/, CONNECTORS.md | C |
| 27 | `source-profiling` skill has no `references/` directory | rag-pipeline | skills/source-profiling/ | C |
| 28 | Missing `LICENSE` file at rag-pipeline repo root | rag-pipeline | repo root | C |

---

## RAG Pipeline Gap Matrix

| Stage | Command | Script | Status |
|-------|---------|--------|--------|
| 1 (Ingest) | `/ingest` | `chunker.py` (464 LOC) | Complete |
| 1 (Publish) | `/ingest` step 5 | `publish_dataset.py` (198 LOC) | Complete |
| 2 (Generate) | `/generate-testset` | `generate_testset.py` (360 LOC) | Complete |
| 3 (Retrieve) | `/run-retrieval` | **MISSING** | Blocking |
| 4 (Evaluate) | `/evaluate` | `run_ragas.py` (156 LOC) | Complete |
| Dashboard | `/build-dashboard` | Inline (by design) | Complete |
| Explore | `/explore-sources` | via `chunker.py --profile-only` | Complete |
| Validate | `/validate` | Pure skill (by design) | Complete |

**Command format**: 7/7 conform to knowledge-work-plugins pattern (YAML frontmatter, numbered workflow, `~~connector` placeholders).

**Standalone+Supercharged readiness**: Substantially Standalone for stages 1, 2, 4. MCP gateway is advertised but not wired into any workflow step.

---

## HF Skills Recommendations Matrix

| Skill | Relevance | Primary Use Case | Projects |
|-------|-----------|------------------|----------|
| **hugging-face-datasets** | Critical | Replace manual publish scripts with DatasetCard API, SQL-first queries via DuckDB `hf://` | 1, 3 |
| **hugging-face-jobs** | High | Offload 20-30min eval harness to cloud (removes Qdrant dependency), schedule GDELT refresh | 1, 2, 3 |
| **hugging-face-cli** | High | Dataset version pinning, `repo info` for manifest validation, offline caching | 1, 3 |
| **hugging-face-paper-publisher** | High | Link datasets to arXiv:2503.07584, implement `/evaluate` TODO on line 82 | 3, 5 |
| **hugging-face-evaluation** | High | Submit RAGAS scores to HF leaderboard format, standardized benchmark output | 1, 3 |
| **hugging-face-tool-builder** | Medium | Package `loaders.py` and `load_gdelt_data()` as callable HF Tools | 1, 2 |
| **huggingface-gradio** | Medium | Replace static dashboard HTML with interactive Gradio app, deploy as HF Space | 3 |
| **hugging-face-trackio** | Low | scvi-tools training metrics only | 4 |
| **hugging-face-model-trainer** | Low | Future: fine-tune reranker on eval data | 1 |

### Top 4 Concrete Actions

1. **Replace inline card strings** with external `.md` files + `load_card()` — **DONE** (this session)
2. **Use `hugging-face-jobs`** to run eval harness as cloud compute (removes local Qdrant dependency)
3. **Use `hugging-face-paper-publisher`** to link 4 published datasets to arXiv:2503.07584
4. **Use `hugging-face-cli`** for `repo info` validation in `validate_manifests.py`

---

## Shared Patterns Suitable for Extraction

### To biosciences-research (docs/methodology)

| Pattern | Source | Why |
|---------|--------|-----|
| 4-dataset RAG evaluation methodology | data-sources `scripts/` | Documented evaluation pipeline: sources → testset → eval inputs → metrics |
| Retriever comparison framework | data-sources `scripts/run_eval_harness.py` | 5-retriever benchmark with RAGAS scoring |
| Competency question alignment | rag-pipeline `scripts/generate_testset.py` | CQ keyword mapping for domain-specific testset generation |

### To biosciences-rag-pipeline (reusable code)

| Pattern | Source | Why |
|---------|--------|-----|
| Generic HF publisher | rag-pipeline already has `publish_dataset.py` | data-sources should backport this pattern |
| Interactive dashboard builder | knowledge-work-plugins `data/skills/interactive-dashboard-builder/` | Chart.js template library for `/build-dashboard` |
| DuckDB `hf://` query patterns | gdelt `processor.py`, rag-pipeline `hf-dataset-queries` skill | Already documented, consolidate as shared utility |

---

## Security Items Requiring Immediate Action

1. **Rotate all 5 API keys** in `/home/donbr/ag-discovery/biosciences-data-sources/.env` (OpenAI, Cohere, HF, LangSmith, Qdrant)
2. **Rotate Neo4j Aura credentials** in `/home/donbr/gdelt_pipeline/.env` and purge from git history
3. **Add allowlist validation** for Cypher label interpolation in `gdelt_pipeline/builders/neo4j.py`
