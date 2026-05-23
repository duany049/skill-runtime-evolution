---
title: "AtomMem: Learnable Dynamic Agentic Memory with Atomic Memory Operation"
slug: atommem-learnable-dynamic-agentic-memory-atomic
arxiv: "2601.08323"
venue: arXiv
year: 2026
tags:
  - agentic-memory
  - reinforcement-learning
  - memory-management
  - long-context-qa
  - web-agents
  - crud-operations
importance: 4
date_added: 2026-05-18
source_type: tex
s2_id: 6f7c447cd5fd3bc7b02fcd795297af305b06c344
tldr: AtomMem reframes agent memory management as a learnable decision-making problem over atomic CRUD operations, trained end-to-end with GRPO to outperform static-workflow baselines on long-context QA and web benchmarks.
contribution_type:
  - method
  - analysis
datasets:
  - HotpotQA
  - 2WikiMultihopQA
  - MuSiQue
  - GAIA
  - WebWalkerQA
  - Asearcher
code_url: "https://github.com/RUCBM/AtomMem"
cited_by: []
---

## Problem & Context

Equipping LLM-based agents with usable memory beyond the context window is a prerequisite for long-horizon tasks, but the dominant designs hard-code their memory workflows. Static expert pipelines fall into two families: *imitation-based* (e.g., MemoryBank, MemGPT borrow human/computer memory metaphors) and *prior-based* (e.g., MemTree, ChatDB, MemoRAG, SCM hand-craft retention rules). Recent RL-enhanced systems—MemAgent, Mem1—still constrain the workflow (e.g., a mandatory summarize-at-every-step routine); tool-style memory systems—Memory-as-Action, AgentFold—expose pruning or folding primitives but design those tools manually. The unifying limitation is an implicit "one-size-fits-all" assumption: a single workflow is forced across heterogeneous task structures, even when the same rule that helps Task A (e.g., exponential forgetting) actively hurts Task B (e.g., long-horizon reasoning that needs early cues). The paper asks: rather than designing the workflow, can the *workflow itself* be learned end-to-end from task feedback?

## Key idea

Reframe agent memory management as a **POMDP over atomic CRUD operations** and let RL optimize the memory policy. The action space `A^mem = {Create, Read, Update, Delete}` is chosen because CRUD is (1) **complete** — any memory state reachable from another via some sequence of CRUD ops; (2) **atomically minimal** — every higher-level memory tool is a structured combination of CRUD calls; and (3) **task-agnostic** — independent of any downstream benchmark. Memory is part of the environment with `s_0^mem = ∅` (reset per task), and the joint action `a_t = (a_t^env, a_t^mem)` interleaves task actions and memory actions, so the agent learns *when* to write/retrieve/revise/erase rather than executing a fixed pipeline.

## Method

**AtomMem** combines an action vocabulary, a hybrid retrieval substrate, and a single-stage RL objective:

- **Atomic CRUD action vocabulary.** Each memory primitive is exposed as a structured XML tag in the model's output: `<create_memory>{content}</create_memory>`, `<read_memory>{query}</read_memory>`, `<update_memory>{memory id: content}</update_memory>`, `<delete_memory>memory id</delete_memory>`. At each decision step the policy emits a sequence `A_t = {a_t^1, …, a_t^{K_t}}` forming a compositional macro-action; non-Read ops execute sequentially on the memory set `M_t = {m_i}_{i=1}^{N_t}`.
- **Hybrid memory retrieval.** Each step the agent receives `o_t = (o_t^env, m_t^scr, M̂_t)`. A **deterministic scratchpad** entry `m_t^scr` is always returned (captures global task state, pivotal step-wise information). A **selective retrieval** path uses the previous step's textual query `q_{t-1}` to TopK-retrieve from a FAISS vector store seeded by Qwen3-embedding-0.6B. The Read operation does not mutate state; non-Read ops do.
- **End-to-end RL with GRPO.** Memory ops are tokens in the model's vocabulary, so optimizing the output sequence likelihood implicitly optimizes the memory policy. Training uses **Group Relative Policy Optimization** with no advantage normalization (Dr.GRPO variant), terminal task-level reward (EM for QA; LLM-as-a-judge for web), advantage `A_i = r_i − mean_j r_j` over a group of repeated rollouts, and a KL regularizer to the reference policy. Advantages are distributed uniformly across all output tokens including the memory ops. Base model is Qwen3-8B; chunk size 4K tokens; default K=6 retrieved entries.

## Experiment & Results

Evaluated on **3 long-context QA benchmarks** (HotpotQA, 2WikiMultihopQA, MuSiQue under a RULER-style needle-in-a-haystack setting; trained on 200 documents ≈ 28K tokens, tested on 800 documents ≈ 112K tokens with multi-question variants of 1–10 questions per task) and **2 multi-turn web benchmarks** (GAIA, WebWalkerQA; trained on Asearcher, up to 40 tool calls per task, Google search + Jina Reader). Baselines: Full Context, Vanilla RAG, Generative Agents, Mem0, A-Mem, and MemAgent (the strongest trained baseline, re-run with matched hyperparameters/seed).

