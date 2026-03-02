# Open Biosciences 🧬

> AI agents grounded in verifiable biological truth — from fuzzy questions to validated knowledge graphs.

Open Biosciences is an open-source platform for accelerating drug discovery and biomedical research by connecting AI agents to the world's authoritative life-sciences databases. Researchers ask questions in natural language. The platform resolves them to canonical identifiers, validates every claim against primary sources, and returns traceable knowledge graphs — not hallucinated summaries.

→ Governance decisions (ADRs, specs, SpecKit): [biosciences-program](https://github.com/open-biosciences/biosciences-program) | Repository Analyzer Framework: [biosciences-architecture](https://github.com/open-biosciences/biosciences-architecture)

---

## Why This Exists

Life sciences researchers spend 60–80% of their time on data wrangling: querying fragmented APIs, reconciling incompatible identifiers (BRCA1 is `HGNC:1100` in HGNC, `P38398` in UniProt, `ENSG00000012048` in Ensembl), and stitching together facts that should flow seamlessly. AI assistants promise to automate this — but a bare LLM will confidently invent drug mechanisms, fabricate trial IDs, and conflate gene symbols. The cost of a wrong answer in biomedicine is not a bad chatbot experience; it is wasted lab time or a misleading grant application.

The platform enforces a **Fuzzy-to-Fact protocol**: every entity is first searched in natural language, then resolved to a canonical identifier (a CURIE), then validated against the authoritative database. No claim leaves the system unverified.

→ Technical specification: [ADR-001 v1.4](https://github.com/open-biosciences/biosciences-program)

---

## Origin

This platform evolved through three distinct implementations before becoming a public organization.

**Chapter 1 — Prototype (Claude Code Skills, 2025):** Six domain skills (`.claude/skills/`) in a single private repo proved the Fuzzy-to-Fact protocol worked. Researchers could resolve gene symbols, fetch drug-target associations, and build knowledge graph fragments — all from the Claude Code CLI. The limitation: everything was session-scoped and terminal-only.

**Chapter 2 — Interactive Research (Deep Agents):** The prototype grew into a LangGraph supervisor with 7 specialist subagents — one per research phase (ANCHOR → ENRICH → EXPAND → TRAVERSE → VALIDATE → PERSIST). A streaming React UI made the reasoning visible. Think-Act-Observe loops made it rigorous. The FOP drug-discovery walkthrough (resolving "ACVR1" to `HGNC:171`, finding Palovarotene in Open Targets, verifying `NCT03312634` in ClinicalTrials.gov) demonstrated the protocol at clinical depth.

**Chapter 3 — Production (Temporal Workflows):** PydanticAI agents wired to Temporal.io activities brought durable execution: workflows survive crashes, retry failed API calls, and run batch pipelines at scale. The same 12-database MCP backbone underlies all three implementations.

**The org migration** is the moment all three converge. `lifesciences-research` became `biosciences-mcp` (the API layer) and `biosciences-program` (the governance). The agents became `biosciences-deepagents` and `biosciences-temporal`. The skills became `biosciences-skills`. A private experiment became a public platform.

→ Migration progress: [migration-tracker.md](migration-tracker.md)

---

## The Platform

Thirteen repositories under the [open-biosciences](https://github.com/open-biosciences) GitHub organization, operated by a 9-agent team with clear ownership boundaries.

| Repository | Role | Wave |
|------------|------|------|
| [biosciences-program](https://github.com/open-biosciences/biosciences-program) | ADRs, specs, SpecKit governance, migration tracking, agent team definitions | ✅ Wave 1 |
| [biosciences-architecture](https://github.com/open-biosciences/biosciences-architecture) | Repository Analyzer Framework | ✅ Wave 1 |
| [biosciences-skills](https://github.com/open-biosciences/biosciences-skills) | 6 domain skills, Graphiti commands | ✅ Wave 1 |
| [platform-skills](https://github.com/open-biosciences/platform-skills) | Scaffold commands, security review (platform skills) | ✅ Wave 1 |
| [biosciences-mcp](https://github.com/open-biosciences/biosciences-mcp) | 12 FastMCP servers, 697+ tests, unified gateway | ✅ Wave 2 |
| [biosciences-memory](https://github.com/open-biosciences/biosciences-memory) | Graphiti + Neo4j knowledge graph layer | ✅ Wave 2 |
| [biosciences-deepagents](https://github.com/open-biosciences/biosciences-deepagents) | LangGraph supervisor + 7 specialists, React UI | ✅ Wave 3 |
| [biosciences-temporal](https://github.com/open-biosciences/biosciences-temporal) | PydanticAI agents, Temporal.io durable workflows | ✅ Wave 3 |
| [biosciences-evaluation](https://github.com/open-biosciences/biosciences-evaluation) | Evaluation rubrics, quality metrics | ⬜ Wave 4 |
| [biosciences-research](https://github.com/open-biosciences/biosciences-research) | Competency questions, graph-builder workflows | 🟡 Wave 4 |
| [biosciences-education](https://github.com/open-biosciences/biosciences-education) | Training materials, tutorials, onboarding | ⬜ Wave 4 |
| [biosciences-workspace-template](https://github.com/open-biosciences/biosciences-workspace-template) | Bootstrap scripts, workspace config templates | ⬜ Wave 4 |
| [knowledge-work-plugins](https://github.com/anthropics/knowledge-work-plugins) | Cowork plugin collection (bio-research plugin wraps biosciences-mcp) | ✅ |

→ Full agent team definitions: [AGENTS.md](AGENTS.md)

---

## Workspace

Eleven active repositories organized into four migration waves, plus four legacy predecessor repos kept as read-only references.

```
🧬 open-biosciences/
│
├── 🏛️ FOUNDATION  (Wave 1 ✅ complete)
│   📋 program          coordination, migration tracking, agent team
│   🏗️ architecture     Repository Analyzer Framework
│   ⚡ skills           6 domain skills, Graphiti commands
│   🔧 platform-skills  scaffold commands, security review
│
├── 🔌 PLATFORM  (Wave 2 ✅ complete)
│   🔌 mcp              12 FastMCP servers · 697+ tests · unified gateway
│   🧠 memory           Graphiti + Neo4j knowledge graph layer
│
├── 🤖 ORCHESTRATION  (Wave 3 ✅ complete)
│   🤖 deepagents       LangGraph supervisor · 7 specialists · React UI
│   ⏱️ temporal         PydanticAI agents · Temporal.io durable workflows
│
├── 🔬 VALIDATION & EDUCATION  (Wave 4 🟡 in progress)
│   🔬 research         competency questions · graph-builder workflows
│   📊 evaluation       quality metrics · evaluation rubrics
│   📚 education        tutorials · onboarding guides
│   🧰 workspace-template  bootstrap scripts · workspace config
│
└── 📦 LEGACY PREDECESSORS  (read-only migration sources)
    📦 lifesciences-research    → architecture · mcp · skills
    📦 lifesciences-deepagents  → deepagents
    📦 lifesciences-temporal    → temporal
    📦 graphiti-fastmcp         → memory (Wave 4)
```

---

## Research Scenarios

The platform is built around concrete research questions: *"What are the synthetic lethal partners for ARID1A in ovarian cancer?"* or *"What drugs target the ACVR1 pathway in fibrodysplasia ossificans progressiva?"* Each question exercises the full 12-database stack — gene resolution, protein interactions, drug candidates, clinical trials — and produces a validated knowledge graph with full provenance.

→ Competency questions catalog: [biosciences-research](https://github.com/open-biosciences/biosciences-research)
→ Live API layer (12 servers): [biosciences-mcp](https://github.com/open-biosciences/biosciences-mcp)

---

## Get Involved

The platform is MIT-licensed and structured for external contribution. The [knowledge-work-plugins](https://github.com/anthropics/knowledge-work-plugins) collection includes a bio-research plugin that wraps the biosciences-mcp 34-tool gateway. Here are three entry points depending on your background:

**Researcher** — Start with the competency questions catalog in [biosciences-research](https://github.com/open-biosciences/biosciences-research). Each scenario documents the research question, the databases queried, and re-run instructions. You can run any scenario today against the live MCP layer.

**Developer** — The MCP layer is the most accessible contribution surface. Each server is an independent FastMCP wrapper for a public API. If you work with a database not yet covered (KEGG, OMIM, Orphanet), the scaffold commands in [platform-skills](https://github.com/open-biosciences/platform-skills) generate a compliant server skeleton from a spec. Adding a new MCP server is a self-contained, well-scoped contribution.

**AI/LLM Builder** — The Fuzzy-to-Fact protocol (ADR-001) and SpecKit SDLC (ADR-003) are reusable patterns, not just biosciences artifacts. The architecture documents in [biosciences-program](https://github.com/open-biosciences/biosciences-program) explain the design decisions. The 9-agent team topology is a template for how to structure AI-assisted development of complex platforms.

→ Scaffold commands: [platform-skills](https://github.com/open-biosciences/platform-skills)
→ Architecture ADRs: [biosciences-program](https://github.com/open-biosciences/biosciences-program)

---

## Built On Open Science

This platform wraps 20+ years of bioinformatics API work. STRING, STITCH, NCATS Translator, and BioThings Explorer established the patterns for federated biological data access that make this platform possible. We do not redistribute upstream data — we wrap public APIs and return live results. Every MCP server is a lightweight client; the authoritative data remains with the institutions that curate it.

The novel contributions are methodological: the Fuzzy-to-Fact protocol, the SpecKit specification-driven SDLC, the Agentic Biolink schema (a unified JSON envelope across 12 heterogeneous APIs), and the 9-agent team topology for AI-assisted platform development.

→ Prior art and research context: [biosciences-program](https://github.com/open-biosciences/biosciences-program)

---

## Acknowledgements

This project queries public APIs maintained by rigorous scientific institutions. We gratefully acknowledge their contributions:

- **[HGNC](https://www.genenames.org/)** — HUGO Gene Nomenclature Committee, EMBL-EBI
- **[UniProt](https://www.uniprot.org/)** — UniProt Consortium
- **[ChEMBL](https://www.ebi.ac.uk/chembl/)** — European Bioinformatics Institute (EMBL-EBI)
- **[Open Targets](https://platform.opentargets.org/)** — EMBL-EBI, Wellcome Sanger Institute, GSK
- **[STRING](https://string-db.org/)** — STRING Consortium
- **[BioGRID](https://thebiogrid.org/)** — Tyers Lab, University of Montreal
- **[IUPHAR/GtoPdb](https://www.guidetopharmacology.org/)** — International Union of Basic and Clinical Pharmacology
- **[PubChem](https://pubchem.ncbi.nlm.nih.gov/)** — National Center for Biotechnology Information (NCBI)
- **[WikiPathways](https://www.wikipathways.org/)** — WikiPathways Community
- **[ClinicalTrials.gov](https://clinicaltrials.gov/)** — U.S. National Library of Medicine
- **[Ensembl](https://www.ensembl.org/)** — EMBL-EBI
- **[NCBI Gene](https://www.ncbi.nlm.nih.gov/gene)** — National Center for Biotechnology Information

---

## License

MIT. The platform code in this organization is open source under the MIT License. The data returned by each API server remains subject to the terms of the upstream data provider — this platform wraps public APIs and does not redistribute their content.
