---
title: "SkillLearnBench: Benchmarking Continual Learning Methods for Agent Skill Generation on Real-World Tasks"
slug: skilllearnbench-benchmarking-continual-learning-methods-agent
arxiv: "2604.20087"
venue: ""
year: 2026
tags:
  - agent-skills
  - continual-learning
  - benchmark
  - skill-generation
  - llm-agents
  - evaluation
importance: 4
date_added: 2026-05-18
source_type: tex
s2_id: f73d5cb01fb23264ba211fb6305586ed7b31d61c
tldr: "First benchmark for evaluating continual skill learning methods, with 20 verified skill-dependent tasks across 15 sub-domains and a three-level evaluation framework spanning skill quality, execution trajectory, and task outcome."
contribution_type:
  - benchmark
  - analysis
datasets:
  - SkillLearnBench
code_url: "https://github.com/cxcscmu/SkillLearnBench"
cited_by: []
---

## Problem & Context

Agent skills — structured documents that encode task-specific instructions, workflows, and domain knowledge — have become the de facto interface for adapting LLM agents to specialized tasks. Skills have rapidly emerged as an open standard adopted across Claude, Cursor, GitHub, OpenAI Codex, and similar platforms, with growing community-curated skill libraries. Yet for agents that continuously face novel tasks, hand-authored pre-built skills inevitably fall short: the agent must be able to *generate new skills from its own working experience* and add them to a library that grows over time. This generate-store-reuse cycle constitutes a form of continual learning that updates an external skill store rather than model weights.

Before this paper, multiple continual learning methods for skill generation had been proposed independently — one-shot generation, self-feedback refinement, teacher-feedback guidance, and structured pipelines such as Anthropic's Skill Creator — but each was evaluated in its own setting on its own tasks. No principled benchmark existed to compare them under controlled conditions, to disentangle whether failures came from the skill itself or from execution behavior, or to test whether generated skills are *reusable* across instances rather than overfit to a single seed. Prior skill benchmarks (SkillsBench, LangChain's evaluation, Tessl) measured only binary task completion with vs. without a skill, treating the skill as an evaluation object rather than the generation method as the primary target.

## Key idea

Evaluate continual *skill generation methods* (not just skills) as the primary target, on a benchmark of *skill-dependent, multi-instance, verifiable tasks*, through a three-level pipeline that scores (1) the generated skill specification as a textual artifact, (2) the agent's execution trajectory under that skill, and (3) the final task outcome. Separating these levels diagnoses *where* methods fail — bad skill content, low skill adoption, or execution drift — which a pure pass/fail outcome metric cannot.

## Method

The paper contributes three artifacts wired together:

1. **The SkillLearnBench task collection.** 20 tasks across 15 sub-domains drawn from a community-driven taxonomy of real-world skill usage (Ling et al. 2026), spanning six categories: Software Engineering, Information Retrieval, Productivity Tools, Data & Analytics, Content & Creative, and Utilities & Other. 17 tasks are adapted from SkillsBench; 3 (`schedule-planning`, `anthropic-poster-design`, `chinese-poem-generator`) are newly designed to fill missing categories. Each task `t_i = (x_i, v_i, S_i, Q_i)` ships with a natural-language description, a deterministic verifier `v_i`, a human-authored reference skill set `S_i`, and a set of query instances `Q_i` with varying parameters that share the same core task structure. 100 instances total.

2. **Skill-dependency verification.** Every task must (a) be unsolvable without a skill — the no-skill agent's pass rate over R=10 runs must be ≤ α=0.5 on every instance — and (b) solvable with the human-authored skill in at least one of those runs. This filters out tasks the base LLM can already do, ensuring the benchmark isolates skill contribution.

3. **Three-level evaluation framework.**
   - **Level 1: Skill Quality** (textual analysis, no execution). An LLM judge scores Coverage (fraction of reference key points from the human skill recovered by the generated skill), Executability (mean of completeness / determinism / consistency / usability), and Safety (mean of six risk dimensions: data/privacy, prompt injection, illegal content, bias, system integrity, untrusted communication).
   - **Level 2: Trajectory.** Trajectory alignment (mean of key-point recall, execution order, completeness against an oracle trajectory) and Skill usage rate (fraction of generated skills actually invoked at execution time).
   - **Level 3: Outcome.** Task accuracy (mean of `v_i` over all instances) and Solving efficiency (total token consumption).

