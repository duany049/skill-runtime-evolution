---
name: MemEvolve
slug: memevolve
type: system
tags:
  - agentic-memory
  - meta-evolution
  - self-evolving-agents
  - memory-architecture
  - bilevel-optimization
source_papers:
  - memevolve-meta-evolution-agent-memory-systems
parent_methods: []
child_methods: []
code_repo: https://github.com/bingreeky/MemEvolve
date_updated: 2026-05-20
---

## Problem setting

MemEvolve targets the co-evolution of (i) an LLM-based agent's experience memory and (ii) the *architecture* of the memory system itself. The setting is a population of candidate memory architectures `{Ω_j}`, each instantiated as a concrete `(Encode, Store, Retrieve, Manage)` tuple under the EvolveLab `BaseMemoryProvider` interface, running over batches of agent-environment task trajectories. Per-architecture feedback is a 3-D vector `(success, −token cost, −latency)` aggregated across the batch. The objective is to discover memory architectures that (a) deliver substantial gains on deep-research-style agentic benchmarks (GAIA, WebWalkerQA, xBench-DeepSearch, TaskCraft), and (b) transfer across LLM backbones and agent frameworks without re-evolution.

## Mechanism

A bilevel optimization process — see [[dual-evolution-loop]] — over the [[modular-memory-design-space]].

**Inner loop (Experience Evolution).** For each candidate `Ω_j^{(k)}` in the population at iteration `k`, the agent runs over a batch `T_j^{(k)}` of 60 trajectories starting from `M_0 = ∅`. The candidate's Encode/Store/Manage rules populate memory; Retrieve conditions the policy. The batch contains 40 newly sampled tasks plus 20 tasks reused from the previous iteration for inter-iteration calibration. Per-trajectory feedback is `f_j(τ) = (success, −token cost, −latency)`; the batch aggregate is `F_j^{(k)}`.

**Outer loop (Architectural Evolution).** A meta-operator `F` performs two steps:
1. *Architectural Selection.* Rank candidates by non-dominated (Pareto) sort over `F_j^{(k)}`, breaking ties by the primary performance metric. Keep the top-K survivors (K = 1 in main experiments).
2. *Diagnose-and-Design.* For each survivor `Ω_p^{(k)}`:
   - **Diagnosis** — an LLM inspects the parent's trajectory replays via a structured replay interface, examining outcome statistics, retrieval traces, and storage state to produce a defect profile `D(Ω_p^{(k)})` across all four memory slots (retrieval failures, ineffective abstractions, storage inefficiencies, missing management).
   - **Design** — conditioned on `D`, the LLM proposes `S = 3` descendant architectures by modifying only the four permissible implementation sites in the slot tuple. Descendants vary along encoding strategies, storage rules, retrieval constraints, or management policies; all conform to the `BaseMemoryProvider` interface and remain executable.

The descendants form the next population `{Ω_{j'}^{(k+1)}} = ⋃_p {Ω_{p,s}^{(k+1)}}_{s=1}^S`.

## Procedure

```
Initialize J^(0) = {Ω_0}  # singleton: hand-designed baseline
for k = 0, ..., K_max - 1:
    for Ω_j in J^(k):  # inner loop
        M_t = ∅
        Run agent over T_j^(k) (40 new + 20 reused tasks):
            update M_t via Ω_j
            collect f_j(τ) = (success, -cost, -latency) per τ
        F_j^(k) = aggregate({f_j(τ)})
    P^(k) = TopK(J^(k); Pareto-rank, primary-metric tiebreak)  # outer loop: select
    next = ∅
    for Ω_p in P^(k):
        D_p = Diagnose(Ω_p, trajectory replays of T_p^(k))
        for s = 1, ..., S:
            Ω_{p,s} = Design(Ω_p, D_p, s)
            next.add(Ω_{p,s})
    J^(k+1) = next
return best architecture in J^(K_max) by F
```

**Default configuration.** `K_max = 3`; survivor budget `K = 1`; descendants per parent `S = 3`; batch size 60 (40 new + 20 reused). Meta-evolution operator LLM = GPT-5-mini; backbone agents = SmolAgent (2-agent) or Flash-Searcher (single-agent deep research). Held-out transfer LLMs: Kimi K2, DeepSeek V3.2 (no re-evolution).

