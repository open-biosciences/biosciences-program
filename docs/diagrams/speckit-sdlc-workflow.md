# SpecKit SDLC Workflow

> The 7-step specification-driven development lifecycle defined in ADR-003.
> Each step produces a traceable artifact before code is written.
> Commands live in `biosciences-program/.claude/commands/speckit.*.md`.

```mermaid
flowchart TD
    subgraph governance["Governance Layer"]
        constitution["/speckit.constitution<br/>Establish project constraints"]
        const_out(["constitution.md"])
    end

    subgraph specification["Specification Phase"]
        specify["/speckit.specify<br/>Create feature specification"]
        clarify["/speckit.clarify<br/>(optional) Surface underspecified areas"]
        spec_out(["spec.md"])
        clarify_out(["clarifications"])
    end

    subgraph planning["Planning Phase"]
        plan["/speckit.plan<br/>Create implementation plan"]
        tasks["/speckit.tasks<br/>Generate actionable tasks"]
        analyze["/speckit.analyze<br/>(optional) Cross-artifact consistency"]
        plan_out(["plan.md"])
        tasks_out(["tasks.md"])
    end

    subgraph implementation["Implementation Phase"]
        implement["/speckit.implement<br/>Execute bounded implementation"]
        gate{"Human<br/>Approval<br/>Gate"}
    end

    constitution --> const_out
    const_out --> specify
    specify --> spec_out
    spec_out --> clarify
    clarify --> clarify_out
    clarify_out --> plan
    spec_out --> plan
    plan --> plan_out
    plan_out --> tasks
    tasks --> tasks_out
    tasks_out --> analyze
    analyze --> gate
    gate -->|approved| implement
    gate -->|rejected| plan

    style governance fill:#b2f2bb,stroke:#22c55e,color:#1e1e1e
    style specification fill:#a5d8ff,stroke:#4a9eed,color:#1e1e1e
    style planning fill:#d0bfff,stroke:#8b5cf6,color:#1e1e1e
    style implementation fill:#ffd8a8,stroke:#f59e0b,color:#1e1e1e

    style constitution fill:#d4edda,stroke:#22c55e,color:#1e1e1e
    style specify fill:#cfe2ff,stroke:#4a9eed,color:#1e1e1e
    style clarify fill:#cfe2ff,stroke:#4a9eed,color:#1e1e1e,stroke-dasharray: 5 5
    style plan fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e
    style tasks fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e
    style analyze fill:#e8d5ff,stroke:#8b5cf6,color:#1e1e1e,stroke-dasharray: 5 5
    style implement fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e
    style gate fill:#fff3cd,stroke:#f59e0b,color:#1e1e1e

    style const_out fill:#ffffff,stroke:#22c55e,color:#1e1e1e
    style spec_out fill:#ffffff,stroke:#4a9eed,color:#1e1e1e
    style clarify_out fill:#ffffff,stroke:#4a9eed,color:#1e1e1e
    style plan_out fill:#ffffff,stroke:#8b5cf6,color:#1e1e1e
    style tasks_out fill:#ffffff,stroke:#8b5cf6,color:#1e1e1e
```

## Commands

| # | Command | Input | Output | Gate |
|---|---------|-------|--------|------|
| 1 | `/speckit.constitution` | Project principles, ADR constraints | `constitution.md` | **Required for new projects** |
| 2 | `/speckit.specify` | Feature request (natural language) | `spec.md` | -- |
| 3 | `/speckit.clarify` | `spec.md` (auto-triggered) | 5 targeted questions + user answers | Optional |
| 4 | `/speckit.plan` | `spec.md` + `constitution.md` | `plan.md` with architecture decisions | -- |
| 5 | `/speckit.tasks` | `plan.md` | `tasks.md` with dependency order | -- |
| 6 | `/speckit.analyze` | All artifacts | Consistency + Constitution compliance report | **Blocks on violations** |
| 7 | `/speckit.implement` | `tasks.md` + human approval | Code changes (delegates to Platform Skills) | **Human approval required** |

## Key Principles

- **Specification before code**: Every feature flows through structured artifacts before implementation begins
- **Bounded agent design**: `max_tasks_per_session: 5` prevents PR sprawl (ADR-003 Section 4.1)
- **Platform Skill chaining**: `/speckit.implement` delegates to scaffold commands (ADR-002) before writing code directly
- **Single writer ownership**: Each artifact has exactly one owning command (ADR-003 Section 4.2)

## References

- [ADR-003: SpecKit Specification-Driven Development](../../docs/adr/accepted/adr-003-v1.0.md)
- [ADR-002: Project Skills as Platform Engineering](../../docs/adr/accepted/adr-002-v1.0.md)
- SpecKit commands: `biosciences-program/.claude/commands/speckit.*.md`
