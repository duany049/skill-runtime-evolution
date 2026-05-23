---
name: Live-Evo
slug: live-evo
type: system
tags:
  - agentic-memory
  - self-evolving-agents
  - online-learning
  - experience-replay
  - future-prediction
source_papers:
  - live-evo-online-evolution-agentic-memory
parent_methods: []
child_methods: []
code_repo: https://ag2ai.github.io/live-evo-page/
date_updated: 2026-05-18
---

## Problem setting

Live-Evo addresses self-evolving agentic memory under continuous, streaming feedback. The setting is a sequential task stream $\mathcal{Q}$ (e.g. weekly future-prediction batches) with continuous outcome signals (Brier score, market return). The agent must update its memory online so that performance improves over time while past experiences can be reweighted or forgotten as the task distribution drifts. The underlying agent is intentionally simple — only Google Search and Web Fetch tools with strict time-windowed retrieval — which isolates the contribution of the memory architecture.

## Mechanism

Two coupled memory banks plus four closed-loop operators per task.

**Memory banks.**
- Experience Bank $\mathcal{E}$: structured past task interactions with per-entry retrieval weights $\{w_e\}$.
- Meta-Guideline Bank $\mathcal{M}$: composition meta-heuristics specifying how to turn retrieved experiences into a task-adaptive guideline.

**Per-task operators** for each $q \in \mathcal{Q}$:

1. `Retrieve(q, E, M)` — agent generates queries (active retrieval, not direct similarity against $q$), returns top-$k$ experiences ranked by $\mathit{Score} = w_e \cdot \mathit{Sim}(\mathit{exp}, \mathit{query})$ plus a selected meta-guideline $\hat m$.
2. `CompileGuideline(q, E_q, \hat m)` — LLM produces a task-specific guideline $g_q$ by extracting cross-experience regularities under $\hat m$.
3. `ContrastiveEval(q, g_q)` — execute the agent twice: with $g_q$ (memory-on, score $r^{\text{on}}_q$, trajectory $\tau_q$) and without (memory-off, score $r^{\text{off}}_q$).
4. `UpdateWeights(W_{E_q}, r^{\text{on}}_q - r^{\text{off}}_q)` — reinforce experiences with positive gap, decay those with non-positive gap. On non-positive gap, also append `Reflect(q, g_q, E_q)` as a new entry to $\mathcal{M}$.

**Per-batch operator** (after processing batch $\mathcal{Q}$):

5. `SelectWorst(Q, {r^on}, ρ)` picks the worst $\rho$ fraction of memory-on tasks (default $\rho = 0.3$). For each selected $q$: produce $e^{\text{new}}_q$ via `Summarize(q, \tau_q)`, then commit to $\mathcal{E}$ only if `Eval(q, e^{\text{new}}_q) > r^{\text{on}}_q$` — the Verify-Before-Update gate.

## Procedure

```
for q in Q:
    E_q, m_hat = Retrieve(q, E, M)
    g_q       = CompileGuideline(q, E_q, m_hat)
    r_on, r_off, tau_q = ContrastiveEval(q, g_q)
    W_{E_q}   = UpdateWeights(W_{E_q}, r_on - r_off)
    if r_on - r_off <= 0:
        M = M ∪ {Reflect(q, g_q, E_q)}

Q_bad = SelectWorst(Q, {r_on}, ρ)
for q in Q_bad:
    e_new = Summarize(q, tau_q)
    if Eval(q, e_new) > r_on_q:
        E = E ∪ {e_new}

return E, M
```

Default hyperparameters: backbone GPT-4.1-mini, temperature 0.2, `bad_case_percentile` $\rho = 0.3$, `min_brier_improvement` 0.05, `experience_similarity_threshold` 0.5, top-$k$ retrieval.

## Assumptions

- The evaluation signal is **dense and continuous** (Brier, return); sparse binary signals would not provide enough resolution for the contrastive gap.
- Outcomes resolve **promptly** so per-task feedback is available within the loop iteration; the system is not designed for long-delayed labels.
- Tasks within a batch are roughly **comparable** (e.g. weekly tasks of the same type) so that retrieved experiences from prior batches transfer.
- The backbone model is capable of producing **meaningful failure reflections** for $\mathcal{M}$ insertion; weak backbones degrade the meta-guideline quality (consistent with Qwen3-8B showing smaller gains).

## Limitations

- Reliance on dense feedback bounds applicability to settings where a continuous outcome signal exists.
- Verify-Before-Update is conservative; subtle heuristics with modest per-task gains may never cross the admission threshold.
- Contrastive evaluation doubles per-task inference cost (memory-on + memory-off execution).
- Memory-growth dynamics over horizons longer than 10 weeks are not characterized; $\mathcal{M}$ has no explicit pruning mechanism.
- Single-trial verification confounds the candidate's true utility with per-task stochasticity.

## Tradeoff profile

- **Calibration vs. decision quality.** Removing weight-update hurts Brier most (+17.0%); removing guideline-compile hurts market return most (−20.4%). Calibration and decision quality are driven by partially different mechanisms in the system.
- **Bank purity vs. learning speed.** Verify-Before-Update keeps $\mathcal{E}$ clean at the cost of slower adoption of new heuristics. Lowering `min_brier_improvement` trades purity for speed.
- **Compute vs. supervision quality.** Contrastive evaluation costs an extra agent run per task but provides the causal signal that drives weight updates. Single-run alternatives lose the supervision.
- **Backbone strength vs. relative gain.** Live-Evo's relative improvement is largest on weaker backbones (GPT-4.1-mini > GPT-4.1 > GPT-5-mini), because weaker models produce more failure cases and therefore richer reflection signal. Stronger backbones already produce well-calibrated outputs, leaving less headroom.
