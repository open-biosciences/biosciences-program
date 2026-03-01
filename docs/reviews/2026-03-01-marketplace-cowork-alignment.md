# Marketplace Cowork Plugin Alignment

**Date**: 2026-03-01
**Issue**: [AGE-217](https://linear.app/agentic-wisdom/issue/AGE-217/align-marketplace-with-anthropic-cowork-plugin-format)
**PR**: [marketplace#6](https://github.com/open-biosciences/marketplace/pull/6) (merged)
**Reference**: Anthropic `knowledge-work-plugins/bio-research` plugin

---

## Summary

Aligned the `open-biosciences/marketplace` repo with Anthropic's Cowork `.claude-plugin/` format. Comparison against the Cowork reference revealed a critical MCP wiring gap and non-standard fields.

## Changes

### 1. Added `.mcp.json` to 12 MCP server plugins

Cowork expects `.mcp.json` at each plugin root (sibling to `.claude-plugin/`), not MCP config embedded in `plugin.json`.

```json
{
  "mcpServers": {
    "hgnc": {
      "type": "http",
      "url": "https://biosciences-mcp.fastmcp.app/mcp",
      "headers": {
        "Authorization": "Bearer ${BIOSCIENCES_API_KEY}"
      }
    }
  }
}
```

### 2. Stripped non-standard fields from `plugin.json`

Removed `mcp` and `tools` from all 12 MCP server plugin.json files. Cowork expects only: `name`, `version`, `description`, `author`, `repository`.

### 3. Cleaned `marketplace.json`

- Removed root-level `metadata` field
- Removed `category` and `tags` from all plugin entries
- Removed external `biosciences-rag-pipeline` entry (`../` reference)
- Result: 15 entries with `name`, `source`, `description` only

### 4. Updated documentation

- CLAUDE.md, README.md, CONTRIBUTING.md updated with `.mcp.json` conventions

---

## Key Finding: Two-Layer Auth Model

FastMCP Cloud requires a bearer token for gateway access. The Cowork `.mcp.json` format supports two auth mechanisms:

1. **`headers`** — static bearer tokens with `${VAR}` env var interpolation (GitHub, Datadog pattern)
2. **`oauth`** — per-user OAuth flows with `clientId` and `callbackPort` (Slack pattern)

Our marketplace uses the `headers` pattern. The `${BIOSCIENCES_API_KEY}` variable is resolved from the client environment at connection time — the key never appears in the manifest.

### Auth Chain

```
Plugin consumer installs plugin
  → Cowork reads .mcp.json
    → Resolves ${BIOSCIENCES_API_KEY} from user's env
      → HTTP request with Authorization: Bearer <token>
        → FastMCP Cloud validates token
          → Gateway forwards to upstream APIs
            (using server-side BIOGRID_API_KEY, NCBI_API_KEY)
```

### Two Layers, Two Owners

| Key | Owner | Location | Purpose |
|-----|-------|----------|---------|
| `BIOSCIENCES_API_KEY` | Plugin consumer | Client env (`${VAR}` in `.mcp.json`) | FastMCP Cloud gateway auth |
| `BIOGRID_API_KEY` | Platform operator | Server env (FastMCP Cloud) | BioGRID upstream API auth |
| `NCBI_API_KEY` | Platform operator | Server env (FastMCP Cloud) | Entrez rate limit (3→10 req/s) |

10 of 12 upstream APIs are fully public — no keys required.

### Cowork Auth Reference

Source: `cowork-plugin-management/skills/cowork-plugin-customizer/examples/customized-mcp.json`

```json
{
  "github": {
    "type": "http",
    "url": "https://api.githubcopilot.com/mcp/",
    "headers": { "Authorization": "Bearer ${GITHUB_TOKEN}" }
  },
  "datadog": {
    "type": "http",
    "url": "https://api.datadoghq.com/mcp",
    "headers": {
      "DD-API-KEY": "${DATADOG_API_KEY}",
      "DD-APPLICATION-KEY": "${DATADOG_APP_KEY}"
    }
  }
}
```

Slack uses OAuth instead of headers:

```json
{
  "slack": {
    "type": "http",
    "url": "https://mcp.slack.com/mcp",
    "oauth": { "clientId": "1601185624273.8899143856786", "callbackPort": 3118 }
  }
}
```

---

## Upstream API Auth Summary

| Server | Auth | Key Env Var |
|--------|------|-------------|
| HGNC | None (public) | — |
| UniProt | None (public) | — |
| ChEMBL | None (public) | — |
| Open Targets | None (public) | — |
| STRING | None (public) | — |
| Ensembl | None (public) | — |
| PubChem | None (public) | — |
| IUPHAR | None (public) | — |
| WikiPathways | None (public) | — |
| ClinicalTrials | None (public) | — |
| Entrez | Optional (higher rate) | `NCBI_API_KEY` |
| BioGRID | Required | `BIOGRID_API_KEY` |
