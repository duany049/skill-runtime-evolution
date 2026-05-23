---
title: Skill Boundary Check
aliases:
  - capability-aware early stopping
  - knowledge-boundary awareness
  - failed-skill early termination
tags:
  - computer-use-agent
  - inference-efficiency
  - early-stopping
  - test-time-scaling
  - skill-library
maturity: emerging
definition: "An inference-time mechanism in which an agent compares the required skill for the incoming query against its skill set's recorded failure entries and terminates early if that skill was unreachable during prior learning, avoiding wasted test-time-scaling rollouts."
key_papers:
  - osexpert-computer-use-agents-learning-professional
first_introduced: "OSExpert (Liu et al., 2026)"
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

A skill boundary check is the inference-time procedure of mapping the incoming user query (or its decomposed sub-skills) against the *failed* entries of a learned skill library — i.e. unit functions that repeatedly failed under bounded retries during exploration. If a required skill is marked failed, the agent stops early with an error signal rather than spending its remaining test-time-scaling budget on retries that are unlikely to succeed.

## Intuition

Today's CUAs cannot tell when they're in over their head: they keep retrying until the interaction budget runs out. Humans recognize "I don't know how to do this in this tool" almost immediately and either ask for help or give up; a skill boundary check is the same gesture, but implemented as a lookup against a structured record of what the agent already failed at during learning. The signal is cheap — it requires only the skill set the agent already built — but unlocks most of the efficiency win because failed long-horizon tasks are exactly where current CUAs waste the most time.

## Variants

- **Hard early stop**: if any required sub-skill is marked failed, terminate immediately.
- **Soft routing**: failed-skill matches downgrade the agent to a fallback policy (e.g. ask the user, or run a more capable but more expensive backbone) rather than terminating.
- **Per-step boundary check**: instead of a one-shot lookup, the check fires whenever a new sub-skill is dispatched during execution.

## Comparison

Distinct from generic confidence-based early-stopping (e.g. entropy thresholds, value-function gating in RL), which rely on internal model signals; the skill boundary check uses an *external*, empirically-grounded record of past failures. Distinct from blind test-time scaling, which trades latency for marginal recovery probability without consulting capability.

## Known limitations

- Failed entries conflate true infeasibility with agent-policy limits, so the check can over-prune for weak backbones.
- Requires upstream learning that actually catalogues failures; a skill set without explicit failure entries cannot support the check.
- Coverage depends on whether the query's required skill cleanly maps to a stored skill (or stored failure); novel skills bypass the check.

## Open problems

- How to update failed entries when a stronger backbone, a UI patch, or a user hint plausibly invalidates the prior failure.
- Calibrating the check so it cuts latency without measurably hurting success rate.
- Disentangling failure attribution (agent policy vs. environment vs. UI ambiguity) inside the boundary check.

## Relationship to foundations

The mechanism is a specific instantiation of *known-unknown* reasoning: the agent maintains an explicit set of things it has tried and failed at, and uses set membership as a stopping criterion. Related in spirit to abstention in classification, deferral in selective prediction, and the "rejection option" in active learning.

## My understanding

The most striking result in OSExpert is that the skill boundary check supplies most of the latency reduction, not the fast planner. That's a useful signal for the field: capability-aware early stopping is a small mechanism with large downstream impact, and it generalizes beyond GUI agents — any agent with a structured record of past failures can apply it. The tricky engineering question is failure attribution: the value of the check decays sharply once "failed" entries become a graveyard of weak-backbone artifacts.
