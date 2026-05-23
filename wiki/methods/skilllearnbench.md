---
name: SkillLearnBench
slug: skilllearnbench
type: benchmark
tags:
  - agent-skills
  - continual-learning
  - benchmark
  - skill-generation
  - llm-evaluation
source_papers:
  - skilllearnbench-benchmarking-continual-learning-methods-agent
parent_methods: []
child_methods: []
code_repo: "https://github.com/cxcscmu/SkillLearnBench"
date_updated: 2026-05-18
---

## Problem setting

Evaluate methods that generate agent skills from task descriptions and execution experience, under controlled conditions that isolate (a) the quality of the generated skill specification, (b) the agent's execution behavior when using that skill, and (c) the resulting task outcome. The benchmark targets *generation methods* as the primary unit of evaluation, not individual skills.

## Mechanism

SkillLearnBench is a closed benchmark package consisting of:

- **20 tasks across 15 sub-domains and 6 categories** drawn from a community-driven taxonomy of real-world skill usage. Each task ships with a natural-language description, a deterministic verifier, a human-authored reference skill set, and a set of varying-parameter instances (100 instances total).
- **A skill-dependency filter** (see [[skill-dependent-task]]) that every task must pass: unsolvable without a skill at threshold α=0.5 over R=10 runs, and solvable with the human-authored skill in at least one of those runs.
- **A three-level evaluation framework** scored by a fixed solving agent (Claude Sonnet 4.6, temperature 0, 100-turn budget, containerized sandbox) plus an LLM judge (GPT-5-mini).
  - Level 1 — Skill Quality: Coverage, Executability (mean of completeness / determinism / consistency / usability), Safety (mean over six risk dimensions).
  - Level 2 — Trajectory: alignment (key-point recall × execution order × completeness) and skill usage rate.
  - Level 3 — Outcome: task accuracy and solving efficiency (token count).
- **Four reference continual-learning baselines**: One-Shot, Self Feedback (K=2), Teacher Feedback (K=3), and Skill Creator.

## Procedure

To evaluate a continual-learning method `m`:

1. For each task `t_i`, supply the method with `t_i`'s description `x_i` and one seed instance from `Q_i`.
2. The method produces a generated skill set `\hat{S}_i = m(x_i, seed)`. Skill generation happens *once per task*, then is reused across all instances of that task (testing reusability).
3. Run the fixed solving agent on every `q ∈ Q_i` with `\hat{S}_i` injected. Record the execution trajectory and the verifier's pass/fail outcome.
4. Score Level 1 directly on `\hat{S}_i` (no execution).
5. Score Level 2 on the execution trajectory against an oracle trajectory derived from the human-authored skill.
6. Score Level 3 as the average pass rate and total tokens used.

Compare methods across the seven metrics and across LLM backbones; reproducibility requires reporting the LLM family and version used for both generation and solving.

## Assumptions

- The fixed solving agent (Claude Sonnet 4.6) is representative of contemporary deployment-grade LLM agents; results may not transfer to weaker or substantially different backbones.
- The LLM judge (GPT-5-mini) gives a reliable enough signal on textual Level 1 and Level 2 scores; the paper does not measure inter-judge agreement.
- The community taxonomy from Ling et al. 2026 is a reasonable surrogate for the long tail of real-world skill demands.
- Human-authored skills, while imperfect, define a meaningful upper bound for what generated skills should aspire to.

## Limitations

- Single solver (fixed at Claude Sonnet 4.6) means trajectory-level results are conditioned on one backbone's preferences for invoking skills.
- Single LLM judge; judge bias is not measured.
- Small sample (20 tasks, 100 instances) relative to the diversity of real-world agent work.
- No cross-task transfer evaluation — the benchmark tests reusability *across instances of one task*, not *across tasks*.
- Closed set of reference methods; new methods (EvoSkill, ProcMEM, SkillWeaver) are noted in related work but not evaluated.

## Tradeoff profile

- **Strength.** First principled comparison framework for skill-generation methods; the three-level decomposition lets researchers diagnose *why* a method fails (bad content vs. low adoption vs. execution drift).
- **Strength.** Verifier-driven outcome scoring removes subjective grading from Level 3, the most consequential metric.
- **Cost.** Running the full benchmark across 6 LLM backbones × 4 methods × 100 instances is expensive (the paper reports per-method token costs in the 300K–700K range, plus judge calls).
- **Cost.** Adding a new method requires implementing the generate-then-fix-solver protocol; the benchmark is not plug-and-play with arbitrary online learners.
- **Coverage.** Six categories cover much of the agent-skill demand surface but not exhaustively; cross-domain transfer is not measured.
