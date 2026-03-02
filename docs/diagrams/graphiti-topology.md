# Graphiti / Neo4j Dual-Environment Topology

> The Open Biosciences knowledge graph runs across two Neo4j environments with
> distinct read/write policies. All new writes go to Docker; Aura is read-only (write-frozen).

```mermaid
flowchart TD
    subgraph docker_env["Docker Neo4j (Local)"]
        subgraph priming_ns["Priming Namespace (READ-ONLY)"]
            priming["open-biosciences-migration-<br/>2026-priming<br/><i>Static project context</i>"]
        end
        subgraph working_ns["Working Namespace (MUTABLE)"]
            working["open-biosciences-migration-<br/>2026<br/><i>Active decisions, progress</i>"]
        end
    end

    subgraph aura_env["Neo4j Aura (Cloud)"]
        aura_data["~5k nodes | 1GB free tier<br/><b>WRITE FROZEN</b><br/><i>Reads only</i>"]
    end

    subgraph mcp_servers["MCP Connections"]
        gd["graphiti-docker<br/><i>reads + writes</i>"]
        ga["graphiti-aura<br/><i>reads only</i>"]
        nd["neo4j-docker-cypher<br/><i>reads + writes</i>"]
        na["neo4j-aura-cypher<br/><i>reads only</i>"]
    end

    subgraph consumers["Consumers"]
        session["Claude Code<br/>Session Priming"]
        memory["biosciences-memory<br/>Memory Engineer"]
        research["biosciences-research<br/>Research Workflows"]
        deep["biosciences-deepagents<br/>PERSIST phase"]
    end

    gd -->|search_nodes, add_memory| docker_env
    nd -->|read/write Cypher| docker_env
    ga -->|search_nodes, search_memory_facts| aura_env
    na -->|read Cypher| aura_env

    session --> gd
    memory --> gd
    memory --> nd
    research --> gd
    deep --> gd
    session --> ga
    memory --> ga
    memory --> na

    style docker_env fill:#c3fae8,stroke:#06b6d4,color:#1e1e1e
    style priming_ns fill:#e6fcf5,stroke:#06b6d4,color:#1e1e1e
    style working_ns fill:#e6fcf5,stroke:#06b6d4,color:#1e1e1e
    style aura_env fill:#e0e0e0,stroke:#666666,color:#1e1e1e
    style mcp_servers fill:#cfe2ff,stroke:#4a9eed,color:#1e1e1e
    style consumers fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e

    style priming fill:#c3fae8,stroke:#06b6d4,color:#1e1e1e
    style working fill:#c3fae8,stroke:#06b6d4,color:#1e1e1e
    style aura_data fill:#f0f0f0,stroke:#666666,color:#1e1e1e

    style gd fill:#c3fae8,stroke:#06b6d4,color:#1e1e1e
    style ga fill:#f0f0f0,stroke:#666666,color:#1e1e1e
    style nd fill:#c3fae8,stroke:#06b6d4,color:#1e1e1e
    style na fill:#f0f0f0,stroke:#666666,color:#1e1e1e

    style session fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e
    style memory fill:#cfe2ff,stroke:#4a9eed,color:#1e1e1e
    style research fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e
    style deep fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e
```

## Namespace Policy

| Namespace | Environment | Group ID | Mutable? | Purpose |
|-----------|-------------|----------|----------|---------|
| Priming | Docker | `open-biosciences-migration-2026-priming` | **NO -- read-only, never update** | Static project context for session priming |
| Working | Docker | `open-biosciences-migration-2026` | Yes | Active decisions, progress, evolving context |
| Aura (all) | Cloud | N/A | **NO -- write-frozen** | Historical data; reads for analytics only |

## MCP Connection Matrix

| MCP Server | Environment | Read | Write | Primary Use |
|------------|-------------|------|-------|-------------|
| `graphiti-docker` | Docker Neo4j | Yes | Yes | Session priming, memory storage, knowledge persistence |
| `graphiti-aura` | Neo4j Aura | Yes | **No** | Historical searches, analytics, cross-referencing |
| `neo4j-docker-cypher` | Docker Neo4j | Yes | Yes | Direct Cypher queries, schema inspection |
| `neo4j-aura-cypher` | Neo4j Aura | Yes | **No** | Direct Cypher analytics queries |

## Write-Freeze Policy (Aura)

Neo4j Aura (`graphiti-aura`) is in a **permanent write-freeze state**:

- The free-tier 1GB instance is at capacity (~5k nodes)
- This is a known, accepted state -- do NOT warn about capacity or recommend upgrading
- **NEVER write new data to graphiti-aura** -- no `add_memory`, no `write_neo4j_cypher`
- **Reads from Aura are fine** -- searches, queries, analytics all permitted
- **All new data MUST go to graphiti-docker** (local Docker Neo4j)

## Session Priming Protocol

Every Claude Code session must begin by querying the priming namespace:

```python
# REQUIRED at session start
search_nodes(query="project structure", group_ids=["open-biosciences-migration-2026-priming"])
search_memory_facts(query="agent ownership and dependencies", group_ids=["open-biosciences-migration-2026-priming"])
```

This loads the 9-agent team structure, repo dependencies, migration status, and platform conventions into the session context.
