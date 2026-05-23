---
name: AutoRefine
slug: autorefine
type: system
tags:
  - llm-agents
  - continual-learning
  - skill-library
  - experience-extraction
  - pattern-maintenance
source_papers:
  - autorefine-trajectories-reusable-expertise-continual-llm
parent_methods: []
child_methods: []
code_repo: ""
date_updated: 2026-05-19
---

## Problem setting

AutoRefine targets the **continual LLM-agent refinement** setting: an agent receives a stream of tasks from a domain $\mathcal{D}$, executes each via a trajectory $\tau_i = \langle s_0, a_1, o_1, \ldots, s_T \rangle$, and receives a sparse binary feedback $f_i \in \{\text{success}, \text{failure}\}$ at episode termination. The goal is to *accumulate* transferable knowledge across this task stream — without weight updates — such that performance on later tasks improves while the prompt context does not balloon. Concretely the system must answer three coupled questions: what unit of knowledge to extract from a trajectory, how to keep the resulting repository compact under unbounded task streams, and how to retrieve and apply that knowledge on new tasks. AutoRefine sits in the in-context-learning-with-self-generated-experience corner of the design space — no RL gradients, no external annotation, no human demonstrations.

## Mechanism

A pattern repository $\mathcal{P} = \{p_1, \ldots, p_M\}$ holds two pattern types: **skill patterns** (text guideline or code snippet) and **subagent patterns** (specialized LLM agents with their own memory). Each $p_j$ carries metadata $(d_j, c_j, r_j, u_j, s_j, e_j)$ — description, applicability context, retrieval count, utilization count, success count, embedding.

Three stages interleave:

1. **Task execution with patterns** — pattern retrieval (multi-query reformulation + embedding similarity + optional MMR), pattern integration (skills enter system prompt or tool space; subagents are delegated via hierarchical context transfer), and metadata tracking (increment $r_j$ on retrieval, $u_j$ on verified usage via a dedicated verifier agent, $s_j$ on task success).
2. **Pattern extraction from trajectories** — every $K{=}10$ tasks, an extraction agent contrasts successful and failed trajectories in the recent batch and abstracts recurring success-correlated decision strategies into new patterns. The agent decides skill-vs-subagent based on procedural complexity.
3. **Pattern maintenance** — at exponentially spaced intervals ($10, 20, 40, 80, \ldots$), the framework scores patterns, prunes the bottom 20% by utility, and merges similar same-type patterns via agglomerative clustering guided by a merge agent.

## Procedure

1. Initialize $\mathcal{P} \leftarrow \emptyset$.
2. For each incoming task $t_i$:
   a. Generate $m$ retrieval queries $\{q_1, \ldots, q_m\}$ that reformulate $d_{\text{task}}$.
   b. Embed each $q_k$ with Qwen3-Embedding-4B; for each $p_j \in \mathcal{P}$, compute $\text{sim}(t_i, p_j) = \max_k \cos(e_{q_k}, e_{p_j})$.
   c. Build $\mathcal{P}_{\text{retrieved}} = \text{TopK}(\{p_j : \text{sim}(t_i, p_j) \ge \theta\}, k)$; optionally apply MMR for diversity.
   d. Inject skill-pattern guidelines into the system prompt, register skill-pattern code as callable tools, and bind matched subagent patterns as delegate calls.
   e. Run the main agent on $t_i$. The main agent may delegate matched subtasks to a subagent, which executes in its own context and returns a result.
   f. For each $p_j \in \mathcal{P}_{\text{retrieved}}$: $r_j \mathrel{+}= 1$; if verifier confirms invocation, $u_j \mathrel{+}= 1$; if $f_i = \text{success}$, $s_j \mathrel{+}= 1$.
3. **Every $K$ tasks** (default $K = 10$): partition the recent batch into $\mathcal{H}^+ = \{(\tau_i, 1)\}$, $\mathcal{H}^- = \{(\tau_i, 0)\}$. Feed both into the extraction agent, which:
   a. Identifies recurring action sequences and decision strategies discriminating success from failure.
   b. Writes a natural-language causal explanation.
   c. Abstracts the explanation into a structured pattern with metadata $(d_p, c_p, 0, 0, 0, e_p = \text{Embed}(c_p))$.
   d. Sets pattern type: simple procedural guideline / code → skill; multi-step, state-tracking procedure → subagent.
