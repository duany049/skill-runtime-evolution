---
name: Reflective Step-wise Rewards
slug: reflective-step-wise-rewards
type: evaluation
tags:
  - credit-assignment
  - llm-as-judge
  - reward-shaping
  - sparse-reward
  - reinforcement-learning
source_papers:
  - just-time-reinforcement-learning-continual-learning
parent_methods: []
child_methods: []
code_repo: ""
date_updated: 2026-05-18
---

## Problem setting

In long-horizon RL tasks (web navigation, text-based games), the environment usually returns only an outcome reward at episode termination (success/failure, final game score). This sparse signal is hard to attribute to individual actions across a multi-step trajectory — the classical credit-assignment problem. Trained value networks address this but require gradient updates; training-free agents need an alternative.

## Mechanism

After each episode, an **LLM-based Evaluator** is prompted with the completed trajectory `τ = (s_1, a_1, …, s_T, a_T)` and the outcome signal, and is asked to emit a step-wise reward `r_t` for each action: a scalar quantifying how much `a_t` contributed to the overall episode result. Formally, `E: τ → {r_t}_{t=1}^T`. These step-wise rewards are then aggregated into discounted returns `G_t = Σ_{u=t}^{T} γ^{u−t} r_u`, which are what gets stored in non-parametric experience memory.

## Procedure

1. Run episode to termination, collecting the trajectory.
2. Receive environment outcome reward (success/failure, game score).
3. Prompt the Evaluator (a separate LLM call) with the full trajectory + outcome, requesting step-wise reward annotations.
4. Parse the Evaluator's response into a sequence `{r_t}`.
5. Compute `G_t` via the discount formula above.
6. Store `(s_t, a_t, G_t)` triplets in memory.

The Evaluator can in principle be the same model as the policy LLM or a different one; the paper uses the same backbone family for both.

## Assumptions

- The Evaluator can interpret the trajectory and assign meaningful per-step credit. This is most plausible when the trajectory is structured (action labels, intermediate observations are interpretable text) and the task has identifiable sub-goals.
- The Evaluator's per-step rewards are at least monotonically consistent with the true (unknown) advantage signal — the actual scale matters less than the relative ranking, because the advantage centering `Â = Q̂ − V̂` cancels global biases.
- The outcome reward from the environment is reliable.

## Limitations

- Adds one extra LLM call per episode termination, which can dominate cost on cheap baselines.
- Evaluator hallucination: in trajectories with subtle causal chains, the Evaluator may misattribute credit, propagating noise into the memory.
- No principled calibration of `r_t` magnitudes across episodes; relies on the model's implicit consistency.
- No mechanism for the Evaluator to update its own judgment as the agent learns — the Evaluator is also frozen.

## Tradeoff profile

- **Cost**: ~one Evaluator LLM call per episode; cheap relative to gradient-based credit assignment.
- **Quality**: bounded by the LLM's reasoning over trajectories; best when the trajectory contains explicit sub-goal completion signals.
- **Generality**: works for any task where the trajectory can be serialized into text; does not require environment-specific reward shaping.
- **Composability**: orthogonal to the policy update mechanism — could plug into any retrieval-based RL method that needs return estimates.
