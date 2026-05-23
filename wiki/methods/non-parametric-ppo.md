---
name: Non-Parametric PPO
slug: non-parametric-ppo
type: optimization
tags:
  - non-parametric-optimization
  - prompt-optimization
  - skill-evolution
  - trust-region
  - ppo
source_papers:
  - skill-pro-learning-reusable-skills-experience
parent_methods: []
child_methods: []
code_repo: "https://github.com/Miracle1207/Skill-Pro"
date_updated: 2026-05-18
---

## Problem setting

Optimise a natural-language policy (here, a pool of Procedural Skills `Ω = {ω⁽¹⁾, …, ω⁽ᴷ⁾}` conditioning a frozen LLM action policy) for expected cumulative return, without updating any model parameters. The pool has fixed capacity `K`. Optimisation acts on `Ω` itself, treating each `ω = ⟨I_ω, π_ω, β_ω⟩` as an editable natural-language artefact.

## Mechanism

Two coupled mechanisms replace PPO's parameter update.

1. **Semantic gradient step.** For each invoked Skill `ω` and each trajectory `τ_i` in the batch where it was active, hindsight attribution produces a structured natural-language gradient `g_i = (g_i^{(I)}, g_i^{(π)}, g_i^{(β)})`. An LLM-based `Aggregate(·)` over `B` trajectories yields `ḡ_ω`, a stable batch-level update direction. An LLM rewrite then produces a candidate `ω' = ω ⊕ ḡ_ω`. This is the gradient-ascent analogue.

2. **PPO Gate (trust-region acceptance).** For a batch of historical trajectories under behaviour Skill `ω`, compute per-step importance ratios `ρ_t(ω') = π_LLM(a_t | s_t, ω') / π_LLM(a_t | s_t, ω)` and return-to-go advantages `Â_t = G_t − R̄` (running baseline). The clipped surrogate

   ```
   L_CLIP(ω') = Ê_τ [ (1/|τ|) Σ_t min( ρ_t Â_t , clip(ρ_t, 1−ε, 1+ε) Â_t ) ]
   ```

   is computed as `J(ω')`. Out of `N_c` candidates, the best is selected and admitted iff `J(ω_new) > 0`.

3. **Score-based maintenance.** Each retained Skill carries an online score `Score(ω) = G_b(ω) / max(1, N_b(ω))` where `G(ω; τ)` averages advantage `r_t − r̄` over time-steps when `ω` is active. When the pool exceeds `K`, prune Skills with non-positive score, semantic duplicates (cosine-similar embeddings), and ascending-score tails.

The trust-region clip operates *across* the natural-language edit rather than across a parameter delta, so "small change" is enforced behaviourally (small `ρ_t` deviation on past actions) rather than syntactically.

## Procedure

```
Input : initial pool Ω_0, frozen LLM π_LLM, capacity K, ε, N_c
For n = 1..N:
    # 1. Experience collection
    Collect batch T_B under π_Ω = μ · π_LLM.

    # 2. Semantic-gradient candidate proposal
    For each ω invoked in T_B:
        For each τ_i in T_B containing ω:
            g_i ← hindsight_attribution(τ_i, ω)        # structured (I, π, β) edit
        ḡ_ω ← Aggregate({g_i})                         # LLM consolidation
        Generate N_c candidates {ω'_j = ω ⊕ ḡ_ω}      # LLM rewrite

        # 3. PPO Gate verification
        For each ω'_j:
            Compute J(ω'_j) = L_CLIP(ω'_j)
        ω* ← argmax_j J(ω'_j)
        If J(ω*) > 0:
            Ω ← Ω ∪ {ω*}

    # 4. Score-based maintenance
    Update Score(ω) for all ω ∈ Ω.
    If |Ω| > K:
        Prune Score(ω) ≤ 0, cosine-duplicates, ascending-score tail until |Ω| = K.

Output : Ω_N
```

## Assumptions

- The LLM is treated as a stochastic policy whose action likelihoods `π_LLM(a | s, ω)` are at least *ratio-meaningful*, so that `ρ_t(ω')` is a useful quantity. In practice the paper assumes a tokeniser-level likelihood readout is available.
- Hindsight attribution by the LLM yields informative, low-noise signals on average; sample size `B` is large enough for aggregation to filter trajectory-specific noise.
- Return-to-go with a running baseline is a sufficient advantage estimator (no learned value function).
- Skill activation/execution/termination decompose the credit-assignment problem cleanly enough that *component-level* gradients are meaningful.

## Limitations

- *Importance ratio estimation* over natural-language actions is non-trivial. The paper does not extensively characterise variance behaviour of `ρ_t` for long, free-form action strings; large variance could undermine the gate.
- The advantage estimator is high-variance compared to learned-value PPO; the paper compensates with `ε`-clipping but provides no theoretical guarantee.
- Acceptance criterion `J(ω_new) > 0` is conservative: it rejects all candidates whenever no clear winner exists, so progress can stall on tasks with weak signal.
- Compute cost is dominated by `N_c` LLM rewrites and likelihood evaluations per evolution round; runtime is reported empirically but not contrasted with parametric fine-tuning.
- Method is currently demonstrated on small Skill pools (`K ∈ {5, 10, 20}`). Behaviour at much larger `K` (where Skill collisions become common) is open.

## Tradeoff profile

- Vs *parametric RLHF / PPO*: no weight updates → no catastrophic forgetting, no compute spent on backprop, but bounded by what natural-language edits can express. Cheap to deploy, hard to push past LLM capability ceiling.
- Vs *flat prompt optimisation* (TextGrad, GEPA-style): adds a *trust-region* via the PPO Gate; ablating it (`w/o PPO Gate` in Skill-Pro Table 3) destabilises training. The price is more LLM calls per round.
- Vs *episodic memory growth* (RAG, Reflexion): pool is *bounded* by `K` and *quality-controlled* by score-based pruning, so storage and inference cost stay flat as experience grows. The price is information loss when a Skill is pruned.
- Vs *Best-of-N prompting*: similar candidate-and-select shape, but candidates here come from *targeted* semantic gradients rather than random sampling, and the selector is a *behavioural* surrogate over a held-out historical batch rather than an LLM-as-judge.
- Symmetric to *PPO proper*: the same proximal-step intuition (small policy moves are stable) is preserved; the difference is that "policy" lives in natural language and "step size" is measured by importance-weighted historical-action probability rather than KL on parameters.
