---
title: "MemSkill: Learning and Evolving Memory Skills for Self-Evolving Agents"
slug: memskill-learning-evolving-memory-skills-self
arxiv: "2602.02474"
venue: arXiv
year: 2026
tags:
  - agentic-memory
  - skill-evolution
  - self-improvement
  - memory-management
  - reinforcement-learning
  - llm-agent
importance: 4
date_added: 2026-05-18
source_type: tex
tldr: "MemSkill reframes LLM-agent memory operations as a learnable, evolvable skill bank, training a controller via RL to select skills and a designer LLM to refine/expand the bank from hard cases."
contribution_type:
  - method
  - system
datasets:
  - LoCoMo
  - LongMemEval
  - HotpotQA
  - ALFWorld
code_url: "https://github.com/ViktorAxelsen/MemSkill"
cited_by: []
---

## Problem & Context

Most LLM-agent memory systems still depend on small, hand-designed operation primitives (add/update/delete/skip) and heuristic modules that hard-code human priors about *what* to store, *how* to revise, and *when* to prune. Representative pipelines—MemoryBank, A-MEM, Mem0, MemoryOS, MemGPT, LightMem—periodically extract salient information into an external store, retrieve entries for a new query, then consolidate or prune via fixed routines. These designs are brittle under diverse interaction patterns and scale poorly on long histories. Recent learning-based work such as Memory-R1 and Mem-α adds RL to memory management, but still operates over a static action set. Concurrent self-evolving memory work either benchmarks test-time evolution (Evo-Memory) or meta-optimizes memory *architectures* within a fixed modular space (MemEvolve)—neither evolves the memory *operations* themselves. MemSkill targets exactly that gap: it lifts memory extraction into a learnable abstraction—a set of generic, reusable *memory skills*—that an agent can both select and continually grow from interaction data.

## Key idea

Reframe the canonical operation primitives as items in an **evolving skill bank** and learn two coupled processes on top of it:

1. **Skill use** — a lightweight controller selects a small Top-K subset of skills from the bank conditioned on the current text span and retrieved memories; an LLM executor conditions on those skills to emit memory updates in one pass at span-level granularity (rather than turn-by-turn).
2. **Skill evolution** — periodically, an LLM-based designer mines representative hard cases from a sliding buffer of training failures and proposes refinements to existing skills or entirely new skills, with snapshot-and-rollback if a proposed update degrades performance.

Cycling these two processes yields a closed-loop self-evolving memory system whose action space (the skill bank) is itself shaped by the data, with minimal hand-coded priors.

## Method

MemSkill maintains two stores: a **memory bank** (trace-specific, e.g., per dialogue) and a **skill bank** (shared across all traces, initialized with four primitives—`Insert`, `Update`, `Delete`, `Skip`). Each skill carries a short *description* used for selection and a detailed *content* spec used by the executor.

**Controller (skill-selection policy).** For each text span $x_t$ and retrieved memories $M_t$, the controller computes a state embedding $h_t = f_\text{ctx}(x_t, M_t)$ and a skill embedding $u_i = f_\text{skill}(\text{desc}(s_i))$ for every skill in the current bank (shared Qwen3-Embedding-0.6B encoder). Skill scores are inner products $z_{t,i} = h_t^\top u_i$ followed by softmax. Because scoring is computed against *embeddings of descriptions*, the controller automatically adapts when the skill bank grows or shrinks—no fixed-dimensional action head. The controller draws an ordered Top-K set without replacement via Gumbel-Top-K and forwards it to the executor.

**Executor.** A fixed LLM that consumes $(x_t, M_t, A_t)$ and emits structured memory updates parsed back into the trace's memory bank.

**Controller optimization.** PPO-style RL with downstream task performance (F1 / success rate) as reward. The action is an *ordered Top-K set*, so the joint log-probability is computed under the without-replacement selection process and used with importance weighting and clipping:
$\pi_\theta(A_t \mid s_t) = \prod_{j=1}^{K} \frac{p_\theta(a_{t,j}\mid s_t)}{1-\sum_{\ell<j} p_\theta(a_{t,\ell}\mid s_t)}$

**Designer (skill evolution).** A sliding **hard-case buffer** keeps query-centric failures with metadata (retrieved memories, predictions, repeated-failure count). Cases are KMeans-clustered to reflect distinct error types; within each cluster, difficulty-weighted representative cases are sampled. A two-stage LLM designer first analyzes the selected cases to identify missing/mis-specified memory behaviors, then proposes concrete edits (refine existing skills + add new skills). Best-bank **snapshot-and-rollback** guards against regressions; **exploration boosting** after each evolution step biases selection toward newly added skills so the controller can learn to use them.

**Closed-loop optimization.** Each cycle: train controller on current bank → accumulate hard cases → designer updates the bank (rollback if regression) → next cycle continues controller training with boosted exploration. Skill evolution fires every 100 training steps with at most 3 edits per round.

## Experiment & Results

