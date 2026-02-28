# graphiti-fastmcp Migration Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Migrate the graphiti-fastmcp server into biosciences-memory as a curated, OpenAI+Neo4j-only package with biosciences domain entity types and FastMCP-official test patterns.

**Architecture:** Factory-pattern FastMCP server with async queue-based episode processing, Pydantic-Settings config with YAML+env expansion, and custom Neo4j driver for Aura connection pool management. Both factory (Cloud) and CLI entry points. Tests use FastMCP's in-memory `Client(server)` pattern.

**Tech Stack:** Python >=3.11, fastmcp 2.x, graphiti-core 0.24.3, openai, pydantic v2, pydantic-settings, pytest-asyncio, ruff, pyright, hatchling, uv

**Design doc:** `docs/plans/2026-02-27-graphiti-fastmcp-migration-design.md`

**PII safeguard:** No `.env` files, no real URIs/keys, no personal paths. Only `.env.example` with placeholders and synthetic test data (BRCA1, TP53).

---

### Task 1: Create pyproject.toml

**Files:**
- Create: `biosciences-memory/pyproject.toml`

**Step 1: Write the file**

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "biosciences-memory"
version = "0.1.0"
description = "Graphiti knowledge graph memory layer for the Open Biosciences platform"
requires-python = ">=3.11"
license = "MIT"
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

[tool.hatch.build.targets.wheel]
packages = ["src/biosciences_memory"]

[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "W", "I", "B", "UP"]

[tool.pyright]
pythonVersion = "3.11"
typeCheckingMode = "basic"

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
markers = [
    "unit: Unit tests (no external dependencies)",
    "integration: Integration tests (may need Neo4j or API keys)",
    "e2e: End-to-end tests (full stack)",
]
```

**Step 2: Verify**

Run: `cd /home/donbr/open-biosciences/biosciences-memory && uv sync --extra dev 2>&1 | tail -5`
Expected: Dependencies resolve successfully

**Step 3: Commit**

```bash
cd /home/donbr/open-biosciences/biosciences-memory
git add pyproject.toml
git commit -m "chore: add pyproject.toml with hatchling, ruff, pyright, pytest markers"
```

---

### Task 2: Create package skeleton + response models + formatting utils

**Files:**
- Create: `src/biosciences_memory/__init__.py`
- Create: `src/biosciences_memory/config/__init__.py`
- Create: `src/biosciences_memory/services/__init__.py`
- Create: `src/biosciences_memory/models/__init__.py`
- Create: `src/biosciences_memory/utils/__init__.py`
- Create: `src/biosciences_memory/models/response_types.py` (copy as-is from `graphiti-fastmcp/src/models/response_types.py`)
- Create: `src/biosciences_memory/utils/formatting.py` (copy as-is from `graphiti-fastmcp/src/utils/formatting.py`)

**Step 1: Create directory structure**

```bash
mkdir -p src/biosciences_memory/{config,services,models,utils}
touch src/biosciences_memory/__init__.py
touch src/biosciences_memory/config/__init__.py
touch src/biosciences_memory/services/__init__.py
touch src/biosciences_memory/models/__init__.py
touch src/biosciences_memory/utils/__init__.py
```

**Step 2: Write response_types.py**

Copy verbatim from source: `/home/donbr/graphiti-fastmcp/src/models/response_types.py` (43 LOC, no changes needed)

**Step 3: Write formatting.py**

Copy verbatim from source: `/home/donbr/graphiti-fastmcp/src/utils/formatting.py` (50 LOC, no changes needed)

**Step 4: Verify imports**

Run: `cd /home/donbr/open-biosciences/biosciences-memory && uv run python -c "from biosciences_memory.models.response_types import StatusResponse; print('OK')"`
Expected: `OK`

**Step 5: Commit**

```bash
git add src/
git commit -m "feat: add package skeleton, response models, and formatting utils"
```

---

### Task 3: Create entity types (generic + biosciences)

**Files:**
- Create: `src/biosciences_memory/models/entity_types.py`

**Step 1: Write entity_types.py**

Copy the 9 generic entity type classes from source (`graphiti-fastmcp/src/models/entity_types.py`), then append:

```python
# --- Biosciences Domain Entity Types ---

