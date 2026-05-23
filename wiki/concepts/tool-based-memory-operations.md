---
title: Tool-based Memory Operations
aliases:
  - memory tools
  - memory action space
  - memory operations as tools
tags:
  - agentic-memory
  - tool-use
  - llm-agents
  - memory-architecture
maturity: emerging
definition: "Exposing memory-management operations (Add/Update/Delete for long-term memory; Retrieve/Summary/Filter for short-term memory) as explicit tool calls in the LLM agent's action space, so memory control becomes an intrinsic component of the agent's policy rather than an external pipeline."
key_papers:
  - agentic-memory-learning-unified-long-term
first_introduced: "Agentic Memory (AgeMem), Yu et al. 2026 (arXiv:2601.01885)"
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

Tool-based memory operations are memory-management primitives surfaced to the LLM as named tools (in the same sense as function-calling tools), so the agent can decide *when* and *with what arguments* to invoke them. In AgeMem the six tools split into LTM-targeted (Add a new entry, Update by `memory_id`, Delete by `memory_id`) and STM-targeted (Retrieve top-$k$ from LTM into context, Summary of a context span, Filter out context messages whose similarity to a criterion exceeds threshold $\theta_f$).

## Intuition

Moving memory control into the action space turns "what to store" from an external schedule into a learnable decision under the same RL objective that supervises task answers. The cost is conceptual machinery (six new actions in the policy); the benefit is that the agent can route memory operations through its own context-aware reasoning instead of via a fixed pipeline.

## Variants

- **AgeMem's six-tool kit** — Add/Update/Delete + Retrieve/Summary/Filter. The current state of the art.
- Plausible alternatives that are not yet explored: continuous (non-discrete) memory routing; learned tool *constructors* that create new memory-typing operations on the fly; tool kits that include explicit consolidation operations across episodes.

## Comparison

- **vs. RAG** — RAG is essentially a single hard-coded "Retrieve" call before answer generation; tool-based STM operations let the agent choose to Retrieve, Summary, or Filter, in any order, multiple times.
- **vs. trigger-based LTM** — trigger-based systems run Add/Update/Delete on fixed schedules (e.g., end of every turn). Tool-based LTM lets the agent invoke them on demand.
- **vs. memory-manager LLM** — a separate manager LLM also "chooses" memory ops, but at a different time, with no shared policy. Tool-based ops keep everything inside the task agent.

## Known limitations

The tool set in AgeMem is fixed and coarse-grained (e.g., Delete is whole-entry, not field-level; Filter uses a single similarity threshold). The choice of $\theta_f$ matters for Filter behavior (paper reports stable performance across $\theta_f \in [0.4, 0.8]$ with peak around 0.5). Tool calls also consume tokens — the All-Returns reward uses slightly more tokens than the Answer-Only baseline even after training.

## Open problems

- What is the right granularity for memory tools? More fine-grained operations (e.g., partial Update) might improve memory quality but expand the action space and complicate RL credit assignment.
- Can the tool set be *learned* rather than designed? E.g., the agent discovering new memory operations under a meta-objective.
- How do these tools interact with model-internal memory mechanisms (KV-cache reuse, recurrent state)?

## Relationship to foundations

Builds on tool-augmented LLM agents (function-calling, ReAct-style action interfaces) and on RL credit assignment for discrete action spaces.

## My understanding

The tool-based framing is the cheapest way to make memory management an RL problem: existing tool-use machinery (action tokens, structured outputs, tool-call rewards) carries over directly. The deeper question — and the one AgeMem doesn't fully answer — is whether the tool set should be fixed or co-designed with the model. A fixed kit makes the policy interpretable; a learned kit might be more expressive but harder to evaluate.
