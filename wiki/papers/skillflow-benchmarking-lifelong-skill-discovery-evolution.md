---
title: "SkillFlow: Benchmarking Lifelong Skill Discovery and Evolution for Autonomous Agents"
slug: skillflow-benchmarking-lifelong-skill-discovery-evolution
arxiv: "2604.17308"
venue: arXiv
year: 2026
tags:
  - skill-evolution
  - lifelong-learning
  - agent-benchmark
  - procedural-memory
  - skill-library
  - daef
  - skill-patch
importance: 4
date_added: 2026-05-18
source_type: tex
s2_id: 6158fe73b7cb734c2561ae4b28f37f6ccdf54929
tldr: "A 166-task / 20-family benchmark and Agentic Lifelong Learning protocol that measures whether autonomous agents can discover, patch, and consolidate reusable skills over a sequence of tasks sharing a Domain-Agnostic Execution Flow (DAEF)."
contribution_type:
  - benchmark
  - analysis
  - protocol
datasets:
  - SkillFlow
  - GDPval
  - SkillsBench
code_url: "https://github.com/ZhangZi-a/SkillFlow"
cited_by: []
---

## Problem & Context

Frontier LLM agents — Claude Code, Gemini CLI, Codex CLI, Qwen-Coder — are increasingly deployed as autonomous command-line operators with native support for **plug-and-play external skills**: packaged procedural knowledge (`SKILL.md` files, helper scripts, usage rubrics) that augment a base model on specialized tasks. Prior work establishes two flanking facts. Skills clearly help: [[skillsbench]]-style evaluations show large gains when skills are *provided*. And skills can be *generated* from experience: SkillWeaver, SkillRL, MemSkill, Memento, and related systems all demonstrate that experience-derived skills improve downstream performance.

What sits between those two flanks is the central question of this paper: **can an autonomous agent extract reusable skills from its own task-solving experience, repair them after failure, and maintain an evolving skill library across a sequence of tasks?** Earlier benchmarks evaluate skill *use* but assume the skills already exist; earlier skill-discovery systems evaluate marginal task gains but rarely measure library-level evolution (compactness, repair, transfer across siblings of the same workflow). No prior benchmark systematically holds the workflow constant while letting the agent grow its own library — leaving the lifelong-skill question empirically underspecified.

## Key idea

Construct a benchmark in which each task family shares a **Domain-Agnostic Execution Flow (DAEF)** — a workflow skeleton with the same operation types and dependency structure but different domain-specific grounding — and evaluate agents under an **Agentic Lifelong Learning** protocol where an agent starts with an empty skill library, solves the family's tasks in a fixed difficulty order, and updates its library through explicit **skill patches** derived from execution traces and rubric feedback. The shared DAEF means a skill abstracted on task $T_1$ is *meant* to apply to $T_2 \ldots T_n$ in the same family; the protocol thus measures lifelong learning at the workflow level rather than at the surface lexical level.

## Method

Formally, a domain-grounded workflow is $\mathcal{T} = (V, E, \lambda, \gamma)$ where $V$ is sub-goals, $E$ is precedence edges, $\lambda$ assigns each node a domain-agnostic operation type (drawn from a controlled single-word vocabulary: `read`, `retrieve`, `compute`, `detect`, `output`, …), and $\gamma$ provides task-specific grounding. The DAEF is the abstraction $\mathcal{F} = \phi(\mathcal{T}) = (V_F, E_F, \lambda_F)$ obtained by stripping $\gamma$ — keeping the workflow skeleton but removing concrete files, entities, and business semantics.

**Benchmark construction (four-step pipeline).**
1. *Seed task collection and skill curation* — 64 seed tasks (18 from SkillsBench, 46 from GDPval) plus ~8,000 open-source skills filtered down to 2,318 for matching.
2. *Task–skill pair matching* — Qwen3-embedding-4B retrieves 5–10 candidate skills per seed by semantic similarity.
3. *Iterative task-family generalization* — a dual-agent loop runs inside Cursor: an **Architect Agent** (GPT-5.3-Codex) proposes new tasks conditioned on the target DAEF and the seed reference package; a **Critic Agent** (Claude Opus 4.6) verifies the tasks in real Docker environments, evaluating workflow consistency, difficulty gradient, solvability, verifier correctness, and environment reliability. Up to five revision rounds; families that fail to stabilize are dropped. After acceptance, a second expansion round adds more tasks and establishes a difficulty gradient, yielding 8–9 Harbor-format tasks per family.
4. *Human review* — reviewers inspect each family for instruction leakage, logical soundness, environment correctness, and difficulty calibration. All 20 families pass.

**Agentic Lifelong Learning protocol.** Let $\mathcal{F} = \{T_1, \ldots, T_n\}$ be the ordered family. The agent maintains a skill library $\mathcal{S}_t$ and produces an execution trace $\tau_t$ plus rubric $r_t$ per task. After each task it emits a **skill patch** under a fixed prompt template $g$:
$$\Delta_t = \text{Model}_g(\mathcal{S}_{t-1}, \tau_t, r_t), \quad \mathcal{S}_t = \text{Apply}(\Delta_t, \mathcal{S}_{t-1}).$$
The patch is a minimal three-field record — `summary`, `upsert_files`, `delete_paths` — sufficient to add, revise, or delete `SKILL.md` files and helper scripts. Patch generation uses the model's native capability (not the surrounding harness) to avoid confounding skill quality with instruction-following ability. Each family resets the library, so the protocol isolates within-workflow lifelong learning from cross-workflow skill retrieval.

