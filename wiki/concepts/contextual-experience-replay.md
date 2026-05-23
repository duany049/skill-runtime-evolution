---
title: Contextual Experience Replay
aliases:
  - CER
  - in-context experience replay
  - experience replay for LLM agents
tags:
  - agentic-memory
  - procedural-memory
  - skill-evolution
  - self-improvement
  - in-context-learning
  - experience-replay
maturity: emerging
definition: A training-free paradigm where an LLM agent distills compact natural-language experiences from past trajectories into a dynamic buffer and replays retrieved subsets in its context window to improve decision-making on new tasks.
key_papers:
  - contextual-experience-replay-self-improvement-language
first_introduced: "2025"
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

Contextual Experience Replay (CER) is a class of training-free self-improvement mechanisms for LLM agents in which past task trajectories — successful or failed — are distilled into compact natural-language **experiences** (typically split into environment dynamics and decision-making skills) stored in a dynamic buffer; for each new task a retrieval module selects the top-k most relevant experiences and replays them as context conditioning the base agent's policy.

## Intuition

Standard LLM agents enter every new task without environment-specific knowledge, forcing repeated exploration. Reinforcement-learning experience replay solves the analogous problem by training a policy on stored transitions; CER ports the *spirit* of that idea to the context window: rather than gradient-updating the model, the buffer is mutated and retrieved into context. This lets agents accumulate environment-specific knowledge across tasks without training, while the in-context nature gives implicit reasoning over noisy or contradictory experiences (so failed trajectories can still inform decisions).

The split into *dynamics* (where things are, what pages contain what, how to navigate) and *skills* (reusable procedural patterns for completing sub-goals) reflects two distinct ways past trajectories help the next decision: state-grounding versus action-selection priors.

## Variants

- **Online CER** — buffer is initialized empty and grows from self-generated trajectories during inference.
- **Offline CER** — buffer is pre-populated from human-annotated or LLM-explored trajectories collected before deployment, then frozen for inference.
- **Hybrid CER** — offline warm-up followed by online evolution. Empirically the strongest setting in the introducing paper.
- **Reward-filtered CER** — buffer includes only trajectories that pass a ground-truth evaluator; trades applicability for slightly higher quality.

## Comparison

- vs. **classical experience replay** (Schaul 2016, Rolnick 2019): no gradient updates; experiences are textual abstractions rather than (s, a, r, s') tuples.
- vs. **Agent Workflow Memory / AutoGuide / AutoManual / ExpeL / Synapse**: CER provides separated distillation and retrieval modules (most prior work lacks retrieval), maintains a dynamic accumulating buffer (vs. rewrite-style updates), and works without ground-truth reward.
- vs. **Tree-Search / sampling-based agents**: orthogonal — CER reuses *past* experience, search expands the *current* action space. Empirically they synergize.
- vs. **fine-tuning on trajectories**: training-free; no model weights are updated, so the same buffer transfers across backbone changes.

## Known limitations

- Quality of distilled experiences depends on trajectory structure: random-exploration trajectories degrade performance because the distillation prompt extracts misleading patterns.
- The dynamics representation leans on URL-addressable state; non-web environments need different state abstractions.
- Cold-start: pure online mode has no buffer for the first task.
- Buffer growth and retrieval-quality tradeoffs at scale are unexplored.

## Open problems

- How to filter or segment low-quality trajectories so they contribute without poisoning the buffer.
- Designing state-grounding analogues to "dynamics" for environments without natural addressable state.
- Training-free buffer compression as the experience set grows beyond what retrieval can handle.
- Generalizing distillation/retrieval modules across environments rather than per-domain prompts.

## Relationship to foundations

CER is conceptually downstream of two foundations: **experience replay** from RL (Schaul 2016; Rolnick 2019) and **in-context learning** in LLMs. It can also be viewed as a procedural-memory mechanism in the agentic-memory hierarchy, sitting alongside skill libraries and workflow memories.

## My understanding

CER's core insight is that, in a context-window-rich LLM regime, the *replay* in experience replay does not need to be parameter updates — it can be context augmentation. That collapses the training/inference distinction and makes self-improvement available to closed-weight models. The separation of dynamics vs. skills is the methodologically novel piece: many concurrent approaches mix them, and the ablation in the introducing paper shows both axes carry independent signal.