Main results (Avg. column, same Qwen3-8B backbone): **AtomMem 58.8** beats MemAgent 56.7, A-Mem 51.4, Full Context 46.0, Vanilla RAG 42.2, Mem0 24.2. On 800-doc HotpotQA AtomMem reaches **72.9** (vs MemAgent 71.1); on 800-doc MuSiQue **48.5** (vs 44.5); on GAIA **37.4** (vs 33.0). AtomMem w/o RL averages 50.3, so RL contributes roughly **+8.5 points** end-to-end. The model also scales — trained at 200 docs, it maintains a lead at 800 docs (4× context extension).

**Ablations** (HotpotQA / 2WikiMQA / MuSiQue): removing `Update` drops 6.4 / 4.9 / 7.2 points; removing `Delete` drops only 1.3 / 0.2 / 0.9 (these QA tasks are information-accumulation, not conflict-resolution). Removing the scratchpad drops 6.0 / 11.2 / 9.1; removing external storage drops 8.6 / 8.1 / 11.2; removing **both** collapses by 52.2 / 40.4 / 43.0 — the two memory components are complementary, not substitutable. Variants trained from scratch (scratchpad-only, storage-only) never catch the joint design. Hyperparameter sweep: K=6 sufficient (K=3 hurts; K=12 marginal); chunk size 2048/4096/8192 all comparable. Embedding choice matters — Qwen3-embedding-0.6B gives 66.5 avg, Qwen3-embedding-8B 68.2 avg, random selection drops to 59.1.

**Training dynamics** (Section 4.3): early in training the agent over-uses Read and neglects memory maintenance; as training progresses Read frequency drops sharply while Create/Update/Delete rise — the agent discovers a *task-aligned* policy that maintains a compact, relevant memory. When the task condition changes (e.g., bounded memory budget), the operation frequencies shift differently, indicating the policy is genuinely task-conditioned rather than a single fixed strategy.

## Limitations

- **RL is compute-intensive.** Reaching convergence takes ~2–3 days on an 8-GPU cluster; scaling to even longer horizons or noisier task distributions is a real bottleneck.
- **Uniform advantage assignment is blunt.** Task-level rewards are evenly distributed across all output tokens including individual memory operations. A finer-grained credit-assignment scheme (per-memory-entry contribution to task success) would in principle be more accurate, but the paper deliberately defers this: developing a new RL algorithm for a single downstream task is judged premature, and accurately scoring per-entry value is non-trivial.
- **Delete looks weak under non-conflicting tasks.** Ablation shows Delete contributes little on the QA benchmarks because facts mostly accumulate without contradiction; under bounded-capacity / conflicting-facts settings the relative importance of Delete and Update changes (Appendix experiment), but the main benchmarks don't stress-test this regime.
- **Cross-task-type transfer not evaluated.** QA-trained agents and web-trained agents are evaluated separately; no result on whether a unified policy transfers across QA and web modalities.

## Open questions

- Can per-memory-entry credit assignment (vs uniform advantage) yield meaningfully better policies without bespoke RL algorithm design?
- How does the learned CRUD policy behave under bounded memory capacity? The appendix hints frequencies shift, but the structure of the new policy and any failure modes are not characterized.
- Does an AtomMem-style policy generalize across agent backbones (e.g., does the CRUD vocabulary transfer to a different base LLM without retraining), or is the learned policy backbone-coupled?
- Are atomic CRUD operations the *minimal* useful set, or is a smaller/larger atomic set (e.g., merge `Update` and `Delete`, or split `Read` into deterministic and selective sub-actions) more learnable?
- How does the approach interact with cross-task / lifelong memory (the paper deliberately resets `s_0^mem = ∅` per task, separating itself from Expel/Memp-style cross-task accumulation)?

## My take

The framing — memory management as a learnable POMDP over atomic operations rather than a hand-crafted workflow — is the right level of abstraction for this line of work, and the experiments are honest about which ablations matter (the catastrophic drop when removing *both* memory components, and the training-dynamics trajectories from Read-heavy to balanced CRUD usage, are the strongest evidence). The 8.5-point lift over the no-RL baseline shows that the gain really is in *learning the workflow*, not in the structural design alone. The pieces I'd most want to see followed up are (i) the bounded-capacity regime, where Delete should matter more and the policy should look qualitatively different, and (ii) cross-task / cross-backbone transfer, since "fix the workflow per task" risks reintroducing a different one-size-fits-all problem one rung up.

## Related

- [[atomic-crud-memory-operations]]
- [[learnable-dynamic-memory-management]]
- [[hybrid-memory-retrieval]]
- [[atommem]]
- [[yupeng-huo]]
- [[yankai-lin]]
