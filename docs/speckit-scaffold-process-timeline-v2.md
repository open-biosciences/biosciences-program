# Life Sciences MCP Standardization Process Timeline

This document visualizes the current and potential optimized workflows for the `scaffold-fastmcp` process within the Life Sciences MCP development lifecycle.

## Current Workflow

The current process front-loads the scaffolding before specification. This ensures the directory structure exists before the agent attempts to write specifications or code, but it leaves the generated specification files empty, requiring a separate `/speckit.specify` step.

```mermaid
timeline
    title Life Sciences MCP Development Workflow
    section Preparation
        Step 0 : Parallel Development Setup (Git Worktrees)
    section Scaffolding
        Step 1 : /scaffold-fastmcp [api_name]
               : Creates file stubs (server.py, client.py)
               : Creates spec directory (specs/NNN-api/)
               : Enforces architecture (Envelopes, Async Lock, Validation)
    section Specification
        Step 2 : /speckit.specify (Standard Prompt)
               : Fills spec.md with ADR-based constraints (Constraint Injection)
               : Detailed architectural planning
    section Implementation
        Step 3 : /speckit.plan & /speckit.tasks
               : Breakdown of work
        Step 4 : /speckit.implement
               : Code generation & Refinement
```

## Optimized Workflow (Future State)

A potential optimization is to merge the Scaffolding and Specification steps. Since the Standard Prompt contains structured requirements, `scaffold-fastmcp` could accept a specification file or prompt as input and pre-populate the `spec.md` and even generated code stubs with more specific details (e.g., exact tool names inferred from the spec).

```mermaid
timeline
    title Optimized Development Workflow
    section Preparation
        Step 0 : Parallel Development Setup
    section Integrated Scaffolding
        Step 1 : /scaffold-spec [api_name] "Prompt"
               : Generates spec.md from Standard Prompt (SpecKit)
               : Scaffolds file stubs based on Spec
               : Auto-populates tool signatures from spec
    section Execution
        Step 2 : /speckit.plan & /speckit.tasks
               : Immediate transition to planning
        Step 3 : /speckit.implement
               : Implementation
```

## Analysis

- **Current State Efficiency:** High. The strict separation prevents "blank page syndrome" by ensuring the agent always has a file structure to query and populate.
- **Optimization Value:** Moderate. Merging steps saves one user interaction but complicates the scaffolding logic, which would need to parse natural language specifications. The current "Template -> Fill" pattern is robust and deterministic.
