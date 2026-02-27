# Prior Art & Research Context: Life Sciences API Patterns

**Purpose:** Provide research context for the Life Sciences MCP architecture—positioning the current work within both historical prior art and the emerging 2025-2026 LLM knowledge augmentation landscape.

**Date:** 2026-01-10

**Audience:** Research agents, collaborators, and reviewers seeking to understand how this project fits within the broader field.

---

## Research Context

### The Problem Space

Large Language Models excel at reasoning but lack reliable access to authoritative, up-to-date biomedical knowledge. When asked about drug-target interactions, protein networks, or clinical trials, LLMs either hallucinate or rely on stale training data. The solution requires:

1. **Structured API access** to authoritative databases (STRING, ChEMBL, UniProt, etc.)
2. **Semantic typing** to enable graph-based reasoning across sources
3. **Token efficiency** to maximize useful context within LLM limits
4. **Error recovery** for autonomous agent operation

### This Project's Position

The **Life Sciences MCP Server** project creates a unified interface for 13 biomedical APIs, following the Model Context Protocol (MCP) standard. It sits at the intersection of:

- **Legacy:** 20+ years of bioinformatics API design (STRING, STITCH, NCATS Translator)
- **Present:** Retrieval-Augmented Generation (RAG) for biomedical LLMs
- **Emerging:** Agentic AI systems that autonomously traverse knowledge graphs

### What This Document Covers

| Section | Purpose |
|---------|---------|
| §1-2 | Historical prior art (STRING/STITCH patterns, publications) |
| §3-4 | Current standards (TRAPI, BioThings Explorer) |
| §5 | 2025-2026 LLM + API research landscape |
| §6-7 | Architecture alignment and novel contributions |

---

## Executive Summary

The Fuzzy-to-Fact protocol and Agentic Biolink schema are not novel inventions—they formalize patterns that life sciences databases have evolved over two decades. This document captures the prior art, key publications, and industry standards that validate our architectural decisions while highlighting areas where this project extends beyond existing work.

---

## 1. The STRING API Verb Taxonomy

STRING (Search Tool for the Retrieval of Interacting Genes/Proteins) has been refining its API since 2000. Their endpoint design reveals a mature taxonomy of "API verbs":

| API Verb Category | STRING Endpoint | Semantic Pattern | Our Mapping |
|-------------------|-----------------|------------------|-------------|
| **Resolve** | `/get_string_ids` | Ambiguous input → Canonical IDs | `search_*` (Fuzzy Phase 1) |
| **Retrieve** | `/network`, `/interaction_partners` | Canonical ID → Structured data | `get_*` (Strict Phase 2) |
| **Visualize** | `/image/network`, `/get_link` | Generate representations | Utility tools |
| **Enrich** | `/enrichment`, `/ppi_enrichment` | Derive statistical insights | Extended analysis |
| **Annotate** | `/functional_annotation` | Attach semantic context | Cross-reference population |
| **Homology** | `/homology`, `/homology_best` | Cross-species mapping | Federation pattern |

### Key Insight

STRING's `get_string_ids` endpoint **is** fuzzy search, despite using "get" in the name. It accepts gene names, synonyms, or UniProt IDs and returns ranked STRING identifiers. The verb naming is less important than the contract:

> **Fuzzy Phase:** Ambiguous input in → Ranked candidates out
> **Strict Phase:** Canonical ID in → Authoritative record out

Our choice of `search_*` vs `get_*` is actually more semantically honest than STRING's legacy REST naming.

---

## 2. Published Papers from STRING/STITCH Teams

### STRING Database Evolution

