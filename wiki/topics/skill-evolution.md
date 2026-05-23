---
title: Skill Evolution
tags:
  - skill-evolution
  - self-improvement
  - bottom-up-agent
  - skill-library
key_venues: []
related_topics:
  - procedural-memory
  - agentic-memory
  - continual-learning-evaluation
key_people: []
linked_ideas: []
---

## Overview

Skill evolution treats the *skill library itself* as a living artifact that grows, refines, and prunes over an agent's deployment lifetime. In contrast to one-shot skill generation (which produces a fixed set), evolution-based work asks how new skills *emerge* from observed task pressure, how existing skills *adapt* in response to success/failure trajectories, how *cross-user* and *cross-instance* signals get aggregated into shared updates, and how the library avoids unbounded growth (compression, retirement, deduplication).

A unifying theme is that skill evolution is *bottom-up* — driven by the actual demands of completed and failed trajectories — rather than *top-down* from a designer's curated workflow.

## Timeline

## Seminal works

## SOTA tracker

## Key benchmarks

## Open problems

### Known gaps

- Cross-user skill sharing under privacy / heterogeneity constraints is barely studied.
- Retirement and deprecation policies are usually heuristic; principled approaches are missing.
- Conflict resolution between contradictory skill updates (from different trajectories or users) remains ad hoc.

### Methodological gaps

- Most evaluations compare evolved-skills vs. no-skills; few isolate the marginal value of *evolution* vs. one-shot skill generation.
- Generalization across agent backbones is rarely measured: a skill that helps Claude may hurt Qwen.
