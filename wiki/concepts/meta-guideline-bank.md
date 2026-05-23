---
title: Meta-Guideline Bank
aliases:
  - meta-guideline memory
  - how-to-use experience bank
  - guideline bank
tags:
  - agentic-memory
  - meta-learning
  - hierarchical-memory
  - prompt-design
maturity: emerging
definition: A higher-level memory layer that stores composition instructions (meta-heuristics) for transforming retrieved experiences into a task-adaptive guideline, separately from the experiences themselves.
key_papers:
  - live-evo-online-evolution-agentic-memory
first_introduced: Live-Evo (Zhang et al., 2026)
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

A meta-guideline bank $\mathcal{M}$ is a separate memory store, parallel to the experience bank $\mathcal{E}$, that holds *composition instructions*: meta-heuristics specifying how retrieved experiences should be combined and applied to a current task under different conditions. The entries are not task experiences and not task-specific guidelines — they are instructions about *how to produce* task-specific guidelines from experiences, populated online from failure reflections.

## Intuition

Most agentic memory systems either dump retrieved trajectories into the prompt or apply a fixed abstraction operator (summary, heuristic rule) baked in at design time. Both options conflate two distinguishable things: (i) what the agent has seen, and (ii) the policy by which past observations should influence current decisions. Once these are decoupled, the *how-to-use* layer becomes learnable. The meta-guideline bank is the explicit form of that decoupling: it pulls the compilation policy out of the static prompt and turns it into an updatable memory whose entries grow as the system encounters failure modes.

## Variants

- *Single-tier*: $\mathcal{M}$ contains atomic meta-guidelines, each a textual heuristic about combining retrieved experiences (Live-Evo's instantiation).
- *Conditioned*: meta-guidelines are conditioned on task type or feature, retrieved alongside experiences.
- *Hierarchical*: meta-guidelines themselves could be organized at multiple abstraction levels (meta-meta-guidelines about choosing meta-guidelines); not yet realized in published work.

## Comparison

vs. *static system prompt with composition rules*: a static prompt encodes one fixed compilation policy; a meta-guideline bank holds many policies, retrieves the relevant one, and is updated from environmental feedback.

vs. *plain reflection memory* (e.g. Reflexion, ExpeL): reflections in those systems are stored as task-specific feedback rather than as reusable composition recipes; reflections sit alongside experiences rather than at a separate, higher-level layer.

vs. *prompt programs / hand-coded chains-of-thought*: hand-coded prompt programs encode a single offline-designed compilation strategy; the meta-guideline bank discovers and maintains many such strategies online.

## Known limitations

- The bank can accumulate stale meta-guidelines over long horizons. Live-Evo describes how new entries are added on failure but does not describe a forgetting or pruning mechanism for $\mathcal{M}$ itself.
- The semantic granularity of meta-guidelines is not formally specified — empirically they are short natural-language heuristics produced by `Reflect`, with no constraint on overlap.
- Retrieval from $\mathcal{M}$ is currently coarse (one $\hat m$ selected per task); how multiple meta-guidelines compose is not studied.

## Open problems

- Pruning policy for $\mathcal{M}$: a meta-guideline that referenced a now-decayed experience set may itself need retiring.
- Quantifying meta-guideline quality independently of the experiences they compile.
- Whether $\mathcal{M}$ should be shared across agent instances / users, given that meta-heuristics are arguably more transferable than raw experiences.

## Relationship to foundations

Conceptually a form of meta-learning: $\mathcal{M}$ holds the "learning to learn" policy whose downstream effect is on the compilation step rather than on model weights. The agentic-memory framing keeps the meta-learning fully external and interpretable.

## My understanding

This is the structural move that makes self-evolving memory genuinely about *evolution* rather than accumulation. Separating "what happened" from "how to use it" allows the two layers to be updated by different mechanisms (weight reinforcement on $\mathcal{E}$; failure-triggered insertion on $\mathcal{M}$) — and it explains the ablation pattern, where removing $\mathcal{M}$ hurts performance even though all the raw experiences remain available. The interesting research question is how to give $\mathcal{M}$ its own pruning and revision dynamics; right now it is monotonically growing.
