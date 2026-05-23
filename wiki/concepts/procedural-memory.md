---
title: Procedural Memory
aliases:
  - procedural memory
  - how-to memory
  - experience pool
  - skill memory
  - procedural memory pool
tags:
  - procedural-memory
  - agentic-memory
  - experience-replay
maturity: active
definition: A persistent store of structured, reusable "how-to" experience units distilled from past agent trajectories, indexed for context-conditioned retrieval and updated under quality-control policies, distinct from episodic (verbatim past interactions) and semantic (facts) memory.
key_papers:
  - remember-me-refine-me-dynamic-procedural
first_introduced: cognitive psychology (Anderson & Squire); adopted for LLM agents circa 2023
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

Procedural memory for an LLM agent is a persistent store of structured units that encode *how to act* under specified conditions — typically tuples of (usage scenario, core experience content, keywords/tags, confidence, tools used) — produced by analyzing prior task trajectories and retrieved at task time to bias the agent toward known-good action patterns. It is contrasted with episodic memory (raw past interactions) and semantic memory (factual knowledge).

## Intuition

Most agent failures are not failures of base capability but of *recall*: the model can solve a class of task but, in a fresh context, fails to evoke the relevant strategy. Procedural memory externalizes recurring decision patterns so they survive across sessions and across context-window resets. The unit is "how to handle situations like this", not "what happened last time".

## Variants

- **Granularity**: trajectory-level (whole runs as experiences) vs. keypoint-level (fine-grained decision steps). Empirically, keypoint-level transfers better — irrelevant context obscures core logic in trajectory-level entries.
- **Acquisition**: post-hoc summarization of successes only, vs. paired success/failure analysis, vs. comparative analysis between high- and low-scoring trajectories from the same task.
- **Indexing**: raw query embedding, query-keyword embedding, LLM-generated *generalized query*, or LLM-generated *usage scenario* — the last typically wins because it captures both context and applicability.
- **Update policy**: append-only (passive accumulation), selective addition (success-only), reflection-augmented (failed-then-retry), and utility-based deletion (prune entries with low $u/f$ ratio).
- **Reuse**: retrieve-only, retrieve+rerank, retrieve+rewrite into task-specific guidance — adaptation closes the gap when stored experiences imperfectly match the new task.

## Comparison

- **vs. parametric memory** — procedural memory lives in an external store, leaves the base weights unchanged, and supports per-experience traceability, editability, and selective deletion; parametric (fine-tuning, RLHF) updates the model itself, which is harder to roll back or audit.
- **vs. episodic memory** — episodic stores past *interactions* verbatim for later reference; procedural stores *generalized procedures* abstracted from them. Procedural is strictly downstream of episodic in most implementations.
- **vs. semantic memory** — semantic stores entity-and-relation facts; procedural stores executable decision patterns.
- **vs. workflow libraries (one-shot)** — workflow libraries are typically designed top-down or summarized once from a corpus; procedural-memory frameworks evolve the pool online via additions, reflections, and deletions.

## Known limitations

- Single-failure online distillation is noisy — analyzing a lone failed trajectory in real time often produces misguided rules, unlike batch failure analysis at initial pool construction.
- LLM-as-a-Judge validation lets through some non-actionable or wrong entries; ReMe and similar frameworks rely on later utility-based deletion to clean up.
- Storage grows unless bounded; without deletion thresholds, "toxic noise" accumulates and degrades retrieval precision.
- Cross-backbone reuse of stored experiences is under-tested — an experience written by one summarizer may or may not generalize to a different execute-model.

## Open problems

- Learned vs. heuristic deletion thresholds — should $\alpha$, $\beta$ be tuned per task family or even per experience type?
- Hierarchical procedural memory: most current pools are flat; compositional/hierarchical structures remain largely open.
- Task-state-conditioned retrieval that adapts mid-execution, not just at task start.
- Principled isolation of procedural-memory contribution from base-model capability in evaluation harnesses.

## Relationship to foundations

Procedural memory in agents has its roots in cognitive psychology's distinction between declarative and procedural knowledge (Anderson, ACT-R; Squire's memory taxonomy). The LLM-agent adoption keeps the conceptual distinction but realizes the store as an external, embedding-indexed database rather than a learned weight subsystem.

## My understanding

The center of gravity is shifting from "what to store" to "how to maintain". Early procedural-memory work spent most effort on extraction operators; the more recent generation (including ReMe) treats the *pool as a feedback-driven system*, with addition, deletion, and online reflection as first-class operators. Whether this scales — e.g., whether a 100k-entry pool stays well-shaped under utility-based deletion alone — is the next pressure point.
