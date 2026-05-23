---
title: Skill-Conditioned Memory Construction
aliases:
  - skill-guided memory
  - skill-conditioned extraction
  - span-level memory construction
tags:
  - agentic-memory
  - memory-management
  - llm-agent
maturity: emerging
definition: "A memory-update procedure in which an LLM executor is conditioned on a selected subset of reusable memory skills and a span of interaction context, producing memory updates in one structured generation step rather than via interleaved heuristic operations turn by turn."
key_papers:
  - memskill-learning-evolving-memory-skills-self
first_introduced: "MemSkill (arXiv:2602.02474, 2026)"
date_updated: 2026-05-18
related_concepts:
  - memory-skill
linked_ideas: []
---

## Definition

**Skill-conditioned memory construction** is a memory-update procedure in which an LLM executor receives three inputs—(i) the current text span $x_t$, (ii) retrieved memories $M_t$, and (iii) a selected subset of skills $A_t$ drawn from a shared skill bank—and emits structured memory updates in a single generation. The procedure is *span-level*: rather than processing the interaction trace turn by turn with interleaved heuristic operations, it consumes a configurable-length span (e.g., 512 tokens) and applies all selected skills jointly.

## Intuition

Two design choices motivate the shift away from per-turn pipelines. First, **composition in one call**: handing multiple relevant skills to the same LLM call lets the model trade off between, e.g., insertion and consolidation in context rather than re-reading the same span across multiple operation calls. Second, **decoupled granularity**: span length is a knob, not a built-in assumption, so the same construction procedure scales to long histories where per-turn processing would be wasteful.

## Variants

- **Top-K composition** — controller draws K skills without replacement; K can differ between training (small) and evaluation (larger) to compose more skills at inference.
- **Fixed-skill** baseline — skill set frozen at the initial primitives, only selection learned.
- **Random-skill** ablation — selection randomized, isolates the value of skill-conditioning vs. learned selection.

## Comparison

- vs. **turn-level extraction** (LightMem, MemoryBank): turn-level pipelines process each turn separately; skill-conditioned construction processes a span and composes skills.
- vs. **modular memory pipelines** (MemoryOS): MemoryOS decomposes memory construction into specialized modules invoked sequentially; skill-conditioned construction selects-and-composes in one pass.
- vs. **plain RAG with summarization**: skill-conditioning makes the memory-construction *behavior* explicit and editable, not implicit in a summarization prompt.

## Known limitations

- One-pass composition makes the executor's output harder to audit per-operation; failures cannot be attributed to a single skill.
- Span size is a hyperparameter (MemSkill defaults to 512); too-long spans can dilute the controller's state representation; too-short spans lose the composition advantage.
- Top-K is itself a hyperparameter and the system is mildly sensitive to it (smaller K under-utilizes the bank in long contexts).

## Open problems

- Whether composition order matters when multiple skills are passed in one call, and whether explicit ordering by the controller would help.
- How to attribute downstream failures back to specific skills in the selected subset (credit assignment).
- Adaptive span sizing tied to context content rather than a fixed token budget.

## Relationship to foundations

Sits at the intersection of retrieval-augmented memory construction and skill-library composition. The span-level formulation is closer to chunked document processing than to dialogue-turn processing.

## My understanding

The headline efficiency claim—one LLM call per span instead of multiple per-operation calls per turn—matters less than the *interface* claim: making the memory-construction behavior selectable from a bank instead of fixed in a pipeline is what enables downstream evolution. The composition vs. attribution tradeoff is the main thing to watch.
