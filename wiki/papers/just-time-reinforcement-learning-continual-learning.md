---
title: "Just-In-Time Reinforcement Learning: Continual Learning in LLM Agents Without Gradient Updates"
slug: just-time-reinforcement-learning-continual-learning
arxiv: "2601.18510"
venue: arXiv
year: 2026
tags:
  - continual-learning
  - test-time-adaptation
  - reinforcement-learning
  - llm-agents
  - logit-modulation
  - non-parametric-memory
  - retrieval-augmented
importance: 4
date_added: 2026-05-18
source_type: tex
tldr: "JitRL is a training-free framework that performs RL-style policy optimization at test time by retrieving past trajectories to estimate advantages and adding them directly to LLM output logits — derived as the closed-form solution to KL-constrained policy optimization."
contribution_type:
  - method
  - theory
datasets:
  - WebArena
  - WebArena-Lite
  - Jericho
  - Zork-1
  - Zork-3
  - Library
code_url: "https://github.com/liushiliushi/JitRL"
cited_by: []
---

## Problem & Context

LLM agents are deployed with frozen weights, so they cannot learn from on-the-job experience the way humans can — they re-make the same mistakes on similar tasks. Hendrycks et al. (2025) flag "capability to continually learn new information" as the largest current AGI gap.

Two existing answers are both unsatisfactory. **Gradient-based RL** (e.g. WebRL, ToolRL, ToRL) can in principle update policies from rewards, but it is data-hungry, computationally expensive, prone to catastrophic forgetting, and only modestly effective in continual-adaptation evaluations. **In-context learning** approaches (Reflexion, AWM, MemGPT-style memory) inject textual descriptions of past experience into the prompt, but they degrade as context grows (lost-in-the-middle) and they cannot encode the kind of policy-level skill that emerges only from reward signals — prompts are restricted to what can be written in text. The open question driving the paper is whether agents can learn continually using an RL formulation while avoiding any gradient updates.

## Key idea

