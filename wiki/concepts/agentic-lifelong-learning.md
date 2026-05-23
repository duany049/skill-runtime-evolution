---
title: Agentic Lifelong Learning
aliases:
  - lifelong skill evolution
  - agent lifelong learning
  - sequential skill acquisition
tags:
  - lifelong-learning
  - skill-evolution
  - continual-learning
  - agent-protocol
maturity: emerging
definition: "An evaluation protocol in which an autonomous agent begins a task sequence with no skills, solves tasks in fixed difficulty order, externalizes each trajectory's lesson into a skill patch, and carries the updated skill library forward — measuring whether the library, not just any single task, improves over time."
key_papers:
  - skillflow-benchmarking-lifelong-skill-discovery-evolution
first_introduced: "SkillFlow (2026)"
date_updated: 2026-05-18
related_concepts:
  - skill-patch
  - domain-agnostic-execution-flow
linked_ideas: []
---

## Definition

Agentic Lifelong Learning is a sequential evaluation protocol over an ordered task family $\mathcal{F} = \{T_1, T_2, \ldots, T_n\}$ sorted by within-family difficulty. The agent maintains a mutable skill library $\mathcal{S}_t$ at step $t$. On $T_1$ the agent runs with an empty library; after every task it consumes the trajectory $\tau_t$ and verifier-derived rubric $r_t$ to produce a skill patch $\Delta_t = \text{Model}_g(\mathcal{S}_{t-1}, \tau_t, r_t)$ under a fixed prompt template $g$, and the patched library $\mathcal{S}_t = \text{Apply}(\Delta_t, \mathcal{S}_{t-1})$ is what the agent uses on $T_{t+1}$. The protocol resets the library between families so the unit of evaluation is **library-level improvement within one workflow**, not cross-workflow generalization.

## Intuition

Most skill-augmented agent benchmarks evaluate a snapshot: "given these skills, how well does the agent perform?" Agentic Lifelong Learning evaluates the *delta*: "starting from nothing, does the agent's library help its future self?" By forcing the library to be built incrementally from the agent's own traces — not handed in by humans — the protocol directly measures the discovery + repair + consolidation chain that the skill-evolution literature has been claiming is the next frontier.

The distinguishing piece is that the protocol decouples the patch-generation step from the agent harness. The harness runs tasks; the patch is produced by the **model's native capability** under a fixed prompt template. This isolates skill quality from instruction-following ability of the harness.

## Variants

- Trajectory-and-rubric driven (canonical, from SkillFlow): both $\tau_t$ and $r_t$ are inputs to the patch model. Rubric is normalized text describing missing/incorrect content from the verifier.
- Trajectory-only ablation: $r_t = \emptyset$; tests whether verifier feedback is necessary for useful patches.
- History-context control: instead of skills, prepend the full prior interaction history. On Claude Opus 4.6 this reaches only 51.04% (vs 71.08% with skills), confirming the gain is not from raw context length.

## Comparison

- vs. *one-shot skill generation*: one-shot evaluates "can the agent write a skill once that helps later"; ALL evaluates "can the agent maintain a library across many tasks, repairing earlier mistakes as it learns more".
- vs. *episodic memory replay*: episodic memory stores raw trajectories. ALL stores compressed, executable, optionally-typed skill artifacts — and measures whether the *abstraction* (not the raw record) helps.
- vs. *cross-domain continual learning*: traditional continual-learning benchmarks span heterogeneous tasks; ALL constrains the task family to a shared DAEF so the protocol measures repair and consolidation rather than retrieval and routing.

## Known limitations

- Library reset between families is a design choice — it isolates within-workflow learning but means cross-family skill transfer is not measured.
- The fixed patch prompt template introduces prompt sensitivity that the protocol does not quantify.
- Skill-use detection is via execution-trace events (reads/calls); the protocol cannot distinguish "skill consulted" from "skill caused the action".

## Open problems

- Designing variants that *do* mix families to test cross-DAEF transfer without confounding skill quality with retrieval quality.
- Credit assignment: how much of the gain comes from the patch model versus the execution harness?
- Optimal stopping: when should the agent stop patching and commit to its current library? No principled criterion exists.
- Whether explicit skill repair can be trained for, rather than emerging as a side effect of base-model capacity.

## Relationship to foundations

The protocol is an instantiation of the broader continual-learning setup applied to skill libraries rather than model weights. It does not depend on any foundational machine-learning result beyond the standard sequential-evaluation framing.

## My understanding

Agentic Lifelong Learning is the **scaffolding piece** that the skill-evolution discourse needed. Earlier work made claims like "skills extracted from experience help downstream performance"; ALL pins down exactly what "downstream" means (the next task in a difficulty-ordered family with the same DAEF) and what "help" means (verifier success rate plus library-size compactness). The protocol's biggest contribution may be normative: it implicitly argues that *library quality over time* is the right unit of evaluation for skill systems, not single-task task success. Once that framing is accepted, the negative-result findings in SkillFlow (skill inflation, error propagation, repair-not-writing-as-bottleneck) become legible as quality regressions rather than experimental noise.
