---
name: Contextual Experience Replay
slug: contextual-experience-replay
type: system
tags:
  - agentic-memory
  - procedural-memory
  - web-agents
  - in-context-learning
  - self-improvement
  - experience-replay
source_papers:
  - contextual-experience-replay-self-improvement-language
parent_methods: []
child_methods: []
code_repo: ""
date_updated: 2026-05-18
---

## Problem setting

A language-model agent A faces a stream of sequential decision-making tasks in a complex environment (e.g. WebArena, VisualWebArena). The agent has no environment-specific pre-training and must improve at inference time, without weight updates, by exploiting its own past trajectories or a pre-collected trajectory set.

## Mechanism

CER instantiates four modules around the base agent:

- **Distillation module D** — a VLM prompted to read one trajectory τ and emit two outputs: (a) **dynamics** D_i — a list of {page summary, URL, inferred usage}; and (b) **skills** S_i — a list of {summary, step-by-step natural-language guideline, concrete action examples}. Distillation prompts are conditioned on the existing buffer to avoid repetition.
- **Dynamic experience buffer ε** — accumulates E_i = (D_i, S_i) for all trajectories processed so far.
- **Retrieval module R** — two parallel VLM prompts (one for dynamics, one for skills) that, given the current task goal, website description, and the entire buffer, select the top-k_d and top-k_s most relevant entries.
- **Replay** — selected experiences are mapped to natural-language descriptions E_NL = f(E_selected) and prepended/inserted into the agent's context C, yielding C' = g(C, E_NL).

The base agent A (e.g. GPT-4o with ReAct output format, or any ReAct/sampling/Tree-Search backbone) then issues actions conditioned on C'.

## Procedure

Operating modes are distinguished by trajectory source:

1. **Online**: start with empty buffer ε. For task i: solve with current ε via retrieval+replay; afterwards run D on trajectory τ_i; merge E_i into ε.
2. **Offline**: pre-collect a trajectory set T (human-annotated or LLM-explored); run D on all τ ∈ T to populate ε; freeze ε; solve all test tasks with retrieval+replay over the fixed buffer.
3. **Hybrid**: run the offline stage to warm up ε, then continue online evolution during test.

Reference hyperparameters: k_d = k_s = 5 retrieved experiences per task; temperature 0.1; GPT-4o-2024-0513 backbone; BrowserGym environment harness; max 30 steps per task.

## Assumptions

- The base model supports long-context in-context learning sufficient to absorb 5+5 retrieved experiences plus the regular agent prompt without exceeding context limits.
- Trajectories are goal-oriented (random exploration degrades distillation quality).
- The environment state has some addressable representation (URL, in the web case) that dynamics distillation can encode.
- The agent output format (ReAct) is compatible with action examples embedded in skill guidelines.

## Limitations

- Strong dependence on trajectory quality: random-exploration data hurts hybrid performance below online-only.
- Web-specific dynamics representation; ports to non-URL environments require redesign.
- Cold-start in pure online mode.
- Distillation/retrieval modules add inference-time cost (≈+17% input tokens for hybrid on WebArena) — modest but non-zero.
- Weaker open-source backbones (e.g. Llama-3.1-70B) yield smaller relative gains due to format-following fragility.
- Buffer scaling and retrieval-quality degradation at very large experience counts is unexplored.

## Tradeoff profile

- **Compute vs. quality.** CER trades a modest token overhead (+5.8% offline, +11.2% online, +17.3% hybrid) for a relative success-rate gain of ~37-52% on WebArena over the ReAct baseline. At similar or lower compute than Tree Search, CER achieves higher success on VisualWebArena.
- **Stability vs. plasticity.** Measured on WebArena Forum cross-template SR: stability 93% (retains nearly all baseline-solvable tasks), plasticity 141% (solves 41% more new task types). Robust to noisy/failed trajectories — only ~2 points gap between full CER and reward-filtered CER_success.
- **Generality vs. specialization.** Orthogonal to the base agent algorithm (ReAct, sampling+reranking, Tree Search) — composes additively. CER + sampling reaches 52.6 SR vs. CER alone 37.7 on Forum.
- **Training-free vs. RL.** No weight updates means CER works on closed-weight models and transfers across backbone swaps, but cannot exploit gradient signal from reward.