**Setup.** Four benchmarks: LoCoMo and LongMemEval (long-dialogue memory; F1 and LLM-judge), ALFWorld with Seen/Unseen splits (embodied; success rate and number of environment steps), and HotpotQA (transfer to long-document QA). Base LLMs: LLaMA-3.3-70B-Instruct and Qwen3-Next-80B-A3B-Instruct. Controller is a small MLP; shared encoder Qwen3-Embedding-0.6B; retriever Contriever. Memory budget capped at 20 retrieved items for all methods. Evaluation defaults: K=7 for LoCoMo/LongMemEval, K=5 for ALFWorld, span size 512 tokens. Baselines: No-Memory, Chain-of-Notes, ReadAgent, MemoryBank, A-MEM, Mem0, LangMem, MemoryOS.

**Main results.** MemSkill achieves the strongest overall performance across all three core benchmarks for both base models. On LoCoMo and LongMemEval it tops the LLM-judge score within each base-model block; on ALFWorld it wins success rate on both Seen and Unseen. Notably, **LongMemEval is evaluated in pure transfer**—the skill bank trained on LoCoMo is applied without further training—and still beats every baseline.

**Cross-base-model transfer.** Skills are trained only on LLaMA and applied directly to Qwen without retraining; MemSkill remains competitive and continues to outperform strong baselines, showing the evolved skills capture reusable memory behaviors independent of the underlying LLM.

**Distribution-shift transfer to HotpotQA.** The LoCoMo-trained skill bank transfers to HotpotQA's long-document QA across 50/100/200-document context settings, beating MemoryOS and A-MEM, with the gap widening at 200 documents. Sensitivity sweep over K shows K=7 best across all three context sizes; smaller K underutilizes the bank in longer contexts.

**Ablations on LoCoMo (LLM-judge, both LLaMA and Qwen).** Removing the controller (random skill selection) → clear drop. Removing the designer (skill bank frozen at the 4 primitives) → even larger drop, especially on Qwen. **Refine-only** (no new skills introduced) → consistently beats static skills but still below full MemSkill, particularly on Qwen—confirming that *introducing new skills* yields gains beyond refining the seed primitives.

**Case studies.** Final evolved banks specialize cleanly: LoCoMo skills emphasize temporal context and who-did-what-where-when activity structure; ALFWorld skills emphasize action constraints, object locations, and task-relevant preconditions for multi-step execution.

## Limitations

- The hard-case buffer, KMeans clustering, and designer prompt all introduce hyperparameters (sliding-window size, expiration step gap, cluster count, K, evolution period, max edits per round) whose interaction is not fully ablated.
- The designer is a fixed LLM acting on summarized hard cases; capacity of the designer LLM is itself a confound for what skills can be discovered, but this is not isolated.
- Evolution success depends on snapshot-and-rollback to dodge regressions, suggesting individual designer proposals are noisy and may regress without this guardrail.
- Reported experiments use API-served base LLMs (LLaMA-3.3-70B, Qwen3-Next-80B); generalization to smaller, locally-deployable models is not measured.
- The four-primitive seed bank (`Insert/Update/Delete/Skip`) is itself a hand-coded prior; how sensitive the system is to alternative seed sets is not reported.

## Open questions

- How well do MemSkill-evolved skills transfer across modalities (e.g., from text dialogue to multi-modal or robotic memory)?
- Can skill-bank growth be regulated without an explicit max-edits cap—e.g., via consolidation or retirement policies analogous to those discussed in skill-library work?
- What is the marginal value of designer-driven evolution vs. a strong one-shot skill-generation baseline matched on compute?
- Does cross-user / cross-deployment skill aggregation (instead of single-trajectory evolution) yield further gains, and how should conflicting designer proposals be reconciled?
- How does MemSkill behave under adversarial / noisy memory traces designed to destabilize the controller or trigger pathological skill edits?

## My take

MemSkill's framing—*lift the memory action set from fixed primitives to a learnable, evolvable skill bank*—is exactly the move suggested by the broader "skill library" research line, and the embedding-based controller cleanly handles the variable action space. The ablation that isolates `Refine-only` vs. full evolution is the strongest evidence in the paper: it directly shows that *adding new skills* contributes on top of refining seed primitives, which is the harder claim. The LoCoMo→HotpotQA distribution-shift result is also a high-quality probe—much more telling than within-benchmark transfer—and lends credibility to the "memory skills are reusable" thesis.

The most interesting open territory is whether the snapshot-and-rollback dependence reveals a deeper instability of designer-proposed edits, which would suggest more principled evolution operators (consolidation, retirement, conflict resolution) are needed before MemSkill scales to longer-running deployments.

## Related

- [[agentic-memory]] — MemSkill is a learnable / evolvable point on the agent-memory frontier this topic surveys.
- [[skill-evolution]] — MemSkill is a memory-flavored instance of the bottom-up skill-evolution agenda; its designer is a concrete evolution operator.
- [[procedural-memory]] — Memory skills are procedural artifacts (when-to-apply + how-to-apply) shared across traces.
- [[memory-skill]] — Concept page introduced by this paper.
- [[skill-conditioned-memory-construction]] — Concept page introduced by this paper.
- [[closed-loop-skill-evolution]] — Concept page introduced by this paper.
- [[evolvable-skill-bank-controller]] — Method page introduced by this paper.
- [[designer-driven-skill-bank-evolution]] — Method page introduced by this paper.