4. **At maintenance milestones $n_{\text{threshold}} \in \{10, 20, 40, 80, \ldots\}$:**
   a. Score each pattern $\text{score}(p_j) = \frac{s_j}{u_j + \epsilon} \cdot \log(1 + u_j) \cdot \left(1 + \frac{u_j}{r_j + \epsilon}\right)$ with $\epsilon = 0.01$.
   b. Prune the bottom $\alpha = 20\%$.
   c. Build candidate pairs $\mathcal{C} = \{(p_i, p_j) : \text{sim}(e_i, e_j) \ge 0.85 \land \text{type}(p_i) = \text{type}(p_j)\}$.
   d. For each candidate, ask the merge agent $\mathcal{A}_{\text{merge}}$ whether the patterns address the same subtask, have compatible procedures, and have overlapping contexts. If yes, replace both with $p_{\text{merged}}$: agent-synthesized description and context, recomputed embedding, summed counts.
   e. Repeat the most-similar-pair merge until $\mathcal{C}$ is empty.

## Assumptions

- The task stream provides sparse binary success/failure feedback at episode termination ($f_i \in \{0, 1\}$). The framework does not address shaped or partial rewards.
- A semantic-embedding model (Qwen3-Embedding-4B in the paper) is available for context similarity.
- An LLM strong enough to act as extraction agent and merge agent is available (Claude-sonnet-4 or GPT-4-turbo in the paper).
- Trajectories are observable and storable; the framework keeps the recent batch $\mathcal{H}_{\text{recent}}$ on hand for extraction.
- The task distribution has *some* recurring procedural structure — otherwise extraction returns no generalizable patterns. AutoRefine has not been evaluated under abrupt concept drift.
- Subagent delegation is a single level (main → subagent); deeper hierarchy is not addressed.

## Limitations

- Maintenance schedule, $\theta_{\text{merge}} = 0.85$, prune $\alpha = 20\%$, and batch size $K = 10$ are hand-tuned hyperparameters; the paper does not propose a way to learn them.
- All learning is success-conditioned: failures supply contrastive signal but no patterns are extracted directly from failure modes. The paper flags this as future work.
- Each maintenance cycle requires LLM calls (merge agent) and embedding recomputation; cost is not reported.
- On case-specific hard constraints (TravelPlanner hard-constraint-macro), the auto-extracted system underperforms manually designed agents like ATLAS, because the extracted patterns favor universal rules over case-specific verification logic.
- Cross-domain repository transfer (patterns learned on TravelPlanner reused on ScienceWorld) is untested.
- The verifier agent that decides whether a retrieved pattern was actually used is another LLM, so its error rate propagates into the scoring loop.

## Tradeoff profile

- **Compute vs. prompt length.** Multi-query retrieval, dedicated verifier and merge agents, and subagent contexts cost more LLM calls than naive "stuff all experience into prompt" but keep the main agent's context bounded — which is the whole point as the repository scales.
- **Compactness vs. specificity.** The 20% prune and $\theta_{\text{merge}} = 0.85$ favor a small, general repository. Ablation on TravelPlanner shows the no-maintenance variant grows 4.5× larger and gains slightly on hard constraints (44.4% vs 38.9%) but loses overall (31.1% vs 35.6%). So compactness costs you a little on case-specific tasks.
- **Procedural expressivity vs. design effort.** Subagent patterns cost the extraction agent more reasoning per creation than skill patterns and require a delegation harness, but they handle procedures that flat text cannot. On TravelPlanner this is the dominant contribution (subagent ablation = –22.3 points).
- **Self-improvement vs. interpretability.** Skill patterns are inspectable text; subagent patterns are full agents whose behavior is harder to audit. The framework leans into automation, which trades interpretability for capacity.
- **Convergence with reflection.** AutoRefine alone (27.1% on TravelPlanner test) and AutoRefine+ReAct (34.1%) suggest experience accumulation and within-task reflection sit on complementary axes — the practical recommendation is to combine them rather than pick one.
