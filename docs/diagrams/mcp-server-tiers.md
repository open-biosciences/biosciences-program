# MCP Server Tiers: 12 Databases, 34 Tools

The Open Biosciences platform provides unified access to 12 life sciences databases
organized in 5 tiers, from drug discovery core to interaction networks. Each server
implements the Fuzzy-to-Fact protocol (ADR-001) with `search_*` (fuzzy) and `get_*`
(strict CURIE) tool pairs.

```mermaid
flowchart TD
    subgraph T0["Tier 0 -- Drug Discovery Core"]
        style T0 fill:#fef3c7,stroke:#f59e0b,stroke-width:2px
        ChEMBL["<b>ChEMBL</b><br/>3 tools<br/><i>CHEMBL25</i>"]
        OT["<b>Open Targets</b><br/>3 tools<br/><i>ENSG00000115170</i>"]
    end

    subgraph T1["Tier 1 -- Gene & Protein Identity"]
        style T1 fill:#dcfce7,stroke:#22c55e,stroke-width:2px
        HGNC["<b>HGNC</b><br/>2 tools<br/><i>HGNC:171</i>"]
        UniProt["<b>UniProt</b><br/>2 tools<br/><i>Q04771</i>"]
        Ensembl["<b>Ensembl</b><br/>3 tools<br/><i>ENSG00000115170</i>"]
        Entrez["<b>Entrez / NCBI</b><br/>3 tools<br/><i>90</i>"]
    end

    subgraph T2["Tier 2 -- Pharmacology"]
        style T2 fill:#dbeafe,stroke:#4a9eed,stroke-width:2px
        PubChem["<b>PubChem</b><br/>2 tools<br/><i>CID 5280453</i>"]
        IUPHAR["<b>IUPHAR / GtoPdb</b><br/>4 tools<br/><i>Target ID 1810</i>"]
    end

    subgraph T3["Tier 3 -- Pathways & Clinical Trials"]
        style T3 fill:#ede9fe,stroke:#8b5cf6,stroke-width:2px
        WikiPW["<b>WikiPathways</b><br/>4 tools<br/><i>WP4806</i>"]
        CT["<b>ClinicalTrials.gov</b><br/>3 tools<br/><i>NCT03312634</i>"]
    end

    subgraph T4["Tier 4 -- Interaction Networks"]
        style T4 fill:#ccfbf1,stroke:#06b6d4,stroke-width:2px
        STRING["<b>STRING</b><br/>3 tools<br/><i>9606.ENSP00000263640</i>"]
        BioGRID["<b>BioGRID</b><br/>2 tools<br/><i>107149</i>"]
    end

    T0 --> T1
    T1 --> T2
    T2 --> T3
    T3 --> T4

    HGNC -- "cross_references.uniprot" --> UniProt
    HGNC -- "cross_references.ensembl_gene" --> Ensembl
    UniProt -- "cross_references.string" --> STRING
    ChEMBL -- "cross_references.chembl" --> OT
    OT -- "clinical evidence" --> CT
    WikiPW -- "gene members" --> HGNC
    IUPHAR -- "cross_references.chembl" --> ChEMBL

    classDef t0node fill:#fff3bf,stroke:#f59e0b,stroke-width:2px,color:#1e1e1e
    classDef t1node fill:#b2f2bb,stroke:#22c55e,stroke-width:2px,color:#1e1e1e
    classDef t2node fill:#a5d8ff,stroke:#4a9eed,stroke-width:2px,color:#1e1e1e
    classDef t3node fill:#d0bfff,stroke:#8b5cf6,stroke-width:2px,color:#1e1e1e
    classDef t4node fill:#c3fae8,stroke:#06b6d4,stroke-width:2px,color:#1e1e1e

    class ChEMBL,OT t0node
    class HGNC,UniProt,Ensembl,Entrez t1node
    class PubChem,IUPHAR t2node
    class WikiPW,CT t3node
    class STRING,BioGRID t4node
```

## Server Details

| Tier | Server | Tools | Fuzzy Input | Strict CURIE Format | Key Data |
|------|--------|-------|-------------|---------------------|----------|
| 0 | ChEMBL | `search_compounds`, `get_compound`, `get_compounds_batch` | Drug/compound name | `CHEMBL25` | Drug compounds, assays, mechanisms |
| 0 | Open Targets | `search_targets`, `get_target`, `get_associations` | Gene/disease name | `ENSG00000115170` | Disease-target associations, clinical evidence |
| 1 | HGNC | `search_genes`, `get_gene` | Gene symbol/name | `HGNC:171` | Official gene nomenclature, cross-references |
| 1 | UniProt | `search_proteins`, `get_protein` | Protein name/gene | `Q04771` | Protein sequences, function, structure |
| 1 | Ensembl | `search_genes`, `get_gene`, `get_transcript` | Gene symbol | `ENSG00000115170` | Genomic coordinates, transcripts, variants |
| 1 | Entrez | `search_genes`, `get_gene`, `get_pubmed_links` | Gene symbol/keyword | `90` (NCBI Gene ID) | Gene records, literature links |
| 2 | PubChem | `search_compounds`, `get_compound` | Compound name/SMILES | `CID 5280453` | Chemical structures, bioactivity |
| 2 | IUPHAR/GtoPdb | `search_ligands`, `get_ligand`, `search_targets`, `get_target` | Drug/receptor name | Target ID `1810` | Pharmacological targets, ligand binding |
| 3 | WikiPathways | `search_pathways`, `get_pathway`, `get_pathways_for_gene`, `get_pathway_components` | Pathway/gene name | `WP4806` | Biological pathways, gene members |
| 3 | ClinicalTrials.gov | `search_trials`, `get_trial`, `get_trial_locations` | Condition/drug/NCT ID | `NCT03312634` | Clinical studies, enrollment, outcomes |
| 4 | STRING | `search_proteins`, `get_interactions`, `get_network_image_url` | Protein/gene name | `9606.ENSP00000263640` | Protein-protein interactions, confidence scores |
| 4 | BioGRID | `search_genes`, `get_interactions` | Gene symbol | `107149` | Genetic and physical interactions |

## Cross-Reference Flow

Every entity returned by any MCP server includes a `cross_references` object (ADR-001 Appendix A)
that enables automated graph traversal across databases:

```
HGNC:171 (ACVR1)
  cross_references:
    uniprot: ["Q04771"]
    ensembl_gene: "ENSG00000115170"
    entrez: "90"
    string: "9606.ENSP00000263640"
```

This enables the platform to resolve a single gene name into a fully connected
knowledge graph spanning all 12 databases without manual identifier mapping.
