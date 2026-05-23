---
title: Procedural Memory Framework
aliases:
  - Build-Retrieve-Update framework
  - procedural memory lifecycle
tags:
  - procedural-memory
  - agentic-memory
  - lifecycle
maturity: emerging
definition: "A three-axis factorization of agent procedural memory into independently varied Build, Retrieve, and Update modules, allowing each module's design choices to be ablated in isolation."
key_papers:
  - memp-exploring-agent-procedural-memory
  - memevolve-meta-evolution-agent-memory-systems
first_introduced: "Memp (2025)"
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

The procedural-memory framework decomposes an agent's how-to memory into three orthogonal lifecycle modules:

- **Build** — converts an executed trajectory $\tau$ and its reward $r$ into a stored memory unit $m^p = B(\tau, r)$. Granularity choices range from verbatim trajectory storage to LLM-distilled abstract scripts to their union.
- **Retrieve** — given a new task $t_{\text{new}}$, scores stored memories by vector similarity over some *key* representation (query embedding, keyword averages, etc.) and returns the top-k.
- **Update** — refreshes the memory bank after every $t$ tasks via $M(t+1) = U(M(t), E(t), \tau_t)$, supporting add, delete, and modify operations.

Treating the three modules as independent factors makes the design space measurable.

## Intuition

Prior work on procedural memory (Voyager, AWM, AutoManual) usually picks one operating point in this space and hardwires it. The framing here is that the *axes themselves* are the right object of study — performance is dominated by interactions among Build granularity, Retrieve key, and Update policy, not by any one of them in isolation.

## Variants

- **Build granularity:** trajectory-verbatim / abstract-script / proceduralization (both).
- **Retrieve key:** random sample / raw query / averaged keyword embeddings (AveFact).
- **Update policy:** vanilla append / validation-filtered / reflexion-style adjustment.

## Comparison

Distinct from agent-cognitive frameworks like Soar or Memory Bank, this framework targets *measurable lifecycle dynamics* rather than architectural completeness. Distinct from one-shot skill libraries (Voyager), it explicitly models the Update operator as a learnable knob.

## Known limitations

- The framework assumes the environment provides reward signals so Build and Update can score trajectories.
- Modules are treated as independent in ablation, but interactions (e.g. retrieval key shape × update policy) are not fully explored.
- The Hard-Constraint regression observed on TravelPlanner suggests the framework does not yet model the interaction between retrieved memory and strict-constraint reasoning.

## Open problems

- Can Update operators themselves be learned rather than hand-specified?
- What determines the right number of retrieved memories per task?
- How does the framework behave when reward signals are unavailable (real-world deployment)?

## Relationship to foundations

Grounded in human-procedural-memory analogies (Squire's procedural learning, basal-ganglia chunking) and in MDP-style policy modeling; the framework reformulates the standard policy as $\pi_{m^p}(a_t \mid s_t)$ conditional on the retrieved memory.

## My understanding

The framework is most useful as a *design-space cartography tool*: when reading other procedural-memory work (AWM, AutoManual, Voyager, Reflexion), labelling each system's choice along Build / Retrieve / Update makes their differences crisp and exposes corners of the space no one has tried.
