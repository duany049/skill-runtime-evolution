---
title: "Agentic Memory: Learning Unified Long-Term and Short-Term Memory Management for Large Language Model Agents"
slug: agentic-memory-learning-unified-long-term
arxiv: "2601.01885"
venue: arXiv preprint
year: 2026
tags:
  - agentic-memory
  - long-term-memory
  - short-term-memory
  - memory-management
  - reinforcement-learning
  - grpo
  - tool-use
  - llm-agents
importance: 4
date_added: 2026-05-18
source_type: tex
tldr: "AgeMem unifies long-term and short-term memory management for LLM agents by exposing memory operations as tools and training a single policy with a three-stage progressive RL curriculum plus step-wise GRPO that broadcasts terminal rewards back to memory decisions."
contribution_type:
  - method
  - system
datasets:
  - HotpotQA
  - ALFWorld
  - SciWorld
  - PDDL
  - BabyAI
cited_by: []
---

## Problem & Context

LLM-based agents in long-horizon tasks are bottlenecked by the finite context window, so their effective "memory" must be managed explicitly. The field has split memory into two largely independent pieces: long-term memory (LTM) — a persistent external store of task/user knowledge — and short-term memory (STM) — the content of the active context. Existing LTM systems (LangMem, A-Mem, Mem0, Mem0-graph, Zep) rely on predefined memory schemas or heuristic update rules; existing STM solutions (RAG variants, ReSum-style periodic compression) operate on fixed schedules. Both lines treat memory as an external module bolted onto the agent, optimized separately and combined ad hoc. AgeMem's authors identify three concrete blockers to a unified treatment: (C1) functional heterogeneity — LTM controls what to *store/update/discard* while STM controls what to *retrieve/summarize/filter*; (C2) training mismatch — LTM training and STM training use different supervision signals, and standard RL assumes continuous trajectories while memory operations produce fragmented, sparsely-rewarded experiences; (C3) deployment cost — many systems require an auxiliary expert LLM as a memory manager, multiplying inference and training cost.

## Key idea

Treat memory management as a single learnable control problem for one agent policy. Expose six memory operations as tools in the action space, train a single LLM agent to invoke them via reinforcement learning over a three-stage trajectory curriculum, and use a step-wise GRPO variant that broadcasts the terminal task reward back to every intermediate memory decision so a single delayed signal can shape both early-stage storage and late-stage retrieval/filtering. The unified framing replaces the prior "static STM + agent-based LTM" architecture (and its expert-LLM controllers) with one end-to-end optimized policy.

## Method

AgeMem's policy operates over states $s_t = (C_t, \mathcal{M}_t, \mathcal{T})$ where $C_t$ is the active context (STM), $\mathcal{M}_t$ is the persistent long-term store (LTM), and $\mathcal{T}$ is the task specification. The action space mixes ordinary language generation with six memory tools (see [[tool-based-memory-operations]]):

- **LTM tools** — `Add` (insert entry into $\mathcal{M}_t$), `Update` (modify entry by `memory_id`), `Delete` (remove stale entry).
- **STM tools** — `Retrieve` (bring top-$k$ semantically relevant LTM entries into $C_t$), `Summary` (compress a span of $C_t$ into a concise representation), `Filter` (drop $C_t$ segments whose similarity to a criterion exceeds threshold $\theta_f$).

Training uses the [[three-stage-memory-curriculum]]: Stage 1 exposes the agent to contextual information $I_q$ in casual dialogue and lets it use LTM tools to construct $\mathcal{M}_t$; Stage 2 resets $C_t$ (preserving $\mathcal{M}_t$) and injects distractor utterances, forcing learned STM control via Summary/Filter/Retrieve; Stage 3 issues the formal query and requires coordinated retrieval + reasoning. The reset between Stages 1 and 2 closes the information-leakage path so the agent must actually use LTM rather than residual context. A `DistractorGen` procedure synthesizes Stage-2 noise without dataset labels, making the curriculum portable to non-QA settings.

Policy optimization uses [[step-wise-grpo]]: for the group $G_q = \{\tau_k^{(q)}\}_{k=1}^K$ of $K$ rollouts on task $q$, the terminal reward $r_T^{(k,q)}$ is normalized within the group into an advantage $A_T^{(k,q)} = (r_T - \mu_{G_q})/(\sigma_{G_q}+\epsilon)$, then **broadcast** to every preceding step ($A_t = A_T$). This propagates the late, sparse reward back through Stage-1 storage decisions and Stage-2 STM decisions, addressing the discontinuous-reward problem identified in C2. The objective adds a KL penalty $\beta D_{KL}[\pi_\theta \| \pi_{\text{ref}}]$ for stability.

The composite reward $R(\tau) = \mathbf{w}^\top \mathbf{R} + P_{\text{penalty}}$ has three signal channels: $R_{\text{task}}$ from an LLM judge on final answer correctness, $R_{\text{context}}$ that rewards compression efficiency + preventive summarization + information preservation, and $R_{\text{memory}}$ that rewards high-quality reusable entries + meaningful update/delete operations + semantic relevance of retrieved memories. Penalties target context overflow and excessive tool calls.

## Experiment & Results

