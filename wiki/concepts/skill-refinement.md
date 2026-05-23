---
title: Skill Refinement
aliases:
  - skill adaptation
  - skill rewriting
  - skill improvement
tags:
  - skill-refinement
  - test-time-adaptation
  - skill-quality
  - agentic-skills
maturity: emerging
definition: The process of transforming retrieved or stored agentic skills into more useful forms before or during agent task execution, by improving skill content (clarity, relevance, format) or by synthesizing across multiple skills, with the goal of narrowing the gap between general-purpose skills and task-specific curated ones.
key_papers:
  - how-well-agentic-skills-work-wild
first_introduced: how-well-agentic-skills-work-wild (this paper) — formalization of query-specific vs. query-agnostic refinement strategies with empirical comparison
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

*Skill refinement* turns a candidate skill set — typically the top-k results of [[skill-retrieval]] from a large noisy collection — into a higher-utility set before the agent attempts the task. Operationally it includes: cleaning up format and clarity, distilling relevant snippets from noisy skills, synthesizing complementary information across multiple skills, and reformulating skill metadata to make selection easier. The refining agent may or may not see the target task. [[how-well-agentic-skills-work-wild]] partitions refinement strategies along this *task-awareness* axis into query-agnostic and query-specific.

## Intuition

Curated skills win because they encode exactly the information the task needs, in a form the agent recognizes. General-purpose retrieved skills lose on both axes: they carry tangential content and they are not framed for the immediate task. Refinement asks whether the agent can close that gap automatically. The empirical finding from [[how-well-agentic-skills-work-wild]] is that refinement acts as a *multiplier* on existing skill quality, not as a *generator*: when retrieved skills already cover the task adequately, the agent can synthesize a tailored skill that lifts performance toward the curated upper bound; when relevant skills are absent from the retrieval pool, refinement cannot conjure them and the gain disappears.

## Variants

- *Query-agnostic refinement* — each skill is improved offline, independently of any specific task, using a meta-skill (e.g. Anthropic's `skill-creator`) that generates synthetic test queries, compares agent outputs with/without the skill, and iterates. Cheap at inference time; cannot compose across skills.
- *Query-specific refinement* — at inference time, the agent reads the task, examines retrieved skills, attempts an initial solution, self-evaluates correctness (without ground-truth verifier), reflects on which skills helped, and writes a single tailored skill that merges and discards content from the candidates. Expensive but composes across skills.
- *Hybrid refinement* (open) — task-distribution-aware offline refinement that may be a useful middle ground between the two endpoints.

## Comparison

vs. *prompt rewriting / query refinement*: skill refinement edits the knowledge artifact itself; prompt rewriting changes the user's request. Skill refinement persists if cached; prompt rewriting is per-turn.

vs. *fine-tuning on skill content*: refinement is text-level and reversible; fine-tuning is weight-level and one-way.

vs. *test-time adaptation* (TTA in classification): TTA usually updates model parameters or activations; skill refinement updates external artifacts the model reads.

## Known limitations

- Query-specific refinement is expensive — a full exploration pass per task at inference time — and its cost vs. quality tradeoff is not yet characterized.
- Without ground-truth verification during self-evaluation, the agent's judgement of which skills helped is itself noisy; this can backfire (Kimi K2.5 drops 33.5% → 26.7% on SB w/ curated in [[how-well-agentic-skills-work-wild]]).
- Query-agnostic refinement cannot exploit task structure and yields inconsistent, often marginal gains.
- Refinement does not close the gap when retrieved skills have low task coverage (~3.5/5 in [[how-well-agentic-skills-work-wild]]).

## Open problems

- Can query-agnostic refinement be made effective with access to the *task distribution* rather than the specific query?
- What self-evaluation signal is strong enough to trust without ground-truth (execution traces, unit-test-like checks, agent-internal confidence)?
- Does refinement value depend on the agent's harness, and how does a refined skill transfer across harnesses?
- Should refined skills be written back to the collection (closing an evolution loop), or kept ephemeral?

## Relationship to foundations

Skill refinement combines ideas from self-improvement (the agent revises an artifact based on its own outputs), reflection / verbalized critique, and skill / instruction rewriting. It sits between *fixed knowledge* (RAG) and *learned knowledge* (fine-tuning) on the agent-knowledge spectrum.

## My understanding

The multiplier-not-generator framing is the most actionable result for deployers: refinement at inference time pays off only when the upstream retrieval is already mostly hitting the right neighborhood. That makes retrieval quality the load-bearing investment, and refinement an inference-time amplifier rather than a fallback. The interesting open question is whether a smarter offline (query-agnostic but task-distribution-aware) refinement could push curated-level quality into the bulk of a large skill repository without paying per-query cost — that would change the cost structure of skill ecosystems entirely.
