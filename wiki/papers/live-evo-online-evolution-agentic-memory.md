---
title: "Live-Evo: Online Evolution of Agentic Memory from Continuous Feedback"
slug: live-evo-online-evolution-agentic-memory
arxiv: "2602.02369"
venue: arXiv
year: 2026
tags:
  - agentic-memory
  - self-evolving-agents
  - online-learning
  - experience-replay
  - live-benchmark
  - future-prediction
importance: 4
date_added: 2026-05-18
source_type: tex
tldr: "Live-Evo is an online self-evolving agentic memory system that separates an Experience Bank from a Meta-Guideline Bank, uses contrastive (memory-on vs memory-off) evaluation to reweight experiences from continuous feedback, and verifies new experiences before committing them, yielding 20.8% Brier and 12.9% market-return gains on the Prophet Arena live benchmark."
contribution_type:
  - method
  - system
  - analysis
datasets:
  - Prophet Arena
  - Xbench-DeepResearch
cited_by: []
---

## Problem & Context

LLM agents are increasingly equipped with memory — stored experience and reusable guidelines that improve task-solving performance. Recent self-evolving systems update memory from interaction outcomes, but virtually all existing pipelines are developed for static train/test splits: they approximate "online" learning by folding a static benchmark into sequential pieces. This setup hides two issues that surface in real deployments. First, in true online streams the task distribution drifts over time, so past experience can grow stale or become outright misleading. Second, the dominant evaluation signal in standard reasoning/search benchmarks is sparse and binary (correct/incorrect), which leaves the memory-update loop with too coarse a signal to know which experiences actually help.

Live benchmarks like Prophet Arena and FutureX reframe agent evaluation as a longitudinal future-prediction problem with weekly updates, calibration-based metrics (Brier score), and decision-oriented outcomes (market return). These are dense, continuous signals on a real, drifting task stream — exactly the regime where existing self-evolving memory systems are not designed to operate. Before Live-Evo, only a handful of works (e.g. EvoMemoryBench, AgentWorkflowMemory) studied memory under task streams, and they still emulated streams by folding static datasets. The opening for the paper is: a self-evolving memory system that is grounded in continuous environmental feedback, can decide *when* and *how* to use past experience, and can actively forget misleading entries as the world changes.

## Key idea

Treat agentic memory as an online optimization problem with two decoupled banks rather than a single store. An **Experience Bank** $\mathcal{E}$ holds *what happened* (structured past task interactions with adjustable retrieval weights). A **Meta-Guideline Bank** $\mathcal{M}$ holds *how to use experience* (meta-heuristics that compile retrieved experiences into a task-adaptive guideline). Both banks are updated online from continuous feedback through three coupled mechanisms:

1. **Contrastive evaluation at action time.** For each task, the agent runs twice — once memory-on with the compiled guideline, once memory-off — and the performance gap is the supervision signal for memory updates. Continuous metrics (Brier, return) make this gap informative; binary rewards would not.
2. **Reinforcement-and-decay weight updates.** Experiences whose retrieval helps get their weights increased; experiences whose retrieval hurts get decayed and gradually drift out of retrieval. The dynamic is explicitly analogous to human memory reinforcement/forgetting.
3. **Verify-before-update for write-back.** After processing a batch, only the worst-performing tasks under memory-on are summarized into candidate experiences, and a candidate is committed to $\mathcal{E}$ only if a re-evaluation shows a statistically significant improvement over the original memory-on score. Failure cases additionally trigger a new entry in $\mathcal{M}$ (a new meta-guideline) instead of polluting $\mathcal{E}$ directly.

The result is a memory system that learns not just what to store, but how to use what is stored — and that grounds those updates in environmental feedback rather than LLM self-verification.

## Method

Live-Evo formalizes self-evolving memory as a closed-loop four-stage operator per task: `{Retrieve, Compile, Act, Update}`.

- **Retrieve.** Given task $q$, the agent generates queries (active retrieval, not just similarity matching against $q$ itself) and returns the top-$k$ experiences plus a selected meta-guideline $\hat m$. Ranking uses $\mathit{Score} = \mathit{Weight} \cdot \mathit{Sim}(\mathit{exp}, \mathit{query})$, mixing learned weight with semantic similarity so that decayed experiences disappear from retrieval even when they remain semantically close.
- **Compile.** `CompileGuideline(q, E_q, \hat m)` extracts cross-experience regularities from $E_q$, grounds them in the current task, and produces a task-specific guideline $g_q$. The meta-guideline $\hat m$ steers this compilation, making the inductive bias explicit and learnable rather than fixed.
- **Act.** The agent executes the underlying search policy twice: once conditioned on $g_q$ (memory-on, producing $r^{\text{on}}_q, \tau_q$) and once without (memory-off, producing $r^{\text{off}}_q$). This `ContrastiveEval` quantifies the causal contribution of the guideline.
- **Update.** Per-task: experience weights $W_{E_q}$ are updated using the gap $r^{\text{on}}_q - r^{\text{off}}_q$; if the gap is non-positive, a new meta-guideline entry is added to $\mathcal{M}$ via `Reflect(q, g_q, E_q)` (failure triggers learning at the *how-to-use* layer, not the *what* layer). Per-batch: the worst $\rho$ fraction of memory-on tasks (default $\rho=0.3$) is selected; each is `Summarize`d into a candidate experience $e^{\text{new}}_q$ from its stored trajectory $\tau_q$; the candidate is committed to $\mathcal{E}$ only if `Eval(q, e^{\text{new}}_q) > r^{\text{on}}_q`. This is the *Verify Before Update* gate.

The underlying agent is intentionally simple — two tools (Google Search + Web Fetch) with strict time-based retrieval to prevent information leakage past the prediction close time. This isolates the contribution of the memory system from any tool-use sophistication.

