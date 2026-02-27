# Repository Dependency Graph

> Visual map of how the 12 Open Biosciences repositories depend on each other.
> Arrows point from provider to consumer (A --> B means "B depends on A").

```mermaid
flowchart TD
    subgraph wave1["Wave 1: Foundation"]
        arch["biosciences-architecture\n(ADRs, schemas)"]
        skills["biosciences-skills\n(6 domain skills)"]
        program["biosciences-program\n(coordination)"]
        platskills["platform-skills\n(scaffold, security)"]
    end

    subgraph wave2["Wave 2: Platform"]
        mcp["biosciences-mcp\n(12 FastMCP servers, 697+ tests)"]
        memory["biosciences-memory\n(Graphiti / Neo4j)"]
    end

    subgraph wave3["Wave 3: Orchestration"]
        deep["biosciences-deepagents\n(LangGraph supervisor)"]
        temporal["biosciences-temporal\n(PydanticAI + Temporal.io)"]
    end

    subgraph wave4["Wave 4: Validation"]
        eval["biosciences-evaluation\n(quality metrics)"]
        research["biosciences-research\n(competency questions)"]
        edu["biosciences-education\n(training materials)"]
        workspace["biosciences-workspace-template\n(bootstrap scripts)"]
    end

    arch -->|schemas| mcp
    mcp -->|MCP tools| deep
    mcp -->|MCP tools| temporal
    mcp -->|MCP tools| research
    memory -->|graph persist| deep
    memory -->|graph persist| research

    eval -.->|reads from all| arch
    eval -.->|reads from all| mcp
    eval -.->|reads from all| deep
    eval -.->|reads from all| temporal

    style wave1 fill:#b2f2bb,stroke:#22c55e,color:#1e1e1e
    style wave2 fill:#a5d8ff,stroke:#4a9eed,color:#1e1e1e
    style wave3 fill:#d0bfff,stroke:#8b5cf6,color:#1e1e1e
    style wave4 fill:#ffd8a8,stroke:#f59e0b,color:#1e1e1e,stroke-dasharray: 5 5

    style arch fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style skills fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style program fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style platskills fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style mcp fill:#cfe2ff,stroke:#4a9eed,color:#1e1e1e
    style memory fill:#cfe2ff,stroke:#4a9eed,color:#1e1e1e
    style deep fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e
    style temporal fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e
    style eval fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e
    style research fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e
    style edu fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e
    style workspace fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e
```

## Dependency Rules

| Rule | Description |
|------|-------------|
| **Pure providers** | `biosciences-architecture` and `biosciences-skills` have no upstream dependencies |
| **Schema dependency** | `biosciences-mcp` depends on `biosciences-architecture` for Pydantic schemas and ADR compliance |
| **Tool consumers** | `biosciences-deepagents` and `biosciences-temporal` consume MCP tools via HTTP transport |
| **Graph persistence** | `biosciences-memory` is consumed by `biosciences-research` and `biosciences-deepagents` (PERSIST phase) |
| **Quality observer** | `biosciences-evaluation` reads from all repos but no repo depends on it |
