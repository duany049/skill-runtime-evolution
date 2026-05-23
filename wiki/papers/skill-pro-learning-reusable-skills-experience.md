---
title: "Skill-Pro: Learning Reusable Skills from Experience via Non-Parametric PPO for LLM Agents"
slug: skill-pro-learning-reusable-skills-experience
arxiv: "2602.01869"
venue: ""
year: 2026
tags:
  - procedural-memory
  - skill-evolution
  - agentic-memory
  - non-parametric-optimization
  - llm-agents
importance: 4
date_added: 2026-05-18
source_type: tex
s2_id: 28f0b6f57e57337c7db267ae4ac52c12cbb5d376
tldr: "Skill-Pro lets a frozen LLM agent autonomously grow a small pool of reusable natural-language procedural Skills (activation/execution/termination triples) via a Non-Parametric PPO loop that proposes candidates from semantic gradients and accepts them through a PPO-style trust-region gate plus online score-based pruning."
contribution_type:
  - method
  - benchmark
datasets:
  - ALFWorld
  - TextArena
  - Mastermind-v0
code_url: "https://github.com/Miracle1207/Skill-Pro"
cited_by: []
---

## Problem & Context

LLM-driven agents have become competent at sequential decision-making, but their performance is dominated by *on-the-fly reasoning*: even in recurring situations they re-derive solutions from prompts and feedback rather than reusing a distilled procedure. This is computationally redundant and amplifies error in long-horizon tasks.

Before Skill-Pro, interaction-experience reuse in LLM agents fell into two camps. *Parametric* approaches (RLHF, fine-tuning, RL with verifiable rewards) bake experience into model weights at high cost, with risks of catastrophic forgetting and capability narrowing. *Non-parametric* approaches keep the LLM frozen and place experience in external memory — raw trajectories (RAG, Reflexion-style), distilled insights (Expel), notes (A-MEM), workflow graphs (AWM), or hybrid graph memories (G-Memory). These behave like *episodic* memory: the agent still has to read past cases into a context window and re-derive what to do, so storage grows while per-step inference stays expensive.

The paper's thesis is that the missing piece is *procedural* memory in the human-cognition sense: an implicit (or at least executable) mapping from situation to action pattern that the agent invokes without re-deriving. The challenges are formalising what a procedural unit is (C1: executability), making such units actually reused at decision time with consistent gain (C2: reusability), and learning/refining them without touching LLM weights (C3: non-parametric optimization).

## Key idea

Treat the *Skill pool* itself as the learnable object, optimised by a non-parametric analogue of PPO. Skills are natural-language triples `<activation, execution, termination>`; the agent is modelled as a Skill-augmented MDP whose policy factorises into Skill selection plus an LLM action policy conditioned on the selected Skill. Learning is done entirely by *editing the pool*: candidate Skills are proposed via natural-language "semantic gradients" derived from hindsight attribution over trajectory batches, then admitted only if a PPO-style clipped surrogate evaluated against past behaviour is positive. An online advantage-style score prunes underperforming Skills, keeping the pool small and high-quality.

## Method

Skill-Pro instantiates three coupled objects.

**Skill-MDP.** A standard MDP `(S, A, P, R, γ)` is extended with a dynamic Skill pool `Ω = {ω⁽¹⁾, ..., ω⁽ᴷ⁾}` of fixed capacity `K`. At each step `t`, a Skill-selection policy `μ(ω | s_t, Ω)` picks `ω_t` (paper instantiates two simple `μ`: top-similarity by embedding/LLM-judge, or value-based top-k by `Q(s_t, ω)`). A frozen LLM action policy `π_LLM(a | s_t, ω_t)` then produces primitive actions until the Skill's natural-language termination condition fires. The hierarchical policy factorises as `π_Ω(ω_t, a_t | s_t) = μ · π_LLM`. Optimisation target is expected cumulative discounted return. With `π_LLM` and `μ` frozen, the only learnable quantity is the *Skill pool evolution operator* `E`, applied iteratively over batches.

**Non-Parametric PPO.** A two-stage replacement for PPO's gradient step.

1. *Semantic Gradients.* For each invoked Skill `ω` and each trajectory `τ_i` where it was active, hindsight attribution decomposes the outcome into refinement suggestions for activation, execution, and termination, yielding `g_i = (g_i^I, g_i^π, g_i^β)`. An LLM-based `Aggregate(·)` consolidates a batch of `B` such gradients into a stable `ḡ_ω`, filtering trajectory-specific noise. A candidate is produced by `ω' = ω ⊕ ḡ_ω` — an LLM edit operation that rewrites the three natural-language fields along `ḡ_ω`'s direction. This is the analogue of a gradient-ascent step on a textual policy.
2. *PPO Gate (trust-region verification).* Treating `π_LLM` as the stochastic policy, the importance ratio `ρ_t(ω') = π_LLM(a_t | s_t, ω') / π_LLM(a_t | s_t, ω)` is computed over a batch of historical trajectories collected under behaviour Skill `ω`. With return-to-go advantages `Â_t = G_t − R̄` and ε-clipping, the clipped surrogate `L_CLIP(ω')` (the *PPO Gate*) scores each candidate. Out of `N_c` candidates the best is taken (`ω_new = argmax J(ω')`) and only admitted if `J(ω_new) > 0`.

**Score-based pool maintenance.** Each Skill carries an online score `Score(ω) = G_b(ω) / max(1, N_b(ω))` where `G(ω; τ)` is the average advantage `r_t − r̄` accumulated over time-steps when `ω` was active (or `(R(τ) − R̄)/|τ|` when only a trajectory-level return exists). When the pool exceeds capacity `K`, Skills with non-positive score, duplicates (cosine-similar), and ascending-score tails are pruned.

