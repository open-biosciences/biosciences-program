# Visual Artifact Suite

> Diagrams and visual guides for the Open Biosciences platform, organized by audience.

Three layers of visual communication, each designed to stand alone:

1. **Ecosystem Positioning** — Where Open Biosciences fits in the life sciences landscape
2. **Platform Architecture** — How the 12 repos, 9 agents, and dependencies connect
3. **Research Workflows** — What questions you can answer and how data flows

---

## Layer 1: Ecosystem Positioning

For the open-source bio community — understanding the platform's role and value.

| Artifact | Format | Description |
|----------|--------|-------------|
| [ecosystem-map.excalidraw](ecosystem-map.excalidraw) | Excalidraw | Three-band ecosystem map: authoritative databases (bottom) → Open Biosciences platform (middle) → community touchpoints (top). Shows the "storefront vs. workshop" relationship between `anthropics/life-sciences` marketplace (5 servers) and Open Biosciences (12 servers, dual orchestrators, knowledge graph). |
| [ecosystem-positioning.md](ecosystem-positioning.md) | Mermaid | Two flowcharts: (1) Researcher journey from marketplace discovery to full platform capabilities, (2) Developer/contributor journey from scaffold commands to merged pull request. |

---

## Layer 2: Platform Architecture

For contributors — understanding repo structure, agent ownership, and data flow.

| Artifact | Format | Description |
|----------|--------|-------------|
| [platform-architecture.excalidraw](platform-architecture.excalidraw) | Excalidraw | Four horizontal swim lanes (Foundation → Platform → Orchestration → Validation) with 12 repos as boxes, agent assignments, and dependency arrows between layers. Color-coded by wave. |
| [repo-dependency-graph.md](repo-dependency-graph.md) | Mermaid | Directed graph of all 12 repos showing schema dependencies, MCP tool consumption, and graph persistence edges. Wave-colored subgraphs with dependency rules table. |
| [agent-ownership-map.md](agent-ownership-map.md) | Mermaid | Left-to-right map connecting 9 agents to their 12 repos with responsibility summaries and decision authority matrix. |
| [data-flow.md](data-flow.md) | Mermaid | Sequence diagram of the Fuzzy-to-Fact protocol: natural language → fuzzy search → CURIE resolution → strict lookup → orchestration phases (ANCHOR through PERSIST) → knowledge graph. Uses BRCA1 drug-target example. |

---

## Layer 3: Research Workflows

For researchers — what questions the platform answers and how databases connect.

| Artifact | Format | Description |
|----------|--------|-------------|
| [fuzzy-to-fact-journey.excalidraw](fuzzy-to-fact-journey.excalidraw) | Excalidraw | Left-to-right visual journey through the Fuzzy-to-Fact protocol using the ACVR1/FOP drug discovery example. Fuzzy input → candidate resolution → database fan-out (12 MCP servers) → validated knowledge graph with real CURIEs. |
| [mcp-server-tiers.md](mcp-server-tiers.md) | Mermaid | 5-tier layered diagram of all 12 MCP servers with tool counts, example CURIEs, and cross-reference edges. Includes full server details table. |
| [research-workflow-sequence.md](research-workflow-sequence.md) | Mermaid | Complete sequence diagram of ACVR1/FOP research question through all 7 specialist subagents (Anchor → Enrichment → Expansion → Traversal → Validation → Persistence). Real biological data: HGNC:171, UniProt Q04771, Palovarotene, NCT03312634. |

---

## Formats

**Excalidraw** (`.excalidraw`) — Interactive, hand-drawn style diagrams. Open in [excalidraw.com](https://excalidraw.com) or the VS Code Excalidraw extension. Collaborative and editable — fork and modify freely.

**Mermaid** (`.md`) — Plain text diagrams in markdown code blocks. Render natively on GitHub, in VS Code (with Mermaid extension), or at [mermaid.live](https://mermaid.live). Edit with any text editor.

## Color Scheme

All diagrams use a consistent wave-based color palette:

| Wave | Fill | Stroke | Used For |
|------|------|--------|----------|
| Foundation (Wave 1) | `#b2f2bb` / `#d4edda` | `#22c55e` | architecture, skills, program, platform-skills |
| Platform (Wave 2) | `#a5d8ff` / `#cfe2ff` | `#4a9eed` | mcp, memory |
| Orchestration (Wave 3) | `#d0bfff` / `#e8d5ff` | `#8b5cf6` | deepagents, temporal |
| Validation (Wave 4) | `#ffd8a8` / `#fff3cd` | `#f59e0b` | evaluation, research, education, workspace-template |
| Knowledge Graph | `#c3fae8` | `#06b6d4` | Graphiti, Neo4j persistence |

## Accuracy

All biological data in these diagrams uses real identifiers verified against authoritative sources:

- **ACVR1**: HGNC:171, UniProt Q04771, Ensembl ENSG00000115170
- **BRCA1**: HGNC:1100, UniProt P38398, Ensembl ENSG00000012048
- **Palovarotene**: FDA approved 2023 for FOP (brand name Sohonos), trial NCT03312634
- **Olaparib**: CHEMBL:3545110, PARP inhibitor for BRCA1-related cancers

---

*Generated with Claude Code. All artifacts are MIT-licensed and designed for community contribution.*
