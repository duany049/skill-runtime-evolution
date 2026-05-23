---
title: "LEGOMem: Modular Procedural Memory for Multi-agent LLM Systems for Workflow Automation"
slug: legomem-modular-procedural-memory-multi-agent
arxiv: "2510.04851"
venue: AAMAS
year: 2026
tags:
  - procedural-memory
  - multi-agent-systems
  - llm-agents
  - workflow-automation
  - rag
importance: 4
date_added: 2026-05-18
source_type: tex
tldr: LEGOMem decomposes successful multi-agent trajectories into role-aware memory units (full-task plans for the orchestrator, fine-grained subtask traces for task agents) and shows on OfficeBench that orchestrator-side memory is the dominant lever, with subtask memory delivering additional gains for smaller agents.
contribution_type:
  - method
  - system
  - analysis
datasets:
  - OfficeBench
cited_by: []
---

## Problem & Context

Multi-agent LLM systems (e.g. Magentic-One, AutoGen, UFO) decompose complex productivity workflows — document editing, calendar scheduling, email handling — across an orchestrator and specialized tool-using task agents. Before this work, these systems were **stateless and transactional**: each task is solved from scratch and prior execution experience is discarded, so the same coordination and tool-use mistakes recur. Existing procedural-memory research for LLM agents — Synapse (`zheng2023synapse`) and Agent Workflow Memory / AWM (`wangAgentWorkflowMemory2024`) — targets **single-agent** settings: Synapse stores full successful trajectories as exemplars, AWM induces frequently used subtask sequences as reusable skills. Neither addresses the multi-agent decision surface of *where memory should sit* (orchestrator vs task agents) and *how it should be routed* across role-specialized agents. Episodic/semantic memory systems such as A-MEM, Mem0, MemoryBank focus on conversational histories rather than executable workflow knowledge. LEGOMem is positioned as the first procedural-memory framework explicitly designed for the orchestrator + task-agents architecture.

## Key idea

