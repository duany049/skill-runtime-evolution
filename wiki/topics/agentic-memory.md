---
title: Agentic Memory
tags:
  - agentic-memory
  - episodic-memory
  - long-term-memory
  - memory-architecture
key_venues: []
related_topics:
  - procedural-memory
  - skill-evolution
  - continual-learning-evaluation
key_people: []
linked_ideas: []
---

## Overview

Agentic memory is the umbrella for *all* forms of agent memory beyond the context window: episodic (past interactions), semantic (extracted facts), procedural (how-to knowledge), and the management policies that bind them. Work in this area focuses on architectural questions — what *to store*, what *to forget*, how to *consolidate* short-term experience into long-term retention, how to *retrieve* under context pressure, and how to *update* memory atomically as new information arrives.

The frontier sits in *learnable* and *evolvable* memory: instead of hand-crafted "summarize-then-store" pipelines, the memory operations themselves become learned behaviors that adapt to the task distribution.

## Timeline

## Seminal works

## SOTA tracker

## Key benchmarks

## Open problems

### Known gaps

- Coupling of short-term (context) and long-term (external store) management is mostly siloed; unified policies are emerging but unproven.
- Traceability of how a memory item influenced a decision is rarely engineered, making debugging and trust hard.
- Memory consolidation criteria (what is worth retaining) are typically heuristic.

### Methodological gaps

- Evaluation often confounds *memory quality* with *retrieval quality*; few benchmarks decouple them.
- Long-horizon, multi-session evaluation is expensive and under-standardized.
