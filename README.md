# biosciences-program

Program management and cross-repo coordination for the [Open Biosciences](https://github.com/open-biosciences) platform.

Owned by the **Program Director** agent. This repo contains coordination documents only -- no runnable code.

## Contents

### AGENTS.md

Defines the 9-agent team that operates the platform:

| # | Agent | Primary Repo(s) |
|---|-------|-----------------|
| 1 | Program Director | biosciences-program |
| 2 | Platform Architect | biosciences-architecture |
| 3 | MCP Platform Engineer | biosciences-mcp |
| 4 | Memory Engineer | biosciences-memory |
| 5 | Deep Agents Engineer | biosciences-deepagents |
| 6 | Research Workflows Engineer | biosciences-research |
| 7 | Temporal Engineer | biosciences-temporal |
| 8 | Quality & Skills Engineer | biosciences-evaluation, biosciences-skills |
| 9 | Education & Workspace Engineer | biosciences-education, biosciences-workspace-template |

### migration-tracker.md

Tracks the migration from 3 predecessor repos into 11 new repos across 4 waves:

| Wave | Focus | Target Repos |
|------|-------|-------------|
| 1 - Foundation | ADRs, skills, coordination | architecture, skills, program |
| 2 - Platform | MCP servers, knowledge graph | mcp, memory |
| 3 - Orchestration | LangGraph agents, Temporal workflows | deepagents, temporal |
| 4 - Validation | Evaluation, research workflows | evaluation, research |

Migration progress is also tracked in [Linear](https://linear.app/agentic-wisdom/project/open-biosciences-migration-f926beb444f6) (issues AGE-148 through AGE-181).

## Dependencies

None. This is a coordination repo that references all other repos but has no code dependencies.

## Related Repos

- [biosciences-architecture](https://github.com/open-biosciences/biosciences-architecture) -- ADRs and governance
- [biosciences-skills](https://github.com/open-biosciences/biosciences-skills) -- shared skills and commands
- [biosciences-mcp](https://github.com/open-biosciences/biosciences-mcp) -- MCP server platform

## License

MIT
