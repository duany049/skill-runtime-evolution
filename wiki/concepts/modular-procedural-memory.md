---
title: Modular Procedural Memory
aliases:
  - multi-agent procedural memory
  - role-aware procedural memory
  - LEGO-style procedural memory
tags:
  - procedural-memory
  - multi-agent-systems
  - llm-agents
  - workflow-automation
maturity: emerging
definition: Procedural memory for multi-agent LLM systems decomposed into role-aware units — full-task plans for the orchestrator and fine-grained subtask traces for individual task agents — that can be retrieved and re-routed independently rather than stored as flat trajectories.
key_papers:
  - legomem-modular-procedural-memory-multi-agent
first_introduced: 2026 (LEGOMem, AAMAS)
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

Modular procedural memory is the design pattern of splitting a successful multi-agent task trajectory into separately addressable units aligned to roles in the agent team: a **full-task memory** (the orchestrator's high-level plan, agent-selection trace, and summarized outcome) and a set of **subtask memories** (each task agent's localized tool-use sequence and observations). These units live in role-keyed banks, are indexed by embeddings of task or subtask descriptions, and are retrieved and routed *independently* — the orchestrator pulls relevant full-task memories for planning while each task agent receives only the subtask memories that match its delegated subtask. This contrasts with flat single-agent procedural memory (Synapse-style full trajectory exemplars, AWM-style induced subtask sequences) that exposes the entire trace to one model irrespective of role.

## Motivation

In multi-agent orchestrator-plus-task-agents architectures (e.g. Magentic-One-style), different roles need different memory abstractions: an orchestrator benefits from end-to-end planning context, while a task agent benefits from precise tool-call sequences for its app. Treating all memory as a single global pool either floods the orchestrator with low-level tool details or starves the task agent of execution specificity. Modular procedural memory separates these so each role retrieves at its own granularity.

## Variants

- **Vanilla LEGOMem** — subtask memories statically extracted from the retrieved full-task memories at task start, no runtime re-retrieval.
- **LEGOMem-Dynamic** — just-in-time subtask retrieval against per-agent banks each time a subtask is generated.
- **LEGOMem-QueryRewrite** — pre-plan a draft subtask list via a query-rewriter LLM, retrieve subtask memories ahead of execution.

## Empirical signal

On OfficeBench, modular allocation of memory across orchestrator and task agents lifts overall success rate by 12–14 absolute points over memory-less multi-agent baselines, and the placement of memory (orchestrator vs task agent) matters more than the choice of retrieval granularity. Orchestrator memory carries most of the gain; subtask memory becomes more decisive when task agents are smaller language models. See [[legomem-modular-procedural-memory-multi-agent]].

## Related work

Single-agent precursors include Synapse (full successful trajectories as exemplars) and Agent Workflow Memory / AWM (induced subtask sequences as reusable skills). Modular procedural memory generalizes the *unit* of procedural memory from "one trajectory per task" to "one trajectory per role" and adds an explicit routing decision absent in single-agent settings.

## Open questions

- Whether finer-grained roles (per-tool memory, per-state memory) further improve over the current orchestrator/task-agent split.
- Whether failed trajectories can contribute role-aware "what-not-to-do" memory units symmetric to the current success-only curation.
- Whether modular procedural memory composes with hierarchical or learnable retrieval policies, rather than flat embedding similarity.
