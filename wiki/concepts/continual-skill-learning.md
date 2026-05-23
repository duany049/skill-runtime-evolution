---
title: Continual Skill Learning
aliases:
  - skill continual learning
  - continual learning via skill generation
  - generate-store-reuse cycle
tags:
  - continual-learning
  - skill-generation
  - procedural-memory
  - lifelong-learning
maturity: emerging
definition: A form of continual learning where an LLM agent accumulates capabilities by generating, storing, and reusing skills in an external library rather than updating model weights.
key_papers:
  - skilllearnbench-benchmarking-continual-learning-methods-agent
first_introduced: Used by Wu et al. 2026 "agent" survey and concurrent works (SkillRL, SkillNet, AutoSkill, Memento) to frame skill libraries as the substrate for non-parametric continual learning.
date_updated: 2026-05-18
related_concepts:
  - agent-skill
linked_ideas: []
---

## Definition

Continual skill learning is the paradigm in which an LLM agent acquires new capabilities over a stream of tasks by *generating new skills* from its task experience and *adding them to a growing external library*, instead of by updating the model's parameters. Each completed (or failed) task can yield a candidate skill; the library accumulates, retrieves, and (in some variants) refines skills as the agent operates over time.

## Intuition

Classical continual learning fights *catastrophic forgetting*: when you train a model on task B, the weights that encoded task A get overwritten. Continual skill learning sidesteps this by externalizing the unit of learning. Skills live in a library outside the model. Adding skill 1001 cannot corrupt skill 1; the only failure modes are retrieval misses and skill conflicts at execution time.

This trade is the central appeal: you give up parametric integration (the model never "internalizes" the skill) in exchange for non-destructive accumulation and human auditability.

## Variants

- **One-shot generation.** Skill produced in a single pass from a task description. Baseline; SkillsBench shows it provides little average benefit.
- **Self-feedback refinement.** Agent generates, attempts, reflects, revises (e.g., SkillRL). SkillLearnBench finds this drifts recursively without external grounding.
- **Teacher-feedback refinement.** External expert (often a stronger LLM with access to ground-truth skills) provides directional guidance after failures (e.g., LOTBench-style).
- **Pipeline generation.** Structured multi-stage process (analyze → investigate → write → validate), e.g., Anthropic's Skill Creator.
- **Trajectory induction.** Skills extracted from past trajectories rather than generated from descriptions (e.g., AgentRR, SkillWeaver, ProcMEM).

## Comparison

Distinct from *in-context learning*: ICL updates behavior within a single context window without persistence. Continual skill learning persists across episodes via the library.

Distinct from *retrieval-augmented generation*: RAG retrieves factual passages; continual skill learning retrieves *procedural recipes* tagged with activation conditions and tested against task outcomes.

Sibling to *experience replay* in classical RL — both store and reuse past experience — but the unit of replay is a structured skill, not a transition tuple.

## Known limitations

- **Recursive drift in self-feedback.** Without external signal, iterated self-revision plateaus or degrades; the loop reinforces the model's own biases.
- **Adoption gap.** A skill may be high-quality on paper but the solving agent ignores it. SkillLearnBench measures skill usage rate independently of quality and finds adoption is often the bottleneck.
- **Negative transfer on open-ended tasks.** Rigid procedural skills can hurt on tasks whose solution space is broad (poem generation, creative design).
- **Backbone dependency.** Method rankings reverse across LLM families. A method tuned on Claude may underperform on Gemini.

## Open problems

- What is the minimum *external signal* sufficient to break self-feedback drift? Verifier output, weak teacher hints, peer-agent disagreement?
- How should the library *forget* — when should an old skill be retired, merged, or specialized?
- Cross-task transfer: does a skill learned on task A help task B in the same sub-domain, and if so, how does the library encode that transitivity?
- Cost accounting: continual skill learning trades compute-now for compute-later; what is the right break-even metric?

## Relationship to foundations

Builds on classical continual learning (Shi et al. 2025 survey) but inverts the storage substrate: parameters → external library. Connects to case-based reasoning, episodic memory architectures, and program-synthesis literature.

## My understanding

The paradigm's bet is that *non-destructive accumulation* outweighs *integration depth*. The bet pays off when tasks have clear reusable workflows. SkillLearnBench shows the bet fails when tasks are open-ended or when adoption is brittle — both of which are common in real-world agent deployments. The next round of work will need to grapple with adoption and selective application, not just skill quality.
