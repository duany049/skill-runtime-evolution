---
title: GUI-DFS Exploration
aliases:
  - GUI Depth-First Search
  - GUI-DFS
  - DFS-based UI exploration
  - reverse-traversal UI DFS
tags:
  - computer-use-agent
  - gui-agent
  - environment-learning
  - exploration
  - skill-library
maturity: emerging
definition: "Goal-free, agent-driven depth-first exploration of a GUI environment in which a planner/action/feedback triad iteratively discovers, verifies and condenses unit functions into a reusable skill set."
key_papers:
  - osexpert-computer-use-agents-learning-professional
first_introduced: "OSExpert (Liu et al., 2026)"
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

GUI-DFS exploration is a structured, goal-free traversal of a GUI's state space in which a computer-use agent discovers unit functions by maintaining an explicit DFS stack of exploration-state nodes. Each node bundles a plan sequence `Π` and the action sequence `α` that reaches the state; the agent restarts the environment, replays `α`, lets an action module emit `α'`, and asks a feedback module to classify the outcome as Continue / Final / Error. Continue states push successor plans; Final states distill `(Π, α⊕α')` into a verified unit-function skill; Error states return targeted critiques and trigger bounded retries.

## Intuition

A novice exploring a new application tends to click everything visible at the top level, then recurse into whichever sub-screen each click opens, restarting whenever something goes wrong. GUI-DFS encodes that intuition as a stack discipline: depth-first ensures functions are explored *to completion* before moving sideways, reverse traversal keeps the order logical, and the planner-action-feedback decomposition keeps the per-node decision tractable for current vision-language models. The DFS stack is the agent's working memory; the resulting skill set is its long-term consolidation.

## Variants

- **Strong-explorer configuration** — GPT-5 as planner + feedback, UI-TARS-1.5-7B as action module (used in OSExpert main results).
- **Single-model configuration** — Qwen-3-VL-8B for all three modules (cheaper; achieves comparable results on several OSExpert-Eval subsets).
- **Curriculum-extended GUI-DFS** — after unit-function discovery, the agent self-proposes composite tasks rooted in already-discovered skills and runs a second exploration pass to verify them.

## Comparison

Distinct from prior self-evolving GUI agents (AppAgent-v2, SEAgent, Mobile-Agent-E) which explore along human-provided or tutorial-derived queries: GUI-DFS is *goal-free* and *bottom-up*, so the resulting skill set is not biased by the query distribution. Compared with breadth-first or random exploration, the DFS stack discipline gives logical, function-completing traversal at the cost of depth-first blind spots in highly branching menus.

## Known limitations

- No exhaustive-coverage guarantee under finite budget; deeply nested or rarely-reachable functions can be missed.
- Each node requires an environment reset + action replay, which is expensive for apps with slow startup or persistent state.
- Failure classifications conflate true infeasibility with agent-policy limits, so failed entries can ossify weak-backbone blind spots in downstream skill-boundary checks.

## Open problems

- Replacing the fixed retry budget `R` with a learned stopping criterion conditioned on the error attribution.
- Distinguishing exploration failures caused by the agent from those caused by the environment.
- Sharing partial exploration progress across application versions or sibling applications without redoing the full traversal.

## Relationship to foundations

GUI-DFS extends classical depth-first search to a setting where state transitions are non-deterministic, observation-dependent, and partially observable, and where the "graph" is induced by an agent's plans rather than a closed action API. It also borrows the planner/critic decomposition common in tool-use and self-correction work.

## My understanding

GUI-DFS is, mechanically, a relatively simple wrapper around an LLM-driven UI agent — its novelty is in making the *exploration objective* explicit and decoupled from any downstream task, which is what unlocks broad coverage of unit functions in environments where neither curated demonstrations nor user queries exist. The interesting open question is whether DFS is actually the right strategy here, or whether BFS / priority-based traversal would yield better coverage per unit of compute.
