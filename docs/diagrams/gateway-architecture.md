# FastMCP Cloud Gateway Architecture

> Unified HTTPS gateway serving 12 MCP servers (34 tools) to all platform clients.
> Two-tier auth model: consumer-facing API key + operator-managed upstream keys.

```mermaid
flowchart TD
    subgraph clients["Clients"]
        cc["Claude Code<br/>(developer)"]
        deep["biosciences-deepagents<br/>(LangGraph supervisor)"]
        temp["biosciences-temporal<br/>(PydanticAI workflows)"]
    end

    apikey["BIOSCIENCES_API_KEY<br/>(consumer auth)"]

    subgraph gateway["biosciences-mcp.fastmcp.app"]
        gw["FastMCP Cloud Gateway<br/>HTTPS | 34 tools"]
    end

    subgraph public["10 Public Servers (no upstream keys)"]
        hgnc["HGNC"]
        uniprot["UniProt"]
        chembl["ChEMBL"]
        ot["Open Targets"]
        ensembl["Ensembl"]
        pubchem["PubChem"]
        iuphar["IUPHAR"]
        wikipw["WikiPathways"]
        ct["ClinicalTrials"]
        string["STRING"]
    end

    subgraph keyed["2 Keyed Servers (operator auth)"]
        biogrid["BioGRID<br/><i>BIOGRID_API_KEY</i>"]
        entrez["Entrez<br/><i>NCBI_API_KEY</i>"]
    end

    cc --> apikey
    deep --> apikey
    temp --> apikey
    apikey --> gw

    gw --> hgnc
    gw --> uniprot
    gw --> chembl
    gw --> ot
    gw --> ensembl
    gw --> pubchem
    gw --> iuphar
    gw --> wikipw
    gw --> ct
    gw --> string
    gw --> biogrid
    gw --> entrez

    style clients fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e
    style gateway fill:#a5d8ff,stroke:#4a9eed,color:#1e1e1e
    style public fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style keyed fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e

    style cc fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e
    style deep fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e
    style temp fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e
    style apikey fill:#ffffff,stroke:#f59e0b,color:#1e1e1e
    style gw fill:#cfe2ff,stroke:#4a9eed,color:#1e1e1e

    style hgnc fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style uniprot fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style chembl fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style ot fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style ensembl fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style pubchem fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style iuphar fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style wikipw fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style ct fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style string fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style biogrid fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e
    style entrez fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e
```

## Two-Tier Auth Model

| Tier | Key | Owner | Location | Purpose |
|------|-----|-------|----------|---------|
| **Consumer** | `BIOSCIENCES_API_KEY` | Plugin consumer | Client `.env` (referenced as `${BIOSCIENCES_API_KEY}` in `.mcp.json` headers) | Authenticates to FastMCP Cloud gateway |
| **Operator** | `BIOGRID_API_KEY` | Platform operator | FastMCP Cloud server environment | BioGRID upstream API auth |
| **Operator** | `NCBI_API_KEY` | Platform operator | FastMCP Cloud server environment | Entrez rate limit upgrade (3 to 10 req/s) |

10 of the 12 upstream life sciences APIs are fully public and require no authentication keys.

## Gateway Details

| Property | Value |
|----------|-------|
| Endpoint | `https://biosciences-mcp.fastmcp.app/mcp` |
| Transport | HTTPS (Streamable HTTP) |
| Servers | 12 FastMCP servers |
| Tools | 34 total (see [mcp-server-tiers.md](mcp-server-tiers.md)) |
| Tests | 697+ (in biosciences-mcp repo) |
| Protocol | Fuzzy-to-Fact (ADR-001 Section 3) |

## Client Configuration

All clients connect via `.mcp.json` with the same pattern:

```json
{
  "mcpServers": {
    "biosciences-mcp": {
      "type": "http",
      "url": "https://biosciences-mcp.fastmcp.app/mcp",
      "headers": {
        "Authorization": "Bearer ${BIOSCIENCES_API_KEY}"
      }
    }
  }
}
```

## References

- [ADR-001: Agentic-First Architecture](../../docs/adr/accepted/adr-001-v1.4.md) -- Fuzzy-to-Fact protocol
- [ADR-004: FastMCP Lifecycle Management](../../docs/adr/accepted/adr-004-v1.0.md) -- Server lifecycle and deployment
- [mcp-server-tiers.md](mcp-server-tiers.md) -- Full server/tool inventory
