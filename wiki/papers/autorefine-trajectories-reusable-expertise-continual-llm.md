---
title: "AutoRefine: From Trajectories to Reusable Expertise for Continual LLM Agent Refinement"
slug: autorefine-trajectories-reusable-expertise-continual-llm
arxiv: "2601.22758"
venue: arXiv preprint (ICML 2026 submission format)
year: 2026
tags:
  - llm-agents
  - continual-learning
  - skill-library
  - experience-extraction
  - subagent
  - procedural-memory
importance: 3
date_added: 2026-05-19
source_type: tex
tldr: AutoRefine extracts dual-form Experience Patterns (skills and subagents) from agent trajectories and maintains them via scoring, pruning, and merging, outperforming manually designed multi-agent systems on TravelPlanner (27.1% vs 12.1%).
contribution_type:
  - method
  - system
datasets:
  - ALFWorld
  - ScienceWorld
  - TravelPlanner
code_url: ""
cited_by: []
---

## Problem & Context

LLM-based agents have been deployed on web navigation, robotics, and household assistance, but they typically treat each task as independent — they do not autonomously accumulate transferable knowledge across tasks the way humans do. Prior work that tries to close this gap by extracting knowledge from execution traces (e.g., ExpeL, AutoGuide, AutoManual, Voyager, AdaPlanner, Reflexion, CLIN, RAP, MemGPT) faces two limits the paper names directly:

1. **Flattened textual knowledge is insufficient for procedural logic.** Extracted experience is represented as static text. That representation cannot encode procedural subtasks that involve sequential steps, conditional branching, and state tracking (e.g., hotel booking inside a travel-planning task).
2. **No experience maintenance.** Methods accumulate experience and inject it into the prompt, relying on the LLM to filter relevance. As the repository grows, the prompt blows out, retrieval gets noisier, and obsolete/redundant patterns are never pruned.

Adjacent paradigms — RL-based agent training, prompt optimization with external annotated examples, and reflection-only methods (Reflexion, Self-Refine, Tree-of-Thoughts) — either need substantial compute and supervision or do not accumulate across tasks. The state before this paper, then, is: extraction-based agents that cannot represent procedure and cannot keep their own repository clean.

## Key idea

Represent learned experience as a **dual-form Experience Pattern repository** with continuous maintenance. Two forms cover different kinds of knowledge:

- **Skill patterns** — natural-language guidelines or executable code snippets for static or single-step strategic knowledge.
- **Subagent patterns** — specialized agents with their own memory and reasoning that encapsulate procedural subtasks (e.g., booking, multi-step coordination). The main agent delegates matching subtasks to them as atomic operations.

A continuous maintenance loop (score → prune → merge) at exponentially spaced intervals keeps the repository compact: low-utility patterns are dropped, semantically similar same-type patterns are merged. Pattern selection at execution time uses multi-query reformulation, embedding similarity (Qwen3-Embedding-4B), and optional MMR for diversity.

## Method

The framework has three stages run in a loop.

**Stage 1 — Task execution with patterns.** For each task description $d_{\text{task}}$, an LLM generates $m$ retrieval queries $\{q_1, \dots, q_m\}$ that reformulate the request. Each query is embedded with Qwen3-Embedding-4B and matched against pattern context embeddings via cosine similarity. The per-pattern similarity is the max over queries. The framework selects $\mathcal{P}_{\text{retrieved}} = \text{TopK}(\{p_j : \text{sim}(t_i, p_j) \ge \theta\}, k)$, optionally refined by MMR with $\lambda$ balancing relevance and diversity. Skill patterns enter the system prompt (as guidelines) or the agent's tool space (as callable code). Subagent patterns become hierarchical delegates: the main agent transfers task context to a matched subagent, which executes with its own memory and returns a result. Each pattern carries metadata $m_j = (d_j, c_j, r_j, u_j, s_j, e_j)$ — description, context, retrieval count, utilization count, success count, and embedding. Counts are incremented at retrieval, usage (verified by a dedicated verifier agent), and on $f_i = \text{success}$.

