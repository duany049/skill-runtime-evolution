---
name: AtomMem
slug: atommem
type: training
tags:
  - agentic-memory
  - reinforcement-learning
  - grpo
  - crud-operations
  - long-context
source_papers:
  - atommem-learnable-dynamic-agentic-memory-atomic
parent_methods: []
child_methods: []
code_repo: "https://github.com/RUCBM/AtomMem"
date_updated: 2026-05-18
---

## Overview

AtomMem is an end-to-end framework that trains an LLM agent to manage its own memory by RL over an atomic CRUD action vocabulary. The training pipeline turns memory management from a designer-specified pipeline into a learned policy on top of a Qwen3-8B base model.

## Mechanism

**Action vocabulary.** Memory ops are exposed as structured XML tags emitted by the model: `<create_memory>{content}</create_memory>`, `<read_memory>{query}</read_memory>`, `<update_memory>{memory id: content}</update_memory>`, `<delete_memory>memory id</delete_memory>`. At each step the policy emits a compositional macro-action `A_t = {a_t^1, …, a_t^{K_t}}`; non-Read ops execute sequentially on the memory set.

**Memory substrate.** A FAISS vector database stores entries `M_t = {m_i}_{i=1}^{N_t}`. Hybrid retrieval combines a deterministic scratchpad entry with TopK semantic retrieval (default K=6) using Qwen3-embedding-0.6B. Long inputs are chunked into 4K-token segments and fed step-by-step.

**Optimization.** Group Relative Policy Optimization (GRPO) with no advantage normalization (Dr.GRPO variant). For each task, multiple rollouts form a group `G`; the advantage of trajectory `i` is `A_i = r_i − (1/|G|) Σ_j r_j`. The objective is `J(θ) = E[(1/G) Σ_i ρ_θ^i A_i − β · KL(π_θ || π_ref)]`, with `ρ_θ^i` the importance sampling ratio. Advantages are distributed uniformly over all output tokens including memory ops. Fully on-policy: each rollout used for a single update.

**Rewards.** Terminal-only. For QA tasks: exact match between model answer and ground truth. For web tasks: LLM-as-a-judge. Per-task memory reset: `s_0^mem = ∅`.

## Strengths

- Decouples *workflow design* from *workflow execution* — the workflow is learned, not engineered.
- Strong empirical lift: ~8.5 points over no-RL baseline, beats MemAgent on average under matched backbone/seed.
- Robust to chunk size (2048/4096/8192 comparable); scales from 200-doc training context to 800-doc evaluation.
- Memory components are individually substitutable (graceful degradation when scratchpad or storage is removed individually).

## Limitations

- Convergence requires ~2–3 days on an 8-GPU cluster; compute footprint is high.
- Uniform advantage assignment across all output tokens is a known coarse approximation; per-memory-entry credit assignment is left for future work.
- Delete operation appears underused on the evaluated QA benchmarks because tasks are information-accumulation with non-conflicting facts; benefit shifts under bounded-capacity regimes.
- Per-task memory reset means cross-task / lifelong memory accumulation is out of scope.

## Reproduction notes

- Base model: Qwen3-8B; embedding: Qwen3-embedding-0.6B (Qwen3-embedding-8B gives ~1.7 avg point boost).
- Vector store: FAISS; default chunk size 4096, retrieve K=6.
- Web tasks: max 40 tool calls per task; tools are Google search + Jina URL Reader.
- Training data: HotpotQA / 2WikiMultihopQA / MuSiQue (QA) and Asearcher (web). Evaluation: same QA datasets + GAIA + WebWalkerQA, with 200-doc training / 800-doc eval RULER-style long-context augmentation and 1–10 questions per task.
