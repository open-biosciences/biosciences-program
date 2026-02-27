# Research Workflow: ACVR1/FOP Drug Discovery Journey

A complete research question flow showing how the Open Biosciences platform
transforms a natural language question into a verified knowledge graph. This
example traces the ACVR1/FOP (fibrodysplasia ossificans progressiva) drug
discovery scenario through all 7 specialist subagents.

```mermaid
sequenceDiagram
    actor R as Researcher
    participant S as Supervisor<br/>(LangGraph)
    participant AN as Anchor<br/>Specialist
    participant EN as Enrichment<br/>Specialist
    participant EX as Expansion<br/>Specialist
    participant TD as Traversal<br/>(Drugs)
    participant TT as Traversal<br/>(Trials)
    participant V as Validation<br/>Specialist
    participant P as Persistence<br/>Specialist
    participant KG as Knowledge<br/>Graph

    R->>S: "What drugs target ACVR1 in FOP?"

    Note over S: Supervisor analyzes question<br/>and plans specialist dispatch

    rect rgb(220, 252, 231)
        Note right of S: ANCHOR PHASE
        S->>AN: Resolve gene entity "ACVR1"
        AN->>AN: HGNC search_genes("ACVR1")
        AN-->>S: ACVR1 resolved to HGNC:171<br/>score: 0.98
    end

    rect rgb(219, 234, 254)
        Note right of S: ENRICH PHASE
        S->>EN: Get protein context for HGNC:171
        EN->>EN: UniProt get_protein("Q04771")
        EN->>EN: Ensembl get_gene("ENSG00000115170")
        EN-->>S: Activin receptor type-1<br/>BMP signaling pathway<br/>Ser/Thr kinase domain
    end

    rect rgb(237, 233, 254)
        Note right of S: EXPAND PHASE
        S->>EX: Find interaction partners
        EX->>EX: STRING get_interactions("9606.ENSP00000263640")
        EX->>EX: BioGRID get_interactions("107149")
        EX-->>S: BMPR2 (score 0.999)<br/>SMAD1/5/9 (BMP cascade)<br/>FKBP1A (inhibitory)
    end

    rect rgb(254, 243, 199)
        Note right of S: TRAVERSE PHASE (Drugs)
        S->>TD: Find drugs targeting ACVR1
        TD->>TD: Open Targets get_associations("ENSG00000115170")
        TD->>TD: ChEMBL search_compounds("ACVR1 inhibitor")
        TD-->>S: Palovarotene (retinoic acid agonist)<br/>Saracatinib (ALK2 inhibitor)<br/>LDN-193189 (BMP inhibitor)
    end

    rect rgb(254, 243, 199)
        Note right of S: TRAVERSE PHASE (Trials)
        S->>TT: Find clinical trials for FOP + drugs
        TT->>TT: ClinicalTrials search_trials("fibrodysplasia ossificans progressiva")
        TT->>TT: ClinicalTrials get_trial("NCT03312634")
        TT-->>S: NCT03312634: Palovarotene Phase 3<br/>NCT03706625: Palovarotene long-term<br/>Status: FDA approved 2023
    end

    rect rgb(204, 251, 241)
        Note right of S: VALIDATE PHASE
        S->>V: Cross-check all claims
        V->>V: Verify HGNC:171 cross-refs match UniProt Q04771
        V->>V: Verify Palovarotene in both ChEMBL and ClinicalTrials
        V->>V: Confirm NCT03312634 references ACVR1/FOP
        V-->>S: All 5 entities verified<br/>3 cross-reference chains confirmed<br/>0 unresolved claims
    end

    rect rgb(220, 191, 255)
        Note right of S: PERSIST PHASE
        S->>P: Save verified graph
        P->>KG: Store ACVR1 node (HGNC:171)
        P->>KG: Store protein context (Q04771)
        P->>KG: Store drug relationships (Palovarotene)
        P->>KG: Store trial evidence (NCT03312634)
        KG-->>P: Graph persisted: 5 nodes, 7 edges
        P-->>S: Knowledge graph saved with provenance
    end

    S-->>R: Connected knowledge graph with provenance:<br/>ACVR1 -> Palovarotene -> NCT03312634<br/>All claims verified against source databases
```

## Phase Summary

| Phase | Specialist | MCP Servers Used | Key Output |
|-------|-----------|-----------------|------------|
| ANCHOR | Anchor Specialist | HGNC | ACVR1 resolved to HGNC:171 (canonical CURIE) |
| ENRICH | Enrichment Specialist | UniProt, Ensembl | Protein function: activin receptor type-1, BMP signaling |
| EXPAND | Expansion Specialist | STRING, BioGRID | Interaction partners: BMPR2, SMAD1/5/9, FKBP1A |
| TRAVERSE (Drugs) | Traversal Specialist | Open Targets, ChEMBL | Drugs: Palovarotene, Saracatinib, LDN-193189 |
| TRAVERSE (Trials) | Traversal Specialist | ClinicalTrials.gov | Trials: NCT03312634 (Phase 3), NCT03706625 (long-term) |
| VALIDATE | Validation Specialist | All (cross-check) | 5 entities verified, 3 cross-reference chains confirmed |
| PERSIST | Persistence Specialist | Neo4j/Graphiti | 5 nodes, 7 edges saved with full provenance |

## Biological Context

**ACVR1** (Activin A Receptor Type 1, also known as ALK2) is a bone morphogenetic protein
(BMP) type I receptor. Gain-of-function mutations in ACVR1 (most commonly R206H) cause
**Fibrodysplasia Ossificans Progressiva (FOP)**, an ultra-rare disorder where soft tissue
progressively turns to bone.

**Palovarotene** is a retinoic acid receptor gamma (RAR-gamma) agonist that inhibits
aberrant BMP signaling. It received FDA approval in 2023 (brand name: Sohonos) as the
first treatment specifically for FOP, based on results from clinical trial NCT03312634.

This research workflow demonstrates how the platform can trace the complete path from
gene identity through protein function, interaction networks, drug candidates, and
clinical evidence -- producing a verified knowledge graph with every claim backed by
an authoritative database source.
