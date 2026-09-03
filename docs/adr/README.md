# ADR index and placement rule

This directory holds no ADR files. It records **where ADRs live and why**, so cross-repo readers can find them and authors know where to put the next one.

## Placement rule (decided 2026-09-02)

ADRs are placed by **scope**, at the repository closest to where the decision is used. This is deliberately lightweight for the platform's current maturity; it is revisited when a second platform appears, not before.

| Scope | Lives in | Numbering | Binds |
|---|---|---|---|
| **Platform** — a decision every MCP connector must follow | `biosciences-mcp/docs/adr/accepted/` | `ADR-NNN` (001…) | `biosciences-mcp`, `biosciences-mcp-edge`, `psychology-mcp`, and any future connector repo, except where a repo records a scoped deviation (see below) |
| **Project** — a decision that binds one repo | that repo's `docs/adr/` | prefixed series, e.g. `ADR-MEM-NNN` | that repo only |
| **Program** — cross-repo process, ownership, release, migration | `biosciences-program/docs/adr/` | `ADR-PRG-NNN` when the first one is needed | all repos in the org |
| **Shared non-functional** across programs | a separate shared repo | later | not warranted yet |

Rules that follow from the table:

1. **Consumers inherit by reference.** A repo that follows the platform ADRs links to them from its own `docs/adr/README.md` (as [`psychology-mcp`](https://github.com/open-biosciences/psychology-mcp/blob/main/docs/adr/README.md) does); it does not copy them. `biosciences-mcp-edge` copies the §8 envelope models because it has no cross-repo import, which is a code dependency choice, not a governance one.
2. **Deviations are recorded where they apply.** A repo that departs from a platform ADR by design keeps a `docs/adr/README.md` naming the ADR, the clause, and the reason. `psychology-mcp` records five literature adaptations this way. `biosciences-mcp-edge` departs from ADR-001 §3 (Fuzzy-to-Fact) by design — its tools are post-graph detailed retrievals keyed by an identifier the caller already resolved, and deterministic wrappers for retrievals that skills performed non-deterministically — and should record that in the same form (tracked in AGE-689).
3. **Numbers are assigned once.** Platform numbers are taken in order in `biosciences-mcp`; a project series uses its own prefix so it can never collide. Drafts in reference repositories (`lifesciences-research`, `lifesciences-mcp`) do not reserve platform numbers.
4. **Visibility.** Public org repositories MUST NOT link to file paths or URLs in private org repositories; refer to a private repo by role and carry any needed measurement or text in the public document. Private repositories may link to public ones. (AGE-696.) As of 2026-09-03 every `open-biosciences` repository is public; the rule stands for the next private one.
5. **The Platform Architect authors platform ADRs in `biosciences-mcp`.** `biosciences-architecture` holds the Repository Analyzer Framework and workspace architecture snapshots, not governance artifacts.

## Where things are

| Series | Location | Notes |
|---|---|---|
| Platform ADR-001 … ADR-006 | [`biosciences-mcp/docs/adr/accepted/`](https://github.com/open-biosciences/biosciences-mcp/tree/main/docs/adr/accepted) | Moved here 2026-03-03 (program `83d6c67`, mcp `5278c4e`) after two earlier moves; settled by the placement rule above as the closest fit to use. |
| Platform ADR-007 — Gateway Rate Resilience | [`biosciences-mcp/docs/adr/accepted/adr-007-v1.0.md`](https://github.com/open-biosciences/biosciences-mcp/blob/main/docs/adr/accepted/adr-007-v1.0.md) | Accepted 2026-09-03 (AGE-690), no clause changes from the 2026-09-02 draft. The `proposed/adr-007-v0.1.md` path forwards to it. Codifies the backoff and headerless-throttle behaviour already implemented in the connectors and cited by [`psychology-mcp` constitution](https://github.com/open-biosciences/psychology-mcp/blob/main/.specify/memory/constitution.md) Principle VIII. |
| `ADR-MEM-001` — Constrain Edge Invalidation with a Typed Ontology and Reference-Time Discipline | [`biosciences-memory/docs/adr/proposed/adr-mem-001-v1.0.md`](https://github.com/open-biosciences/biosciences-memory/blob/main/docs/adr/proposed/adr-mem-001-v1.0.md) | Project-scoped `MEM-` series. Relocated from this repo (PR #3) to avoid the ADR-001 number collision. |
| Decision record: Agentic Biolink (ADR-001 §4) | [`lifesciences-research/docs/adr/critique/adr-001-agentic-biolink-decision-record-2026-09-02.md`](https://github.com/donbr/lifesciences-research/blob/main/docs/adr/critique/adr-001-agentic-biolink-decision-record-2026-09-02.md) | Reference copy; input to ADR-001 v1.5. To be mirrored into `biosciences-mcp/docs/adr/critique/`. |
| Specs and Spec Kit config for the 13 servers | [`biosciences-mcp/specs/`](https://github.com/open-biosciences/biosciences-mcp/tree/main/specs), `biosciences-mcp/.specify/` | Moved with the ADRs. |
