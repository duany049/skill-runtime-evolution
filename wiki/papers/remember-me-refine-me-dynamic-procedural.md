---
title: "Remember Me, Refine Me: A Dynamic Procedural Memory Framework for Experience-Driven Agent Evolution"
slug: remember-me-refine-me-dynamic-procedural
arxiv: "2512.10696"
venue: arXiv
year: 2025
tags:
  - procedural-memory
  - agentic-memory
  - experience-replay
  - tool-use
  - llm-agents
  - memory-refinement
importance: 3
date_added: 2026-05-18
source_type: tex
tldr: ReMe is a procedural-memory framework where an LLM agent distills success/failure trajectories into structured experiences, retrieves and rewrites them context-adaptively, and uses utility-based deletion plus failure-aware reflection to keep the experience pool compact and effective.
contribution_type:
  - method
  - benchmark
  - analysis
datasets:
  - BFCL-V3
  - AppWorld
  - reme.library
code_url: "https://github.com/agentscope-ai/ReMe"
cited_by: []
---

## Problem & Context

Procedural memory — the encoding of "how-to" knowledge from prior task trajectories — has emerged as a key substrate for evolving LLM agents without retraining. Before ReMe, the dominant pattern was *passive accumulation*: agents stored either raw whole trajectories (Synapse, HiAgent) or summarized workflows (Agent KB, CER) into an append-only pool and retrieved by similarity at task time. Three structural problems plagued this paradigm:

1. **Coarse granularity.** Whole-trajectory experiences carry irrelevant noise and obscure the core decision logic the agent actually needs.
2. **Static reuse.** Retrieved experiences are injected verbatim, with no adaptation to the current task's constraints, so even close matches fail under slight context shifts.
3. **Unbounded growth without quality control.** With no removal mechanism, the pool degrades into a mixture of valid insights and "toxic noise" — including outdated entries and validated-but-wrong distillations.

The paper frames the right system as an *evolving cognitive substrate* satisfying three criteria: high-quality extraction, task-grounded utilization, and progressive optimization. Existing systems satisfy at most one or two.

## Key idea

Treat procedural memory as a closed loop with three coordinated mechanisms instead of an append-only archive:

- **Multi-faceted distillation** — extract fine-grained, structured experience units from both successes and failures, including comparative insights between paired trajectories.
- **Context-adaptive reuse** — index by an LLM-generated "usage scenario" rather than raw query text, then optionally rerank and rewrite the retrieved set into task-specific guidance.
- **Utility-based refinement** — track the per-experience utility online (recall count $f$ and success contribution count $u$), and remove any entry whose $u/f$ ratio falls below threshold $\beta$ after at least $\alpha$ retrievals.

A "memory scaling effect" emerges: Qwen3-8B + ReMe outperforms vanilla Qwen3-14B, and Qwen3-14B + ReMe surpasses vanilla Qwen3-32B — suggesting memory quality can substitute for parameter scale.

## Method

ReMe operates in three interconnected phases over a shared experience pool $\mathcal{E}$. Each experience is a 5-tuple $E = \langle \omega, e, \kappa, c, \tau \rangle$: usage scenario $\omega$ (when to apply), core content $e$, keyword set $\kappa$, confidence $c \in [0,1]$, and tool set $\tau$.

**Experience Acquisition.** For each training task $q$, sample $N=8$ trajectories at temperature $0.9$ from $\text{LLM}_{execute}$. Within each task group, the highest- and lowest-scoring trajectories are passed to a summarizer $\text{LLM}_{summ}$, which performs three complementary analyses:

- *Success pattern recognition* — distill underlying principles from trajectories scoring above threshold (empirically $1.0$).
- *Failure analysis* — identify the earliest decision step that drove a suboptimal outcome.
- *Comparative insight generation* — when there is a reward gap, articulate which specific decision distinguishes the higher- from the lower-scoring path.

A LLM-as-a-Judge validation step filters out non-actionable extractions, followed by cosine-similarity deduplication against the existing pool. Retained entries are indexed by an embedding of the *usage scenario* $\omega$ (not the task query) using text-embedding-v4 (1024-dim).

**Experience Reuse.** Given a new query, the retriever computes cosine similarity against indexed scenarios and returns the top-$K=5$. An optional context-aware reranker $\text{LLM}_{rerank}$ re-orders by relevance to the current task's constraints. A rewriting module then reorganizes the (possibly multiple) retrieved experiences into one cohesive, task-specific guidance block injected into the agent's context.

**Experience Refinement.** Two design choices drive lifelong quality:

- *Selective addition* — only experiences distilled from new *successful* trajectories enter the pool. The paper compares this against *full addition* and shows full-addition hurts because single-failed-trajectory analysis is unreliable in online execution (unlike the batch failure analysis available during initial pool construction).
- *Failure-aware reflection* — on a new failure, $\text{LLM}_{summ}$ extracts lessons and $\text{LLM}_{execute}$ retries with up to 3 reflection rounds; lessons are added to memory only if a retry succeeds.
- *Utility-based deletion* — remove $E$ when $f(E) \geq \alpha$ and $u(E)/f(E) \leq \beta$. Settings: $\alpha = 5$, $\beta = 0.5$.

The "dynamic" variant runs refinement during test-time execution; the "fixed" variant freezes the pool after the initial construction.

## Experiment & Results