**Metrics.** Task success rate (verifier-based), efficiency (turns, monetary cost, output tokens), and skill statistics (final library size and reuse rate of stored skills on later tasks).

## Experiment & Results

11 model variants evaluated across four native harnesses (Claude Code, Codex CLI, Qwen-Coder, Kimi-CLI): Claude Sonnet 4.5 / Opus 4.5 / Sonnet 4.6 / Opus 4.6, MiniMax M2.5 / M2.7, GPT 5.4, GPT 5.3 Codex, Qwen-Coder-Next, Qwen3-Coder-480B, Kimi K2.5.

**Headline numbers (vanilla → lifelong skill evolution, task success rate, %):**

| Model | Vanilla | +Skill Evo | Δ |
|---|---|---|---|
| Claude Opus 4.6 | 62.65 | **71.08** | **+8.43** |
| MiniMax M2.5 | 28.31 | 34.94 | +6.63 |
| Claude Sonnet 4.5 | 49.40 | 55.42 | +6.02 |
| GPT 5.4 | 33.13 | 36.75 | +3.62 |
| Claude Opus 4.5 | 58.43 | 60.84 | +2.41 |
| Kimi K2.5 | 55.42 | 56.02 | +0.60 (66.87% skill-use rate) |
| Claude Sonnet 4.6 | 56.63 | 56.63 | 0.00 |
| MiniMax M2.7 | 37.35 | 36.75 | -0.60 |
| Qwen3-Coder-480B | 24.70 | 24.10 | -0.60 |
| Qwen-Coder-Next | 45.18 | 44.58 | -0.60 |
| GPT 5.3 Codex | 52.41 | 46.39 | **-6.02** |

A control experiment on Claude Opus 4.6 that simply prepends the full prior interaction history as context reaches only 51.04% — below both vanilla (62.65%) and the lifelong protocol (71.08%) — confirming the gain comes from structured skill consolidation, not raw context length.

**Six findings.** (1) Opus 4.6 is the only configuration approaching stable library-level improvement (compact library, repaired skills reused after failures). (2) Once an incorrect skill enters the library, later tasks systematically inherit the flawed abstraction — local errors become sequence-level patterns. (3) Stronger settings end with **smaller** final libraries; consolidation beats proliferation. (4) Qwen variants and some MiniMax variants fail by **skill inflation** — accumulated skill count grows almost monotonically with task index, but completion does not. (5) Codex consolidates variants into evolving core skills well, but the compact library does not translate to stronger end-to-end completion. (6) The decisive capability gap is **skill repair**, not skill writing — most models can write some skill, but few can recognize and fix a bad one.

Gains are broadly distributed across domains (Finance & Economics, Operations & Supply Chain, Healthcare & Life Sciences, Governance & Strategy, Data & Document Intelligence) rather than concentrated in any one category, with Data & Document Intelligence showing more positive transfer and Finance & Economics more negative gains.

## Limitations

- Each family resets the library — by design, cross-workflow skill retrieval is not measured. Long-tail generalization across heterogeneous workflows remains untested.
- Closed-model evaluation is sensitive to version drift; the snapshot of 11 model variants will age out as vendors push silent updates.
- All execution happens inside Docker via Harbor; tasks requiring open-network services or interactive UI are excluded by construction.
- Patch generation uses a single fixed prompt template; prompt sensitivity of the skill-patch interface is unexplored.
- Skill-use detection is via execution-trace events (reads / calls), which conflates "agent consulted skill" with "skill caused the action".

## Open questions

- What is the right unit for a *cross-family* skill? DAEF abstracts within a workflow; nothing in this benchmark abstracts across workflows.
- Can a model's skill-repair capability be induced by training signals (RL on patch quality, supervised data of bad-skill → good-skill diffs), or does it co-emerge with general reasoning capacity?
- What policy minimizes skill inflation without losing useful specialization? Patch-driven `delete_paths` is available but not learned.
- How robust is the **same workflow ⇒ same skill** assumption when domain grounding $\gamma$ is far from any seed in training distribution?
- Does the per-family library-size sweet spot scale with task difficulty, model size, or harness?

## My take

SkillFlow is the most concrete operationalization to date of the "lifelong skill" question that the agentic-memory literature has been circling. The DAEF abstraction is the right lever: by holding the workflow skeleton constant, it converts a fuzzy claim ("agents accumulate experience") into a measurable one ("can the same workflow scaffold absorb increasing difficulty"). Finding 6 — repair, not writing, is the bottleneck — feels under-appreciated in the broader memory-augmented-agent discourse, where most papers still report skill *generation* as the headline. The protocol's intentional within-family reset is honest about scope but leaves the harder cross-family generalization question open for follow-up work.

The negative results — Qwen's skill inflation, GPT 5.3 Codex's regression, Kimi K2.5's high-use / low-gain pattern — are arguably more informative than the Opus 4.6 win, because they isolate failure modes that any future skill-library design has to defend against.

## Related

- Topic: [[skill-evolution]]
- Topic: [[continual-learning-evaluation]]
- Topic: [[procedural-memory]]
- Topic: [[agentic-memory]]
- Concept: [[domain-agnostic-execution-flow]]
- Concept: [[agentic-lifelong-learning]]
- Concept: [[skill-patch]]
- Method: [[skillflow-benchmark]]
- Author: [[ziao-zhang]]
- Author: [[feng-zhao]]
