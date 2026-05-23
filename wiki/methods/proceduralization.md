---
name: Proceduralization
slug: proceduralization
type: prompting
tags:
  - procedural-memory
  - trajectory-distillation
  - context-engineering
source_papers:
  - memp-exploring-agent-procedural-memory
parent_methods: []
child_methods: []
code_repo: https://github.com/zjunlp/MemP
date_updated: 2026-05-18
---

## Problem setting

An agent has access to a bank of past trajectories. Deciding *how to present* relevant past experience to a new task is non-trivial: verbatim trajectories are rich but bulky and overfit to the past episode; LLM-distilled abstract scripts are compact and generalize but throw away executable detail. The question is whether either extreme is the right choice.

## Mechanism

Proceduralization is a Build-time strategy that stores **both** representations of each trajectory and concatenates them at retrieval time:

1. The raw trajectory $\tau = (s_0, a_0, o_1, s_1, \ldots)$ is kept verbatim round-by-round.
2. The same trajectory is processed by an LLM that distills it into a high-level *script*: an abstract step-by-step procedure stripped of episode-specific facts.
3. At task time, the top-k retrieved entries each contribute both their trajectory and their script to the agent's context.

The hypothesis is that scripts provide *generalization scaffolding* while trajectories provide *concrete grounding*, and the combination dominates either alone.

## Procedure

1. **Distillation:** prompt the base LLM with $\tau$ and ask it to extract a numbered abstract procedure. Cache as the script $s_\tau$.
2. **Storage:** insert $(\tau, s_\tau)$ into the memory bank under the trajectory's chosen retrieval key.
3. **Retrieval:** at inference, retrieve top-k entries by cosine similarity; concatenate scripts then trajectories into the agent's context, prefixed by an instructional header.
4. **Execution:** the agent runs ReAct-style with the proceduralized context as prior knowledge.

## Assumptions

- The base LLM can distill a trajectory into a faithful script without losing critical control-flow information.
- The agent's context window is large enough to hold both trajectory and script for top-k retrieved entries.
- Cosine similarity over the retrieval key surfaces *useful* prior episodes (this is upstream of Proceduralization itself).

## Limitations

- Doubles the context cost vs. Trajectory-only or Script-only storage.
- Script quality depends on the donor model's distillation ability; weaker donors may yield uninformative scripts.
- Empirical gains on Hard-Constraint scores in TravelPlanner are mixed — the extra script context sometimes displaces the strict constraints in the agent's reasoning.

## Tradeoff profile

| | Trajectory-only | Script-only | Proceduralization |
|---|---|---|---|
| Context cost | medium | low | high |
| Generalization | low (overfits) | high | high |
| Concrete grounding | high | low | high |
| Build cost | low | medium (extra LLM call) | medium |

Best when context budget is not the binding constraint and tasks vary in surface form but share deep structure. Worst when the agent must satisfy strict constraints that get crowded out by retrieved context.
