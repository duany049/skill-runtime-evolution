---
name: Query-Specific Skill Refinement
slug: query-specific-skill-refinement
type: inference
tags:
  - skill-refinement
  - test-time-adaptation
  - skill-synthesis
  - agentic-skills
source_papers:
  - how-well-agentic-skills-work-wild
parent_methods: []
child_methods: []
code_repo: "https://github.com/UCSB-NLP-Chang/Skill-Usage"
date_updated: 2026-05-18
---

## Problem setting

When agents retrieve agentic skills from a large noisy collection, the retrieved skills often only partially align with the target task: useful information is scattered across multiple skills, formatted inconsistently, and mixed with irrelevant content. Loading all retrieved skills wastes context and can mislead the agent. Loading none gives up the upside. Static reranking does not help because relevance at the snippet level depends on the task. *Query-specific skill refinement* lets the agent transform the retrieved set into a single tailored skill at inference time, conditioned on the specific task.

## Mechanism

The agent acts as an inference-time skill author. After standard retrieval returns top-k candidates, the agent: (1) reads the task instructions, (2) examines every retrieved skill in full, (3) attempts an initial solution as exploration, (4) self-evaluates correctness without access to a ground-truth verifier, (5) reflects on which retrieved skills contributed and which misled, (6) writes a single new skill that merges the useful portions, discards the noise, and reformulates content to match the task's specific requirements. The agent has access to a meta-skill (Anthropic's `skill-creator`) that encodes best practices for skill authoring. The output is then loaded as the agent's skill input for the actual task attempt.

## Procedure

1. Run [[skill-retrieval]] to obtain top-k candidate skills (k=5 in [[how-well-agentic-skills-work-wild]]).
2. Provide the agent with the task description, all k retrieved skills, and the `skill-creator` meta-skill.
3. Agent attempts initial solution → self-evaluates → identifies useful vs. misleading skills (no ground-truth verifier).
4. Agent composes a refined `SKILL.md` that synthesizes across retrieved candidates.
5. Replace the retrieved skill set with the single refined skill in the agent's environment.
6. Agent retries the task with the refined skill as its only skill input.

Example from `Terminal-Bench 2.0` tensor parallelism task: retrieved skills `torch-tensor-parallel` (weight sharding, missing differentiable collectives) and `pytorch-research` (custom `autograd.Function` patterns) are merged into a single refined skill that adds differentiable collective wrappers to the weight-sharding workflow, producing an implementation that passes all tests.

## Assumptions

- Retrieved skills contain *enough* relevant signal to be amplified — refinement is a multiplier, not a generator (confirmed empirically in [[how-well-agentic-skills-work-wild]]: gains vanish when coverage score ≤ 3.49/5).
- The agent's self-evaluation is at least directionally correct (false signal can backfire — Kimi K2.5 drops 33.5% → 26.7% on SB w/ curated).
- The agent harness supports authoring and loading a new skill mid-trajectory.
- The exploration pass and the final attempt do not share state in problematic ways (no leakage of partial solutions).

## Limitations

- Expensive: a full exploration pass per task at inference time roughly doubles compute relative to direct retrieval.
- Fails when relevant skills are absent from the retrieval pool — modest or negative gains on `SkillsBench` retrieved (w/o curated).
- Self-evaluation without ground truth is fragile; weaker or differently-tuned models can mis-judge and produce worse refined skills than the originals.
- The refined skill is task-specific by construction and not directly reusable across tasks unless additional generalization steps are added.
- Effect is harness-sensitive: agent harnesses without subagent support (e.g. Terminus-2) cannot run the query-agnostic counterpart, complicating cross-strategy comparisons.

## Tradeoff profile

- *Quality vs. cost*: substantially recovers curated-level performance under high retrieval coverage (Claude 40.1 → 48.2 on SB w/ curated; 57.7 → 65.5 on Terminal-Bench 2.0) at the price of a full exploration pass per task.
- *Generality vs. specificity*: pays off most on tasks where retrieved skills partially cover the requirements; pays nothing when relevant skills are absent.
- *Inference vs. offline*: complement to query-agnostic refinement — the latter is cheap at inference but cannot compose across skills; query-specific can compose but is expensive.
- *Robustness*: cleanest gains on the strongest base agent (Claude Opus 4.6); weaker models can suffer from refinement, indicating model capability is a prerequisite.
