---
title: Non-Parametric Experience Memory
aliases:
  - non-parametric memory
  - trajectory memory
  - experience bank
  - episodic trajectory store
  - retrieval-based value memory
tags:
  - agentic-memory
  - non-parametric
  - retrieval
  - value-estimation
  - reinforcement-learning
maturity: emerging
definition: "An external, non-parametric store of past (state, action, return) triplets that supports inference-time value or advantage estimation via retrieval — replacing the role of a trained value network."
key_papers:
  - just-time-reinforcement-learning-continual-learning
first_introduced: ""
date_updated: 2026-05-18
related_concepts:
  - test-time-policy-optimization
linked_ideas: []
---

## Definition

A non-parametric experience memory `M = {(s_i, a_i, G_i)}_{i=1}^N` stores trajectories (or trajectory fragments) outside the model's weights, with no learned compression. The store grows as the agent gathers experience and is queried by similarity to the current state to estimate quantities — typically state values, action values, or advantages — that a trained value network would otherwise provide.

## Intuition

Conventional actor-critic RL co-trains a critic `Q_φ(s,a)` to estimate expected return. The critic is parametric: its compute is amortized into training, but its capacity is finite and it must be retrained as the environment shifts. Non-parametric experience memory replaces the critic with a query against stored returns: `V̂(s) = (1/|N(s)|) Σ G_i` for `i ∈ N(s)` and similarly for `Q̂(s,a)`. Memory size and retrieval `k` become the new hyperparameters governing the bias-variance trade-off.

Under standard regularity assumptions (Lipschitz value function, well-mixed memory) these estimators are consistent: as the memory grows and `k` is chosen appropriately, retrieved estimates converge to the true `V^π` and `Q^π` even when the policy itself drifts over time.

## Variants

- **Flat trajectory store** — store individual `(s, a, G)` triplets indexed by state similarity.
- **Episodic memory with workflows** (AWM, Voyager) — store reusable workflow descriptions rather than raw triplets; closer to procedural memory.
- **Hierarchical memory** (MemGPT) — separate short-term and long-term tiers with management policies.
- **Network-style memory** (A-mem) — connected interaction graph indexed dynamically.

JitRL takes the *flat trajectory store* variant and uses it specifically as a value-estimation substrate, not as a context-augmentation source.

## Comparison

Versus **parametric value networks**: non-parametric memory is interpretable (every estimate decomposes into specific stored experiences) and trivially online (append, no gradient step). It trades model capacity for storage: memory grows linearly with experience, and retrieval cost grows accordingly.

Versus **context-window-as-memory** (Reflexion, FIFO-Memory): context-window methods can hold only a small recent prefix; non-parametric external memory can scale to millions of triplets, with retrieval doing the selection. The cost is moving from natural-language pattern matching to explicit similarity metrics over structured state representations.

## Known limitations

- State abstraction is critical: raw HTML or game text is too noisy for direct retrieval, so functionally-equivalent states must be mapped to similar representations. This is often hand-engineered per environment.
- Unbounded growth: without retirement, memory size and retrieval latency grow without bound across episodes.
- Cold start: when memory is empty or sparse, retrieved estimates are unreliable; methods need an explicit exploration / optimism mechanism.

## Open problems

- Learning state abstraction for retrieval rather than hand-crafting it.
- Memory consolidation criteria — what is worth retaining beyond raw `(s, a, G)` triplets?
- Cross-task generalization of stored procedural knowledge.

## Relationship to foundations

Connects to nearest-neighbor regression (the consistency results of k-NN estimators carry over) and to instance-based learning in classical ML.

## My understanding

Non-parametric experience memory is the most direct substrate for test-time policy optimization: it gives an explicit, queryable representation of the agent's empirical return distribution. The interesting research direction is not "should we use it" — flat stores already work — but how to learn the *state abstraction* that makes retrieval meaningful across tasks.
