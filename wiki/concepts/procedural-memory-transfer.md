---
title: Procedural Memory Transfer
aliases:
  - cross-model memory transfer
  - memory portability
tags:
  - procedural-memory
  - knowledge-transfer
  - distillation
maturity: emerging
definition: "The phenomenon that procedural memory distilled by one LLM agent can be loaded into another (often weaker) LLM agent and still raise task success and reduce step count, without retraining either model."
key_papers:
  - memp-exploring-agent-procedural-memory
first_introduced: "Memp (2025)"
date_updated: 2026-05-18
related_concepts:
  - procedural-memory-framework
linked_ideas: []
---

## Definition

Procedural memory transfer is the empirical claim that a memory bank $Mem$ produced by a builder $B$ operating on top of a strong agent (e.g. GPT-4o) can be loaded as the conditioning context for a different, weaker agent (e.g. Qwen2.5-14B) and improve that weaker agent's performance on the same task family.

## Intuition

If the procedural memory captures *task structure* rather than *model-specific quirks*, then it should be portable across backbones — analogous to a recipe that helps any cook, not only the chef who wrote it. Memp tests this by moving GPT-4o-built memory into Qwen2.5-14B and measuring downstream gains.

## Variants

- **Strong → weak transfer** (the main case studied): the donor model is stronger; transfer is expected to elevate the recipient.
- **Cross-family transfer** (Claude → Qwen, etc.): tests whether the memory format is family-agnostic.
- **Self-transfer**: load a model's own memory at a different deployment timestep — the trivial baseline.

## Comparison

Distinct from knowledge distillation (which compresses *model weights*), procedural-memory transfer compresses *experience traces* into a re-usable artifact at the prompt/context layer. No fine-tuning of the recipient is involved.

## Known limitations

- Demonstrated only on two benchmarks (TravelPlanner, ALFWorld) and three backbones.
- Memory format is text-vector — portability to agents with different tool APIs or different observation modalities is open.
- The retained value when transferring from a *weak* donor to a *strong* recipient is not measured.

## Open problems

- What properties of the memory bank (format, abstraction level, key granularity) predict transfer success?
- Can transfer be improved by jointly optimizing Build for portability instead of for the donor's own use?
- How does transfer degrade across larger backbone gaps (e.g. GPT-4o → 1B-parameter model)?

## Relationship to foundations

Linked to the broader knowledge-transfer literature and to teacher-student paradigms, but operates at the context-engineering layer rather than via weight updates.

## My understanding

If this generalizes, it changes the economics of running a fleet of agents: one strong agent can amortize its trajectory cost across many weaker agents that consume its memory. The open question is whether the gains plateau or compound as the donor's memory bank matures.