class Gene(BaseModel):
    """A gene entity from a biological organism.

    Instructions for identifying and extracting genes:
    1. Look for gene symbols (BRCA1, TP53, EGFR) and full gene names
    2. Identify HGNC identifiers when available (HGNC:1100)
    3. Extract Ensembl gene IDs (ENSG00000012048) when referenced
    4. Capture gene-disease associations and gene functions
    """

    symbol: str | None = Field(None, description="HGNC gene symbol (e.g., BRCA1)")
    hgnc_id: str | None = Field(None, description="HGNC identifier (e.g., HGNC:1100)")
    ensembl_id: str | None = Field(None, description="Ensembl gene ID (e.g., ENSG00000012048)")


class Protein(BaseModel):
    """A protein entity with structural and functional annotations.

    Instructions for identifying and extracting proteins:
    1. Look for UniProt accessions (P38398) and protein names
    2. Identify protein families and domains when mentioned
    3. Capture protein-protein interactions and functional roles
    """

    uniprot_id: str | None = Field(None, description="UniProt accession (e.g., P38398)")
    name: str | None = Field(None, description="Protein name")


class Drug(BaseModel):
    """A drug or chemical compound entity.

    Instructions for identifying and extracting drugs:
    1. Look for ChEMBL IDs (CHEMBL25) and drug names
    2. Identify clinical trial phases and approval status
    3. Capture mechanism of action and target information
    """

    chembl_id: str | None = Field(None, description="ChEMBL compound ID (e.g., CHEMBL25)")
    name: str | None = Field(None, description="Drug or compound name")
    phase: int | None = Field(None, description="Clinical trial phase (1-4)")


class Disease(BaseModel):
    """A disease or medical condition entity.

    Instructions for identifying and extracting diseases:
    1. Look for disease names and MONDO ontology IDs
    2. Identify disease-gene and disease-drug associations
    3. Capture prevalence, symptoms, and classification when mentioned
    """

    name: str | None = Field(None, description="Disease name")
    mondo_id: str | None = Field(None, description="MONDO disease ontology ID")


class Pathway(BaseModel):
    """A biological pathway entity.

    Instructions for identifying and extracting pathways:
    1. Look for pathway names and WikiPathways IDs (WP534)
    2. Identify signaling cascades and metabolic pathways
    3. Capture pathway components (genes, proteins) when referenced
    """

    name: str | None = Field(None, description="Pathway name")
    wikipathways_id: str | None = Field(None, description="WikiPathways ID (e.g., WP534)")


# --- Biosciences Domain Edge Types ---

class TargetOf(BaseModel):
    """Drug targets a gene or protein."""
    mechanism: str | None = Field(None, description="Mechanism of action")


class AssociatedWith(BaseModel):
    """General association between biological entities."""
    score: float | None = Field(None, description="Association score (0-1)")


class TreatedBy(BaseModel):
    """Disease is treated by a drug."""
    indication: str | None = Field(None, description="Treatment indication")


class InvolvedIn(BaseModel):
    """Gene or protein is involved in a pathway."""
    role: str | None = Field(None, description="Role in pathway")


class InteractsWith(BaseModel):
    """Protein-protein or gene-gene interaction."""
    interaction_type: str | None = Field(None, description="Type of interaction")


# --- Registries ---

ENTITY_TYPES: dict[str, type[BaseModel]] = {
    # Generic types
    "Requirement": Requirement,
    "Preference": Preference,
    "Procedure": Procedure,
    "Location": Location,
    "Event": Event,
    "Object": Object,
    "Topic": Topic,
    "Organization": Organization,
    "Document": Document,
    # Biosciences types
    "Gene": Gene,
    "Protein": Protein,
    "Drug": Drug,
    "Disease": Disease,
    "Pathway": Pathway,
}