Procedural memory in multi-agent systems should be **modular and role-aware**: a successful trajectory is split into (i) a *full-task memory* (high-level plan + summarized execution trace) and (ii) one or more *subtask memories* (a single agent's localized tool-use steps and observations). At inference time, the orchestrator receives full-task memories to support task decomposition / delegation, while each task agent receives only the subtask memories relevant to its delegated subtask. This is implemented as a RAG layer over an existing multi-agent framework: a procedural memory bank indexed by semantic embeddings, queried by either the task description or the subtask description. The paper uses this modular split to systematically probe three questions — *placement* (orchestrator vs task agent), *retrieval granularity* (full-task vs subtask), and *team capability* (LLM-only, hybrid, SLM-only) — and finds that orchestrator-side memory is the dominant lever, with finer-grained subtask retrieval mattering most when task agents are smaller and global planning support is weak.

## Method

LEGOMem operates in two phases over a Magentic-One-style architecture with orchestrator $A_\text{orch}$, task agents $A = \{A_1, \dots, A_k\}$ (Word, Excel, Calendar, Email, System, OCR-PDF), and a sandboxed Docker environment $\mathcal{E}$.

**Offline memory construction.** Run the full LLM team without memory, filter for successful trajectories, and use an LLM curation prompt (Appendix A.1) to distill each trajectory into a structured JSON memory unit: `high_level_plan`, a list of `subtasks` (each with `agent`, `description`, `steps` in `<think>…</think><action>…</action>` form, `observations`), `final_answer`, and `reflections`. Full-task memories are stored in a global bank $\mathcal{M}$ indexed by $\phi(d)$ (OpenAI text-embedding-3-large) over the task description $d$; subtask memories are also indexed per agent by $\phi(d_\text{subtask})$ in per-agent banks $\mathcal{M}_{A_j}$.

**Memory-augmented inference.** Three variants differ in how subtask memory is routed:

- **Vanilla LEGOMem.** Retrieve top-$K$ full-task memories from $\mathcal{M}$ using $\phi(d_\text{new})$. The orchestrator gets the retrieved full memories; subtask memories are *statically extracted* from those same full memories and pinned to the corresponding agents before execution begins.
- **LEGOMem-Dynamic.** Orchestrator side unchanged. When the orchestrator generates subtask $s_t$ for agent $A_t$, embed $\phi(s_t)$ and *just-in-time* query $\mathcal{M}_{A_t}$ for the most similar subtask traces. This decouples subtask retrieval from the structure of the retrieved full memory.
- **LEGOMem-QueryRewrite.** Pre-plan retrieval up front: after pulling full-task memories, a query-rewriter LLM $\psi$ drafts a candidate plan $\pi'_\text{draft} = \{s'_1, \dots, s'_n\}$; each $s'_j$ embeds into $\phi(s'_j)$ and retrieves from the per-agent bank ahead of execution, avoiding repeated retrievals at runtime.

At each orchestration step the orchestrator (i) proposes $s_t$ given current state $\sigma_t$ and any retrieved memory, (ii) selects an agent, (iii) the agent executes via tool APIs and returns observation $o_t$ and summary $r_t$, (iv) the orchestrator updates $\sigma_{t+1}$ and re-plans if progress stalls. The framework supports vector store via FAISS (`douze2024faiss`); the prompt schema for curation and query rewriting is in the appendix.

## Experiment & Results

**Benchmark.** OfficeBench (`wang2024officebench`) — 300 multi-step office automation tasks across Level 1 (single app), Level 2 (two apps), Level 3 (multi-app workflows). Split: 148 train (memory curation) / 152 test. Success rate is computed programmatically against final environment state. Each datapoint is averaged over 3 random seeds.

**Team configurations.** LLM team (GPT-4o everywhere), Hybrid (GPT-4o orchestrator + GPT-4o-mini agents), SLM team (GPT-4o-mini everywhere). OCR uses Phi-3.5-mini. Memory: 5 entries for the orchestrator, 3 per task agent, drawn from 93 successful full-task memories and 250 derived subtask memories.

**Main results (overall success rate).** All three LEGOMem variants beat all baselines on every team:

- LLM team: No memory 45.83 → vanilla LEGOMem **58.44** (+12.6); Synapse 58.11, AWM 48.03.
- Hybrid team: No memory 35.31 → LEGOMem-QueryRewrite **50.22** (+12.7); Synapse 46.49, AWM 37.50.
- SLM team: No memory 24.78 → vanilla LEGOMem **38.16** (+13.4); Synapse 32.24, AWM 26.97.

A Hybrid team with QueryRewrite (50.22) exceeds a memory-less LLM team (45.83); a full SLM team with vanilla LEGOMem (38.16) exceeds a memory-less Hybrid team (35.31) — demonstrating that procedural memory can substitute for raw model capability on this benchmark.

**Placement ablation.** Removing orchestrator memory and keeping only task-agent memory drops overall success substantially: LLM team 58.44 → 49.78, Hybrid 48.03 → 35.31 (matches the no-memory baseline). Keeping only orchestrator memory retains most of the gain (LLM 53.29, Hybrid 47.59). Orchestrator-side memory is the dominant lever; task-agent memory adds execution-level precision on top.

**Retrieval granularity ablation.** In the task-agent-only setting the dynamic and query-rewrite variants outperform vanilla by 4–5 points on Hybrid teams, where smaller task agents benefit most from subtask-targeted retrieval.

**Reasoning ablation.** Augmenting memory units with explicit `reflections` reasoning shifts overall scores by ≤2 points either direction — the modular structure itself carries most of the procedural signal.

**Execution efficiency.** LEGOMem cuts average execution steps by up to 16.2% on Level 3 (26.5 → 22.2 for LLM team) and reduces per-step failure rate from 0.275 to 0.225 at Level 3.

## Limitations

- Evaluated on a single benchmark family (OfficeBench) and a single orchestrator architecture (Magentic-One-style central planner + tool-using task agents); generalization to peer-to-peer or decentralized multi-agent topologies is untested.
- Memory is curated *only from successful trajectories* — failed runs are discarded. The paper explicitly flags continual learning from failures as future work.
- Retrieval is purely embedding-similarity over the task or subtask description; no state-conditioned or hierarchical retrieval policy is learned.
- All experiments use OpenAI GPT-4o / GPT-4o-mini; whether the orchestrator-vs-agent placement finding holds across other model families is open.
- Memory bank size is fixed (93 full, 250 subtask) — scaling behavior to thousands of memories and the resulting retrieval-noise / specificity tradeoffs are not explored.

## Open questions

- Can procedural memory be learned *jointly* from successes and failures, with failure traces used to suppress wrong-tool-use patterns rather than discarded?
- How does the orchestrator-memory dominance result change when the orchestrator itself is a small model — does subtask memory become primary?
- Is there a principled way to choose retrieval granularity per task (full-task vs subtask) rather than fixing it at framework-design time?
- How does memory placement interact with explicit skill-library structures (hierarchical, compositional procedures) instead of flat banks?

## My take

LEGOMem's contribution is less a new memory algorithm and more a clean *axis of inquiry* for multi-agent procedural memory — the placement and routing question — backed by a reproducible OfficeBench evaluation. The headline empirical finding (orchestrator memory dominates, subtask memory matters when task agents are weak) is the kind of result that should constrain future multi-agent memory designs: pushing memory into task agents alone is the most-tempting and least-effective option. The framework is intentionally lightweight (RAG over curated trajectories, FAISS, no learned retrieval) which makes it a strong baseline rather than a frontier system; the natural next steps are learnable retrieval, failure-aware curation, and hierarchical memory composition.

## Related

- [[modular-procedural-memory]]
- [[legomem]]
- [[procedural-memory]]
- [[agentic-memory]]
- [[dongge-han]]
