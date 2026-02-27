# Ecosystem Positioning: From Storefront to Workshop

How researchers and developers discover and engage with the Open Biosciences platform.
The marketplace provides a curated **storefront** (5 MCP servers, 4 skills), while
the GitHub organization provides the full **workshop** (12 MCP servers, 6 domain skills,
dual orchestrators, knowledge graph, workspace template).

## Researcher Journey

```mermaid
flowchart TD
    R["Researcher\nAsks a biology question"]
    M["anthropics/life-sciences\nMarketplace (Storefront)"]
    S5["5 Curated MCP Servers\nHGNC, UniProt, ChEMBL,\nOpen Targets, STRING"]
    S4["4 Domain Skills\nFuzzy-to-Fact protocol"]
    OB["Open Biosciences\nGitHub Organization (Workshop)"]
    F12["Full 12-Server Platform\n+ Ensembl, Entrez, PubChem,\nIUPHAR, WikiPathways,\nClinicalTrials.gov, BioGRID"]
    ORCH["Dual Orchestrators"]
    LG["LangGraph Supervisor\n7 specialists + React UI\n(interactive research)"]
    TMP["Temporal.io Workflows\nPydanticAI agents\n(durable batch pipelines)"]
    KG["Knowledge Graph\nGraphiti + Neo4j\nProvenance tracking"]
    WS["Workspace Template\nBootstrap scripts\nMulti-repo setup"]

    R -->|discovers| M
    M -->|browse| S5
    M -->|browse| S4
    S5 -->|"wants full platform"| OB
    S4 -->|"wants full platform"| OB
    OB --> F12
    OB --> ORCH
    ORCH --> LG
    ORCH --> TMP
    OB --> KG
    OB --> WS

    style R fill:#e0f2fe,stroke:#0284c7,stroke-width:2px
    style M fill:#fef3c7,stroke:#f59e0b,stroke-width:2px
    style S5 fill:#fef3c7,stroke:#f59e0b,stroke-width:2px
    style S4 fill:#fef3c7,stroke:#f59e0b,stroke-width:2px
    style OB fill:#dbeafe,stroke:#3b82f6,stroke-width:2px
    style F12 fill:#dcfce7,stroke:#22c55e,stroke-width:2px
    style ORCH fill:#e9d5ff,stroke:#8b5cf6,stroke-width:2px
    style LG fill:#e9d5ff,stroke:#8b5cf6,stroke-width:2px
    style TMP fill:#e9d5ff,stroke:#8b5cf6,stroke-width:2px
    style KG fill:#ccfbf1,stroke:#06b6d4,stroke-width:2px
    style WS fill:#f3f4f6,stroke:#6b7280,stroke-width:2px
```

## Developer / Contributor Journey

```mermaid
flowchart TD
    D["Developer / Contributor\nWants to extend the platform"]
    GH["GitHub open-biosciences\n12 MIT-licensed repos"]
    PS["platform-skills\nScaffold commands"]
    SC["/scaffold-fastmcp\nGenerate compliant\nMCP server skeleton"]
    ADR["biosciences-architecture\nADRs + Agentic Biolink schema"]
    SK["biosciences-skills\n6 domain skills library"]
    NEW["New MCP Server\nFollows Golden Path patterns\nFuzzy-to-Fact compliant"]
    PR["Pull Request\nSecurity review skill\n697+ test patterns"]
    MERGE["Merged into Platform\nAvailable to all researchers"]

    D -->|finds| GH
    GH -->|reads| ADR
    GH -->|uses| PS
    PS --> SC
    SC -->|generates| NEW
    ADR -->|"informs patterns"| NEW
    SK -->|"provides examples"| NEW
    NEW -->|submits| PR
    PR -->|accepted| MERGE

    style D fill:#e0f2fe,stroke:#0284c7,stroke-width:2px
    style GH fill:#dbeafe,stroke:#3b82f6,stroke-width:2px
    style PS fill:#f3f4f6,stroke:#6b7280,stroke-width:2px
    style SC fill:#f3f4f6,stroke:#6b7280,stroke-width:2px
    style ADR fill:#fef3c7,stroke:#f59e0b,stroke-width:2px
    style SK fill:#e9d5ff,stroke:#8b5cf6,stroke-width:2px
    style NEW fill:#dcfce7,stroke:#22c55e,stroke-width:2px
    style PR fill:#dcfce7,stroke:#22c55e,stroke-width:2px
    style MERGE fill:#dcfce7,stroke:#22c55e,stroke-width:2px
```