**Stage 2 — Pattern extraction from trajectories.** Every $K{=}10$ tasks, an extraction agent $\mathcal{A}_{\text{extract}}$ inspects the recent batch $\mathcal{H}_{\text{recent}}$ partitioned into successes $\mathcal{H}^+$ and failures $\mathcal{H}^-$. The agent performs contrastive analysis: it identifies recurring action sequences and decision strategies whose presence correlates with success and absence with failure, produces a natural-language causal explanation, and abstracts it into a structured pattern with metadata initialized to zero counts and embedding $e_p = \text{Embed}(c_p)$. The agent decides pattern type by complexity: simple guideline/code → skill pattern; multi-step procedure needing sustained reasoning → subagent pattern. Sparse-reward setting ($f_i \in \{0, 1\}$), following ExpeL and AutoGuide.

**Stage 3 — Pattern maintenance.** At exponentially spaced intervals ($n_{\text{threshold}} = 10, 20, 40, 80, \ldots$, following Sarukkai et al. 2025), the framework runs scoring, pruning, and merging.

Score:

$$\text{score}(p_j) = \frac{s_j}{u_j + \epsilon} \cdot \log(1 + u_j) \cdot \left(1 + \frac{u_j}{r_j + \epsilon}\right)$$

with $\epsilon = 0.01$. Three factors: effectiveness (success rate among uses), frequency (log-scaled usage), precision (utilization-to-retrieval ratio). The bottom $\alpha = 20\%$ are pruned.

Merging is two-stage. Vector-similarity filtering builds candidate set $\mathcal{C} = \{(p_i, p_j) : \text{sim}(e_i, e_j) \ge \theta_{\text{merge}} \land \text{type}(p_i) = \text{type}(p_j)\}$ with $\theta_{\text{merge}} = 0.85$ (same-type only). Then a merge agent $\mathcal{A}_{\text{merge}}$ checks whether each candidate pair (i) addresses the same subtask, (ii) has compatible procedural steps, (iii) has overlapping applicability contexts. Confirmed pairs are merged: agent-synthesized description and context, recomputed embedding, aggregated counts $r_{\text{merged}} = r_i + r_j$, $u_{\text{merged}} = u_i + u_j$, $s_{\text{merged}} = s_i + s_j$. Agglomerative iteration: repeatedly merge the most similar admissible pair until none remain.

## Experiment & Results

Benchmarks: ALFWorld (140 test, household), ScienceWorld (30 "boil"/"grow" tasks), TravelPlanner (180 validation, 1000 test scenarios). Backbone: Claude-sonnet-4, temperature 0.7 by default; GPT-4-turbo for the ALFWorld subtask comparison to align with prior work's setting. Three seeds with different task orderings; paired t-test at $p < 0.05$.

Main results (success rate / steps):

- ALFWorld: **98.4% ± 1.5** / 12.8 ± 0.8 steps vs. ReAct+Reflexion 95.5% / 16.1 steps — +2.9 SR, 20.5% step reduction.
- ScienceWorld (Pass@1): **70.4% ± 1.9** / 16.5 steps vs. ReAct+Reflexion 69.2% / 40.2 steps — +1.2 SR, 59.0% step reduction.
- TravelPlanner (test): **27.1% ± 2.4** / 21.8 steps vs. ReAct+Reflexion 9.1% / 80.2 steps — +18.0 SR, 72.8% step reduction.

ALFWorld subtask breakdown (GPT-4-turbo, validation unseen, 134 tasks): AutoRefine zero-shot reaches 97.0% overall; AutoManual with 1 manual example reaches 97.4%. AutoRefine matches AutoManual on Put, Heat, Examine (100.0%) and trails by 0.4–2.0 on Clean (96.8 vs 98.9), Cool (95.2 vs 95.4), Put Two (88.2 vs 90.2). Beats ExpeL (12 examples, 79.2%) and AdaPlanner (6 examples, 76.4%) without any manual examples.