**Benchmarks.** BFCL-V3 (Berkeley Function Calling Leaderboard; 50 base multi-turn tasks for pool construction, 150 evaluation tasks) and AppWorld (90 training tasks, 168 test-normal tasks; 9 simulated apps with 457 APIs). Metrics: Avg@4 and Pass@4 over 4 trials, averaged across 3 runs with standard deviation reported.

**Backbones.** Primary: Qwen3-8B / 14B / 32B (instruct). Generalization: GPT-4.1, o4-mini, Qwen3-Max-Preview, Kimi-K2-Thinking, DeepSeek-V3.2, GLM-4.7.

**Baselines.** No Memory, A-Mem (agentic memory system), LangMem (LangChain's episodic-memory module).

**Headline numbers.**

- Qwen3-8B + ReMe (dynamic): BFCL-V3 Avg@4 = 45.17 (vs. 40.33 No Memory), Pass@4 = 68.00 (vs. 59.55). AppWorld Avg@4 = 24.70 (vs. 14.97), Pass@4 = 42.06 (vs. 32.85). Average gain across both: +8.83 Avg@4, +7.29 Pass@4 absolute.
- Qwen3-14B + ReMe (dynamic): Avg@4 = 55.00 / 34.32, Pass@4 = 74.44 / 52.98. Memory-scaling effect: Qwen3-8B + ReMe Avg Pass@4 (55.03) exceeds vanilla Qwen3-14B (54.65); Qwen3-14B + ReMe Avg Pass@4 (63.71) exceeds vanilla Qwen3-32B (61.52).
- Qwen3-32B + ReMe (dynamic): BFCL-V3 Avg@4 = 56.17 / Pass@4 = 76.44; AppWorld Avg@4 = 42.02 / Pass@4 = 63.49.
- Across 6 additional backbones, ReMe-dynamic adds +4.27 to +8.83 Avg@4 (BFCL-V3); largest gain on Kimi-K2-Thinking (+8.83), smallest on GLM-4.7 (already strong at 68%) which still gains +5.83.

**Ablations.**

- *Granularity*: keypoint-level beats trajectory-level by ~3.7 Avg@4 on Qwen3-8B (+4.17 vs. +2.67 over No Memory). Fine-grained units transfer better.
- *Components on Qwen3-8B BFCL-V3*: full-addition 40.83 → selective-addition 44.33 (+3.50) → + reflection 45.00 → + deletion 45.17 / 68.00 Pass@4 (+3.34 over reflection-only Pass@4 of 64.66). Each refinement component contributes; deletion mainly lifts Pass@4.
- *Rerank + rewrite*: each module alone adds ~2 Avg@4; together +1.83 Avg@4 / +6.01 Pass@4 over baseline ReMe on a Qwen3-8B no-thinking config.
- *Retrieval key*: scenario-based indexing > generalized-query > extracted-keywords > raw task description.
- *Summarizer scale*: keeping $\text{LLM}_{execute}$ = Qwen3-8B but scaling $\text{LLM}_{summ}$ to 14B / 32B yields +1.83 / +3.33 Avg@4. Distillation quality compounds.
- *K*: performance rises then saturates; degradation past saturation attributed to noisier retrieved entries. $K=5$ is chosen.
- *Latency*: +2.54 s per task on Qwen3-8B AppWorld (21.42 s → 23.96 s) — modest.

**Error analysis.** On Qwen3-8B BFCL-V3, total failures drop from 62 (No Memory) to 47 (ReMe). ReMe corrects 17 baseline-specific errors and introduces only 2 new ones. The largest reduction is in *Reasoning Error* (22 → 14); *Action Omission* also drops moderately.

## Limitations

- Retrieval is performed once at task start — a context-aware, mid-execution retrieval policy is not explored.
- Experience validation relies on LLM-as-a-Judge, which may miss nuanced quality issues; richer validation is left for future work.
- Larger summarizer models give larger gains, but the paper does not propose a way to recover those gains with a small summarizer (e.g., distillation, multi-pass refinement).
- Benchmarks are tool-augmented function-calling tasks (BFCL-V3, AppWorld); generalization to long-horizon planning, multi-agent settings, or open-ended creative tasks is untested.

## Open questions

- Can the utility threshold $\beta$ be learned or adapted per task family instead of fixed at 0.5?
- The "memory-scaling effect" — does it hold for non-Qwen families and at larger scales (70B+)? Does it compound with parametric scaling, or saturate?
- How does the experience pool behave under deliberately adversarial / poisoned trajectories? Reflection-based addition could be exploited.
- Is keypoint-level distillation a transferable artifact (a `reme.library` paper-released dataset) or backbone-specific? The paper releases reme.library but does not stress-test cross-backbone reuse of stored experiences.

## My take

The strongest contribution is not any single mechanism but the *closure of the loop*: addition + reflection + deletion are simple individually, yet their combination yields a self-correcting pool. The empirical memory-scaling claim (8B + ReMe ≥ vanilla 14B) is the most provocative bit — if it survives replication on other model families and longer horizons, it reframes procedural memory as a deployment-time compute substitute, not just an accuracy add-on. The result that *single-failure online analysis hurts more than it helps* (motivating selective-addition) is an under-discussed practical lesson and deserves more attention from systems that automatically write to memory on every trajectory.

## Related

- [[reme-remember-me-refine-me]]
- [[procedural-memory]]
