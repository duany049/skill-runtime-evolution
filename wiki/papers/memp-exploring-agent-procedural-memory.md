---
title: "Memp: Exploring Agent Procedural Memory"
slug: memp-exploring-agent-procedural-memory
arxiv: "2508.06433"
venue: arXiv
year: 2025
tags:
  - procedural-memory
  - agentic-memory
  - skill-evolution
  - continual-learning
  - llm-agents
importance: 4
date_added: 2026-05-18
source_type: tex
s2_id: e2d3e8abe0fad7210a16be02528370ea03299d64
tldr: "Memp treats agent procedural memory as a first-class optimization object, systematically exploring Build / Retrieve / Update strategies and showing that distilled trajectories transfer from strong to weaker models on ALFWorld and TravelPlanner."
contribution_type:
  - method
  - analysis
datasets:
  - TravelPlanner
  - ALFWorld
code_url: https://github.com/zjunlp/MemP
cited_by: []
---

## Problem & Context

LLM-based agents tackle long-horizon tasks that take dozens of steps under unstable conditions (network glitches, UI changes, schema drift). Restarting from scratch every time is wasteful, yet **procedural memory** in contemporary agents is either hand-crafted (prompt templates), entangled in model parameters, or limited to coarse buffers in frameworks such as LangGraph, AutoGPT, Memory Bank, and Soar. Prior procedural-memory work (Voyager, AWM, AutoManual) shows reuse can help on similar tasks, but no systematic study examines how to *build*, *retrieve*, and *update* such memory over an agent's lifetime, nor how it interacts with model strength. The paper carves out this gap: treat procedural memory as a first-class optimization target with measurable lifecycle dynamics rather than a side-effect of trajectory logging.

## Key idea

Procedural memory is modeled as a learnable module that transforms the standard policy $\pi(a_t|s_t)$ into $\pi_{m^p}(a_t|s_t)$, where $m^p$ is built from past trajectories via a *builder* $B(\tau_t, r_t)$ and refreshed via an *updater* $U(M(t), E(t), \tau_t)$ that supports add / delete / modify. The framework — denoted [[procedural-memory-framework]] — factors three sub-decisions: (a) **Build granularity** (raw trajectory vs. distilled script vs. their union "Proceduralization"), (b) **Retrieval key** (random, query-vector, AveFact keyword averaging), and (c) **Update policy** (vanilla append, validation-filtered, reflexion-style adjustment). The same framework instance ports across GPT-4o, Claude-3.5-sonnet, and Qwen2.5-72B without re-engineering.

## Method

The paper instantiates the framework as a closed loop on top of a base LLM agent acting in an environment formulated as an MDP $\tau = (s_0, a_0, o_1, s_1, \ldots, s_T)$:

- **Build.** From a training-set trajectory $\tau$ and its environment-supplied reward $r$, the builder emits a procedural-memory entry $m^p = B(\tau, r)$ in one of three granularities — `Trajectory` (verbatim round-by-round), `Script` (LLM-distilled abstract procedure), or `Proceduralization` (both, concatenated as memory context).
- **Retrieve.** A new task $t_{\text{new}}$ is encoded by a vector model $\phi$ and matched against stored memories by cosine similarity. Three key strategies are compared: `Random Sample` (no key), `Key=Query` (raw task description as key), `Key=AveFact` (LLM extracts keywords, averages their embeddings). Top-k retrieved memories are concatenated into the context.
- **Update.** Memory refreshes every $t$ test tasks. Strategies: `Vanilla Memory Update` (append all consolidated trajectories), `Validation` (consolidate only successful trajectories), `Adjustment` (reflexion-style — on retrieval failure, merge erroneous trajectory with original and rewrite in place).

The procedural memory $Mem = \sum_t m^{p_t}$ is a vector-keyed library that grows, prunes, and rewrites as the agent acts.

## Experiment & Results

**Backbones:** GPT-4o, Claude-3.5-sonnet, Qwen2.5-72B-Instruct. **Benchmarks:** [[TravelPlanner]] (long-horizon constrained planning; CS = Commonsense, HC = Hard Constraint scores) and [[ALFWorld]] (long-horizon embodied housework, Dev/Test success rates). Steps reported per episode.

