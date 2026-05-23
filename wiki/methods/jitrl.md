---
name: JitRL (Just-In-Time Reinforcement Learning)
slug: jitrl
type: inference
tags:
  - test-time-rl
  - logit-modulation
  - non-parametric-memory
  - retrieval-augmented
  - llm-agents
source_papers:
  - just-time-reinforcement-learning-continual-learning
parent_methods: []
child_methods: []
code_repo: "https://github.com/liushiliushi/JitRL"
date_updated: 2026-05-18
---

## Problem setting

Improve an LLM agent's policy continually at deployment time on long-horizon decision tasks (web navigation, text-based games) without modifying the base model's parameters. The agent must (i) accumulate experience across episodes, (ii) generalize across tasks within a benchmark, (iii) avoid the cost and catastrophic-forgetting risk of gradient-based RL, and (iv) work with both white-box (log-prob-exposing) and black-box LLMs.

## Mechanism

JitRL is a three-component continual loop wrapped around a frozen LLM:

1. **Non-parametric experience memory** `M = {(s_t, a_t, G_t)}` storing structured states, actions, and discounted returns. A reflective evaluator turns raw end-of-episode reward into step-wise returns (see `[[reflective-step-wise-rewards]]`).
2. **Retrieval-based value estimation** at inference. For the current state `s`, retrieve `k` nearest neighbors `N(s)`, estimate `V̂(s)` as the average return over `N(s)`, and estimate `Q̂(s,a)` per candidate action `a`. For unseen actions, apply optimism-under-uncertainty (with probability `λ`, bonus `α/|N(s)|`; with probability `1−λ`, zero) to encourage exploration without unbounded variance.
3. **Closed-form logit update** `z'(s,a) = z(s,a) + β · Â(s,a)` where `Â(s,a) = Q̂(s,a) − V̂(s)`. This is proved (Theorem 4.1) to be the exact solution of the KL-constrained policy optimization objective `max E_{a∼π'}[A(s,a)] − (1/β) D_KL(π' ‖ π_θ)`. Two deployment variants — **Token-level logit** (use log-probs of index tokens for candidate actions) and **Verbalized logit** (elicit a 0–100 confidence and transform into pseudo-logits) — handle white-box vs black-box backbones.

## Procedure

At deployment, for each episode:

1. **Inference loop.** For each step `t`:
   - Abstract raw observation into structured state `s_t`.
   - Retrieve top-`k` neighbors `N(s_t)` from `M`.
   - Compute `V̂(s_t)` over `N(s_t)`.
   - For each candidate action `a_j` in the LLM's proposal set:
     - If `|N(s_t, a_j)| > 0`: `Q̂ = avg return over N(s_t, a_j)`.
     - Else (with prob `λ`): `Q̂ = V̂(s_t) + α/|N(s_t)|`. (With prob `1−λ`: `Q̂ = 0`.)
   - `Â(s_t, a_j) = Q̂(s_t, a_j) − V̂(s_t)`.
   - Update logits: `z'(s_t, a_j) = z(s_t, a_j) + β · Â(s_t, a_j)`.
   - Sample action from softmax(z').
2. **Memory update.** After episode termination:
   - The LLM-based Evaluator generates step-wise rewards `{r_t}` from the trajectory.
   - Compute `G_t = Σ_{u=t}^{T} γ^{u−t} r_u`.
   - Append each `(s_t, a_t, G_t)` to `M`.

Hyperparameters: `β` (logit temperature), `k` (retrieval neighbors, robust region 8–14), `γ` (discount), `λ` (exploration probability for unseen actions), `α` (optimism bonus scale).

## Assumptions

- The base LLM exposes either token-level log-probabilities or a usable verbalized-confidence channel.
- A structured state abstraction is available that maps functionally-equivalent observations to similar representations (URL normalization for web, entity/action extraction for text games).
- The LLM Evaluator can generate plausible step-wise rewards from a completed trajectory.
- The task admits multi-episode evaluation — the agent is given repeated attempts on each task.

## Limitations

- State abstraction is hand-engineered per environment, not learned.
- Memory grows unboundedly across episodes; no consolidation or retirement policy.
- Verbalized-Logit variant depends on the LLM's ability to faithfully report confidence — degraded performance on weakly-calibrated models is plausible.
- Evaluator quality bounds credit-assignment quality; the paper does not isolate this dependency.
- The closed-form derivation assumes a fixed action set per step; continuous or open-ended action spaces require additional engineering.

## Tradeoff profile

- **Cost**: ~$98 API cost on WebArena vs ~$10,000 estimated training cost for WebRL (per the paper) — **>30× cheaper** than weight-update baselines while matching/exceeding their accuracy.
- **Latency**: extra retrieval pass per step (top-`k` nearest-neighbor lookup over `|M|` triplets) adds modest overhead vs Static; negligible compared to LLM inference time.
- **Memory footprint**: scales linearly with experience volume; no built-in compression.
- **Expressiveness**: bounded by `π_θ`'s reachable distribution after additive logit nudge; cannot synthesize structurally new behaviors.
- **Generality**: model-agnostic (validated on Gemini-2.5-flash, GPT-5-mini, DeepSeek-V3.2); environment-agnostic *given* a usable state abstractor.