**Setup.** Two backbones, Qwen2.5-7B-Instruct and Qwen3-4B-Instruct. RL fine-tuning is done *only* on HotpotQA (whose dataset structure provides natural Stage-1 supporting facts), then evaluated zero-shot on ALFWorld, SciWorld, PDDL, BabyAI. Metrics: Success Rate (ALFWorld, SciWorld, BabyAI), Progress Rate (PDDL), LLM-as-a-Judge (HotpotQA), plus Memory Quality (MQ) for stored LTM. Baselines: No-Memory, LangMem, A-Mem, Mem0, Mem0-graph, and AgeMem-noRL. Built on Agentscope + Trinity frameworks.

**Main results.** AgeMem achieves the best average score on both backbones — Qwen2.5-7B: 41.96% (vs. best baseline Mem0 at 37.14%, +4.82pp; vs. No-Memory +13.91pp absolute / +49.59% relative); Qwen3-4B: 54.31% (vs. best baseline A-Mem at 45.74%, +8.57pp; vs. No-Memory +10.34pp absolute / +23.52% relative). It wins on 4/5 datasets on Qwen2.5-7B (loses PDDL by 1.08pp to A-Mem) and 5/5 on Qwen3-4B. RL accounts for 8.53pp (Qwen2.5) and 8.72pp (Qwen3) over AgeMem-noRL, validating the three-stage curriculum.

**Memory quality.** Highest MQ scores on HotpotQA: 0.533 (Qwen2.5-7B) and 0.605 (Qwen3-4B), above all baselines — the unified framework stores higher-quality reusable knowledge.

**STM efficiency.** Compared to a -RAG ablation that replaces STM tools with retrieval, AgeMem reduces prompt tokens 3.1% on Qwen2.5-7B (2117 vs. 2186) and 5.1% on Qwen3-4B (2191 vs. 2310) while maintaining task performance.

**Tool usage analysis.** RL training shifts the tool-use distribution: on Qwen2.5-7B, `Add` operations rise from 0.92 to 1.64/episode, `Update` from ~0 to 0.13, `Filter` from 0.02 to 0.31; `Retrieve` drops slightly (2.31 → 1.95). The authors interpret this as a *qualitative* shift — pre-RL the agent retrieves reactively to compensate for poor Stage-1 storage; post-RL retrieval is more selective because LTM quality is better. The drop in retrieval frequency coincides with higher task scores and MQ.

**Ablations.** On Qwen2.5-7B/HotpotQA: +LTM-only gives +10.6/+14.2/+7.4pp on three datasets; +LTM+RL adds +6.3pp on HotpotQA; +LTM+STM+RL (full AgeMem) reaches +13.9/+21.7/+16.1pp. STM tools give the biggest boost on SciWorld (+3.1) and HotpotQA (+2.4) over RAG. **Reward function** ablation: All-Returns (full composite reward) beats Answer-Only ($R_{\text{task}}$ only) — Judge score 0.544 vs. 0.509, MQ 0.533 vs. 0.479; despite slightly higher token use (2117 vs. 2078) the full reward is decisively better. **FILTER threshold** $\theta_f$: performance is stable across $[0.4, 0.8]$ (Judge 0.524–0.551, MQ 0.510–0.550), with $\theta_f = 0.5$ best on this dataset.

## Limitations

The action space is a fixed set of six memory tools; the authors note finer-grained controls (e.g., partial-update operations, hierarchical memory structures) could be added but aren't explored. Evaluation is across five controlled long-horizon benchmarks, not persistent multi-session real-user deployments — cross-domain transfer is demonstrated *under controlled conditions*. RL training data comes only from HotpotQA, so the three-stage trajectory generation depends on HotpotQA's supporting-fact structure; extending to other curricula with richer interaction patterns is left for future work. The All-Returns reward uses slightly more tokens than the task-only variant, suggesting the unified policy occasionally over-uses memory operations even after training.

## Open questions

- How does AgeMem behave under truly persistent multi-session deployment where LTM grows across many episodes and forgetting/consolidation become first-order concerns? The benchmarks here are within-episode long-horizon, not cross-session.
- Can the curriculum be applied without HotpotQA-style supporting-fact labels? The paper claims the three-stage structure only needs temporal separation between exposure and execution, but doesn't demonstrate this with a non-QA training source.
- Is the broadcast-advantage assumption (every step in a trajectory gets the same advantage) optimal for memory operations, or would a learned step-level credit-assignment scheme (e.g., counterfactual advantages over memory tool calls) improve LTM quality further?
- The Update tool is invoked very sparsely (0.13–0.34/episode after RL). Is this because Update is genuinely rarely useful, or because the reward function under-incentivizes it? The MQ metric doesn't directly probe this.

## My take

The core empirical claim — that unifying LTM/STM control under one policy + one terminal reward beats specialized pipelines — is supported well by the cross-backbone, cross-benchmark gains. The cleanest evidence is the All-Returns vs. Answer-Only ablation: with only the task reward, MQ collapses, so the memory-quality and context-management reward channels are *load-bearing*, not decorative. The step-wise GRPO trick is also more general than just memory management — it's a generic recipe for any RL setting where intermediate actions have to be learned under a single late, sparse reward. The main caveat: training-on-HotpotQA-only and the controlled benchmark suite means the "unified memory" claim is currently a within-task statement, not a true persistent-memory statement; the user-facing long-term-memory regime where AgeMem-style policies would shine is exactly the regime that isn't tested here.

## Related

- [[unified-memory-management]]
- [[tool-based-memory-operations]]
- [[three-stage-memory-curriculum]]
- [[agemem]]
- [[step-wise-grpo]]
- [[agentic-memory]]
- [[continual-learning-evaluation]]
