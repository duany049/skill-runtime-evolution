---
title: Skill-Dependent Task
aliases:
  - skill dependency verification
  - skill-essential task
  - skill-required task
tags:
  - benchmark-design
  - skill-evaluation
  - task-validity
maturity: emerging
definition: A benchmark task that the base LLM cannot reliably solve without a skill but can solve when given a verified human-authored skill, formalized as a pass-rate inequality.
key_papers:
  - skilllearnbench-benchmarking-continual-learning-methods-agent
first_introduced: SkillLearnBench (Zhong et al. 2026)
date_updated: 2026-05-18
related_concepts:
  - agent-skill
  - continual-skill-learning
linked_ideas: []
---

## Definition

A task `t_i = (x_i, v_i, S_i, Q_i)` is *skill-dependent* if it satisfies two conditions over its instance set `Q_i`:

1. **Skill-essential.** Without any skill, the fixed agent's pass rate over `R` independent runs is at most a threshold `α` on every instance: `(1/R) Σ_r v_i(a_r(q, ∅)) ≤ α` for all `q ∈ Q_i`. SkillLearnBench uses `R = 10` and `α = 0.5`.
2. **Skill-solvable.** With the human-authored reference skills `S_i`, the same fixed agent (or a stronger one) successfully completes each instance at least once.

A deterministic verifier `v_i` produces the pass/fail signal, so the inequality can be checked without subjective judging.

## Intuition

The condition is a *validity filter* on benchmark tasks for skill-generation research. If a task can be solved without a skill, including it pollutes the benchmark: any apparent benefit from a generated skill might just be variance. If a task cannot be solved even with the human-authored skill, the benchmark cannot distinguish good from bad skills (everything fails). Imposing both bounds isolates the *marginal contribution of the skill* as the signal under measurement.

## Variants

- **Hard threshold variant.** Use `α = 0` (truly unsolvable without a skill). Stricter; smaller task pool.
- **Stronger-LLM variant.** Verify skill-solvability with a stronger LLM than the solving agent, to avoid coupling task selection to one model's idiosyncrasies. SkillLearnBench permits this.
- **Multi-skill variant.** Allow `S_i` to contain multiple skills; useful for compositional tasks.

## Comparison

Contrast with the standard with/without-skill protocol (SkillsBench, LangChain, Tessl): those compare two pass rates *post hoc* without using either as a *gating filter*. Skill-dependence formalizes the gate.

Sibling to the *prompt-essential* criterion in chain-of-thought benchmarks (a task is CoT-relevant only if zero-shot is much worse than CoT), and to *retrieval-essential* in RAG benchmarks.

## Known limitations

- The choice of `α` (here 0.5) is empirical and the result space is sensitive to it.
- The condition is checked relative to *one fixed agent*; a task might be skill-essential for Claude Sonnet but solvable without a skill by Opus.
- It does not capture *gradient of difficulty*: two tasks both passing the filter may differ in how much value the skill adds.

## Open problems

- Should `α` be task-adaptive? A 0.5 threshold is more permissive on easy tasks than on hard ones.
- Can skill-dependence be defined *probabilistically* rather than via thresholds on a fixed-`R` Bernoulli estimate?
- Can the filter be generalized to *continual* settings, where the agent's "without skill" baseline is itself drifting as the library grows?

## Relationship to foundations

Echoes the methodology behind *contrast-set* evaluation: define a task by what *changes* the outcome rather than by absolute difficulty. Related to causal inference framings of "treatment effect" — the skill is the treatment, the verifier is the outcome.

## My understanding

This is the single methodological move that makes SkillLearnBench a coherent benchmark rather than a task collection. Future skill-learning benchmarks will likely inherit some version of this filter, possibly under different names. The interesting design question is whether the next version makes `α` task-specific or stays with a global threshold.
