---
name: Bottom-Up Skill Evolution
slug: bottom-up-skill-evolution
type: system
tags:
  - llm-agent
  - skill-library
  - mcts
  - visual-grounding
  - bottom-up-agent
source_papers:
  - rethinking-agent-design-top-down-workflows
parent_methods: []
child_methods: []
code_repo: "https://github.com/AngusDujw/Bottom-Up-Agent"
date_updated: 2026-05-18
---

## Problem setting

The agent operates in an open-ended environment modeled as a POMDP `(X, A, T, R)` with the following constraints:

- Observations `x_t ∈ X` are **visual only** — raw screenshots, no privileged state or structured game data.
- Atomic actions `A` are **low-level human-like primitives** — mouse clicks, drags, key presses.
- No explicit reward `R` is exposed; no APIs, no subgoals, no game-specific prompts are available.
- The agent is powered by an LLM `M` that serves multiple prompted roles (proposal, selection, refinement, description, semantic-diff judging).

The goal is to grow a skill library `S = {σ_1, …, σ_n}` from `S = ∅` such that the agent can sustain progress in the environment using only atomic actions and LLM reasoning.

## Mechanism

Each skill `σ = (a_1, …, a_k)` is a sequence of atomic actions, paired with an LLM-generated semantic descriptor `d_σ`. The library evolves under a three-term reward:

`R_skill(σ, S, x_t, T) = R_diversity(σ, S) + R_efficiency(σ) + R_semantics(d_σ, T)`

In the published implementation, only `R_semantics` is operationalized; the other two terms are conceptual.

Three coordinated modules drive evolution:

1. **Skill Augmentation** — When the agent cannot find a viable existing skill (`S_t = ∅`), it grows new skills by progressively appending atomic actions: `σ^(k) = Augment(σ^(k-1), a_k)`, starting at `k = 1`. A candidate sequence is retained only if it triggers a **recognizable visual change** in the environment. Each retained sequence receives an LLM-generated semantic descriptor.
2. **Skill Invocation** — Given a new observation `x_t`, the LLM samples a candidate set `S_t = M(p_select, x_t, S)`. **Monte Carlo Tree Search (MCTS)** then simulates short rollouts to estimate the expected utility of each candidate, and the best-scoring skill `σ*` is executed. If `S_t = ∅`, the agent falls back to Skill Augmentation.
3. **Skill Evaluation and Refinement** — After execution, the LLM computes the **implicit semantic reward** `R_semantics = M(p_differ, σ, x_t, x_{t+1})` from the visual diff between pre- and post-execution screenshots. Skills that score poorly across multiple invocations or across multiple agents are pruned from the library. When viable skills are sparse, the LLM rewrites a candidate as `σ' = M(p_refine, (x_t, σ, T))`, replacing the original only if the rewrite shows improved alignment or efficiency.

**Visual grounding.** Because click targets could theoretically land anywhere on screen, the **Segment Anything Model (SAM)** is used to identify and segment interactable UI elements, drastically reducing the effective action space. Recognized UI elements are also added to the skill library.

**Environment-agnostic prompting.** All prompts (`p_select`, `p_describe`, `p_refine`, `p_differ`) are explicitly designed to contain no game-specific knowledge, so the identical codebase runs across multiple environments.

## Procedure

The full algorithm (Algorithm 1 in the source paper):

```
Input: environment (X, A, T, R), skill library S = ∅, LLM M
for each agent:
  while episode not terminated:
    observe x_t ∈ X
    sample S_t = M(p_select, x_t, S)            # candidate skills
    if S_t = ∅:                                 # AUGMENT
      for k = 1 to k_max:
        generate σ^(k) = Augment(σ^(k-1), a_k)
        if effect(σ^(k)) is recognizable:
          d_σ = M(p_describe, (x_t, σ^(k), T))
          S = S ∪ {(σ^(k), d_σ)}
          break
    else:                                       # INVOKE
      evaluate S_t with MCTS
      execute best skill σ*
    observe trajectory T, compute R_semantics
    if R_semantics low across agents:
      remove σ from S                            # PRUNE
    if skill count < threshold:
      σ' = M(p_refine, (x_t, σ, T))              # REFINE
      replace σ with σ' if better
```

In the published experiments, agents run for up to 1,000 atomic-action steps (~6.5 hours wall-clock) per episode, 3 episodes per environment, using GPT-4o as `M`.

## Assumptions

- The environment exposes a **screen** and accepts **mouse/keyboard** input; embodied or non-visual environments need a different perception/action interface.
- The LLM is capable of zero-shot **VLM-style semantic-diff judgement** on two screenshots — without this, `R_semantics` is unreliable.
- Behaviors of interest produce **observable visual change**; subtle defensive or long-horizon strategies are systematically under-rewarded.
- A single shared skill library is assumed coherent; concurrent edits across multiple agents are not handled by the published implementation.

## Limitations

- **Exploration overhead.** Bootstrapping from zero prior requires ~2-2.5× more environment steps than prior-assisted baselines.
- **Flat skill representation.** Skills are record-and-replay action sequences, not parameterized callable abstractions; cross-environment transfer is therefore minimal.
- **Reward myopia.** Visual-diff implicit reward fails to capture delayed or strategic effects.
- **No reset protocol.** Open-ended games lack reproducible reset/seed mechanisms, introducing variance in evaluation.
- **Real-time inapplicability.** The method is restricted to turn-based environments; current LLM inference latency precludes real-time control.

## Tradeoff profile

| Tradeoff | Position | Comment |
|---|---|---|
| Generality across environments | **High** | Same architecture and prompts work for Slay the Spire and Civilization V |
| Sample efficiency | **Low** | Zero prior + visual-only observation makes bootstrapping slow |
| Per-action token cost | **Moderate-to-high** | Multiple LLM calls per step (select, describe, differ, refine, MCTS rollouts) |
| Reward fidelity | **Low** | Single-step visual-diff reward misses delayed effects |
| Engineering footprint | **Low for the agent, high for visual grounding** | The agent is one codebase; SAM and the screen-interaction harness carry most of the operational complexity |
| Multi-agent coordination | **Unsolved** | Shared-library design is assumed, not implemented |
