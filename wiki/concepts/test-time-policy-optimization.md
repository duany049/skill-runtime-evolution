---
title: Test-Time Policy Optimization
aliases:
  - test-time RL
  - inference-time policy optimization
  - training-free policy improvement
  - gradient-free RL
tags:
  - test-time-adaptation
  - reinforcement-learning
  - llm-agents
  - inference-time-compute
maturity: emerging
definition: "Improving an agent's policy at inference time without any gradient updates to the underlying model parameters, by modulating the policy distribution using signals derived from past experience or other auxiliary computations."
key_papers:
  - just-time-reinforcement-learning-continual-learning
first_introduced: ""
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

Test-time policy optimization is the class of techniques that adjust an agent's action distribution at deployment time — without modifying the weights of the underlying model — to maximize some task-relevant objective. The defining constraint is that the base model `π_θ` is frozen; improvement comes from operations applied around `π_θ` (logit modulation, prompt augmentation, verbal feedback, retrieval-augmented context) rather than through SGD on `θ`.

## Intuition

Conventional RL conflates *learning* with *parameter update*. Test-time policy optimization separates the two: the model retains its general competence, and the deployed policy adapts using local signals (recent failures, retrieved trajectories, verbalized self-reflection) that need not survive to the next deployment. The trade-off is between *flexibility* (the agent can adapt instantly to any feedback channel) and *expressiveness* (without parameter updates, the policy is constrained to the manifold of distributions reachable from `π_θ` via the chosen modulation).

The KL-constrained framing makes the trade-off explicit: any test-time policy `π'` is judged against `π_θ` via `D_KL(π' ‖ π_θ)` plus an objective term. Different test-time methods correspond to different choices of objective and constraint.

## Variants

- **Logit modulation** — directly add a signal to the LLM's output logits. JitRL's `z'(s,a) = z(s,a) + β Â(s,a)` is one instance; the additive form is the exact KL-projection.
- **Prompt augmentation** — append retrieved or generated text to the input. Most ICL-based methods (AWM, Reflexion appended history) fall here.
- **Verbal self-feedback** — generate textual critiques of past trajectories and condition future actions on them (Reflexion).
- **Sampling-time modification** — re-weight or filter samples drawn from `π_θ` (rejection sampling, best-of-N, MCTS-style planning) without modifying logits.
- **Optimism injection** — bias toward exploration when memory coverage is sparse (JitRL's `α/|N(s)|` bonus).

## Comparison

Versus **gradient-based RL fine-tuning** (WebRL, ToolRL, GRPO): test-time methods are radically cheaper but cannot improve beyond what `π_θ`'s reachable distribution and the local signal permit. Gradient-based methods can in principle learn structurally new behaviors but risk catastrophic forgetting and need much more compute.

Versus **pure in-context learning**: test-time policy optimization is the strict superset. ICL is the special case where the modulation channel is "append text to context"; logit modulation, sampling modification, and verbal feedback are additional channels with different cost/expressiveness profiles.

## Known limitations

- Bounded by `π_θ` — if the optimal action has near-zero base probability, no finite logit nudge will sample it reliably.
- Local — typical implementations adapt to recent trajectories only; long-tail tasks may receive no useful retrieval.
- Black-box backbones may not expose logits, forcing approximations (verbalized confidence, sampling-based estimation).

## Open problems

- Principled comparison of modulation channels (logit vs prompt vs verbal) on identical retrieval signals.
- Convergence rates under non-stationary task distributions.
- How to compose multiple test-time modulation channels safely.

## Relationship to foundations

Builds on the KL-regularized policy optimization objective from RL theory and on retrieval-augmented inference patterns.

## My understanding

The concept matters because it forces a precise question: *what is the optimal test-time update under a stated regularizer?* JitRL answers that for one specific objective; future work in this space will define new objectives (different constraints, different reward proxies) and derive their closed-form modulations.
