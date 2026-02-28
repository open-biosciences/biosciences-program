# Design: graphiti-fastmcp Migration to biosciences-memory

**Date:** 2026-02-27
**Approach:** Curated Migration (copy + strip unused providers + add biosciences entities)
**Source:** `/home/donbr/graphiti-fastmcp` (v1.0.1, 2,932 LOC)
**Target:** `/home/donbr/open-biosciences/biosciences-memory`
**Owner:** Memory Engineer (Agent 4)

---

## Constraints

- **No PII in commits** — only `.env.example` with placeholders, synthetic test data
- **OpenAI + Neo4j only** — strip all other LLM/DB providers
- **Both entry points** — factory (FastMCP Cloud) + CLI (argparse + YAML)
- **Entity types** — 9 generic + 5 biosciences-specific + edge types + edge_type_map
- **Pin fastmcp<3** — v3 is RC, source uses v2 patterns
- **Pin graphiti-core==0.24.3** — strict match with source

---

## Package Structure

```
biosciences-memory/
├── pyproject.toml
├── config/
│   └── config.yaml
├── src/
│   └── biosciences_memory/
│       ├── __init__.py
│       ├── server.py                       # Factory entry point (FastMCP Cloud)
│       ├── cli.py                          # CLI entry point (argparse + YAML)
│       ├── config/
│       │   ├── __init__.py
│       │   └── schema.py                   # Pydantic-Settings (OpenAI + Neo4j)
│       ├── services/
│       │   ├── __init__.py
│       │   ├── factories.py                # LLM + Embedder + Database factories
│       │   └── queue_service.py            # Async per-group episode processing
│       ├── models/
│       │   ├── __init__.py
│       │   ├── response_types.py           # TypedDict responses
│       │   └── entity_types.py             # 9 generic + 5 biosciences entities
│       └── utils/
│           ├── __init__.py
│           └── formatting.py               # Node/edge serialization
├── tests/
│   ├── conftest.py
│   ├── unit/
│   │   ├── test_config.py
│   │   ├── test_factories.py
│   │   ├── test_queue_service.py
│   │   ├── test_formatting.py
│   │   └── test_entity_types.py
│   └── integration/
│       ├── test_mcp_tools.py               # In-memory Client(server) tool tests
│       └── test_neo4j_connection.py
└── (existing: .mcp.json, .env.example, CLAUDE.md, README.md, LICENSE)
```

---

## Migration Map

### Migrated (curated)

| Source File | Target | LOC | Changes |
|-------------|--------|-----|---------|
| `src/server.py` | `server.py` | ~400 | Strip FalkorDB paths, keep Neo4j + queue + 9 tools + health routes |
| `src/graphiti_mcp_server.py` | `cli.py` | ~200 | Extract argparse + YAML loading, delegate to shared server creation |
| `src/config/schema.py` | `config/schema.py` | ~150 | Remove FalkorDB, Anthropic, Gemini, Groq, Voyage, Azure configs |
| `src/services/factories.py` | `services/factories.py` | ~100 | Keep OpenAI LLM + OpenAI Embedder + Neo4j Database only |
| `src/services/queue_service.py` | `services/queue_service.py` | 152 | Copy as-is |
| `src/models/response_types.py` | `models/response_types.py` | 43 | Copy as-is |
| `src/models/entity_types.py` | `models/entity_types.py` | ~300 | Keep 9 generic, add 5 biosciences entities + edges + map |
| `src/utils/formatting.py` | `utils/formatting.py` | 50 | Copy as-is |
| `config/config.yaml` | `config/config.yaml` | ~30 | Strip to OpenAI + Neo4j sections |
| `Neo4jDriverWithPoolConfig` | In `server.py` | ~30 | Preserve — critical for Aura pool management |

**Estimated total:** ~1,200 LOC source + ~400 LOC tests

### Dropped

| Source | Reason |
|--------|--------|
| `src/utils/utils.py` (Azure helper) | No Azure provider |
| FalkorDB config + factory branches | Neo4j only |
| Anthropic/Gemini/Groq/Voyage branches | OpenAI only |
| `claude-agent-sdk`, `langchain-*`, `reportlab`, `langsmith-fetch` | Not used by memory server |
| 19 existing test modules | Rewritten with FastMCP official patterns |

### New

| File | Content |
|------|---------|
| `pyproject.toml` | hatchling, ruff, pyright, pytest markers |
| 5 biosciences entity types | Gene, Protein, Drug, Disease, Pathway |
| 5 biosciences edge types | TargetOf, AssociatedWith, TreatedBy, InvolvedIn, InteractsWith |
| `edge_type_map` | Domain connection rules |
| Unit tests (5 modules) | Config, factories, queue, formatting, entities |
| Integration tests (2 modules) | In-memory MCP tools, live Neo4j |

---

## Biosciences Entity Types

