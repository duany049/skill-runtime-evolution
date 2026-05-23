---
title: Memory Update Policies
aliases:
  - memory lifecycle operators
  - update strategies
tags:
  - procedural-memory
  - lifecycle
  - skill-evolution
maturity: emerging
definition: "The family of online operators that govern how an agent's procedural memory bank evolves between task batches — including append-only merging, validation-filtered consolidation, and reflexion-style in-place revision."
key_papers:
  - memp-exploring-agent-procedural-memory
first_introduced: "Memp (2025) formalizes the policy family $U(M(t), E(t), \\tau_t)$"
date_updated: 2026-05-18
related_concepts:
  - procedural-memory-framework
linked_ideas: []
---

## Definition

A memory update policy is a function $U(M(t), E(t), \tau_t) \to M(t+1)$ that takes the current procedural memory $M(t)$, the execution feedback $E(t)$ (success/failure, performance metrics), and the recently completed trajectories $\tau_t$, and produces the next state of the memory bank. The general form admits three primitive operations:

$$U = \text{Add}(M_{\text{new}}) \ominus \text{Del}(M_{\text{obs}}) \oplus \text{Update}(M_{\text{est}})$$

## Intuition

Most prior procedural-memory systems treat update as append-only — a "merge" of newly observed trajectories into the bank. This silently corrupts the bank with failed or redundant trajectories and never deprecates outdated procedures. Update *policies* recognize that the lifecycle operators (add / delete / revise) are independent design choices.

## Variants

- **Vanilla** — append every consolidated trajectory regardless of outcome (the baseline).
- **Validation** — only successful trajectories are abstracted and added; failures and redundant items are discarded.
- **Adjustment** (reflexion-style) — when a retrieved memory leads to a failure, merge the failing trajectory with the original memory and rewrite the entry in place.

## Comparison

Within procedural-memory work, Vanilla is the implicit default. Validation is a quality-gated variant. Adjustment goes further by closing the loop with error correction. Each variant captures a different theory of where memory bank quality comes from: *quantity* (Vanilla), *purity* (Validation), or *iteration* (Adjustment).

## Known limitations

- All variants depend on environment-supplied reward to score $E(t)$; without it the policy degenerates to Vanilla.
- The update interval $t$ (number of tasks between refreshes) is a hyperparameter, not learned.
- No principled deprecation criterion — entries can become stale even under Adjustment.

## Open problems

- Can the update operator itself be learned from data (e.g. via an off-policy bandit over update strategies)?
- How should Adjustment behave when multiple retrieved memories collectively fail — is local in-place rewrite sufficient?
- What is the right metric for memory-bank "health" beyond downstream task accuracy?

## Relationship to foundations

Connects to classical memory consolidation in psychology (selective retention of successful experiences) and to reinforcement-learning replay-buffer management (which trajectories to keep, replay, prune).

## My understanding

Update policies are the locus of skill evolution in procedural-memory systems. If Build and Retrieve set the stage, Update is what makes the agent learn over deployment time. The reflexion-style Adjustment result in Memp suggests in-place revision is meaningfully stronger than append-only — which means future continual-learning protocols should test against Adjustment, not Vanilla.
