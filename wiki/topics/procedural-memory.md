---
title: Procedural Memory
tags:
  - procedural-memory
  - agent-skills
  - skill-library
  - lifelong-learning
key_venues: []
related_topics:
  - skill-evolution
  - agentic-memory
  - continual-learning-evaluation
key_people: []
linked_ideas: []
---

## Overview

Procedural memory for LLM agents is the encoding of *executable how-to knowledge* extracted from past task trajectories — distinct from episodic memory (retrieve past interactions verbatim) and from semantic memory (factual knowledge). The goal is to convert recurring problem-solving patterns into compact, reusable, conditionally-applicable procedures so that an agent does not have to re-derive solutions on every encounter.

Work in this area asks: what *unit* should be stored (workflow, skill, sub-routine, scratchpad), what *abstraction level* is most reusable, what mechanism *extracts* procedures from raw trajectories, how does an agent *retrieve* the right procedure for a given context, and how does the procedural library *update* itself as the agent gains experience.

## Timeline

## Seminal works

## SOTA tracker

## Key benchmarks

## Open problems

### Known gaps

- Procedural memory often relies on hand-crafted extraction operators; learning extraction itself remains under-studied.
- Most systems treat the procedural library as flat; hierarchical / compositional procedural memory is largely open.
- Retrieval over procedural memory tends to use surface lexical or embedding match — task-state-conditioned retrieval is underdeveloped.

### Methodological gaps

- Few benchmarks isolate procedural-memory contribution from base-model improvement.
- Long-horizon evaluation protocols are inconsistent across systems; comparability suffers.