Treat the LLM as a fixed reference policy `π_θ` and perform test-time policy optimization in **logit space**. Maintain a dynamic non-parametric memory `M = {(s_i, a_i, G_i)}` of past `(state, action, discounted return)` triplets. At inference, retrieve `k` nearest neighbors of the current state, compute an empirical state value `V̂(s)` and action value `Q̂(s,a)` by averaging returns, derive an advantage `Â(s,a) = Q̂(s,a) − V̂(s)`, and **add it (scaled by a temperature `β`) directly to the LLM's logits**: `z'(s,a) = z(s,a) + β · Â(s,a)`. This additive update is proved to be the exact closed-form solution of the KL-constrained policy optimization objective `arg max_{π'} E_{a∼π'}[A(s,a)] − (1/β) D_KL(π' ‖ π_θ)`, so no gradient descent is needed.

## Method

JitRL operates in a continuous Inference → Memory-Update loop with three components:

**1. Memory construction.** At the end of each episode, an LLM-based **Evaluator** turns the completed trajectory `τ = (s_1, a_1, …, s_T, a_T)` into step-wise rewards `{r_t}` (reflective step-wise rewards — the paper's credit-assignment trick under sparse outcome rewards). These are aggregated into discounted returns `G_t = Σ_{u=t}^{T} γ^{u−t} r_u`. Raw states (HTML DOM trees, game text) are abstracted into compact structured states `s_t` that map functionally-equivalent states to similar representations (e.g. URL normalization in web navigation, structured entity/action extraction in text games). Each transition `(s_t, a_t, G_t)` is appended to memory `M`.

**2. Test-time value estimation.** Given current state `s`, retrieve top-`k` neighbors `N(s)` from `M`. Compute `V̂(s) = (1/|N(s)|) Σ_{i∈N(s)} G_i`. For each candidate action `a`, partition `N(s,a) = {(s_i, a_i, G_i) ∈ N(s) : a_i = a}` and:
  - **Known actions** (`|N(s,a)| > 0`): `Q̂(s,a) = (1/|N(s,a)|) Σ_{j∈N(s,a)} G_j`.
  - **Unseen actions** (`|N(s,a)| = 0`): with probability `λ` apply optimism-under-uncertainty `Q̂(s,a) = V̂(s) + α/|N(s)|` (the `α/|N(s)|` bonus shrinks as memory grows), and with probability `1−λ` assign `Q̂(s,a) = 0` to avoid over-exploration.
  - Advantage: `Â(s,a) = Q̂(s,a) − V̂(s)`.

**3. Policy update.** Solve `π* = arg max_{π'} E_{a∼π'}[A(s,a)] − (1/β) D_KL(π' ‖ π_θ)`. The closed form is `π*(a|s) ∝ π_θ(a|s) exp(β A(s,a))`, equivalent in logit space to `z'(s,a) = z(s,a) + β Â(s,a)`. Two deployment variants accommodate black-box backbones: **Token-level logit** (prompt the model to emit an index token, extract log-prob), and **Verbalized logit** (prompt the model for a 0–100 confidence and transform to logits) when log-probs are unavailable.

**Theoretical analysis (Theorems 4.1–4.3).** (i) The logit update is the exact closed-form solution to the KL-constrained problem. (ii) `V̂_t`, `Q̂_t`, `Â_t` converge in probability to true values `V^{π_t}`, `Q^{π_t}`, `A^{π_t}` even under non-stationary policies. (iii) Combining the two, `π̂_t(·|s) →_p π*_t(·|s)`.

## Experiment & Results

**Benchmarks.** WebArena (realistic multi-site web navigation: Shopping, Reddit, Map, Admin/CMS, GitLab) and Jericho text adventures (Library, Zork 1, Zork 3).

**Backbones.** Training-free methods (JitRL plus Static, Memory FIFO, Reflexion, AWM, EvoTest) use `Gemini-2.5-flash`; cross-backbone generalization uses `GPT-5-mini` and `DeepSeek-V3.2`. Weight-update baselines use the released `Llama-3.1-70B-Instruct` checkpoints for SFT/WebRL (WebArena) and per-game `Qwen3-32B` checkpoints trained with GRPO (Jericho).

**Protocol.** Each task is run for L consecutive episodes (L=5 on WebArena, 50 on Jericho). Metrics: Average Success/Score (mean over all attempts → learning efficiency) and Final Success/Score (mean of the L-th attempt → converged performance). A wide Final–Avg gap signals successful on-the-fly learning.

**RQ1 — WebArena vs training-free baselines.** JitRL is SOTA on both Avg and Final success rates across all WebArena sites. Largest gains in structured domains: **Shopping +73.2% over Static**. Reflexion suffers from "reflection noise" on sites like Map.

**RQ1 — WebArena vs weight-update.** On the held-out WebArena-Lite subset, JitRL matches/exceeds WebRL while using only inference-time optimization. The paper claims **>30× lower monetary cost** than WebRL training (the cost-comparison table in the appendix sources estimates from H200 GPU training expenses for WebRL vs API usage for JitRL).

**RQ1 — Jericho.** JitRL achieves the highest Avg and Final scores on all three games, beating GRPO-trained checkpoints. Learning curves show: rapid initial learning (10–15 episodes), widening gap over baselines, and reduced variance in later episodes. GRPO shows high variance throughout, indicating gradient methods struggle with sparse text-game rewards. AWM/Memory plateau early due to over-reliance on fixed memory patterns.

**RQ2 — Generalization.** JitRL holds SOTA across `Gemini-2.5-flash`, `GPT-5-mini`, `DeepSeek-V3.2` (model-agnostic). Cold-start cross-task generalization on WebArena (memory restricted to disjoint tasks) still beats baselines, indicating transfer of abstract procedural knowledge rather than memorization. Cross-task retrieval contributes ~50% of retrieved memory on average (62.19% on Shopping).

**RQ3 — Qualitative case studies.** Memory overrides the base model's intuition in three patterns: **Site Functionality** (e.g. ignore the intuitive "Catalog" link, navigate "Marketing"), **Navigation Precision** (prefer deterministic nav links over noisy global search), **UI Mechanics** (use hover-to-reveal subcategories instead of click-into-category).

**RQ4 — Ablations.** **Logit-Update beats Prompt-Update** (same retrieved memory but appended to prompt instead of added to logits) on Admin and Reddit — confirming the gain comes from direct distribution modulation, not just retrieval. **Retrieval neighbor count `k`** has a clear info-noise trade-off: robust region is k=8–14; too few causes high variance from insufficient evidence, too many adds noise that delays convergence.

## Limitations

- The non-parametric memory may inadvertently store PII (HTML DOM trees, interaction history) in real deployments — the paper flags this as a privacy risk requiring anonymization/filtering protocols.
- State abstraction is task-specific (URL normalization for web, entity extraction for text games) — the construction of "functionally equivalent states" is hand-engineered per environment, not learned.
- The logit-update mechanism requires either log-prob access or the ability to elicit verbalized confidences from the LLM. While the Verbalized Logit variant addresses black-box models, it relies on the model following confidence-output instructions reliably.
- Reflective step-wise rewards depend on the LLM Evaluator's judgement; in trajectories where outcome reward signals are subtle or contested, the Evaluator may misattribute credit. The paper does not isolate Evaluator quality.
- The Unseen-Actions branch uses a fixed `λ`-coin-flip between an optimism bonus and zero. The schedule for `λ`, `α`, `β`, `γ`, `k` is treated as a hyperparameter sweep, not a learned/adaptive mechanism.
- Memory grows unboundedly across episodes — there is no consolidation, retirement, or pruning policy described in the main text. This is acceptable for the benchmarks studied but is a deployment concern.

## Open questions

- Can state abstraction be learned end-to-end rather than hand-engineered per environment?
- How does JitRL behave when the environment distribution shifts during deployment (non-stationary tasks vs non-stationary policy)? The convergence theorem assumes underlying-true-value stability across the retrieval neighborhood.
- What is the interaction between memory size `|M|`, retrieval `k`, and effective sample complexity — is there a regime where storing more hurts because retrieval becomes too noisy?
- Can the closed-form KL-constrained update be combined with parameter-efficient fine-tuning (e.g. LoRA) to get the best of both training-free and gradient-based approaches?
- How does JitRL compose with other test-time policies (Reflexion-style verbal feedback, AWM-style procedural memory) — are they additive?

## My take

The technical core is elegant: deriving advantage-modulated logit addition as the exact closed-form solution of KL-constrained policy improvement reframes a lot of retrieval-augmented test-time tricks under a single principled objective. The theory is not deep — Theorem 4.1 is a textbook KL-projection result, applied carefully — but the *framing* is the contribution: it shows that "add retrieved-advantage to logits" is not a heuristic but the optimal training-free RL update under a specific (and reasonable) regularizer.

Empirically the >30× cost reduction relative to WebRL is the headline number, and the comparison is structurally fair (held-out WebArena-Lite). The cross-task memory analysis (≈50% of retrieved triplets come from different tasks) is the most interesting finding for the procedural-memory line: it suggests that what JitRL learns is closer to *abstract task structure* than verbatim trajectory recall, which connects naturally to skill-evolution work.

The main weakness is environment-specific state abstraction — for a method that claims model-agnostic training-free generality, hand-crafted state extractors are a real engineering tax. If the next paper in this line learns state abstraction (e.g. via contrastive representation of functionally-equivalent states), JitRL becomes much more deployable.

## Related

- [[jitrl]]
- [[reflective-step-wise-rewards]]
- [[test-time-policy-optimization]]
- [[non-parametric-experience-memory]]
- [[kl-constrained-logit-update]]
- [[bryan-hooi]]