**Discovered architectures.** Three named systems emerge along the search trajectory:
- **Lightweight** — evolves from a MemoryBank-style few-shot trajectory baseline into a stage-aware system that delivers memory at multiple granularities (plan-level, tool-use-level, working-memory).
- **Riva** — initialized from AgentKB but evolves more agentic encode/retrieve while staying online and lightweight, dropping the costly offline KB.
- **Cerebra** — extends Riva with tool distillation and periodic working-memory maintenance.

The recurring evolutionary signature is *agentic-ization*: encode and retrieve shift from pre-defined pipelines to LLM-driven decisions across outer rounds, often introducing hierarchical memory and meta-guardrails along the way.

## Assumptions

- The agent framework exposes a `BaseMemoryProvider`-compatible hook so any candidate `(E, U, R, G)` tuple is plug-and-play. (Realized via the EvolveLab codebase — see [[evolvelab]].)
- The meta-evolution LLM is strong enough to produce coherent diagnose-and-design rewrites; behavior with weaker meta-operators is unstudied.
- The task family is roughly homogeneous within an evolution run — explicitly disclaimed for cross-family transfer (e.g. embodied → deep research).
- Feedback signals (success, cost, latency) are available and reasonably noiseless per batch; benchmarks used (GAIA, WebWalkerQA, xBench-DS, TaskCraft) all provide binary or LLM-judged success plus measurable cost/latency.
- The inner-loop batch (60 tasks, 40 new) is large enough to suppress per-task variance in the architectural-selection signal — sensitivity to batch size is not separately reported.

## Limitations

- Shallow search: rank-1 + 3 descendants × 3 outer rounds explores a small slice of the design space; broader population pressure may find better Pareto frontiers.
- Single-LLM dependency: the meta-operator and the inner-loop policy can both be the same backbone (GPT-5-mini in main runs); cross-LLM ablations only at the inner loop, not the meta-operator.
- Cross-family generalization explicitly disclaimed; transfer is shown only within the deep-research / web-browsing family.
- 12-baseline re-implementations are claimed faithful but underperformance of any specific baseline (ExpeL flagged in the paper) may partly reflect re-implementation drift, not separately ablated.
- No reported variance across evolutionary seeds — one trajectory per (framework, benchmark) reported.
- Compute cost of the evolution itself (inner+outer LLM calls × generations × candidates × batch) is not separately tabulated beyond per-task post-evolution metrics.
- Inner-loop `M_0 = ∅` reset means steady-state behavior over long horizons is not directly part of the fitness signal.

## Tradeoff profile

- **Performance vs. cost neutrality.** MemEvolve+Flash-Searcher matches No-Memory cost per task (e.g. GAIA $0.085 vs. $0.086) while delivering +4.24 GAIA, +5.0 xBench-DS, +3.53 WebWalkerQA pass@1. The gains are not bought with extra inference cost at evaluation time — the cost is paid up-front in the evolution process.
- **Cross-task generalization vs. task-specific optimization.** Architectures evolved on TaskCraft transfer to WebWalkerQA, xBench-DS, and held-out CK-Pro / OWL frameworks with 2.0–9.09 pp gains, but the authors explicitly caution this generalization holds within a task family, not across radically different families.
- **Search depth vs. compute.** K=1 / S=3 / K_max=3 minimizes evolution compute; richer K and S would expand the Pareto-frontier search at multiplicative cost.
- **Diagnose-and-design vs. random mutation.** Diagnose-and-design uses structured LLM reasoning to localize defects in the parent before mutating, paying a meta-operator inference cost in exchange for targeted (rather than random) descendants — a worthwhile trade for shallow population pressure where each descendant is expensive to evaluate.
- **Pareto multi-objective vs. scalar fitness.** Three-dimensional fitness `(success, −cost, −latency)` with non-dominated sorting prefers architectures that are not strictly dominated on any axis; scalar weighting would force a single Pareto-optimal point and lose the ability to keep cost/latency-favorable architectures alive.
