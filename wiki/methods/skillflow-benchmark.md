---
name: SkillFlow Benchmark
slug: skillflow-benchmark
type: benchmark
tags:
  - agent-benchmark
  - lifelong-learning
  - skill-evolution
  - daef
  - docker-tasks
  - harbor-format
source_papers:
  - skillflow-benchmarking-lifelong-skill-discovery-evolution
parent_methods: []
child_methods: []
code_repo: "https://github.com/ZhangZi-a/SkillFlow"
date_updated: 2026-05-18
---

## Problem setting

Measure whether autonomous LLM agents can **discover** reusable skills from their own task-solving experience, **repair** them after failure, and **consolidate** them into a stable library across a sequence of tasks. Prior benchmarks evaluate skill *use* given pre-built skills, or measure marginal task gains from one-shot skill generation. SkillFlow targets the gap between those two: lifelong, sequential, library-level evaluation under a shared workflow.

## Mechanism

The benchmark consists of **20 task families and 166 runnable Docker tasks** in the Harbor format, spanning five domains (Finance & Economics, Operations & Supply Chain, Healthcare & Life Sciences, Governance & Strategy, Data & Document Intelligence). Each family is built around a **Domain-Agnostic Execution Flow (DAEF)** — a workflow skeleton with a controlled operation-type vocabulary and a fixed dependency structure. Within a family, tasks share the DAEF but differ in concrete grounding (files, entities, business semantics), forming a difficulty-ordered sequence of 8–9 tasks.

Evaluation proceeds under the **Agentic Lifelong Learning** protocol: the agent starts each family with an empty skill library, completes tasks in order, and after each task produces a **skill patch** (`summary`, `upsert_files`, `delete_paths`) that mutates the library. The library is reset between families. Metrics cover task success rate (verifier-judged), efficiency (turns, monetary cost, output tokens), and skill statistics (final library size and reuse rate of stored skills on later tasks).

## Procedure

**Construction pipeline (four stages).**

1. *Seed task collection and skill curation.* 64 seed tasks (18 from SkillsBench, 46 from GDPval) plus 2,318 open-source skills filtered from ~8,000 public skills.
2. *Task–skill pair matching.* Qwen3-embedding-4B retrieves 5–10 candidate skills per seed by semantic similarity, producing 64 task–skill reference pairs.
3. *Iterative task-family generalization.* A dual-agent loop (Architect: GPT-5.3-Codex inside Cursor; Critic: Claude Opus 4.6) constructs new tasks and verifies them in real Docker environments. Up to five revision rounds per family; failures are abandoned. A second expansion round adds tasks and establishes a difficulty gradient. Yields 8–9 Harbor-format tasks per family across 20 retained families.
4. *Human review.* Reviewers inspect each family for instruction leakage, logical soundness, environment correctness, and difficulty calibration. All 20 families pass after revision.

**Evaluation use.** Pair each model with its strongest practical harness (Claude Code, Codex CLI, Qwen-Coder, Kimi-CLI). Standardize only the patch-generation interface as single-turn output. Within each family, run vanilla (no skill evolution) and lifelong-skill-evolution conditions to compute the gain $\Delta$ for each (model, harness) setting.

## Assumptions

- Verifier-based task success is a sufficient proxy for skill utility — i.e. a skill that helps the agent pass the verifier is actually useful.
- The DAEF abstraction map $\phi$ is correctly identified by human annotators (Stage 1 of the construction protocol relies on independent annotators agreeing on meta-step extraction).
- A fixed within-family task order with monotonically increasing difficulty isolates the contribution of accumulated skills from the contribution of base-model capability.
- Skill-patch generation is decoupled enough from the agent harness that library quality is attributable to the model's native abstraction capability, not the harness's planning policy.

## Limitations

- Library reset between families means cross-family skill transfer is not measured.
- The 11 evaluated models are closed-weight commercial systems whose snapshots will age out.
- Tasks requiring open-network services or interactive UI are excluded by Docker/Harbor construction.
- Skill-use detection (reads/calls in execution traces) conflates "skill consulted" with "skill caused the action".
- The fixed patch prompt template introduces unmeasured prompt sensitivity.

## Tradeoff profile

The benchmark trades **breadth of workflow types** (only 20 DAEFs) for **depth of lifelong evaluation** (full 8–9 task sequences per family with verifier-rich rubrics and an auditable skill-patch history). It trades **open-ended environment realism** (no live web, no UI) for **reproducible verifier-based judgement** (Docker-isolated, Harbor-format). And it trades **maximum harness diversity** for **per-model strongest practical setting** (each model paired with its native CLI harness), which makes cross-model comparison fair within each (model, harness) pairing but slightly less so across pairings.
