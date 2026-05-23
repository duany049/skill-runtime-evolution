---
title: Procedural Skill
aliases:
  - Skill (Skill-Pro)
  - Reusable Procedural Unit
  - activation-execution-termination triple
tags:
  - procedural-memory
  - skill-library
  - llm-agents
  - skill-evolution
maturity: emerging
definition: "A reusable, natural-language procedural unit `ω = ⟨I_ω, π_ω, β_ω⟩` that specifies when to activate, how to act while active, and when to return control — the basic learnable element of a Skill-MDP."
key_papers:
  - skill-pro-learning-reusable-skills-experience
first_introduced: "Skill-Pro (Mi et al., 2026)"
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

A Procedural Skill `ω = ⟨I_ω, π_ω, β_ω⟩` is a tuple of three natural-language components:

1. **Activation condition `I_ω`** — a textual description of the observable context patterns in which `ω` should be invoked. The Skill-selection policy `μ` uses `I_ω` to pick `ω_t` from the current pool.
2. **Execution procedure `π_ω`** — an ordered, natural-language sequence of instructions; while `ω` is active the frozen LLM samples primitive actions `a_t ~ π_LLM(a | s_t, π_ω)`.
3. **Termination condition `β_ω`** — a textual predicate evaluated on the current state; `β_ω(s_t) = 1` returns control to `μ` for the next Skill selection.

## Intuition

The triple deliberately mirrors the *options framework* (initiation set, intra-option policy, termination function) but instantiated in natural language so a frozen LLM can interpret each part directly. The activation/termination pair is what distinguishes this from typical RAG-style retrievable units: the Skill itself decides when it is no longer useful, so retrieval need not happen every step. This makes the unit *temporally extended* — a single retrieval at activation amortises over many primitive steps.

## Variants

- *Named* Skills with stable identifiers (`StratPlan`, `FBInference`) versus *anonymous* candidates produced mid-evolution.
- *Explicit* representation (the default in Skill-Pro, comparable to Claude Agent Skills) versus *implicit* hidden-at-execution variants discussed in the paper's Appendix F.
- *Initialised seeds* (general procedural templates) versus *evolved* Skills refined by semantic gradients on task experience.

## Comparison

- vs *workflows* (AWM): workflows are typically full task-completion paths; Procedural Skills are sub-policies with explicit activation and termination, so multiple can compose within a single trajectory.
- vs *distilled insights* (Expel) and *notes* (A-MEM): both are passive textual artefacts retrieved into context. A Procedural Skill is *executed*: the activation/termination machinery means it acts as a policy fragment, not as documentation.
- vs *trajectories* (RAG / Reflexion-style memory): a Procedural Skill is a *compressed*, generalised abstraction across many trajectories rather than a single past episode replayed verbatim.
- vs *Voyager skills*, *Cradle skills*, *Memp*: structurally similar in being executable code/text snippets, but Skill-Pro adds explicit `(activation, termination)` framing and the Non-Parametric PPO learning loop.

## Known limitations

- Currently *explicit* and human-readable; the paper notes that fully implicit (sub-symbolic) procedural representations may be more aligned with human procedural memory but are deferred.
- The three-component split is task-agnostic but may be too coarse for domains where intermediate guards or sub-skill calls would be natural.
- Skills do not nest hierarchically in the released framework (no skill calls another skill).

## Open problems

- A principled story for *compositional* Procedural Skills: when should two Skills be merged into a higher-order one?
- Lifecycle management beyond online-score pruning: explicit Skill *retirement* policies tied to drift in the task distribution.
- Cross-agent transfer: how to detect that a Skill's activation condition is portable across backbones with different observation phrasing.

## Relationship to foundations

- Direct linguistic restatement of the *options framework* in hierarchical RL.
- Connects to *procedural memory* in cognitive science (Squire 2004; Cohen & Squire 1980) as an explicit-encoding analogue.

## My understanding

The smartest design choice is having `β_ω` at all. Most LLM-memory work implicitly retrieves at every step; making termination a *Skill-internal* decision is what lets retrieval ratio drop below 1.0 and what gives Skill-Pro its efficiency win. The unit's three-part structure also gives semantic gradients a natural place to land — you can attribute outcomes to a specific component and propose a targeted edit. Without that decomposition, prompt-level optimisation collapses into "rewrite the whole prompt" which is too coarse to learn from a small batch.
