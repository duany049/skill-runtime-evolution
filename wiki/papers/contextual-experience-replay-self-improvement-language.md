---
title: Contextual Experience Replay for Self-Improvement of Language Agents
slug: contextual-experience-replay-self-improvement-language
arxiv: "2506.06698"
venue: ACL
year: 2025
tags:
  - agentic-memory
  - procedural-memory
  - skill-evolution
  - self-improvement
  - experience-replay
  - web-agents
  - in-context-learning
importance: 4
date_added: 2026-05-18
source_type: tex
s2_id: "204246f01e8870d7244450e5042a32c560263089"
tldr: A training-free framework that distills environment dynamics and decision-making skills from past web-agent trajectories into a dynamic buffer, then retrieves and replays them in context to improve GPT-4o by 51% on WebArena and reach 31.9% on VisualWebArena.
contribution_type:
  - method
  - analysis
datasets:
  - WebArena
  - VisualWebArena
code_url: ""
cited_by: []
---

## Problem & Context

LLM agents tackling sequential decision-making in complex environments — especially realistic web navigation benchmarks like WebArena and VisualWebArena — fail because they lack environment-specific knowledge. Before this work, frontier-model agents reached only ~20% success while humans achieved 78–89%, and existing systems explored each environment from scratch for every task. Training in each environment was costly, and the dominant strategies (ReAct, Reflexion, Tree-Search) offered no efficient mechanism for *continual* learning at inference time. Prior memory-of-trajectories systems (Voyager, ExpeL, Synapse, AutoGuide, AutoManual, Agent Workflow Memory) either targeted simple environments, used raw long observation-action pairs as exemplars, or required ground-truth reward to function, which limited real-world applicability.

## Key idea

Adapt the reinforcement-learning experience-replay idea to LLM agents at the *context* level: instead of training on stored trajectories, distill them into compact natural-language **experiences** (environment dynamics + decision-making skills), maintain a **dynamic buffer**, retrieve the top-k relevant items for each new task, and replay them in the agent's context window. The framework is training-free, works for offline/online/hybrid trajectory sources, and handles both successful and failed trajectories without ground-truth reward.

## Method

CER comprises four modules built on a VLM (GPT-4o in practice): a base decision-making agent A, a distillation module D, a retrieval module R, and a dynamic experience buffer ε.

- **Distillation.** Two separate VLM prompts process each trajectory τ_i to extract E_i = (D_i, S_i). The dynamics module emits a list of page summaries + URLs + inferred usages. The skill module emits abstract skill summaries (e.g. "navigate to forum {forum name}") with step-by-step natural-language guidelines and concrete action examples in ReAct format. Both prompts are conditioned on the existing buffer to avoid repetitive distillation, supporting continual accumulation.
- **Retrieval.** Two parallel VLM-based retrieval modules read the task goal, website description, and full buffer, returning top-k_d dynamics and top-k_s skills (k_d = k_s = 5).
- **Replay.** Selected experiences are programmatically mapped to natural-language descriptions E_NL = f(E) and concatenated into the agent context C' = g(C, E_NL). The base policy then issues actions conditioned on the augmented context.
- **Trajectory sources.** Online (self-generated from past tasks), offline (pre-collected human or LLM-explored), and hybrid (offline warm-up + online evolution).

## Experiment & Results

- **WebArena (812 tasks across 5 sites).** GPT-4o + BrowserGym baseline scores 24.3 average success. CER_offline = 33.4, CER_online = 33.2, CER_hybrid = **36.7** — a relative improvement of 51.0% over baseline. Token overhead for hybrid: +20,631 input tokens / +17.3%.
- **VisualWebArena (910 multimodal tasks across 3 sites).** CER_online reaches **31.9%** average vs. BrowserGym baseline 26.2 and Tree Search 26.4 — outperforming Tree Search at ≥3× lower token cost (CER caps at 30 steps; Tree Search uses 20× sampling × 5 steps + value-function calls).
- **Cross-template generalization (Forum split).** Cross-template success rate jumps from 44.7 (baseline) to 60.0 (CER), with stability = 93% and plasticity = 141% — i.e., 41% additional new task-types solved while retaining most baseline capabilities.
- **Open-source backbone.** With Llama-3.1-70B on Gitlab split, CER_hybrid 22.0 vs. ReAct 17.3 (+26.5% relative); weaker formatting limits skill distillation quality.
- **Synergy with sampling+reranking.** On WebArena Forum, CER + sampling reaches 52.6 vs. CER alone 37.7 (+39.5% relative).
- **Reward access ablation.** CER 31.4 → CER_success 33.5 — modest gap shows robustness to failed/noisy trajectories.
- **Module ablation.** Removing skills: 37.7 → 33.3; removing dynamics: 37.7 → 35.1. Both modules contribute.

## Limitations

- Sensitivity to trajectory quality: random-exploration trajectories degrade hybrid performance below online-only; goal-oriented trajectories are required for high-quality distillation.
- Heavy reliance on environment dynamics + URL navigation: not obvious how to port the dynamics module to non-web environments (e.g. ALFWorld, real-world navigation).
- Cold-start in pure online mode — no experiences available for the very first task.
- Offline mode depends on human-annotated trajectories; automation of high-quality trajectory generation remains open.
- Weaker-model degradation: Llama-3.1-70B sees smaller relative gains due to output-formatting fragility in long, complex action spaces.

## Open questions

- Can low-quality / random-exploration trajectories be salvaged via finer-grained filtering or trajectory segmentation?
- How to design analogous dynamics representations for non-URL environments where state is harder to address atomically?
- How does the buffer scale across many environments — does cross-domain experience transfer or interfere?
- Is the stability–plasticity tradeoff stable as the buffer grows over thousands of tasks?
- Can the distillation/retrieval modules themselves be trained or self-improved, rather than fixed prompts?

## My take

The contribution is conceptually simple but methodologically important: it cleanly separates *what to remember* (dynamics vs. skills) from *when to retrieve* and *how to replay*, and shows training-free in-context experience accumulation can rival or beat search-heavy baselines at far lower token cost. The stability-plasticity framing (Grossberg 1982; Rolnick 2019) gives an interpretable measurement protocol — 93% stability + 141% plasticity is a sharper claim than "average SR went up". The big questions left open are environment-portability (the dynamics module leans heavily on web URL structure) and how the system behaves as the buffer grows long enough that retrieval-quality becomes the bottleneck.

## Related

- [[contextual-experience-replay]]
- [[stability-plasticity-tradeoff]]
- [[yitao-liu]]
- [[shunyu-yao]]