The benchmark then evaluates four continual-learning methods under a fixed solving agent (Claude Sonnet 4.6, temperature 0, 100-turn budget, containerized sandbox): **One-Shot** (single-pass generation, baseline from SkillsBench), **Self Feedback** (K=2 self-revision loop, adapted from SkillRL), **Teacher Feedback** (K=3 with up to K−1 expert-QA rounds, based on LOTBench), and **Skill Creator** (multi-stage analyze/investigate/write/validate pipeline from Anthropic). Each method generates skills with six LLM backbones — Claude Haiku 4.5, Sonnet 4.6, Opus 4.6, Gemini 3.1 Flash Lite, Gemini 3 Flash, Gemini 3.1 Pro. Level 1 and Level 2 LLM-judge metrics use GPT-5-mini.

## Experiment & Results

The main result table reveals a sobering picture: all four methods improve over the no-skill baseline (10.17% accuracy) but all fall well short of the human-authored ceiling (74.50%). Across all LLMs averaged:

- **Self Feedback** averages 31.08% accuracy at the lowest token cost (390K tokens) — best on outcome.
- **One-Shot** averages 30.44% at 461K tokens — close behind Self Feedback.
- **Teacher Feedback** averages 27.47% at 528K tokens — most expensive, mid-pack on accuracy.
- **Skill Creator** averages 27.33% at 406K tokens but achieves the highest skill usage rate (84.47%) and highest coverage (41.12%).
- The **best method covers only ~45% of the gap** between no-skill and human-authored performance.

Critical secondary findings:
- **Method rankings flip across LLMs.** No method dominates across all backbones — Teacher Feedback wins on Claude Haiku 4.5 (34.0%) but loses on Gemini 3.1 Pro (17.83%).
- **Stronger LLMs do not reliably produce better skills.** Claude Opus 4.6 generally does worse than Sonnet 4.6 as the skill generator; method-LLM interaction effects dominate.
- **Open-ended tasks can be *hurt* by learned skills.** Tasks with clear reusable workflows benefit; open-ended generation tasks (e.g., poem generation, poster design) sometimes degrade when a rigid skill overconstrains the agent.
- **Self-feedback drifts; external feedback compounds.** Iterating Self Feedback for more rounds fails to improve accuracy (recursive drift), while Teacher Feedback's external signal yields genuine round-over-round improvement.
- **Adoption gap.** Teacher Feedback produces the lowest skill usage rate (60.20% averaged), meaning the agent often ignores the generated skill at execution time — content quality alone is insufficient; the skill must also be adoptable.

## Limitations

- The solving agent is fixed to Claude Sonnet 4.6, so trajectory-level results are conditioned on one backbone's preferences for invoking skills. Cross-solver generalization is not tested.
- LLM-judge metrics (Coverage, Executability, Safety, Trajectory alignment) use a single model (GPT-5-mini); judge bias and reproducibility across judges are not measured.
- Only four continual-learning methods are evaluated — recent approaches such as EvoSkill, ProcMEM, and SkillWeaver appear in related work but are not included in the controlled comparison.
- The 20 tasks, though grounded in a community taxonomy, are a small sample of the long tail of real-world skill demands; the taxonomy's six categories may not survey the full distribution.
- The benchmark generates a skill *once per task* and tests reusability across instances of that task, but does not evaluate cross-task transfer (skill learned on task A applied to task B).

## Open questions

- What skill *representations* (procedural, declarative, exemplar-based, mixed) most reliably yield high adoption by the solving agent? Skill Creator wins usage rate but loses outcome — what gap does that imply?
- Can the recursive-drift failure mode of self-feedback be repaired by injecting weak external signals (e.g., automated verifier output) without a full teacher?
- For open-ended tasks where rigid skills hurt, is the right move softer skills (guidelines instead of procedures), or learning *when not to invoke a skill*?
- How does evaluating *generation methods* rather than *skills* scale to long-horizon continual settings where the library grows over hundreds of tasks?

## My take

This is the field's first principled apparatus for comparing skill-generation methods, and the headline results are an unflattering mirror: every method beats no-skill, none approaches human-authored, and the gap is large enough that "more compute / bigger backbone" does not paper it over. The three-level decomposition is the contribution that will outlast the specific methods — it operationalizes the intuition that *content quality is not enough; the agent must adopt the skill* and provides a metric (skill usage rate) that makes the adoption gap visible. Pairing accuracy with a usage-rate metric is the kind of methodological move that should propagate to other procedural-memory benchmarks.

The negative result about self-feedback recursive drift vs. external-feedback compounding is the most actionable single takeaway for downstream method designers: cheap self-revision loops will plateau; the path forward needs external grounding.

## Related

- [[agent-skill]] (concept introduced by Anthropic, central to this paper's setup)
- [[continual-skill-learning]] (concept the paper extensively uses and tests)
- [[skill-dependent-task]] (concept this paper introduces to define benchmark task validity)
- [[skilllearnbench]] (the benchmark method this paper introduces)
- [[chenyan-xiong]] (senior author)
- [[continual-learning-evaluation]] · [[skill-evolution]] · [[procedural-memory]] (related topics)