```python
# Entity types
class Gene(BaseModel):
    symbol: str | None = Field(None, description="HGNC gene symbol")
    hgnc_id: str | None = Field(None, description="HGNC identifier (e.g., HGNC:1100)")
    ensembl_id: str | None = Field(None, description="Ensembl gene ID")

class Protein(BaseModel):
    uniprot_id: str | None = Field(None, description="UniProt accession")
    name: str | None = Field(None, description="Protein name")

class Drug(BaseModel):
    chembl_id: str | None = Field(None, description="ChEMBL compound ID")
    name: str | None = Field(None, description="Drug/compound name")
    phase: int | None = Field(None, description="Clinical trial phase")

class Disease(BaseModel):
    name: str | None = Field(None, description="Disease name")
    mondo_id: str | None = Field(None, description="MONDO disease ontology ID")

class Pathway(BaseModel):
    name: str | None = Field(None, description="Pathway name")
    wikipathways_id: str | None = Field(None, description="WikiPathways ID")

# Edge types
class TargetOf(BaseModel):
    mechanism: str | None = Field(None, description="Mechanism of action")

class AssociatedWith(BaseModel):
    score: float | None = Field(None, description="Association score")

class TreatedBy(BaseModel):
    indication: str | None = Field(None, description="Treatment indication")

class InvolvedIn(BaseModel):
    role: str | None = Field(None, description="Role in pathway")

class InteractsWith(BaseModel):
    interaction_type: str | None = Field(None, description="Type of interaction")

# Edge type map
edge_type_map = {
    ("Gene", "Protein"): ["AssociatedWith"],
    ("Drug", "Gene"): ["TargetOf"],
    ("Drug", "Protein"): ["TargetOf"],
    ("Drug", "Disease"): ["TreatedBy"],
    ("Gene", "Disease"): ["AssociatedWith"],
    ("Gene", "Pathway"): ["InvolvedIn"],
    ("Protein", "Pathway"): ["InvolvedIn"],
    ("Protein", "Protein"): ["InteractsWith"],
    ("Gene", "Gene"): ["InteractsWith"],
    ("Entity", "Entity"): ["AssociatedWith"],  # fallback
}
```

---

## Dependencies

```toml
[project]
name = "biosciences-memory"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "fastmcp>=2.13.3,<3",
    "graphiti-core==0.24.3",
    "openai>=1.91.0",
    "pydantic>=2.0.0",
    "pydantic-settings>=2.0.0",
    "pyyaml>=6.0",
    "python-dotenv>=1.0.0",
    "typing-extensions>=4.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.21.0",
    "pytest-timeout>=2.4.0",
    "ruff>=0.7.1",
    "pyright>=1.1.404",
]
```

---

## Test Strategy

Based on FastMCP official testing patterns (gofastmcp.com/patterns/testing):

**Unit tests** (`tests/unit/`, marker: `@pytest.mark.unit`):
- Config: Validate defaults, env var overrides, YAML loading, missing key errors
- Factories: Mock OpenAIClient, OpenAIEmbedder, Neo4jDriver — verify factory dispatch
- Queue: Test sequential processing, parallel group_ids, queue size tracking
- Formatting: Verify node/edge serialization output
- Entity types: Validate Pydantic model construction, optional fields

**Integration tests** (`tests/integration/`, marker: `@pytest.mark.integration`):
- In-memory MCP tools: `Client(server)` pattern — call all 9 tools with mocked Graphiti client
- Live Neo4j: `@pytest.mark.skipif` when `NEO4J_URI` not set

**pytest config:**
```toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
markers = [
    "unit: Unit tests (no external dependencies)",
    "integration: Integration tests (may need Neo4j or API keys)",
    "e2e: End-to-end tests (full stack)",
]
```

---

## PII Safeguards

| Risk | Mitigation |
|------|------------|
| API keys in `.env` | `.gitignore` excludes `.env`; only `.env.example` committed |
| Real Neo4j URIs | Placeholder values in `.env.example` |
| Personal paths in config | `config.yaml` uses `${ENV_VAR}` expansion only |
| Entity types with personal data | Only generic + biosciences domain types |
| Test fixtures | Synthetic data only (BRCA1, TP53 — public gene symbols) |

---

## Key Patterns Preserved

1. **Neo4jDriverWithPoolConfig** — Custom driver subclass with connection pool tuning for Aura
2. **QueueService** — Per-group_id sequential episode processing, cross-group parallel
3. **SEMAPHORE_LIMIT** — Configurable concurrency control for LLM rate limits
4. **Health/status routes** — `GET /health` and `GET /status` custom endpoints
5. **YamlSettingsSource** — Pydantic-Settings with `${VAR:default}` expansion
6. **Index race condition handling** — Catch `EquivalentSchemaRuleAlreadyExists` on Neo4j 5.x

---

## Research Sources

- FastMCP testing patterns: https://gofastmcp.com/patterns/testing
- FastMCP test organization: https://gofastmcp.com/development/tests
- Graphiti custom entity types: https://help.getzep.com/graphiti/core-concepts/custom-entity-and-edge-types
- pytest markers: https://docs.pytest.org/en/stable/
