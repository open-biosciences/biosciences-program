# Fuzzy-to-Fact Data Flow

> Sequence diagram showing how a research question flows through the Open Biosciences platform,
> from natural language input to validated knowledge graph output. This is the core data pattern
> defined in ADR-001 Section 3.

```mermaid
sequenceDiagram
    participant R as Researcher
    participant O as Agent Orchestrator<br/>(LangGraph / Temporal)
    participant M as MCP Server<br/>(12 FastMCP servers)
    participant KG as Knowledge Graph<br/>(Graphiti / Neo4j)

    R->>O: Natural language question<br/>"What drugs target BRCA1?"

    Note over O: ANCHOR Phase

    O->>M: Fuzzy search<br/>search_genes(query="BRCA1")
    M-->>O: Ranked candidates with CURIEs<br/>[{id: "HGNC:1100", name: "BRCA1", score: 0.99}]

    Note over O: CURIE Resolution

    O->>M: Strict lookup<br/>get_gene(hgnc_id="HGNC:1100")
    M-->>O: Full gene record<br/>{symbol: "BRCA1", locus_type: "gene with protein product"}

    Note over O: ENRICH Phase

    O->>M: Protein lookup<br/>get_protein(uniprot_id="UniProtKB:P38398")
    M-->>O: Protein functional data<br/>{function: "E3 ubiquitin ligase", pathways: [...]}

    Note over O: EXPAND Phase

    O->>M: Interaction search<br/>get_interactions(protein="ENSP00000350283")
    M-->>O: Protein-protein interactions<br/>[{partner: "TP53", score: 0.999}, ...]

    Note over O: TRAVERSE Phase

    O->>M: Drug search<br/>search_compounds(query="BRCA1 inhibitor")
    M-->>O: Drug candidates with CURIEs<br/>[{id: "CHEMBL:3545110", name: "Olaparib"}]

    O->>M: Clinical trials<br/>search_trials(query="BRCA1 Olaparib")
    M-->>O: Trial records<br/>[{nct_id: "NCT:02000622", phase: "Phase 3"}]

    Note over O: VALIDATE Phase

    O->>M: Cross-reference verification<br/>get_compound(chembl_id="CHEMBL:3545110")
    M-->>O: Validated compound record

    Note over O: PERSIST Phase

    O->>KG: Store validated entities and relationships
    KG-->>O: Graph nodes and edges created

    O-->>R: Structured research answer<br/>with provenance and CURIEs

    Note over R,KG: All entities stored with CURIEs for reproducibility
```

## Protocol Summary

The **Fuzzy-to-Fact protocol** (ADR-001 Section 3) enforces a bi-modal workflow across all 12 MCP servers:

| Phase | Mode | Input | Output | Error on Violation |
|-------|------|-------|--------|-------------------|
| Discovery | Fuzzy | Natural language string | Ranked candidates with CURIEs and confidence scores | -- |
| Resolution | Strict | Resolved CURIE (e.g., `HGNC:1100`) | Full canonical record | `UNRESOLVED_ENTITY` |

## Orchestration Phases

The agent orchestrator (LangGraph supervisor or Temporal workflow) coordinates 6 phases:

| Phase | Purpose | MCP Servers Used |
|-------|---------|-----------------|
| **ANCHOR** | Resolve gene symbols to canonical identifiers | HGNC, Entrez, Ensembl |
| **ENRICH** | Get protein functional context | UniProt |
| **EXPAND** | Find protein-protein and genetic interactions | STRING, BioGRID |
| **TRAVERSE** | Search drugs and clinical trials | ChEMBL, PubChem, IUPHAR, ClinicalTrials, WikiPathways |
| **VALIDATE** | Cross-check claims against source databases | All servers (verification queries) |
| **PERSIST** | Store validated knowledge graph | Graphiti / Neo4j |
