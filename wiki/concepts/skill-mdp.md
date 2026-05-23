---
title: Skill-MDP
aliases:
  - Skill-augmented Markov Decision Process
  - Skill-Augmented MDP
tags:
  - markov-decision-process
  - skill-evolution
  - procedural-memory
  - hierarchical-policy
maturity: emerging
definition: "A Markov Decision Process extended with a dynamic pool of natural-language Skills that the agent selects and executes hierarchically, separating Skill-pool learning from base-policy learning."
key_papers:
  - skill-pro-learning-reusable-skills-experience
first_introduced: "Skill-Pro (Mi et al., 2026)"
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

A Skill-MDP `M_Ω = (S, A, Ω, P, R, γ)` augments a standard MDP with a *dynamic Skill pool* `Ω = {ω⁽¹⁾, …, ω⁽ᴷ⁾}` of fixed capacity `K`. The agent's policy factorises hierarchically into a Skill-selection policy `μ(ω | s_t, Ω)` and an LLM action policy `π_LLM(a | s_t, ω_t)`, with `Ω` itself treated as the learnable object.

## Intuition

Classical MDP solutions train a parameter-tied policy. In LLM agents the base policy is a frozen LLM, so there is nothing to "train" in the usual sense — but the natural-language *prompts that condition it* are themselves a kind of policy. Skill-MDP makes that explicit: it elevates the prompt pool to a first-class component of the MDP, with a discrete-time horizon over which the pool itself evolves under interaction experience. Skills are temporally extended (each runs until its termination condition fires), so a Skill-MDP is also a semi-MDP in the hierarchical-RL sense, with natural-language options playing the role of macro-actions.

## Variants

- *Selection by similarity*: `ω_t = argmax_ω Sim(s_t, I_ω)` using cosine similarity over embeddings or an LLM-as-judge.
- *Selection by value*: form top-k by similarity, then pick `argmax Q(s_t, ω)` within the candidate set.
- Both are simple instantiations; RL-trained selection policies are flagged as future work.

## Comparison

- vs *Options framework* (Sutton et al.): structurally identical (initiation set, intra-option policy, termination function), but the components are natural-language strings interpreted by a frozen LLM rather than parameterised functions.
- vs *Memory-augmented LLM agents* (RAG / Reflexion / AWM / G-Memory): those typically optimise the *retrieval* policy `μ` over a fixed memory store; Skill-MDP fixes `μ` (and the LLM) and optimises the *store* `Ω`.
- vs *Claude Agent Skills*: same explicit-procedure representation, but Skill-MDP is autonomously learned from trajectories rather than hand-authored.

## Known limitations

- The Skill-selection policy `μ` is assumed simple and fixed; non-trivial selection learning is unstudied.
- Pool capacity `K` is a hard hyper-parameter, not adapted online.
- Skill granularity (how long termination conditions wait) interacts with task structure but is not theoretically characterised.

## Open problems

- A version with co-evolving `μ` and `Ω` under a single non-parametric objective.
- Conditions under which `Ω*` converges (existence, uniqueness, stability across batches).
- How to incorporate trajectory-level structured rewards (not just scalar return-to-go) into Skill-pool optimisation.

## Relationship to foundations

- Builds on the *options framework* and *semi-MDP* theory in hierarchical RL.
- Inherits the MDP optimality objective; the proximal trust-region machinery of PPO is grafted on at the pool-update level rather than at parameter updates.

## My understanding

The interesting move is treating `Ω` as a *first-class* MDP component rather than as decoration around a base policy. That single change is what makes a PPO-style stability argument transfer: you can talk about a "trust-region around the current `Ω`" because there is a well-defined object to perturb. Most prompt-optimisation work optimises a single prompt per task; this is closer to a small-cardinality policy class with its own selection problem.
