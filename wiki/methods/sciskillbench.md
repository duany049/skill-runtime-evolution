---
name: SciSkillBench
slug: sciskillbench
type: benchmark
tags:
  - benchmark
  - materials-science
  - chemistry
  - llm-agent
  - tool-use
  - skill-evolution
  - scientific-discovery
source_papers:
  - cascade-cumulative-agentic-skill-creation-through
parent_methods: []
child_methods: []
code_repo: "https://doi.org/10.6084/m9.figshare.30924998"
date_updated: 2026-05-18
---

## Problem setting

SciSkillBench evaluates whether an LLM-based agent system can autonomously chain diverse external tools (databases, scientific packages, simulation software) to solve real materials-science and chemistry research tasks. Unlike code-only benchmarks (HumanEval, SWE-Bench), tasks here demand interacting with newly-released or undocumented libraries, navigating misleading documentation, and combining multiple functions within a single tool. Unlike one-shot tool-use benchmarks, tasks vary in autonomy level (whether the user specifies key procedural steps) and difficulty (P-value-based), so the benchmark stresses both knowledge breadth and adaptive problem-solving.

## Mechanism

The benchmark is a curated set of **116 tasks** split across two principal categories and six subcategories:

- **Data-oriented (76 tasks):** 22 data-retrieval (Materials Project, ICSD, Matminer, MPContribs, MDF, OQMD, OBELiX, GMAE), 24 data-analysis (pymatgen, Matminer, SMACT, Materials Project, OBELiX), 4 data-management (pymatgen-db, MongoDB), 26 data-processing (RDKit, Matminer, Magpie, Robocrystallographer, pymatgen, enumlib, spglib).
- **Computation-oriented (40 tasks):** 32 simulation (xTB, ORCA, ASE, LAMMPS, CHGNet), 8 specialized models/toolkits (CHGNet, MACE, mlip).

Each task has two stratification axes:

- **Autonomy Level.** Level 0 (58 tasks): user specifies key functions or core procedural components. Level 1 (58 tasks): only the high-level objective is given; greater autonomy is required. Level 0 queries resemble computational-scientist prompts; Level 1 resemble experimental-scientist prompts.
- **Difficulty (post-hoc).** P-value computed across 28 model configurations on Level-1 questions using pass@3 as the per-model "did this model solve it" signal. P ≥ 0.5 → Simple (37 questions); P < 0.5 → Difficult (21 questions).

Each benchmark task ships with a required output format, unit, and a tolerance threshold. Evaluation is outcome-based: the agent's processed answer is compared to ground truth within the tolerance.

## Procedure

For each system configuration and each of the 116 tasks:

1. Run **3 independent repetitions** in separate Python subprocesses, with the Supabase code-extraction cache cleared and temp files removed between runs, and memory-based tools disabled across all benchmark systems.
2. For each repetition: invoke the agent on the task query (specifying required output format and unit), capture the processed final answer.
3. Compute success per attempt against ground truth within the task's tolerance band. Stringent tolerance for deterministic operations (e.g., data retrieval); broader tolerance for inherently variable simulations.
4. Report **Success Rate** (overall accuracy across all attempts) and **pass@k** for k=1,2,3 ("at least one of first k attempts correct").
5. For Level-0 / Level-1 breakdown and difficulty breakdown, repeat the aggregation within the relevant subset.
6. Exclude rare workflow failures (e.g., invalid-JSON parse errors) from the denominator.

Total reported experiments: **16,008** = 116 tasks × 3 reps × 46 system configurations.

## Assumptions

- Tasks have a well-defined ground-truth answer that admits a tolerance band. (Excludes tasks where the answer is a workflow trace, a plot, or a qualitative argument — those are addressed in CASCADE's real-world demos, not in SciSkillBench.)
- Memory-disabled comparisons cleanly isolate the contribution of inference-time meta-skills (continuous learning, self-reflection) from accumulated memory.
- The 28 models used for P-value computation are representative enough that their average solve rate ranks task difficulty reliably.
- Outcome-based grading is acceptable: it does not verify intermediate steps but does verify the user-relevant deliverable.

## Limitations

- **Domain-bounded.** Materials science + chemistry only. Generalization claims to software engineering, biology, etc., rest on CASCADE's domain-agnostic design rather than benchmark coverage.
- **Outcome-only grading.** Cannot distinguish a correct answer reached by the right reasoning from a correct answer reached by accident.
- **P-value calibration sensitivity.** Difficulty bucketing depends on the 28-model panel; a future panel could relabel some tasks.
- **English-only / static.** Task wording is fixed; robustness to prompt variation is not measured.
- **No human-in-the-loop track.** Multi-turn collaboration scenarios are demonstrated outside SciSkillBench (real-world demos), so the benchmark does not measure that mode.
- **Closed-model drift.** Pass@k for closed models (GPT-5, Claude-Sonnet) is tied to a specific test-time snapshot; reproducibility weakens over time.

## Tradeoff profile

Compared to general-purpose code benchmarks (HumanEval, MBPP): SciSkillBench trades clean unit-test grading for **realistic, multi-tool scientific tasks** where the difficulty comes from tool discovery and chaining rather than algorithmic complexity. Compared to ScienceAgentBench and similar agentic benchmarks: SciSkillBench specifically emphasizes (a) breadth across six skill categories within a single domain, (b) explicit autonomy-level stratification, and (c) P-value-based difficulty stratification — at the cost of a narrower domain and no qualitative grading. Compared to closed enterprise evals: the benchmark is openly hosted on Figshare and provides a reusable evaluation harness, trading task-distribution coverage for community accessibility.
