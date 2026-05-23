---
title: Atomic CRUD Memory Operations
aliases:
  - atomic memory operations
  - CRUD memory primitives
  - memory CRUD
tags:
  - agentic-memory
  - action-space
  - memory-primitives
maturity: emerging
definition: A minimal, complete, task-agnostic action space for agent memory consisting of Create, Read, Update, Delete primitives that compose into arbitrary higher-level memory workflows.
key_papers:
  - atommem-learnable-dynamic-agentic-memory-atomic
first_introduced: AtomMem (Huo et al., 2026)
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

Atomic CRUD memory operations are the irreducible primitives — `Create`, `Read`, `Update`, `Delete` — over an explicit memory store, used as the action vocabulary that an LLM agent operates with at every step. Three properties motivate the choice: **completeness** (any reachable memory state from any prior state can be synthesized by some CRUD sequence), **atomic minimality** (no CRUD primitive can be further decomposed into a finer reusable memory tool, and every higher-level memory tool can be expressed as a structured CRUD combination), and **task-agnosticness** (the primitives are not specialized to any downstream benchmark; their effectiveness reduces to the policy that selects them).

## Variants

- **Hybrid retrieval inside Read** — Read can be split into a *deterministic* path (always-return scratchpad entry) and a *selective* path (query-based vector lookup), as in AtomMem.
- **Compositional macro-actions** — at a single decision step the agent emits a sequence `A_t = {a_t^1, …, a_t^{K_t}}` of CRUD ops, executed in order; this exposes the underlying primitives while keeping per-step throughput practical.

## Why it matters

Treating memory ops as atomic primitives is what enables the broader **memory-as-decision-making** framing: when the action set is irreducible, the workflow is no longer a designer's pipeline but a learnable policy. Higher-level memory tools (summarize, fold, prune) become emergent behaviors of CRUD policies, not separately engineered modules.

## Related ideas

(none yet)