TravelPlanner test (final pass rate): AutoRefine 27.10%, AutoRefine+ReAct 34.10%; ATLAS 12.12%, ReAct 10.40%, ReAct+Reflexion 9.13%. Breakdown: AutoRefine commonsense-macro 37.90% vs ATLAS 15.59%; AutoRefine hard-constraint-macro 32.60% vs ATLAS 33.56%. So AutoRefine wins on universal commonsense constraints but loses to manually designed ATLAS on case-specific hard constraints; the combination with ReAct closes both gaps.

Ablation on TravelPlanner validation (full pass: 35.56%):

- w/o subagents → 13.3% (–22.3); commonsense drops 54.4 → 23.3 (–57% relative), hard 38.9 → 29.4 (–24% relative). Largest single-component impact.
- w/o batch extraction → 18.3% (–17.3); repository balloons to 58 patterns (2.4× full); patterns overfit single tasks.
- w/o maintenance → 31.1% (–4.5); repository grows linearly to 108 patterns (4.5× full); utilization rate drops 0.71 → 0.08 (8.9× degradation).

## Limitations

- The paper's gains depend on the benchmarks' having recurring procedural structure that can be encapsulated as subagents; on benchmarks dominated by one-off case-specific verification, manually designed agents (ATLAS hard-constraint micro) still match or exceed.
- The maintenance schedule is heuristic (exponential spacing borrowed from Sarukkai et al.); thresholds $\theta_{\text{merge}} = 0.85$, prune $\alpha = 20\%$, $K = 10$ are hand-tuned and not learned.
- All learning is from successes (sparse-reward, $f_i \in \{0, 1\}$). The conclusion explicitly flags "extending the framework to learn from failures" as future work.
- The Qwen3-Embedding-4B dependency and the agent-driven extraction/merge passes incur extra LLM and embedding calls per maintenance cycle; the paper does not report dollar cost.
- ScienceWorld evaluation is restricted to "boil" and "grow" categories — 30 tasks — limiting the breadth of the scientific-reasoning claim.
- Cross-domain transfer of the repository (patterns learned in TravelPlanner used in ScienceWorld, etc.) is not evaluated.

## Open questions

- Can pattern extraction be made symmetric between successes and failures rather than success-only?
- Are the maintenance hyperparameters learnable from data (e.g., adapting $\theta_{\text{merge}}$ to repository density)?
- Does the subagent abstraction scale to deeper hierarchies (sub-subagents) or hit a delegation-overhead wall?
- How do repository quality and utilization rate co-evolve when the task distribution shifts (concept drift)?
- The verifier agent that confirms actual pattern utilization is itself an LLM — what is the false-positive/negative rate, and how sensitive is the scoring loop to verifier errors?

## My take

The paper reads as a clean integration step rather than a conceptual leap: it stitches together known ingredients — contrastive extraction from successes vs failures (AutoGuide, ExpeL), exponential-interval maintenance (Sarukkai et al.), embedding retrieval with MMR — and adds the genuinely new piece of treating procedural subtasks as full subagents with their own memory rather than text strings. The TravelPlanner result is the load-bearing number: 27.1% on the test set versus ATLAS's 12.1% suggests that "subagent as compiled procedure" beats "manually designed multi-agent stack" on commonsense procedural coordination, which is the strongest empirical signal in the paper. The hard-constraint result also tells you the honest version of the story: this is not a universal upgrade over hand-designed systems; it is specifically a win on universal procedural patterns and a loss on case-specific verification. The combination with ReAct (Ours+ReAct = 34.1%) is the practical recommendation. From a research-direction standpoint, the most interesting question the paper raises is whether the subagent boundary should be a learned object — right now it is decided by the extraction agent at pattern-creation time, but the framework's success suggests this boundary carries most of the signal.

## Related

- [[subagent-pattern]]
- [[autorefine]]
- [[agentic-memory]]
- [[procedural-memory]]
- [[skill-evolution]]
- [[continual-learning-evaluation]]
