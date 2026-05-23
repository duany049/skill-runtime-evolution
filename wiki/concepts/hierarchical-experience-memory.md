---
title: Hierarchical Experience Memory
aliases:
  - hierarchical memory
  - experience memory pool
  - structured experience memory
tags:
  - procedural-memory
  - agentic-memory
  - skill-library
  - gui-agents
maturity: emerging
definition: "A three-tier experience store (high-level workflows, mid-level subtask skills, failure patterns) where each entry is stored as a parameterised template indexed by embedding similarity and retrieved with success-history + exploration scoring, intended to replace raw replay buffers for cross-task and cross-application transfer in agentic RL."
key_papers:
  - ui-mem-self-evolving-experience-memory
first_introduced: ui-mem-self-evolving-experience-memory
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

Hierarchical Experience Memory (HEM) is a multi-level structured store for agentic experience that holds three categorically different artefacts side by side: high-level workflow plans, mid-level subtask skills, and failure patterns. Each artefact is parameterised — concrete values such as filenames, query strings, or contact names are replaced with semantic placeholders (e.g. `{{filename}}`, `{{recipient}}`) — so a single entry covers many surface instantiations. Retrieval combines embedding similarity over task / subtask descriptions with usage statistics, enabling both exploitation of high-success plans and exploration of under-tried alternatives.

HEM contrasts with two adjacent designs: (a) **raw experience replay**, which stores complete trajectories verbatim and suffers from poor generalisation across tasks, and (b) **flat skill libraries**, which mix all reusable knowledge into a single retrievable bag without distinguishing planning, execution, and error knowledge.

## Intuition

The motivation is cognitive: human GUI users keep different *levels* of knowledge about their apps. They know that "Send Email" decomposes into a small number of subtasks (workflow), they know how the *Search* widget on a given app behaves in detail (skill), and they remember specific traps ("don't click Save before naming the file") that block tasks (failure pattern). Conflating these into one representation makes retrieval clumsy: the right granularity for guiding "what should the agent plan to do next" is different from the right granularity for "what micro-mistakes should it avoid right now".

By separating the levels and storing each as parameterised templates, HEM produces both:

- **denser credit assignment** (the workflow gives the agent a sub-task structure against which a progress reward can be computed)
- **cross-task transfer** (templates abstracted over placeholders match novel tasks whose surface form differs but whose underlying structure is similar)

The three tiers are not orthogonal — a single rollout typically retrieves at most one workflow but many skills and several failure patterns, and the workflow may *index* which skills are relevant for each of its subtasks.

## Variants

- **Strong-guidance prompt**: agent receives the full $(\mathcal{W}, \Sigma, \mathcal{F})$ bundle.
- **Weak-guidance prompt**: agent receives only the workflow $\mathcal{W}$ — execution details are left to the policy.
- **No-guidance prompt**: agent operates without any retrieved memory.

The same memory pool can also be queried at inference time after training to produce additional gains, separately from its training-time role.

## Comparison

- vs. **traditional experience replay** (storing raw trajectories): HEM trades verbatim recall for transferability — the template abstraction is lossy on specifics but compounds reusability.
- vs. **flat skill libraries** (e.g. Voyager-style skill bags): HEM separates planning from execution from error knowledge, addressing a known retrieval-precision problem in flat libraries where similar surface skills overwrite each other.
- vs. **chain-of-thought / scratchpad memory**: HEM is durable and update-able across episodes, not just a within-episode reasoning trace.

## Known limitations

- Bootstrapping the workflow level requires either annotated subtask traces or a non-trivial cold-start regime; HEM does not specify how to begin from zero.
- The failure-pattern tier uses recency bias for retrieval but has no specified compaction policy; long-running deployments may accumulate stale or contradictory entries.
- The three-tier split is justified by ablations on GUI agents specifically; whether the same split is optimal in code, web browsing, or scientific reasoning is open.

## Open problems

- Principled forgetting / compaction policies for each tier.
- Whether the tiers should themselves be discovered (learned) rather than hand-designed.
- Cross-domain reuse: does a workflow learned in mobile GUIs help web agents at all?

## Relationship to foundations

HEM is a *concrete instance* of the broader procedural-memory and skill-library tradition. The placeholder-template abstraction is loosely related to schema theory in cognitive science, but the paper does not formalise this connection.

## My understanding

HEM's most defensible contribution is the *separation* of planning, execution, and error knowledge. Many prior skill libraries collapsed these into a single store and suffered from retrieval ambiguity (the same query matches a high-level plan and a low-level action recipe with comparable similarity). The placeholder-template abstraction is a useful but secondary mechanism — it primarily addresses storage efficiency and cross-task transfer; the tier structure is what unlocks denser credit assignment for long-horizon RL.
