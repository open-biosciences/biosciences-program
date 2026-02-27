# Platform Engineering for AI-Augmented Life Sciences

---

## Executive Summary

**The Problem**: Life sciences researchers spend 60-80% of their time on data wrangling—querying fragmented APIs, reconciling incompatible identifiers, and manually stitching together information that should flow seamlessly.

**The Opportunity**: AI agents can automate this integration work, but only if they're guided by architectural guardrails. Without them, AI velocity creates chaos faster than humans can verify.

**Our Approach**: We apply Platform Engineering principles from [Team Topologies](https://teamtopologies.com/) to create a "thick platform" that enables AI agents to navigate 12+ life sciences databases with consistency, reliability, and domain expertise built in.

**The Result**: Researchers ask questions in natural language. The platform handles the complexity. Knowledge graphs emerge from the answers.

---

## Introduction: The Data Integration Nightmare

Picture a computational biologist investigating a promising cancer target. She needs to:

1. **Find the gene** → Query HGNC for the official symbol
2. **Get the protein** → Cross-reference to UniProt for structure and function
3. **Check interactions** → Search STRING and BioGRID for binding partners
4. **Find drugs** → Query ChEMBL for compounds that modulate the target
5. **Assess clinical relevance** → Search ClinicalTrials.gov for active studies
6. **Build context** → Pull pathway information from WikiPathways

Each database has different:
- Authentication requirements
- Rate limiting policies
- Response formats
- Identifier schemes (HGNC:1100 vs ENSG00000012048 vs P38398)

She spends days writing integration code. The code breaks when APIs change. Identifiers don't match across systems. By the time she has clean data, the research question has evolved.

**This is the status quo. It doesn't have to be.**

---

## The Hero's Journey: From Chaos to Coherence

### Act I: The Ordinary World (Before)

Life sciences data lives in silos. Brilliant databases—each a monument to decades of curation—speak different languages:

| Database | Focus | Identifier Format |
|----------|-------|-------------------|
| HGNC | Gene nomenclature | HGNC:1100 |
| UniProt | Protein sequences | UniProtKB:P38398 |
| ChEMBL | Drug compounds | CHEMBL:25 |
| STRING | Protein interactions | STRING:9606.ENSP00000269305 |
| ClinicalTrials.gov | Clinical studies | NCT:00461032 |

Researchers become integration engineers. Scientists write glue code. The cognitive load is crushing.

### Act II: The Call to Adventure

What if AI agents could handle this complexity?

The promise of Claude Code, ChatGPT, and other AI assistants is that they can automate tedious work. But here's the trap: **AI without guardrails creates chaos faster than humans can verify.**

Ask an AI to "create an MCP server for PubChem" without guidance, and you'll get:
- Invented directory structures (different every time)
- Missed error handling patterns
- Inconsistent identifier formats
- No cross-reference mapping

The AI is fast. It's also wrong in ways that compound over time.

### Act III: Crossing the Threshold

We discovered that **Platform Engineering**—the discipline of building internal developer platforms—applies directly to AI-augmented workflows.

From [Team Topologies](https://teamtopologies.com/) (Pais & Skelton, 2019):

> "Platform Teams reduce cognitive load by providing internal products that enable Stream-aligned Teams to deliver faster."

Translated to AI:

> "Platform Skills reduce AI chaos by providing guardrails that enable researchers to get consistent results."

### Act IV: The Road of Trials

Building this platform required solving hard problems:

**Trial 1: The Identifier Problem**
Every database uses different identifiers for the same entity. BRCA1 is `HGNC:1100` in HGNC, `P38398` in UniProt, `ENSG00000012048` in Ensembl.

*Solution*: The **Fuzzy-to-Fact Protocol**. Phase 1 searches with natural language. Phase 2 resolves to canonical CURIEs. Cross-references map between systems.

**Trial 2: The Consistency Problem**
AI agents invent new patterns each time. No two implementations look alike.

*Solution*: **Golden Path Skills**. `/scaffold-fastmcp` generates consistent project structure. Templates encode architectural decisions. The AI follows rails instead of inventing roads.

**Trial 3: The Verification Problem**
AI can write code faster than humans can review it. How do you maintain quality at AI speed?

*Solution*: **Specification-Driven Development**. `/speckit.specify` → `/speckit.plan` → `/speckit.implement`. Each phase has explicit gates. Humans verify specifications. AI executes bounded tasks.

**Trial 4: The Schema Problem**
Each API returns data in its own format. Integration code explodes combinatorially.

*Solution*: **Agentic Biolink Schema**. All tools return the same flattened JSON structure with `cross_references` for database links. One schema to rule them all.

### Act V: The Transformation

What emerges is a platform where:

```
Researcher: "What are the synthetic lethal partners for ARID1A in ovarian cancer?"

Platform:
  → HGNC: Resolve ARID1A → HGNC:11110
  → UniProt: Get protein function → SWI/SNF chromatin remodeling
  → STRING: Find interactions → EZH2 (score 0.999)
  → ChEMBL: Search inhibitors → Tazemetostat (CHEMBL:3414621)
  → ClinicalTrials: Find studies → NCT:03348631 (Phase 2)

Result: Connected knowledge graph with provenance
```

The researcher asks a question. The platform handles the complexity. Knowledge emerges.

---

## The Architecture: Three Layers of Enablement

```
┌─────────────────────────────────────────────────────────────────┐
│              STREAM-ALIGNED LAYER (Business Value)              │
│                                                                 │
│   "What therapeutic strategies exist for ARID1A-deficient      │
│    cancers using synthetic lethality?"                         │
│                                                                 │
│   Researchers · Domain Experts · Discovery Scientists           │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              SPECIFICATION LAYER (Quality Gates)                │
│                                                                 │
│   /speckit.specify → /speckit.plan → /speckit.implement        │
│                                                                 │
│   Transforms questions into bounded, verifiable tasks           │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              PLATFORM LAYER (Encoded Expertise)                 │
│                                                                 │
│   12 MCP Servers · Fuzzy-to-Fact Protocol · Agentic Biolink    │
│                                                                 │
│   HGNC · UniProt · ChEMBL · STRING · BioGRID · Ensembl         │
│   Entrez · PubChem · IUPHAR · WikiPathways · ClinicalTrials    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Conclusion: The New World

### What's Now Possible

1. **Natural Language to Knowledge Graphs**
   Ask a research question. Get a connected graph with provenance.

2. **Reproducible Workflows**
   The [Competency Questions Catalog](competency-questions/competency-questions-catalog.md) documents 7 research scenarios that can be re-run at any time.

3. **Consistent AI Outputs**
   Platform Skills ensure every MCP server follows the same patterns. No more integration surprises.

4. **Reduced Cognitive Load**
   Researchers focus on science. The platform handles plumbing.

### The Invitation

This repository is an experiment in applying Platform Engineering to AI-augmented research. We've learned that:

- **AI velocity requires guardrails** — Speed without structure creates debt
- **Team Topologies scales** — The stream-aligned/platform distinction works for human-AI collaboration
- **Specifications matter more with AI** — Garbage in, garbage out happens faster

We invite you to explore, extend, and challenge these patterns.

---

## Further Reading

| Document | Purpose |
|----------|---------|
| [ADR-002: Project Skills](adr/accepted/adr-002-v1.0.md) | Skills as Platform Engineering (the "tools") |
| [ADR-003: SpecKit SDLC](adr/accepted/adr-003-v1.0.md) | Specification-Driven Development (the "process") |
| [ADR-001: Agentic Biolink](adr/accepted/adr-001-v1.4.md) | The binding technical specification |
| [Competency Questions Catalog](competency-questions/competency-questions-catalog.md) | Research questions for knowledge graph building |
| *Team Topologies* (Pais & Skelton, 2019) | The original framework |

---

**Last Updated**: 2026-01-09
