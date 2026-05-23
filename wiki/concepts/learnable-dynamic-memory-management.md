---
title: Learnable Dynamic Memory Management
aliases:
  - memory-as-decision-making
  - learnable agent memory
  - RL-trained memory policy
  - dynamic agentic memory
tags:
  - agentic-memory
  - reinforcement-learning
  - decision-making
  - policy-learning
maturity: emerging
definition: A framing of agent memory management as a partially observable Markov decision process in which the memory workflow is learned end-to-end by reinforcement learning rather than specified by a static expert workflow.
key_papers:
  - atommem-learnable-dynamic-agentic-memory-atomic
first_introduced: AtomMem (Huo et al., 2026)
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

Learnable dynamic memory management replaces hand-crafted memory workflows with a learned policy over memory actions. The setup is a POMDP `(S, A, P, Ω, O, R, γ)` where the global state `s_t = (s_t^env, s_t^mem)` and joint action `a_t = (a_t^env, a_t^mem)` make memory an explicit, controllable part of the environment. Memory is partially observable — `o_t^mem` is determined by prior memory actions — so retrieval itself is a decision variable. With a tractable atomic action space (e.g., CRUD primitives), the memory workflow is no longer "the rules a designer chose" but "the policy the model learned from task-level feedback."

## Variants

- **Reset-per-task vs cross-task memory.** AtomMem sets `s_0^mem = ∅` at the start of each task, distinguishing this setting from cross-task experience accumulation (e.g., Expel, Memp).
- **Terminal-only reward vs intermediate rewards.** AtomMem uses task-level terminal rewards (EM for QA, LLM-as-judge for web); intermediate memory-quality rewards are a natural alternative but introduce their own credit-assignment problems.
- **Uniform vs per-entry advantage assignment.** Distributing task-level advantage uniformly across all output tokens (including memory ops) is the AtomMem default; per-memory-entry credit assignment is flagged as future work.

## Why it matters

Hand-designed memory workflows encode an implicit "one-size-fits-all" assumption: a rule that helps one task can hurt another. Letting the workflow itself be learned removes the bottleneck of expert design and aligns memory behavior to actual task feedback. Empirically, agents trained this way **discover** structured policies (e.g., shifting from Read-heavy to balanced CRUD usage) rather than executing a designer's recipe.

## Related ideas

(none yet)