## Experiment & Results

**Setups.** Prophet Arena (Brier score lower is better; market return higher is better) over the latest 10 weeks, 500 tasks total, each with a bid-price snapshot 6 hours before close. Xbench-DeepResearch (accuracy) for generalization to traditional deep research, with the static benchmark split into 10 sequential folds for online learning. Default backbone GPT-4.1-mini, temperature 0.2, `bad_case_percentile` 0.3, `min_brier_improvement` 0.05, `experience_similarity_threshold` 0.5.

**Main results — Prophet Arena (Brier score, GPT-4.1-mini).** Live-Evo averages **0.14** Brier across W1–W10, vs base GPT-4.1-mini at 0.22, MiroFlow at 0.32, Qwen Deep Research at 0.20, Live-Evo without experience at 0.19, and ReMem (self-evolving baseline) at 0.16. Live-Evo wins or ties the best score in 7 of 10 weeks. Headline: **20.8% Brier improvement and 12.9% market-return improvement** over the same backbone without memory; under a \$100/week investment strategy this compounds to **\$150 additional cumulative return over 10 weeks**, with the gap widening over time as memory accumulates.

**Cross-model generalization.** With GPT-4.1, GPT-5-mini, and Qwen3-8B as backbones, Live-Evo improves Brier and market return over each model's base configuration (e.g. GPT-4.1: 0.18→0.17 Brier, 1.13→1.18 return; GPT-5-mini: 0.16→0.15, 1.34→1.36; Qwen3-8B: 0.19→0.18, 1.20→1.21). Largest relative gains on the weakest backbone (GPT-4.1-mini), consistent with the headroom argument: weak models produce more failure cases per week, which feed richer reflection signals.

**Xbench-DeepResearch (accuracy, GPT-4.1-mini).** Live-Evo 0.46 beats MiroFlow 0.45, Qwen-DeepResearch 0.43, and ReMem 0.40, despite Live-Evo not being designed for deep-research tasks.

**Ablations vs. full Live-Evo (Brier ↓ / return ↑, GPT-4.1-mini).**
- w/o weight-update: Brier 0.14→0.17 (+17.0%), return 1.46→1.34 (−8.0%).
- w/o meta-guideline: Brier 0.14→0.16 (+10.9%), return 1.46→1.41 (−3.4%).
- w/o guideline-compile: Brier 0.14→0.16 (+11.6%), return 1.46→1.16 (**−20.4%**, the largest single-component drop on return).
- w/o active-retrieve: Brier 0.14→0.17 (+15.0%), return 1.46→1.22 (−16.8%).

Each component is necessary; guideline compilation matters most for decision quality (market return), weight updates matter most for calibration (Brier). The case study traces a concrete high-weight (reusable, task-aligned) vs. low-weight (hallucinated) experience, showing the weight dynamics actually filter the bank as intended.

## Limitations

- Reliance on **dense environmental feedback** (Brier, return) — applicability is constrained in settings with sparse or subjective signals where the contrastive evaluation gap is noisier than the per-task variance.
- **Verify-before-update is conservative.** The protocol admits a new experience only when a re-evaluation shows a statistically significant gain, which delays adoption of subtle or emerging heuristics. There is an explicit speed-vs-precision tradeoff in the write-back gate.
- Doubling inference cost per task from the contrastive (memory-on + memory-off) execution is not discussed; the system requires roughly 2x token cost relative to a non-contrastive baseline.
- All evaluation runs over 10 weeks. Behavior over substantially longer horizons (memory growth, drift, possible saturation of the meta-guideline bank) is not characterized.

## Open questions

- How does the system behave when feedback is delayed, biased, or partially observed (e.g. some week's labels are unavailable)? The current contrastive evaluation assumes prompt outcome resolution.
- Can the verify-before-update threshold be made adaptive (e.g. tighter when the bank is small, looser when it is mature)?
- Is the reinforcement-and-decay dynamic stable when the task distribution shifts abruptly rather than gradually (regime change, not drift)?
- The meta-guideline bank itself has no explicit forgetting mechanism described — what stops $\mathcal{M}$ from accumulating stale meta-heuristics over very long horizons?

## My take

The cleanest contribution is structural: separating "what happened" ($\mathcal{E}$) from "how to use it" ($\mathcal{M}$) and letting both be updated by *different* mechanisms (weight reinforcement for $\mathcal{E}$, reflection insertion for $\mathcal{M}$). That decomposition makes the inductive bias of memory usage *learnable* rather than hard-coded. The contrastive evaluation trick is the second key move — it provides a real, causal supervision signal at the per-task level, which is what enables grounded reinforcement rather than LLM-only self-verification. The fact that the largest ablation drop comes from removing guideline-compile (not weight-update) suggests the system's improvements are dominated by the *how-to-use* axis, which is the underexplored direction in the literature. The methodology of pairing live benchmarks with contrastive memory evaluation is potentially exportable to other online agentic settings (live coding benchmarks, streaming RAG, online tool selection).

The result deserves replication on benchmarks with sparser feedback (binary deep-research accuracy splits) and longer horizons (≥6 months). The doubled inference cost from contrastive evaluation is a real deployment friction worth flagging.

## Related

- Builds on the [[agentic-memory]] research area and the live benchmarking direction.
- Introduces [[online-self-evolving-agentic-memory]] as a distinct regime within self-evolving agents.
- Introduces [[meta-guideline-bank]] as a higher-level "how-to-use experience" memory layer separate from raw experience storage.
- Introduces [[verify-before-update]] as a write-back protocol gating new experiences by re-evaluation gain.
- Implements the [[live-evo]] system that operationalizes the above.
- Authors: [[yaolun-zhang]], [[yiran-wu]].
