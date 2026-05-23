---
name: Progressive Skill Evaluation
slug: progressive-skill-evaluation
type: evaluation
tags:
  - benchmarking
  - skill-evaluation
  - agentic-skills
  - realistic-evaluation
source_papers:
  - how-well-agentic-skills-work-wild
parent_methods: []
child_methods: []
code_repo: "https://github.com/UCSB-NLP-Chang/Skill-Usage"
date_updated: 2026-05-18
---

## Problem setting

Evaluating whether an agent benefits from agentic skills requires more than measuring pass-rate-with-skills minus pass-rate-without-skills on a curated benchmark — that measurement assumes away the practical challenges of skill discovery and adaptation. *Progressive skill evaluation* targets the gap between idealized benchmarks and real deployment by systematically introducing the realistic challenges one at a time, so that each drop in pass rate can be attributed to a specific factor.

## Mechanism

The methodology defines an ordered sequence of evaluation conditions on the same task set, varying three independent factors:

1. *Skill availability* — whether curated skills exist for each task or only general-purpose retrieved skills.
2. *Skill discovery* — whether skills are user-provided in context or must be retrieved by the agent from a large pool.
3. *Skill selection* — whether the agent is forced to load all provided skills or must decide on its own.

Each condition isolates the contribution of one factor by changing only that factor relative to the previous condition. The expected output is a monotone decreasing pass-rate curve; deviations from monotonicity surface bottlenecks (e.g. an agent harness that over-loads skills).

## Procedure

Instantiated in [[how-well-agentic-skills-work-wild]] as six conditions on `SkillsBench`:

1. **Curated + forced load.** All curated skills placed in the agent's environment; agent instructed to load all of them. Upper bound on curated utility.
2. **Curated.** Same skills available; load decision left to the agent. Isolates *skill selection*.
3. **Curated + distractors.** Curated skills plus retrieval-mined distractors, total k=5. Intensifies selection.
4. **Retrieved (w/ curated).** Top-k from the full collection (which includes curated). Introduces *skill retrieval*.
5. **Retrieved (w/o curated).** Top-k from the collection with curated skills removed. Introduces *skill adaptation*.
6. **No skills.** Baseline.

Each condition is run multiple times per task across multiple models, with skill-loading rate reported alongside pass rate to expose the selection bottleneck. The protocol generalizes to non-skill-purpose-built benchmarks (demonstrated on `Terminal-Bench 2.0`) by simply dropping the curated-skill conditions and keeping the retrieved / no-skills pair.

## Assumptions

- A meaningful curated skill set exists for the upper-bound conditions (or those conditions can be dropped).
- The retrieval pool is large enough (10⁴ +) to make retrieval non-trivial.
- The agent harness can be configured to either force-load or self-select skills.
- Skill-loading rate is observable from the agent trajectory.

## Limitations

- Requires multiple full evaluation runs per task, multiplying compute cost.
- Assigning a drop in pass rate to a specific factor is correlational, not causal — confounding between selection and retrieval cannot be fully ruled out.
- Conditions are tied to a specific retrieval system; a different retriever could shift the curve.
- The methodology says nothing about *why* a particular agent or harness fails at a given stage; mechanistic diagnosis still requires separate analysis.

## Tradeoff profile

- *Coverage vs. cost*: more conditions sharpen attribution but inflate compute. Six conditions × three models × 84 tasks × 3 repeats ≈ 4500 trajectories.
- *Generality vs. specificity*: works on any agent-skill benchmark, but the absolute numbers are not directly comparable across benchmarks with different difficulty.
- *Methodology vs. implementation*: the protocol is the contribution; the specific retriever and refinement strategies are interchangeable.
