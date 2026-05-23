---
title: Subagent Pattern
aliases:
  - subagent
  - specialized subagent
  - subtask agent
  - procedural subagent
tags:
  - llm-agents
  - procedural-memory
  - skill-library
  - hierarchical-delegation
maturity: emerging
definition: A reusable unit of procedural knowledge represented not as text but as a specialized LLM-driven agent with its own memory and reasoning, automatically extracted from execution trajectories and invoked by a main agent via hierarchical delegation for a matched subtask.
key_papers:
  - autorefine-trajectories-reusable-expertise-continual-llm
first_introduced: "AutoRefine (2026)"
date_updated: 2026-05-19
related_concepts: []
linked_ideas: []
---

## Definition

A **subagent pattern** is a unit of procedural knowledge stored in an agent's experience repository, expressed as a specialized LLM-driven agent rather than as flat text. It is automatically extracted from successful task trajectories (typically via contrastive analysis against failed trajectories), retrieved by semantic similarity at execution time, and invoked by a main agent through hierarchical delegation — the main agent transfers task context to the subagent, the subagent operates with its own memory and reasoning, and the result is returned as an atomic operation to the main agent.

In the AutoRefine framework, subagent patterns are one of two pattern types in the repository (the other being skill patterns, which remain text/code). The boundary between them is decided by the extraction agent at creation time based on whether the procedure requires sustained multi-step reasoning.

## Intuition

Most procedural-memory work for LLM agents represents experience as text — guidelines, rules, or code snippets. That representation flattens out procedures that involve sequential steps, conditional branching, and internal state tracking (e.g., a "hotel booking" subtask). When the procedure is reinjected into the main agent's prompt, the main agent must re-interpret it every time, paying a context-length and reasoning-load cost. A subagent pattern moves the procedure into its own LLM context with its own memory, so the main agent treats the procedure as an atomic operation — semantically similar to calling a function whose implementation happens to be another LLM. The intuition is the same as one for hierarchical RL or hierarchical task networks, transposed to a prompting / retrieval substrate.

## Variants

- **Skill pattern (sibling form, not a subagent variant).** Static guideline or code snippet for simpler strategic knowledge; injected into the main agent's system prompt or registered as a callable tool. Documented here only as the contrast that motivates the subagent form.
- **Skill-as-tool.** A degenerate boundary case: executable code skills are registered as tools the main agent can invoke. This blurs the line between skill and subagent — both are "callable from the main agent" — but skills as tools do not carry independent memory or reasoning.
- **Manually designed multi-agent.** Pre-AutoRefine work (ATLAS) hand-designs the specialized agents up front. The subagent-pattern proposal is to extract these from trajectories automatically.

## Comparison

| Representation | Carries procedural state | Has independent reasoning | Source |
| --- | --- | --- | --- |
| Free-text guideline (ExpeL, AutoGuide, AutoManual) | no | no | trajectory extraction |
| Code snippet (Voyager, AdaPlanner skills) | sometimes (program variables) | no | trajectory extraction |
| Skill pattern (AutoRefine) | no | no | trajectory extraction |
| Subagent pattern (AutoRefine) | yes (own memory) | yes (own context) | trajectory extraction |
| Manually designed sub-agent (ATLAS) | yes | yes | hand-designed |

The empirical evidence in AutoRefine that motivates the table: on TravelPlanner's commonsense-macro metric, the subagent form scores 37.90% while ATLAS (manual specialized agents) scores 15.59%, and removing subagents alone from AutoRefine drops the full pass rate by 22.3 points.

## Known limitations

- The boundary between skill and subagent is decided heuristically by the extraction agent; it is not learned and can mis-classify.
- Each subagent introduces an additional LLM context window at runtime, which inflates compute cost relative to a pure-text repository.
- Subagent advantage is largest on **universal procedural patterns**; on case-specific hard constraints, manually designed agents (ATLAS) still match or exceed because they can encode explicit verification logic per constraint.
- Delegation depth is a single level in current work (main agent → subagent). Hierarchical chains of subagents are not yet evaluated.

## Open problems

- Should the skill / subagent boundary be a learned hyperparameter rather than an extraction-agent judgment?
- Can subagents themselves invoke sub-subagents safely without context-budget blow-up?
- How do subagent patterns survive concept drift in the task distribution — does maintenance prune them appropriately, or are they sticky?
- Can subagent extraction be made to learn from failure trajectories as well as success, given that the current contrastive analysis is success-conditioned?

## Relationship to foundations

The subagent pattern is a concrete instantiation of hierarchical task delegation atop LLM-driven decision making; it does not depend on any single foundational result. Its closest classical analogue is hierarchical reinforcement learning's notion of an option / sub-policy with its own internal state, transposed from value-function learning to prompt-based reasoning.

## My understanding

The interesting claim in subagent patterns is not "subagents work" — multi-agent systems are well-established — but that the *boundary* of a subagent (which subtask gets encapsulated, how its context is bounded) can be **discovered from trajectories** instead of designed up front. The strong TravelPlanner result against ATLAS provides the first empirical evidence that automatic extraction can not only match but exceed careful manual design on a non-trivial procedural benchmark. The current limitation is that the boundary decision is made by another LLM (the extraction agent) using a complexity heuristic; making this decision principled — perhaps via cost/benefit analysis on past trajectories — looks like the natural next step. The skill vs subagent typology is also under-tested: the current evidence shows that having subagents helps, but does not yet show that excluding the subagent form (i.e., forcing everything into the skill form) is the only failure mode worth eliminating.