The full loop (Algorithm 1) is: collect batch → extract semantic gradients per invoked Skill → propose `N_c` candidates → PPO-Gate filter → update online scores → prune. The LLM is never updated.

## Experiment & Results

**Setup.** Two benchmarks: ALFWorld (train vs OOD held-out tasks) and TextArena's Mastermind-v0 (three difficulty tiers: base / Hard / Extreme). Memory is *built* on the in-domain task, then *reused* on harder variants (cross-task) and on other LLM backbones (cross-agent). Each setting averaged over 50 episodes. Backbones: Gemma-2-9B builds Skills; reuse evaluated with Gemma-3-4B, Qwen3-32B, LLaMA-3.3-70B-Instruct (TextArena) and Qwen3-32B (ALFWorld). Baselines: RAG, Expel, A-MEM, AWM, G-Memory (memory-augmented); CoT, ReAct, plain State agent (no memory).

**Reuse and efficiency (Table 1).** Skill-Pro reuse rate is 0.925 in-domain on Mastermind-v0, 0.825 / 0.900 on Hard / Extreme cross-task, 0.850 / 0.875 on Gemma-3-4B / Qwen3-32B cross-agent. Strongest baseline (RAG) sits at 0.349 in-domain; G-Memory at 0.091. Storage is 816 total tokens (vs 40,510 for G-Memory and 391,706 for AWM), at 102 tokens / Skill. ΔPrompt tokens per step is 273 (RAG: 2698, Expel: 5210). Retrieval ratio is 0.591 because Skills span multiple primitive steps; most baselines retrieve every step (ratio = 1.0).

**Performance (Table 2).** Skill-Pro reaches 0.900 on ALFWorld train and 0.909 on OOD (best in both columns). On Mastermind-v0 / Hard / Extreme average return is 0.606 / 0.463 / 0.333; G-Memory edges it on Extreme (0.356) but at ~50× storage. Cross-agent on Mastermind-v0 it wins on all three backbones (0.444 / 0.615 / 0.647 for Gemma-3-4B / Qwen3-32B / LLaMA-3.3-70B).

**Ablations (Table 3, Mastermind-v0).** *w/o Skill* drops performance 0.606 → 0.388 (-36%). *w/o NP-PPO* (initial seeds, no evolution) drops reuse 39% and performance 21%. Within NP-PPO: *w/o SG* (replace semantic gradients with trajectory summaries) drops PPO Gate Pass Rate by 30% (59.49% → 41.54%) and online score by 96%. *w/o PPO Gate* destabilises training (curves diverge) although Pass Rate trivially hits 100%. *w/o Score (FIFO)* is the worst: online score goes negative (-0.0064), reuse rate falls to 0.131.

**Distribution and evolution (Figs 4–5).** Skill-invocation distribution is invariant across Mastermind difficulty tiers, suggesting the learned Skills capture task-level logic; across backbones distributions differ (Gemma-2-9B is heavy on `FBInference`, others lean on `StratPlan`). Evolutionary lineage plots show repeated refinement events and score-based prunings producing the final compact pool.

## Limitations

- Skills remain *explicit* natural-language artefacts (cf. Claude Agent Skills). Truly implicit / sub-symbolic procedural memory is deferred to future work (Appendix F).
- The Skill-selection policy `μ` is treated as fixed and simple (similarity or value top-k); coupling Skill *selection* learning with Skill *pool* learning is left open.
- `π_LLM` is frozen by construction. Whether the gains stack with a co-evolving fine-tuned LLM is not measured.
- Evaluation focuses on benchmarks where a small number of procedural patterns suffice (Mastermind, ALFWorld). Open-ended or non-recurring tasks may benefit less.
- The PPO Gate uses return-to-go with a running baseline rather than a learned value function; variance/bias trade-offs of this estimator are not characterised.
- Compute cost of running the LLM-based `Aggregate(·)` and candidate generation is reported as a fixed budget but not profiled against the savings from reduced retrieval.

## Open questions

- Can `μ` (Skill selection) itself be learned by an analogous non-parametric mechanism, or must it be RL-trained?
- How does pool size `K` interact with task diversity? Section 4.2 reports K=5 / K=10 / K=20 only on Mastermind.
- Does the semantic-gradient framework transfer to non-decision-making tasks (e.g. coding, theorem proving) where "trajectories" are less natural?
- What is the right hybridisation with episodic memory: when should Skill-Pro fall back to RAG-style retrieval?

## My take

The contribution that should outlive the framework is the *formalism*: casting prompt-level edits as a PPO-style trust-region update gives a principled stopping criterion (admit only when `J(ω') > 0`) that most prompt-optimisation work lacks. The Skill triple is also a clean unit for cross-method comparison — RAG/Expel/AWM all collapse to "no termination condition", which explains why their retrieval ratio is 1.0. The strongest empirical signal is the 50× storage compression with matched-or-better performance on cross-task evaluation; this is consistent with the "compact procedural memory" hypothesis rather than just a benchmark artefact. The weakest part is that the LLM-as-policy importance-ratio computation is glossed: how `π_LLM(a | s, ω)` is actually estimated for a discrete natural-language action `a` matters a lot for whether the gate is well-defined, and the paper is light on this.

## Related

- [[procedural-memory]] (topic)
- [[skill-evolution]] (topic)
- [[agentic-memory]] (topic)
- [[skill-mdp]] (introduced concept — Skill-augmented MDP formalism)
- [[semantic-gradient]] (introduced concept — natural-language gradients)
- [[procedural-skill]] (introduced concept — `<activation, execution, termination>` triple)
- [[non-parametric-ppo]] (introduced method)
- Authors: [[qirui-mi]] (first), [[jun-wang]] (corresp.), [[haifeng-zhang]] (corresp.)
