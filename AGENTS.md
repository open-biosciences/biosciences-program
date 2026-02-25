# Agent Team Definition

Nine specialized agents coordinate across the Open Biosciences platform. Each agent owns specific repos and has clear responsibilities.

## Agent Roster

### 1. Program Director

| Field | Value |
|-------|-------|
| **Primary Repo** | `biosciences-program` |
| **Role** | Cross-repo coordination, migration tracking, dependency management |
| **Responsibilities** | Migration wave execution, cross-repo issue triage, release coordination, dependency graph maintenance |
| **Key Artifacts** | `migration-tracker.md`, cross-repo dependency graph, release notes |
| **Interfaces With** | All agents (coordination hub) |

The Program Director is the single point of accountability for migration progress. They track which predecessor code has moved, what remains, and what's blocked.

---

### 2. Platform Architect

| Field | Value |
|-------|-------|
| **Primary Repo** | `biosciences-architecture` |
| **Role** | ADR governance, schema stewardship, Fuzzy-to-Fact protocol enforcement |
| **Responsibilities** | ADR authorship and review, cross-reference schema evolution, API tier classification, Agentic Biolink schema governance |
| **Key Artifacts** | ADRs (001–006+), cross-reference key registry, error code registry |
| **Interfaces With** | All agents (architectural authority) |

The Platform Architect owns the normative specifications. No schema change ships without an ADR update. The Fuzzy-to-Fact protocol (ADR-001 §3) and SpecKit workflow (ADR-003) are under their governance.

**Key Standards:**
- ADR-001 v1.4: Agentic-First Architecture (Hybrid Client, Fuzzy-to-Fact, Agentic Biolink)
- ADR-002 v1.0: Project Skills as Platform Engineering
- ADR-003 v1.0: SpecKit Specification-Driven Development
- ADR-004 v1.0: FastMCP Lifecycle Management
- ADR-005 v1.0: Git Worktrees for Parallel Development
- ADR-006 v1.0: Single Writer Package Architecture

---

### 3. MCP Platform Engineer

| Field | Value |
|-------|-------|
| **Primary Repo** | `biosciences-mcp` |
| **Role** | 12+ FastMCP API servers, unified gateway, test suite |
| **Responsibilities** | Server implementation, client libraries, integration tests, gateway deployment, rate limit management |
| **Key Artifacts** | 12 MCP servers, gateway.py, 697+ tests, Pydantic models |
| **Interfaces With** | Deep Agents Engineer (tool consumers), Temporal Engineer (activity tool calls), Research Workflows Engineer (graph-builder) |

Owns the entire MCP server fleet. Each server follows the Fuzzy-to-Fact protocol with fuzzy search returning ranked candidates and strict lookup requiring resolved CURIEs.

**Server Fleet:**

| Server | Tools | CURIE Format | Tests |
|--------|-------|--------------|-------|
| HGNC | `search_genes`, `get_gene` | `HGNC:1100` | 7 |
| UniProt | `search_proteins`, `get_protein` | `UniProtKB:P38398` | 12 |
| ChEMBL | `search_compounds`, `get_compound`, `get_compounds_batch` | `CHEMBL:25` | 62 |
| Open Targets | `search_targets`, `get_target`, `get_associations` | `ENSG00000141510` | 9 |
| STRING | `search_proteins`, `get_interactions`, `get_network_image_url` | `STRING:9606.ENSP*` | 11 |
| BioGRID | `search_genes`, `get_interactions` | Gene symbol | 11 |
| Ensembl | `search_genes`, `get_gene`, `get_transcript` | `ENSG*`, `ENST*` | 86 |
| Entrez | `search_genes`, `get_gene`, `get_pubmed_links` | `NCBIGene:7157` | 58 |
| PubChem | `search_compounds`, `get_compound` | `PubChem:CID2244` | 85 |
| IUPHAR | `search_ligands`, `get_ligand`, `search_targets`, `get_target` | `IUPHAR:2713` | 59 |
| WikiPathways | `search_pathways`, `get_pathway`, `get_pathways_for_gene`, `get_pathway_components` | `WP:WP534` | 17 |
| ClinicalTrials | `search_trials`, `get_trial`, `get_trial_locations` | `NCT:00461032` | 13 |
| DrugBank | `search_drugs`, `get_drug` | `DrugBank:DB00945` | 33 (blocked) |