EDGE_TYPES: dict[str, type[BaseModel]] = {
    "TargetOf": TargetOf,
    "AssociatedWith": AssociatedWith,
    "TreatedBy": TreatedBy,
    "InvolvedIn": InvolvedIn,
    "InteractsWith": InteractsWith,
}

EDGE_TYPE_MAP: dict[tuple[str, str], list[str]] = {
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

**Step 2: Verify**

Run: `uv run python -c "from biosciences_memory.models.entity_types import ENTITY_TYPES, EDGE_TYPES, EDGE_TYPE_MAP; print(f'{len(ENTITY_TYPES)} entity types, {len(EDGE_TYPES)} edge types, {len(EDGE_TYPE_MAP)} mappings')"`
Expected: `14 entity types, 5 edge types, 10 mappings`

**Step 3: Commit**

```bash
git add src/biosciences_memory/models/entity_types.py
git commit -m "feat: add entity types — 9 generic + 5 biosciences + edge types and map"
```

---

### Task 4: Create config schema (OpenAI + Neo4j only)

**Files:**
- Create: `src/biosciences_memory/config/schema.py`

**Step 1: Write schema.py**

Curate from source `graphiti-fastmcp/src/config/schema.py`. Keep:
- `YamlSettingsSource` class (verbatim — 57 LOC)
- `ServerConfig` (verbatim)
- `OpenAIProviderConfig` (verbatim)
- `LLMConfig` — remove multi-provider `LLMProvidersConfig`, simplify to just `openai: OpenAIProviderConfig`
- `EmbedderConfig` — same simplification
- `Neo4jProviderConfig` (verbatim — includes pool settings)
- `DatabaseConfig` — remove `DatabaseProvidersConfig`, simplify to just `neo4j: Neo4jProviderConfig`
- `GraphitiAppConfig` (verbatim)
- `GraphitiConfig` (verbatim — Pydantic BaseSettings root)
- `EntityTypeConfig` (verbatim)

Drop: `AzureOpenAIProviderConfig`, `AnthropicProviderConfig`, `GeminiProviderConfig`, `GroqProviderConfig`, `VoyageProviderConfig`, `FalkorDBProviderConfig`.

Change `DatabaseConfig.provider` default from `"falkordb"` to `"neo4j"`.

**Step 2: Verify**

Run: `uv run python -c "from biosciences_memory.config.schema import GraphitiConfig; c = GraphitiConfig(); print(f'DB={c.database.provider}, LLM={c.llm.provider}')"`
Expected: `DB=neo4j, LLM=openai`

**Step 3: Commit**

```bash
git add src/biosciences_memory/config/schema.py
git commit -m "feat: add config schema — OpenAI + Neo4j only with YAML env expansion"
```

---

### Task 5: Create factories (OpenAI + Neo4j only)

**Files:**
- Create: `src/biosciences_memory/services/factories.py`

**Step 1: Write factories.py**

Curate from source `graphiti-fastmcp/src/services/factories.py`. Keep only the OpenAI and Neo4j branches.

```python
"""Factory classes for creating LLM, Embedder, and Database clients."""

import logging
import os

from graphiti_core.embedder import EmbedderClient, OpenAIEmbedder
from graphiti_core.llm_client import LLMClient, OpenAIClient
from graphiti_core.llm_client.config import LLMConfig as CoreLLMConfig

from biosciences_memory.config.schema import DatabaseConfig, EmbedderConfig, LLMConfig

logger = logging.getLogger(__name__)


def _validate_api_key(provider_name: str, api_key: str | None) -> str:
    if not api_key:
        raise ValueError(
            f"{provider_name} API key is not configured. "
            "Please set the appropriate environment variable."
        )
    logger.info(f"Creating {provider_name} client")
    return api_key


class LLMClientFactory:
    """Factory for creating OpenAI LLM clients."""

    @staticmethod
    def create(config: LLMConfig) -> LLMClient:
        if not config.openai:
            raise ValueError("OpenAI provider configuration not found")

        api_key = _validate_api_key("OpenAI", config.openai.api_key)

        is_reasoning_model = (
            config.model.startswith("gpt-5")
            or config.model.startswith("o1")
            or config.model.startswith("o3")
        )
        small_model = "gpt-5-nano" if is_reasoning_model else "gpt-4.1-mini"

        llm_config = CoreLLMConfig(
            api_key=api_key,
            model=config.model,
            small_model=small_model,
            temperature=config.temperature,
            max_tokens=config.max_tokens,
        )

        if is_reasoning_model:
            return OpenAIClient(config=llm_config, reasoning="minimal", verbosity="low")
        return OpenAIClient(config=llm_config, reasoning=None, verbosity=None)


class EmbedderFactory:
    """Factory for creating OpenAI Embedder clients."""

    @staticmethod
    def create(config: EmbedderConfig) -> EmbedderClient:
        if not config.openai:
            raise ValueError("OpenAI provider configuration not found")

        api_key = _validate_api_key("OpenAI Embedder", config.openai.api_key)

        from graphiti_core.embedder.openai import OpenAIEmbedderConfig

        embedder_config = OpenAIEmbedderConfig(
            api_key=api_key,
            embedding_model=config.model,
            embedding_dim=config.dimensions,
        )
        return OpenAIEmbedder(config=embedder_config)


class DatabaseDriverFactory:
    """Factory for creating Neo4j database configuration."""

    @staticmethod
    def create_config(config: DatabaseConfig) -> dict:
        neo4j_config = config.neo4j
        if not neo4j_config:
            from biosciences_memory.config.schema import Neo4jProviderConfig
            neo4j_config = Neo4jProviderConfig()

        uri = os.environ.get("NEO4J_URI", neo4j_config.uri)
        username = os.environ.get("NEO4J_USER", neo4j_config.username)
        password = os.environ.get("NEO4J_PASSWORD", neo4j_config.password)

        return {
            "uri": uri,
            "user": username,
            "password": password,
            "database": neo4j_config.database,
            "max_connection_lifetime": neo4j_config.max_connection_lifetime,
            "max_connection_pool_size": neo4j_config.max_connection_pool_size,
            "connection_acquisition_timeout": neo4j_config.connection_acquisition_timeout,
        }
```

**Step 2: Verify**

Run: `uv run python -c "from biosciences_memory.services.factories import LLMClientFactory, EmbedderFactory, DatabaseDriverFactory; print('OK')"`
Expected: `OK`

**Step 3: Commit**

```bash
git add src/biosciences_memory/services/factories.py
git commit -m "feat: add factories — OpenAI LLM + OpenAI Embedder + Neo4j only"
```

---

### Task 6: Create queue service

**Files:**
- Create: `src/biosciences_memory/services/queue_service.py`

**Step 1: Write queue_service.py**

Copy verbatim from source: `/home/donbr/graphiti-fastmcp/src/services/queue_service.py` (152 LOC, no changes needed — this is battle-tested async code)

**Step 2: Verify**

Run: `uv run python -c "from biosciences_memory.services.queue_service import QueueService; q = QueueService(); print(f'queue_size={q.get_queue_size(\"test\")}')"`
Expected: `queue_size=0`

**Step 3: Commit**

```bash
git add src/biosciences_memory/services/queue_service.py
git commit -m "feat: add async queue service for sequential episode processing"
```

---

### Task 7: Create server.py (factory entry point + all 9 MCP tools)

**Files:**
- Create: `src/biosciences_memory/server.py`

**Step 1: Write server.py**

Curate from source `graphiti-fastmcp/src/server.py`. Key changes:
- All imports use `biosciences_memory.*` package paths
- Remove FalkorDB branch from `GraphitiService.initialize()` — Neo4j only
- Keep `Neo4jDriverWithPoolConfig` class
- Keep `GraphitiService` class
- Keep `create_server()` factory function
- Keep `_register_tools()` with all 9 tools
- Keep `/health` and `/status` custom routes
- Keep `run_local()` for `__main__` execution
- Update server name from `"Graphiti Agent Memory"` to `"Biosciences Memory Server"`
- Use `ENTITY_TYPES` from `biosciences_memory.models.entity_types` as default when no config entity types

**Step 2: Verify factory import**

Run: `uv run python -c "from biosciences_memory.server import create_server; print('OK')"`
Expected: `OK`

**Step 3: Commit**

```bash
git add src/biosciences_memory/server.py
git commit -m "feat: add MCP server with 9 tools, Neo4j driver, and factory entry point"
```

---

### Task 8: Create cli.py (CLI entry point)

**Files:**
- Create: `src/biosciences_memory/cli.py`

**Step 1: Write cli.py**

Extract from source `graphiti-fastmcp/src/graphiti_mcp_server.py` the argparse setup and main() function. Delegate to `create_server()` from `server.py`.

Key args to keep: `--transport`, `--host`, `--port`, `--config`, `--model`, `--group-id`, `--destroy-graph`

Drop: `--llm-provider`, `--database-provider`, `--embedder-provider`, `--embedder-model` (OpenAI + Neo4j only)

**Step 2: Verify**

Run: `uv run python -m biosciences_memory.cli --help`
Expected: Shows argparse help with available options

**Step 3: Commit**

```bash
git add src/biosciences_memory/cli.py
git commit -m "feat: add CLI entry point with argparse + YAML config support"
```

---

### Task 9: Create config.yaml

**Files:**
- Create: `config/config.yaml`

**Step 1: Write config.yaml**

Curate from source `graphiti-fastmcp/config/config.yaml`. Keep only OpenAI + Neo4j sections:

```yaml
# Biosciences Memory Server Configuration
# Supports ${VAR_NAME} and ${VAR_NAME:default_value} env var expansion

server:
  transport: "http"
  host: "0.0.0.0"
  port: 8000

llm:
  provider: "openai"
  model: "gpt-4.1-mini"
  max_tokens: 4096
  openai:
    api_key: ${OPENAI_API_KEY}
    api_url: ${OPENAI_API_URL:https://api.openai.com/v1}

embedder:
  provider: "openai"
  model: "text-embedding-3-small"
  dimensions: 1024
  openai:
    api_key: ${OPENAI_API_KEY}
    api_url: ${OPENAI_API_URL:https://api.openai.com/v1}

database:
  provider: "neo4j"
  neo4j:
    uri: ${NEO4J_URI:bolt://localhost:7687}
    username: ${NEO4J_USER:neo4j}
    password: ${NEO4J_PASSWORD}
    database: ${NEO4J_DATABASE:neo4j}

graphiti:
  group_id: ${GRAPHITI_GROUP_ID:main}
  user_id: ${USER_ID:mcp_user}
```

**Step 2: Commit**

```bash
git add config/config.yaml
git commit -m "feat: add config.yaml — OpenAI + Neo4j with env var expansion"
```

---

### Task 10: Write unit tests

**Files:**
- Create: `tests/__init__.py`
- Create: `tests/conftest.py`
- Create: `tests/unit/__init__.py`
- Create: `tests/unit/test_entity_types.py`
- Create: `tests/unit/test_queue_service.py`
- Create: `tests/unit/test_config.py`
- Create: `tests/unit/test_factories.py`
- Create: `tests/unit/test_formatting.py`

**Step 1: Write conftest.py**

```python
"""Shared pytest fixtures for biosciences-memory tests."""

import pytest

from biosciences_memory.config.schema import GraphitiConfig


@pytest.fixture
def default_config():
    """Create a default GraphitiConfig for testing."""
    return GraphitiConfig()
```

**Step 2: Write test_entity_types.py**

```python
"""Unit tests for entity type models."""

import pytest

from biosciences_memory.models.entity_types import (
    EDGE_TYPE_MAP,
    EDGE_TYPES,
    ENTITY_TYPES,
    Drug,
    Gene,
    Pathway,
    Protein,
    Disease,
)


@pytest.mark.unit
class TestEntityTypes:
    def test_entity_types_count(self):
        assert len(ENTITY_TYPES) == 14  # 9 generic + 5 biosciences

    def test_edge_types_count(self):
        assert len(EDGE_TYPES) == 5

    def test_edge_type_map_count(self):
        assert len(EDGE_TYPE_MAP) == 10

    def test_gene_optional_fields(self):
        gene = Gene()
        assert gene.symbol is None
        assert gene.hgnc_id is None
        assert gene.ensembl_id is None

    def test_gene_with_values(self):
        gene = Gene(symbol="BRCA1", hgnc_id="HGNC:1100", ensembl_id="ENSG00000012048")
        assert gene.symbol == "BRCA1"

    def test_drug_with_phase(self):
        drug = Drug(name="Aspirin", chembl_id="CHEMBL25", phase=4)
        assert drug.phase == 4

    def test_protein_creation(self):
        protein = Protein(uniprot_id="P38398", name="BRCA1")
        assert protein.uniprot_id == "P38398"

    def test_disease_creation(self):
        disease = Disease(name="Breast cancer")
        assert disease.name == "Breast cancer"
        assert disease.mondo_id is None

    def test_pathway_creation(self):
        pathway = Pathway(name="MAPK signaling", wikipathways_id="WP382")
        assert pathway.wikipathways_id == "WP382"

    def test_biosciences_types_in_registry(self):
        for name in ["Gene", "Protein", "Drug", "Disease", "Pathway"]:
            assert name in ENTITY_TYPES

    def test_drug_gene_edge_mapping(self):
        assert ("Drug", "Gene") in EDGE_TYPE_MAP
        assert "TargetOf" in EDGE_TYPE_MAP[("Drug", "Gene")]
```

**Step 3: Write test_queue_service.py**

```python
"""Unit tests for the queue service."""

import asyncio

import pytest

from biosciences_memory.services.queue_service import QueueService


@pytest.mark.unit
class TestQueueService:
    async def test_initial_queue_size(self):
        qs = QueueService()
        assert qs.get_queue_size("test") == 0

    async def test_worker_not_running_initially(self):
        qs = QueueService()
        assert qs.is_worker_running("test") is False

    async def test_add_episode_task_starts_worker(self):
        qs = QueueService()
        processed = []

        async def process():
            processed.append(True)

        await qs.add_episode_task("group1", process)
        await asyncio.sleep(0.1)  # Let worker process
        assert len(processed) == 1

    async def test_sequential_processing_within_group(self):
        qs = QueueService()
        order = []

        async def make_task(n):
            async def process():
                order.append(n)
            return process

        await qs.add_episode_task("g1", await make_task(1))
        await qs.add_episode_task("g1", await make_task(2))
        await qs.add_episode_task("g1", await make_task(3))
        await asyncio.sleep(0.2)
        assert order == [1, 2, 3]

    async def test_uninitialized_add_episode_raises(self):
        qs = QueueService()
        with pytest.raises(RuntimeError, match="not initialized"):
            await qs.add_episode(
                group_id="test",
                name="test",
                content="test",
                source_description="test",
                episode_type=None,
                entity_types=None,
                uuid=None,
            )
```

**Step 4: Write test_config.py**

```python
"""Unit tests for configuration schema."""

import pytest

from biosciences_memory.config.schema import (
    GraphitiConfig,
    Neo4jProviderConfig,
    ServerConfig,
)


@pytest.mark.unit
class TestConfig:
    def test_default_config(self):
        config = GraphitiConfig()
        assert config.server.transport == "http"
        assert config.server.port == 8000
        assert config.llm.provider == "openai"
        assert config.llm.model == "gpt-4.1"
        assert config.database.provider == "neo4j"

    def test_server_defaults(self):
        server = ServerConfig()
        assert server.host == "0.0.0.0"
        assert server.port == 8000

    def test_neo4j_pool_defaults(self):
        neo4j = Neo4jProviderConfig()
        assert neo4j.max_connection_lifetime == 300
        assert neo4j.max_connection_pool_size == 50
        assert neo4j.connection_acquisition_timeout == 60.0

    def test_graphiti_group_id_default(self):
        config = GraphitiConfig()
        assert config.graphiti.group_id == "main"
        assert config.graphiti.user_id == "mcp_user"
```

**Step 5: Write test_factories.py**

```python
"""Unit tests for factory classes."""

import pytest

from biosciences_memory.config.schema import DatabaseConfig, Neo4jProviderConfig
from biosciences_memory.services.factories import DatabaseDriverFactory


@pytest.mark.unit
class TestDatabaseDriverFactory:
    def test_neo4j_config_defaults(self):
        config = DatabaseConfig()
        result = DatabaseDriverFactory.create_config(config)
        assert result["uri"] == "bolt://localhost:7687"
        assert result["user"] == "neo4j"
        assert result["database"] == "neo4j"
        assert result["max_connection_lifetime"] == 300

    def test_neo4j_config_custom(self):
        neo4j = Neo4jProviderConfig(
            uri="neo4j+s://custom:7687",
            username="admin",
            password="secret",
            max_connection_pool_size=100,
        )
        config = DatabaseConfig(neo4j=neo4j)
        result = DatabaseDriverFactory.create_config(config)
        assert result["uri"] == "neo4j+s://custom:7687"
        assert result["user"] == "admin"
        assert result["max_connection_pool_size"] == 100
```

**Step 6: Write test_formatting.py**

```python
"""Unit tests for formatting utilities."""

import pytest
from unittest.mock import MagicMock


@pytest.mark.unit
class TestFormatting:
    def test_format_node_excludes_embedding(self):
        mock_node = MagicMock()
        mock_node.model_dump.return_value = {
            "uuid": "123",
            "name": "BRCA1",
            "attributes": {"name_embedding": [0.1, 0.2]},
        }

        from biosciences_memory.utils.formatting import format_node_result

        result = format_node_result(mock_node)
        assert "name_embedding" not in result.get("attributes", {})

    def test_format_fact_excludes_embedding(self):
        mock_edge = MagicMock()
        mock_edge.model_dump.return_value = {
            "uuid": "456",
            "fact": "BRCA1 interacts with TP53",
            "attributes": {"fact_embedding": [0.3, 0.4]},
        }

        from biosciences_memory.utils.formatting import format_fact_result

        result = format_fact_result(mock_edge)
        assert "fact_embedding" not in result.get("attributes", {})
```

**Step 7: Run all unit tests**

Run: `cd /home/donbr/open-biosciences/biosciences-memory && uv run pytest tests/unit/ -m unit -v`
Expected: All tests pass

**Step 8: Commit**

```bash
git add tests/
git commit -m "test: add unit tests — entity types, queue service, config, factories, formatting"
```

---

### Task 11: Write integration tests (in-memory MCP tools)

**Files:**
- Create: `tests/integration/__init__.py`
- Create: `tests/integration/test_mcp_tools.py`

**Step 1: Write test_mcp_tools.py**

Following FastMCP's official `Client(server)` in-memory testing pattern:

```python
"""Integration tests for MCP tools using FastMCP in-memory Client pattern.

These tests create the FastMCP server and call tools directly without
requiring a running Neo4j instance — the Graphiti service is mocked.
"""

import pytest
from unittest.mock import AsyncMock, MagicMock, patch

from fastmcp import FastMCP


@pytest.fixture
async def mcp_server():
    """Create a test MCP server with mocked Graphiti service."""
    # We test tool registration and response format, not Graphiti internals
    with patch("biosciences_memory.server.GraphitiService") as MockService:
        mock_service = MockService.return_value
        mock_service.initialize = AsyncMock()
        mock_service.get_client = AsyncMock()
        mock_service.entity_types = None

        with patch("biosciences_memory.server.QueueService") as MockQueue:
            mock_queue = MockQueue.return_value
            mock_queue.initialize = AsyncMock()
            mock_queue.add_episode = AsyncMock(return_value=1)

            from biosciences_memory.server import create_server

            server = await create_server()
            # Attach mocks for assertions
            server._test_service = mock_service
            server._test_queue = mock_queue
            yield server


@pytest.mark.integration
class TestMCPTools:
    async def test_server_has_all_tools(self, mcp_server):
        """Verify all 9 MCP tools are registered."""
        from fastmcp.client import Client

        async with Client(mcp_server) as client:
            tools = await client.list_tools()
            tool_names = {t.name for t in tools}
            expected = {
                "add_memory",
                "search_nodes",
                "search_memory_facts",
                "get_episodes",
                "get_entity_edge",
                "delete_entity_edge",
                "delete_episode",
                "clear_graph",
                "get_status",
            }
            assert tool_names == expected

    async def test_add_memory_returns_success(self, mcp_server):
        """Verify add_memory queues episode and returns success."""
        from fastmcp.client import Client

        async with Client(mcp_server) as client:
            result = await client.call_tool(
                "add_memory",
                {"name": "Test Episode", "episode_body": "BRCA1 interacts with TP53"},
            )
            assert result is not None
```

**Step 2: Run integration tests**

Run: `uv run pytest tests/integration/ -m integration -v`
Expected: Tests pass (Graphiti mocked, no external deps needed)

**Step 3: Commit**

```bash
git add tests/integration/
git commit -m "test: add integration tests — in-memory MCP tool verification"
```

---

### Task 12: Lint, type check, and final verification

**Files:**
- Modify: any files with lint/type issues

**Step 1: Run ruff**

Run: `uv run ruff check --fix . && uv run ruff format .`
Fix any issues found.

**Step 2: Run pyright**

Run: `uv run pyright src/`
Note pre-existing graphiti-core type issues (expected, same as biosciences-mcp).

**Step 3: Run full test suite**

Run: `uv run pytest -v`
Expected: All unit and integration tests pass.

**Step 4: Commit any lint fixes**

```bash
git add -A
git commit -m "chore: apply ruff lint and format fixes"
```

---

### Task 13: Update .env.example and README

**Files:**
- Modify: `biosciences-memory/.env.example`
- Modify: `biosciences-memory/README.md`

**Step 1: Update .env.example**

Ensure it has placeholder values for all required env vars (no real keys):

```
# Neo4j Connection
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your-password-here
NEO4J_DATABASE=neo4j

# OpenAI
OPENAI_API_KEY=sk-your-key-here

# Server
SEMAPHORE_LIMIT=10
GRAPHITI_GROUP_ID=main

# Logging
OPENAI_LOG_LEVEL=DEBUG
HTTPX_LOG_LEVEL=INFO
GRAPHITI_LOG_LEVEL=DEBUG
```

**Step 2: Update README**

Add sections for: installation, running the server (factory + CLI), running tests, configuration.

**Step 3: Commit**

```bash
git add .env.example README.md
git commit -m "docs: update .env.example and README for graphiti-fastmcp migration"
```

---

### Task 14: Update migration tracker + Linear + Graphiti

**Files:**
- Modify: `biosciences-program/migration-tracker.md`

**Step 1: Update Wave 4 tracker**

Mark graphiti-fastmcp rows as complete. Update commit reference.

**Step 2: Comment on AGE-195**

Use `mcp__plugin_linear_linear__create_comment` with migration details.

**Step 3: Add Graphiti episode**

Use `mcp__graphiti-docker__add_memory` to record migration completion.

**Step 4: Push both repos**

```bash
cd /home/donbr/open-biosciences/biosciences-memory && git push origin main
cd /home/donbr/open-biosciences/biosciences-program && git push origin main
```

**Step 5: Commit tracker**

```bash
cd /home/donbr/open-biosciences/biosciences-program
git add migration-tracker.md
git commit -m "docs: update migration tracker — graphiti-fastmcp migration complete"
git push origin main
```