| Year | Paper | DOI | Key Contribution |
|------|-------|-----|------------------|
| **2025** | [STRING in 2025: protein networks with directionality of regulation](https://academic.oup.com/nar/article/53/D1/D730/7903368) | 10.1093/nar/gkae1113 | Regulatory networks with directionality; **fine-tuned language models** for literature parsing |
| **2023** | [STRING in 2023](https://academic.oup.com/nar/article/51/D1/D638/6825349) | 10.1093/nar/gkac1000 | Variational auto-encoders for co-expression; any-genome network creation |
| 2019 | STRING v11 | 10.1093/nar/gky1131 | Physical vs functional network distinction |
| 2017 | STRING v10.5 | 10.1093/nar/gkw937 | Improved scoring, organism coverage |
| 2015 | STRING v10 | 10.1093/nar/gku1003 | New evidence channels |

### STITCH Database (Chemical-Protein Interactions)

| Year | Paper | DOI | Key Contribution |
|------|-------|-----|------------------|
| **2016** | [STITCH 5](https://academic.oup.com/nar/article/44/D1/D380/2503089) | 10.1093/nar/gkv1277 | Tissue filtering, binding affinity visualization |
| 2014 | STITCH 4 | 10.1093/nar/gkt1207 | User data integration |
| 2012 | STITCH 3 | 10.1093/nar/gkr1011 | Improved predictions |
| 2010 | STITCH 2 | 10.1093/nar/gkp962 | Pathway integration |
| 2008 | STITCH (original) | 10.1093/nar/gkm795 | Chemical-protein interaction networks |

**Note:** STITCH has not been updated since 2016. Its functionality has been largely absorbed by ChEMBL and PubChem with better maintenance. Our decision to mark STITCH as "out of scope" is validated by this publication gap.

### STRING 2025: LLM Integration Precedent

The STRING 2025 paper is particularly relevant:

> "We gather evidence on the type and directionality of interactions using curated pathway databases and **a fine-tuned language model parsing the literature**."

This validates our "LLM as consumer of structured API data" pattern. STRING's own team now uses language models for knowledge extraction, creating a feedback loop between structured databases and LLM capabilities.

---

## 3. NCATS Translator and TRAPI Standard

The [Biomedical Data Translator](https://ncatstranslator.github.io/TranslatorTechnicalDocumentation/) program, funded by NCATS, has formalized biomedical API patterns into the **Translator Reasoner API (TRAPI)**:

### TRAPI Core Principles

| Principle | TRAPI Implementation | Our Implementation |
|-----------|---------------------|-------------------|
| **Canonical Identifiers** | CURIEs required for all IDs | ADR-001 §5: `PREFIX:LOCAL_ID` format |
| **Semantic Typing** | Biolink model categories | Agentic Biolink schema |
| **Query/Result Binding** | Graph pattern matching | Fuzzy-to-Fact protocol |
| **Structured Responses** | JSON message format | `PaginationEnvelope`, `ErrorEnvelope` |

### TRAPI OpenAPI Specification

TRAPI defines standard operations:
- `/query` - Submit knowledge queries
- `/meta_knowledge_graph` - Describe available data
- `/overlay` - Compute enrichments

Source: [NCATS ReasonerAPI GitHub](https://github.com/NCATSTranslator/ReasonerAPI)

---

## 4. BioThings Explorer: API Federation

[BioThings Explorer](https://academic.oup.com/bioinformatics/article/39/9/btad570/7273783) (Callaghan et al., 2023) provides the theoretical foundation for API federation:

> "BioThings Explorer is an application that can query a virtual, federated knowledge graph derived from the aggregated information in a network of biomedical web services."

### Key Architecture Patterns

1. **Semantic Annotations** - Precise input/output typing for each API
2. **Chained Queries** - Automated multi-step graph traversal
3. **Dynamic Retrieval** - No centralized database; query-time federation

### BTE-RAG (2025)

The 2025 [Federated Knowledge Retrieval](https://www.biorxiv.org/content/10.1101/2025.08.01.668022v1.full) paper demonstrates BioThings Explorer with LLMs:

> "BTE-RAG integrating 61 authoritative biomedical APIs... increased accuracy from 51% to 75.8% for GPT-4o mini and from 69.8% to 78.6% for GPT-4o."

This validates our MCP server approach—structured API access demonstrably improves LLM performance on biomedical tasks.

---

## 4.5 The CURIE Standard Foundation

CURIEs (Compact URIs) provide the identifier interoperability layer for this architecture. Understanding the formal specifications underpinning CURIEs is essential for proper implementation.

### W3C Specification

The [W3C CURIE Syntax 1.0](https://www.w3.org/TR/2010/NOTE-curie-20101216/) (2010) formally defines:
- **Format:** `PREFIX:LOCAL_ID` (e.g., `HGNC:1100`, `UniProtKB:P38398`)
- **Safe CURIEs:** Bracketed form `[PREFIX:LOCAL_ID]` for unambiguous parsing
- Integration with [JSON-LD 1.1](https://www.w3.org/TR/json-ld11/) for Linked Data contexts

### Authoritative Registries

| Registry | Purpose | URL |
|----------|---------|-----|
| **Bioregistry** | Canonical prefix definitions for life sciences | https://bioregistry.io/ |
| **Identifiers.org** | CURIE resolution service | https://identifiers.org/ |
| **Node Normalizer** | NCATS Translator ID transformation | https://github.com/TranslatorSRI/NodeNormalization |

### Transformation Pattern

CURIEs are used at the API layer for interoperability, then transformed for underlying database calls:

```
MCP Tool Layer:     get_gene("HGNC:11110")
                          │
                          ▼ strip prefix
HGNC REST API:      /fetch/hgnc_id/11110
```

This pattern is explicitly documented in BioThings Explorer (Callaghan et al., 2023):

> "BioThings Explorer also performs **ID-to-object translation**, which facilitates the chaining of API calls from one step in the query-path to the next step. This ID translation step is critical when successive APIs in the query-path plan use different identifiers to represent the same biomedical entity."

### Validation Resources

| Resource | Purpose |
|----------|---------|
| [Bioregistry Prefix Lookup](https://bioregistry.io/registry/) | Validate CURIE prefix canonicalization |
| [Biopragmatics CURIE Guide](https://cthoyt.com/2021/09/14/curies.html) | Practical tutorial on CURIE usage |
| ADR-001 §A Cross-Reference Registry | Project-specific CURIE format specifications |

---

## 5. 2025-2026 LLM + API Research

### Key Papers on Knowledge-Augmented LLMs

| Paper | Finding | Relevance |
|-------|---------|-----------|
| [RAG in Biomedicine Systematic Review](https://academic.oup.com/jamia/article/32/4/605/7954485) (2025) | RAG shows 1.35 odds ratio improvement over baseline LLMs | Validates structured retrieval approach |
| [Gene-to-Phenotype with Full-Text Access](https://www.biorxiv.org/content/10.1101/2025.06.11.659165v1.full) (2025) | Specialized LLMs with databases achieved >80% accuracy | Database access is critical for accuracy |
| [DALK: Dynamic Co-Augmentation](https://arxiv.org/html/2405.04819v1) (2024) | LLM + KG co-augmentation for Alzheimer's research | Demonstrates domain-specific KG value |
| [BioLunar Framework](https://arxiv.org/abs/2406.18626) (2024) | Molecular evidence enrichment for biomarker discovery | Complex reasoning across evidence spaces |

### Emerging Best Practices

1. **Structured JSON over free text** - APIs returning semantic triples outperform document retrieval
2. **Evidence attribution** - Including source citations reduces hallucination
3. **Domain-specific databases** - Curated sources (IUPHAR, ChEMBL) outperform general web search
4. **Token efficiency** - Slim/compact responses essential for context window management

---

## 6. Architecture Alignment Summary

| Our Pattern | Industry Precedent | Validation |
|-------------|-------------------|------------|
| `search_*` (fuzzy) | STRING `/resolve`, TRAPI query binding | 20+ years of refinement |
| `get_*` (strict + CURIE) | STRING `/network`, TRAPI knowledge retrieval | Canonical ID requirement |
| `PaginationEnvelope` | Standard REST cursor pagination | Industry standard |
| `ErrorEnvelope` with recovery hints | TRAPI result metadata | Extended for agentic use |
| `cross_references` | Biolink cross-references, BTE federation | Enables graph traversal |
| `slim=True` | **Novel** - Token budgeting for LLMs | Our contribution |
| Recovery hints | **Extended** - Agentic self-healing | Our contribution |

---

## 7. Novel Contributions of This Project

While grounded in prior art, this project makes several contributions that extend beyond existing work:

### 7.1 Token Budgeting (`slim` Parameter)

**Prior Art:** Traditional APIs return full records regardless of use case.

**Our Innovation:** Every `search_*` and `get_*` tool accepts a `slim=True` parameter that returns minimal fields (~20 tokens/entity vs ~115-300 tokens). This is born from LLM context window constraints—a concern that didn't exist when STRING or TRAPI were designed.

**Impact:** Enables batch operations (e.g., resolving 50 gene symbols) within a single LLM turn without context overflow.

### 7.2 Recovery Hints in Error Envelopes

**Prior Art:** TRAPI and REST APIs return error codes with messages. Self-correction is left to the client.

**Our Innovation:** Every `ErrorEnvelope` includes a `recovery_hint` field that tells an autonomous agent exactly how to recover:

```json
{
  "error": {
    "code": "UNRESOLVED_ENTITY",
    "message": "Invalid CURIE format: 'brca1'",
    "recovery_hint": "Use search_genes('brca1') first to resolve to HGNC CURIE, then call get_gene() with the resolved ID."
  }
}
```

**Impact:** Agents can self-correct without human intervention, enabling autonomous multi-step workflows.

### 7.3 Fuzzy-to-Fact as Explicit Protocol

**Prior Art:** STRING has `/get_string_ids` → `/network` but doesn't name or formalize the pattern.

**Our Innovation:** We explicitly name and document the **Fuzzy-to-Fact Protocol** as an architectural principle:
- Phase 1 (Fuzzy): Ambiguous input → Ranked candidates with CURIEs
- Phase 2 (Strict): CURIE → Authoritative record

**Impact:** Makes the pattern teachable, testable, and enforceable across all 13 servers.

### 7.4 Cross-Reference Registry

**Prior Art:** Each database maintains its own cross-reference format.

**Our Innovation:** The **Agentic Biolink** schema defines a 23-key `cross_references` object that standardizes how entities link across databases:

```python
cross_references: {
    "hgnc": "HGNC:1100",
    "uniprot": ["P38398"],
    "ensembl_gene": "ENSG00000012048",
    "chembl": ["CHEMBL1824"],
    ...
}
```

**Impact:** Enables automated graph traversal: Gene → Protein → Drug → Trial without manual ID mapping.

### 7.5 Separation of Node Tools from Edge Skills

**Prior Art:** Traditional bioinformatics tools bundle node retrieval and relationship traversal together. BioThings Explorer federates at the API level, but doesn't distinguish between node and edge operations in its abstraction.

**Our Innovation:** The architecture separates concerns into distinct layers:

| Layer | Implementation | Purpose |
|-------|----------------|---------|
| **MCP Tools** | FastMCP servers | Verified node retrieval (entities with canonical CURIEs) |
| **Platform Skills** | `.claude/skills/` curl workflows | Relationship edge discovery, enrichment analysis |
| **Research Memory** | Graphiti MCP | Document research findings and validated facts |

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         GRAPH CONSTRUCTION KIT                          │
├─────────────────────────────────────────────────────────────────────────┤
│  TIER 1: MCP TOOLS (Verified Nodes)                                     │
│  ├── HGNC, UniProt, ChEMBL, STRING, Open Targets, WikiPathways          │
│  └── Purpose: Canonical entity resolution with cross-references         │
├─────────────────────────────────────────────────────────────────────────┤
│  TIER 2: CURL SKILLS (Relationship Edges)                               │
│  ├── ChEMBL /mechanism: Drug → Target                                   │
│  ├── Open Targets GraphQL: Gene → Disease associations                  │
│  ├── STRING /enrichment: Protein Set → GO/KEGG terms                    │
│  └── Purpose: Graph traversal without MCP overhead                      │
├─────────────────────────────────────────────────────────────────────────┤
│  TIER 3: GRAPHITI (Research Memory)                                     │
│  └── add_memory: Document validated findings as structured episodes     │
└─────────────────────────────────────────────────────────────────────────┘
```

**Rationale:** MCP tools carry protocol overhead (JSON-RPC, tool schemas, envelope parsing). For high-volume edge discovery operations (e.g., fetching 50 drug mechanisms), direct curl is more efficient. Skills provide these curl patterns as documented, copy-paste recipes.

**Note on Graphiti:** Graphiti is used here as a **research memory tool**—not to construct a production knowledge graph, but to document research findings, validated relationships, and discovered facts. It serves as a structured journal for the research process rather than a graph database for downstream applications.

**Available Skills:**
- `lifesciences-genomics` - Ensembl, NCBI, HGNC curl endpoints
- `lifesciences-proteomics` - UniProt, STRING, BioGRID curl endpoints
- `lifesciences-pharmacology` - ChEMBL, PubChem, IUPHAR curl endpoints
- `lifesciences-clinical` - Open Targets, ClinicalTrials.gov curl endpoints
- `lifesciences-graph-builder` - Orchestration workflow combining all tiers

**Impact:** Agents can choose the right tool for the job: MCP for verified entities, curl for bulk edges, Graphiti for research documentation.

### 7.6 Competency Questions as Validation Framework

**Prior Art:** The [BTE-RAG paper (2025)](https://www.biorxiv.org/content/10.1101/2025.08.01.668022v1.full) created benchmark datasets from DrugMechDB to evaluate LLM performance:

| Benchmark Type | Questions | Example Pattern |
|----------------|-----------|-----------------|
| Gene-centric mechanisms | 798 | "What is the mechanism by which [drug] affects [gene]?" |
| Metabolite effects | 201 | "How does [compound] influence [metabolic process]?" |
| Drug-biological process | 842 | "What biological process does [drug] modulate?" |

The [Knowledge Graph-Based Thought framework (2025)](https://academic.oup.com/gigascience/article/doi/10.1093/gigascience/giae082/7943459) uses KGs to validate LLM responses for drug-cancer associations.

The [Hybrid LLM-KG Framework (2025)](https://jastt.org/index.php/jasttpath/article/view/404) operates on 65,000+ entities and 3M relationships, translating natural language to Cypher queries.

**Our Innovation:** The `competency-questions-catalog.md` defines scenario-based research questions that drive graph construction:

| Scenario | Question Type | Validation Target |
|----------|---------------|-------------------|
| Synthetic Lethality | "How can we identify therapeutic strategies for ARID1A-deficient cancer?" | Gene→Gene (SL pairs) → Drug |
| Drug Safety Profiling | "What are off-target risks of Dasatinib?" | Drug → Off-targets → Safety genes |
| Orphan Drug Discovery | "What novel targets exist for Huntington's Disease?" | Disease → Gap analysis → Novel targets |
| Pathway Validation | "How do we validate the p53-MDM2-Nutlin axis?" | Gene → Gene → Drug (mechanism) |

**Alignment with Industry:**

| BTE-RAG Benchmark | Our Competency Question |
|-------------------|-------------------------|
| Gene-centric mechanisms | Scenario 1 (ARID1A → EZH2), Scenario 4 (TP53 → MDM2) |
| Drug-biological process | Scenario 2 (Dasatinib → DDR2 cardiotoxicity) |
| Metabolite effects | Scenario 3 (HTT → GLUT3 metabolism) |

**Impact:** Competency questions serve dual purposes:
1. **Scope Definition** - Define what the knowledge graph should answer
2. **Validation Framework** - Benchmark queries to verify graph completeness

See: `docs/competency-questions/competency-questions-catalog.md`

### 7.7 Self-Healing Knowledge Graphs

**Prior Art:** "Static" knowledge graphs (e.g., download an RDF dump) suffer from "Data Rot" (link rot, obsolete IDs) within months.

**Our Innovation:** The "Self-Healing" pattern (validated in CQ7) treats ID resolution failures not as exceptions, but as triggers for re-grounding:
1.  **Detect**: `MONDO:0014109` -> "Obsolete"
2.  **Search**: Web Search/LLM lookup for "MONDO:0014109 replacement"
3.  **Heal**: Update graph node to `MONDO:0800044` (NGLY1-deficiency)

**External Validation (2025):**
This aligns with 2024-2025 research on "Automated Construction and Refinement" of Medical KGs. The **Harvard Knowledge Graph Agent** (2025) and **KARE Framework** (2024) similarly use LLM-driven agents to continuously refine graphs, mitigating the "static" limitation of traditional databases.

### 7.8 Triangulated Validation

**Prior Art:** Systems rely on single-source truth (e.g., "If ChEMBL says X, X is true").

**Our Innovation:** We recognize that purely structured sources have "Silent Data Gaps" (e.g., recent Phase 3 drugs like Retatrutide often have 0 mechanisms in ChEMBL). The **Triangulated Validation** pattern mandates:
- **Source A**: Structured API (ChEMBL)
- **Source B**: Unstructured Search (Web/PubMed)
- **Logic**: If A is empty but B is rich -> Use B, flag A as outdated.

---

## 8. Why This Matters: Evidence for Structured API Access

### Quantitative Evidence

The 2025 research provides strong quantitative evidence that structured API access outperforms unstructured text retrieval:

| Study | Baseline | With Structured APIs | Improvement |
|-------|----------|---------------------|-------------|
| BTE-RAG (GPT-4o mini) | 51% | 75.8% | **+24.8 pp** |
| BTE-RAG (GPT-4o) | 69.8% | 78.6% | **+8.8 pp** |
| RAG Systematic Review | 1.0x | 1.35x | **35% odds ratio** |

This validates the core architectural principle: **canonical CURIEs, typed entities, and evidence scores aren't just convenient—they're what benchmarks prove works.**

### The Value of Prior Art Documentation

This document serves multiple purposes:

1. **For Researchers:** Understand established patterns before claiming novelty
2. **For Collaborators:** See how this project aligns with (and extends) industry standards
3. **For the Community:** Educational resource documenting which patterns have consensus

**Pattern of acknowledgment:**
- First: Document prior art and established patterns (§1-6)
- Then: Articulate unique value propositions where they exist (§7)
- Finally: Identify alignment as a feature, not a limitation (§8)

Embracing alignment with standards like TRAPI, Biolink, and the Fuzzy-to-Fact pattern is a strength—it means the work builds on proven foundations rather than reinventing them.

---

## 9. Research Directions Informed by Prior Art

### Immediate Opportunities

1. **TRAPI Compatibility** - Consider aligning envelope structure with TRAPI's `message.knowledge_graph` for federation compatibility

2. **Evidence Channels** - STRING's 7-channel evidence model (nscore, fscore, pscore, ascore, escore, dscore, tscore) is essential for agent reasoning about confidence

3. **Directionality** - STRING 2025 adds regulatory direction; consider extending interaction models for cause→effect relationships

### Future Considerations

1. **Network Embeddings** - STRING now offers downloadable embeddings for ML; could enable vector similarity search across proteins

2. **Cross-Species Alignment** - STRING's FedCoder-based ortholog mapping could enable comparative biology queries

3. **Fine-Tuned Literature Parsing** - STRING 2025 uses fine-tuned LLMs for evidence extraction; consider similar approach for novel relationship discovery

---

## References

### Primary Sources

- [STRING API Documentation](https://string-db.org/help/api/)
- [NCATS TRAPI GitHub](https://github.com/NCATSTranslator/ReasonerAPI)
- [BioThings Explorer](https://explorer.biothings.io)

### Key Papers

1. Szklarczyk D, et al. (2025). "The STRING database in 2025: protein networks with directionality of regulation." *Nucleic Acids Research*, 53(D1):D730-D737. [DOI: 10.1093/nar/gkae1113](https://doi.org/10.1093/nar/gkae1113)

2. Szklarczyk D, et al. (2023). "The STRING database in 2023." *Nucleic Acids Research*, 51(D1):D638-D646. [DOI: 10.1093/nar/gkac1000](https://doi.org/10.1093/nar/gkac1000)

3. Kuhn M, et al. (2016). "STITCH 5: augmenting protein–chemical interaction networks with tissue and affinity data." *Nucleic Acids Research*, 44(D1):D380-D384. [DOI: 10.1093/nar/gkv1277](https://doi.org/10.1093/nar/gkv1277)

4. Callaghan J, et al. (2023). "BioThings Explorer: a query engine for a federated knowledge graph of biomedical APIs." *Bioinformatics*, 39(9):btad570. [DOI: 10.1093/bioinformatics/btad570](https://doi.org/10.1093/bioinformatics/btad570)

5. Xu CH, et al. (2025). "Federated Knowledge Retrieval Elevates Large Language Model Performance on Biomedical Benchmarks." *bioRxiv*. [DOI: 10.1101/2025.08.01.668022](https://doi.org/10.1101/2025.08.01.668022)

6. Wang R, et al. (2025). "Improving large language model applications in biomedicine with retrieval-augmented generation." *JAMIA*, 32(4):605. [DOI: 10.1093/jamia/ocaf021](https://doi.org/10.1093/jamia/ocaf021)

7. **KARE Framework** (2024). "Knowledge Graph Community-level Retrieval for LLM Reasoning." *OpenReview*. [link](https://openreview.net/forum?id=r8qN9J6aKj)

8. **Harvard Knowledge Graph Agent** (2025). "Systematic generation, review, and revision of Medical KGs." *Predictive Systems AI*. [link](https://predictivesystems.ai/research)

### Standards and Registries

9. **W3C CURIE Syntax 1.0** (2010). "CURIE Syntax 1.0: A syntax for expressing Compact URIs." W3C Working Group Note. [https://www.w3.org/TR/2010/NOTE-curie-20101216/](https://www.w3.org/TR/2010/NOTE-curie-20101216/)

10. Hoyt CT, et al. (2022). "Unifying the identification of biomedical entities with the Bioregistry." *Scientific Data*, 9:714. [DOI: 10.1038/s41597-022-01807-3](https://doi.org/10.1038/s41597-022-01807-3)

11. **Node Normalizer** (2022). "NCATS Translator Node Normalization Service." GitHub. [https://github.com/TranslatorSRI/NodeNormalization](https://github.com/TranslatorSRI/NodeNormalization)

---

## Appendix: Key Terms for Research Agents

| Term | Definition | Example |
|------|------------|---------|
| **CURIE** | Compact URI (PREFIX:LOCAL_ID) per [W3C CURIE Syntax 1.0](https://www.w3.org/TR/curie/) | `HGNC:1100`, `UniProtKB:P38398` |
| **Fuzzy-to-Fact** | Two-phase resolution: ambiguous → candidates → strict lookup | `search_genes("brca1")` → `get_gene("HGNC:1100")` |
| **Agentic Biolink** | Flattened JSON schema with standardized `cross_references` | See ADR-001 §4-5 |
| **MCP** | Model Context Protocol (Anthropic standard for LLM tool access) | FastMCP server implementation |
| **TRAPI** | Translator Reasoner API (NCATS standard for biomedical KG queries) | OpenAPI-based federation |
| **Token Budgeting** | Minimizing response size for LLM context efficiency | `slim=True` parameter |
| **Recovery Hint** | Error metadata enabling autonomous self-correction | `"Use search_genes() first..."` |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2026-01-10 | Initial prior art documentation |
| 1.1.0 | 2026-01-10 | Added Research Context framing, Novel Contributions section, and Key Terms appendix |
| 1.2.0 | 2026-01-10 | Added §7.5 Node/Edge separation pattern; updated cross-ref count to 23 |
| 1.3.0 | 2026-01-10 | Added §7.6 Competency Questions alignment with BTE-RAG benchmarks |
| 1.4.0 | 2026-01-10 | Added §8 quantitative evidence and prior art documentation value |
| 1.5.0 | 2026-01-24 | Added §4.5 CURIE Standard Foundation; W3C and Bioregistry references; fixed ChEMBL CURIE format |