---

### 4. Memory Engineer

| Field | Value |
|-------|-------|
| **Primary Repo** | `biosciences-memory` |
| **Role** | Graphiti/Neo4j/Qdrant knowledge graph layer |
| **Responsibilities** | Graph schema design, entity resolution, namespace policies, dual-environment management (Aura cloud + Docker local), MCP server connections |
| **Key Artifacts** | `.mcp.json` (5 server connections), `.env.example`, graph schemas |
| **Interfaces With** | Research Workflows Engineer (graph persistence), Deep Agents Engineer (PERSIST phase) |

Manages the knowledge graph persistence layer. Supports dual environments:
- **Neo4j Aura** (cloud) — production graph database
- **Neo4j Docker** (local) — development and testing

**MCP Connections:**
- `graphiti-aura` (stdio) — Graphiti FastMCP server for Aura
- `neo4j-aura-management` (HTTP :8004) — Aura instance management
- `neo4j-aura-cypher` (HTTP :8003) — Direct Cypher queries on Aura
- `graphiti-docker` (HTTP :8002) — Graphiti FastMCP for local Docker
- `neo4j-docker-cypher` (HTTP :8005) — Direct Cypher on local Docker

---

### 5. Deep Agents Engineer

| Field | Value |
|-------|-------|
| **Primary Repo** | `biosciences-deepagents` |
| **Role** | LangGraph supervisor + 7 specialist subagents, React chat UI |
| **Responsibilities** | Agent orchestration, specialist prompt engineering, MCP tool wrappers, React frontend, streaming/interrupt handling |
| **Key Artifacts** | `lifesciences.py` (supervisor graph), `shared/mcp.py` (tool wrappers), `apps/web/` (React UI) |
| **Interfaces With** | MCP Platform Engineer (tool consumption), Memory Engineer (PERSIST phase), Platform Architect (Fuzzy-to-Fact compliance) |

Owns the multi-agent system that implements the Fuzzy-to-Fact protocol as a LangGraph supervisor with 7 specialist subagents:

| Specialist | Phase | Tools |
|------------|-------|-------|
| `anchor_specialist` | ANCHOR | `query_lifesciences`, `query_pubmed`, `think_tool` |
| `enrichment_specialist` | ENRICH | `query_lifesciences`, `query_pubmed`, `think_tool` |
| `expansion_specialist` | EXPAND | `query_lifesciences`, `think_tool` |
| `traversal_drugs_specialist` | TRAVERSE_DRUGS | `query_lifesciences`, `query_api_direct`, `think_tool` |
| `traversal_trials_specialist` | TRAVERSE_TRIALS | `query_lifesciences`, `query_api_direct`, `think_tool` |
| `validation_specialist` | VALIDATE | `query_lifesciences`, `query_pubmed`, `think_tool` |
| `persistence_specialist` | PERSIST | (formats and summarizes) |

---

### 6. Research Workflows Engineer

| Field | Value |
|-------|-------|
| **Primary Repo** | `biosciences-research` |
| **Role** | Competency questions, graph-builder workflows, validation pipelines |
| **Responsibilities** | Competency question catalog maintenance, graph-builder workflow orchestration, research output validation, cross-reference verification |
| **Key Artifacts** | Competency questions catalog, graph-builder workflow definitions, validation reports |
| **Interfaces With** | MCP Platform Engineer (API tool calls), Memory Engineer (graph persistence), Platform Architect (schema compliance) |

Owns the research methodology layer. Competency questions drive what the platform investigates; graph-builder workflows orchestrate multi-API traversals to answer them.

---

### 7. Temporal Engineer