**Build granularity (Table 1).** All three memory variants beat No-Memory on both benchmarks across all three backbones. `Proceduralization` (trajectory + script) is best almost everywhere — e.g. GPT-4o on ALFWorld: Dev 87.14 / Test 77.86 / Steps 15.01 vs. No-Memory 39.28 / 42.14 / 23.76. On TravelPlanner with GPT-4o, CS rises from 71.93 (No-Memory) to 79.94 (Proceduralization); steps fall from 17.84 to 14.62.

**Retrieval key (Table 2).** On TravelPlanner, `Key=AveFact` dominates: GPT-4o CS 76.02 vs. Random Sample 74.59 and No-Memory 71.93; Qwen2.5-72B CS 63.41 vs. 56.57 baseline. Hard-constraint scores show a different story — No-Memory often has the highest HC (e.g. Claude HC 33.06 vs. AveFact 29.61), suggesting retrieval can over-anchor and miss strict constraints.

**Update policy.** Across iterative trajectory-group updates, reflexion-style `Adjustment` outperforms `Vanilla` and `Validation`. On the final group it beats the second-best strategy by +0.7 points and cuts ~14 steps, indicating that error-correcting updates compound over time while plain append plateaus.

**Cross-model transfer.** Procedural memory built by GPT-4o and used by Qwen2.5-14B raises Qwen's TravelPlanner success by 5% and cuts ~1.6 average steps; analogous gains on ALFWorld. This is the paper's most provocative claim: procedural knowledge is portable across models.

**Comparison vs. baselines on ALFWorld (GPT-4o):** ReAct < Expel < AWM < Memp, with Memp attaining the highest Dev/Test success at the lowest step count.

## Limitations

The authors flag two: (1) retrieval is confined to vector-similarity over manually crafted keys — classical IR methods like BM25 are untested; (2) the framework consumes explicit benchmark-supplied reward signals, so it cannot judge success in real-world settings where rewards are sparse or absent. From the data, two further limitations are visible but not discussed: (3) on TravelPlanner, Hard-Constraint scores frequently *drop* under memory-augmented setups (the No-Memory baseline often leads on HC), suggesting retrieved memories can crowd out strict-constraint reasoning; (4) the scaling analysis shows performance *degrades* past a certain number of retrieved memories due to context dilution — no principled stopping rule is proposed.

## Open questions

- How should retrieval interact with constraint satisfaction so that retrieved memories don't displace hard-constraint enforcement (the HC regression on TravelPlanner)?
- What governs the optimal number of retrieved memories per task — can it be predicted from task complexity rather than tuned empirically?
- Can the validation / adjustment update operators themselves be *learned* rather than hand-specified?
- For cross-model transfer, what properties of the procedural memory bank (format, abstraction level, key granularity) determine portability across heterogeneous backbones?
- How does Memp behave in environments without ground-truth reward — can an LLM-as-judge substitute reliably, as the authors speculate?

## My take

Memp is the cleanest existing ablation of the procedural-memory lifecycle: the Build × Retrieve × Update grid is exactly the design space prior work (Voyager, AWM, AutoManual) implicitly chose from but rarely justified. The cross-model transfer result is the most useful piece for our agenda — it implies procedural memory is not just an artifact of a specific backbone's quirks but a quasi-portable artifact. The negative finding on Hard Constraints is under-emphasized in the paper but is exactly the kind of failure mode a serious continual-learning protocol needs to surface: memory-augmented agents can be *better on average* and *worse on the strict-correctness axis* simultaneously. Future work that takes this seriously should not report aggregate accuracy alone.

## Related

- [[procedural-memory]] — Memp is positioned squarely in this topic.
- [[skill-evolution]] — Memp's Update phase (reflexion-based Adjustment, deprecation) is exactly the evolution mechanism the topic catalogs.
- [[continual-learning-evaluation]] — Memp's trajectory-group protocol is one concrete way to measure skill-utility-over-time.
- [[agentic-memory]] — Memp is a specific instantiation of the broader memory-architecture program.
