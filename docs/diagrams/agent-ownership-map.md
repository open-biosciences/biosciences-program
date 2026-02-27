# Agent Ownership Map

> Maps each of the 9 agents in the Open Biosciences platform to the repositories they own
> and their primary responsibilities. Color-coded by migration wave.

```mermaid
flowchart LR
    subgraph agents["Agent Team"]
        a1["Agent 1<br/>Program Director"]
        a2["Agent 2<br/>Platform Architect"]
        a3["Agent 3<br/>MCP Platform Engineer"]
        a4["Agent 4<br/>Memory Engineer"]
        a5["Agent 5<br/>Deep Agents Engineer"]
        a6["Agent 6<br/>Research Workflows"]
        a7["Agent 7<br/>Temporal Engineer"]
        a8["Agent 8<br/>Quality & Skills"]
        a9["Agent 9<br/>Education & Workspace"]
    end

    subgraph repos["Repositories"]
        r_prog["biosciences-program"]
        r_arch["biosciences-architecture"]
        r_mcp["biosciences-mcp"]
        r_mem["biosciences-memory"]
        r_deep["biosciences-deepagents"]
        r_res["biosciences-research"]
        r_temp["biosciences-temporal"]
        r_eval["biosciences-evaluation"]
        r_skills["biosciences-skills"]
        r_platskills["platform-skills"]
        r_edu["biosciences-education"]
        r_ws["biosciences-workspace-template"]
    end

    a1 --> r_prog
    a2 --> r_prog
    a2 --> r_arch
    a3 --> r_mcp
    a4 --> r_mem
    a5 --> r_deep
    a6 --> r_res
    a7 --> r_temp
    a8 --> r_eval
    a8 --> r_skills
    a8 --> r_platskills
    a9 --> r_edu
    a9 --> r_ws

    style a1 fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style a2 fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style a3 fill:#cfe2ff,stroke:#4a9eed,color:#1e1e1e
    style a4 fill:#cfe2ff,stroke:#4a9eed,color:#1e1e1e
    style a5 fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e
    style a6 fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e
    style a7 fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e
    style a8 fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e
    style a9 fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e

    style r_prog fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style r_arch fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style r_mcp fill:#cfe2ff,stroke:#4a9eed,color:#1e1e1e
    style r_mem fill:#cfe2ff,stroke:#4a9eed,color:#1e1e1e
    style r_deep fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e
    style r_res fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e
    style r_temp fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e
    style r_eval fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e
    style r_skills fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style r_platskills fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style r_edu fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e
    style r_ws fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e
```

## Agent Responsibilities

| # | Agent | Repos | Key Responsibility |
|---|-------|-------|-------------------|
| 1 | Program Director | biosciences-program | Cross-repo coordination, migration tracking |
| 2 | Platform Architect | biosciences-architecture, biosciences-program | ADR authoring (via program repo), schema stewardship, Fuzzy-to-Fact protocol |
| 3 | MCP Platform Engineer | biosciences-mcp | 12 FastMCP servers, unified gateway, 697+ tests |
| 4 | Memory Engineer | biosciences-memory | Graphiti/Neo4j knowledge graph, dual-environment management |
| 5 | Deep Agents Engineer | biosciences-deepagents | LangGraph supervisor + 7 specialist subagents, React UI |
| 6 | Research Workflows | biosciences-research | Competency questions catalog, graph-builder workflows |
| 7 | Temporal Engineer | biosciences-temporal | PydanticAI + Temporal.io durable workflows |
| 8 | Quality & Skills | biosciences-evaluation, biosciences-skills, platform-skills | Evaluation rubrics, 6 domain skills, scaffold commands |
| 9 | Education & Workspace | biosciences-education, biosciences-workspace-template | Training materials, tutorials, bootstrap scripts |

## Decision Authority

| Decision Type | Authority |
|---------------|-----------|
| Schema changes | Platform Architect (Agent 2) |
| New MCP server | MCP Platform Engineer (Agent 3) + Platform Architect (Agent 2) |
| Cross-repo dependencies | Program Director (Agent 1) |
| Test quality standards | Quality & Skills Engineer (Agent 8) |
| Migration wave execution | Program Director (Agent 1) |