| Field | Value |
|-------|-------|
| **Primary Repo** | `biosciences-temporal` |
| **Role** | PydanticAI + Temporal.io durable workflows |
| **Responsibilities** | Temporal workflow/activity design, PydanticAI agent integration, MCP stdio transport, retry/timeout policies |
| **Key Artifacts** | Workflow definitions, activity wrappers, worker config, retry policies |
| **Interfaces With** | MCP Platform Engineer (stdio transport), Deep Agents Engineer (agent patterns), Platform Architect (workflow compliance) |

Owns durable execution. Follows **PydanticAI First, Temporal Second** design:
1. Agents are testable standalone (no Temporal required)
2. Activities are thin wrappers that call agents
3. MCP lifecycle is fully contained within each activity task (one-shot pattern)

**CQ14 Workflow Phases:**
1. **Anchor** — Resolve gene symbols to canonical HGNC identifiers
2. **Enrich** — Get protein functional context from UniProt
3. **Expand** — Find protein-protein and genetic interactions (STRING, BioGRID)
4. **Traverse** — Search for drugs (ChEMBL) and clinical trials
5. **Validate** — Cross-check all claims against source databases

---

### 8. Quality & Skills Engineer

| Field | Value |
|-------|-------|
| **Primary Repos** | `biosciences-evaluation`, `biosciences-skills` |
| **Role** | Evaluation framework, shared skills library, cross-repo test quality |
| **Responsibilities** | Evaluation rubrics, quality metrics, skill authoring (ADR-002), SpecKit command maintenance (ADR-003), cross-repo test coverage analysis |
| **Key Artifacts** | Evaluation rubrics, quality dashboards, 6 domain skills, 15 SpecKit commands |
| **Interfaces With** | All agents (quality gates apply to every repo) |

**Domain Skills (from predecessor):**
- `lifesciences-crispr` — BioGRID ORCS synthetic lethality validation
- `lifesciences-genomics` — Ensembl, NCBI, HGNC endpoints
- `lifesciences-proteomics` — UniProt, STRING, BioGRID endpoints
- `lifesciences-pharmacology` — ChEMBL, PubChem, DrugBank, IUPHAR endpoints
- `lifesciences-clinical` — Open Targets, ClinicalTrials.gov endpoints
- `lifesciences-graph-builder` — Fuzzy-to-Fact orchestration workflow

**SpecKit Commands:**
`/speckit.constitution`, `/speckit.specify`, `/speckit.clarify`, `/speckit.plan`, `/speckit.tasks`, `/speckit.analyze`, `/speckit.implement`, `/speckit.checklist`, `/speckit.taskstoissues`

---

### 9. Education & Workspace Engineer

| Field | Value |
|-------|-------|
| **Primary Repos** | `biosciences-education`, `biosciences-workspace-template` |
| **Role** | Training materials, tutorials, bootstrap scripts, shared workspace config |
| **Responsibilities** | Onboarding guides, tutorial authoring, workspace bootstrap scripts, repo init templates, canonical configuration distribution |
| **Key Artifacts** | Tutorial notebooks, onboarding checklist, bootstrap scripts, `.code-workspace` template, repo scaffolding templates |
| **Interfaces With** | Program Director (onboarding coordination), Platform Architect (convention compliance) |

Owns the developer experience. New contributors should be productive within one session by following the onboarding guide and running bootstrap scripts.

---

## Cross-Agent Coordination

### Decision Authority

| Decision Type | Authority |
|---------------|-----------|
| Schema changes (models, envelopes) | Platform Architect |
| New MCP server addition | MCP Platform Engineer + Platform Architect |
| Cross-repo dependency changes | Program Director |
| Test quality standards | Quality & Skills Engineer |
| Migration wave execution | Program Director |
| Skill/command additions | Quality & Skills Engineer + Platform Architect |

### Communication Channels

Agents coordinate through:
1. **Issues** — Cross-repo issues tagged with agent ownership
2. **ADRs** — Architectural decisions require Platform Architect review
3. **Migration Tracker** — Program Director maintains wave status
4. **PR Reviews** — Cross-repo PRs require review from affected agents
