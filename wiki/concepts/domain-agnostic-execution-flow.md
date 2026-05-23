---
title: Domain-Agnostic Execution Flow
aliases:
  - DAEF
  - execution flow
  - workflow framework
  - workflow skeleton
tags:
  - workflow-abstraction
  - skill-transfer
  - agent-benchmark
  - daef
maturity: emerging
definition: "A workflow skeleton shared by a family of tasks after stripping domain-specific entities, file names, and business semantics — preserving operation types and dependency structure as a scaffold for cross-domain skill transfer and benchmark construction."
key_papers:
  - skillflow-benchmarking-lifelong-skill-discovery-evolution
first_introduced: "SkillFlow (2026)"
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

A Domain-Agnostic Execution Flow (DAEF) is the abstraction of a domain-grounded workflow $\mathcal{T} = (V, E, \lambda, \gamma)$ to $\mathcal{F} = \phi(\mathcal{T}) = (V_F, E_F, \lambda_F)$ obtained by removing the grounding map $\gamma$ — concrete files, entities, fields, business objects — while preserving the node set, the dependency edges, and the operation-type labels. Node labels are drawn from a controlled single-word vocabulary (e.g. `read`, `retrieve`, `compute`, `detect`, `output`) so that the same DAEF can be recognized across distinct domains.

## Intuition

Workflow diagrams in agent literature usually conflate "what gets done" with "to what concrete thing". DAEF separates the two: it keeps the procedural skeleton that any agent solving the same workflow must follow, and discards the surface vocabulary that varies by domain. Two tasks instantiate the same DAEF when their workflow graphs agree after $\phi$, even if their files, entities, and prose are entirely different. That makes DAEF a natural scaffold for **measuring whether a skill abstracted on one task transfers to its sibling task** — sibling defined precisely by sharing the DAEF.

The construct is intentionally narrower than "high-level workflow type" and broader than "single template instance". It is a *graph-level* equivalence class with a fixed controlled vocabulary, which makes it auditable: two annotators can independently extract a DAEF from a task and check agreement.

## Variants

- Strict DAEF as defined in [[skillflow-benchmarking-lifelong-skill-discovery-evolution]] — controlled single-word vocabulary, explicit dependency edges, normalized via a two-stage annotation protocol (meta-step extraction → workflow construction).
- Allowed-variation DAEF — the canonical DAEF retains a list of permitted variation types (along which sibling tasks may differ) so a difficulty gradient can be constructed without leaving the workflow.

## Comparison

- vs. *task family* (informal): a task family is usually defined by surface labels (domain, modality, prompt). DAEF is defined by the workflow graph, so two surface-similar tasks can have different DAEFs and two surface-different tasks can share one.
- vs. *plan template* / *prompt template*: those operate at the input layer; DAEF operates at the execution-graph layer, after the agent has decomposed the task into sub-goals.

## Known limitations

- The controlled vocabulary is fixed by the annotation protocol; extending the vocabulary requires re-annotating earlier DAEFs to maintain comparability.
- The within-family reset in the SkillFlow protocol prevents DAEFs from measuring *cross-DAEF* transfer; that is a separate evaluation question.
- The abstraction map $\phi$ is human-defined; learning $\phi$ from interaction trajectories is an open problem.

## Open problems

- Automatically inducing DAEFs from raw trajectories rather than from human annotation.
- Quantifying when two DAEFs are "near-neighbors" (so partial skill transfer is plausible) versus disjoint.
- DAEF as a retrieval key for procedural-memory lookup, instead of (or alongside) embedding similarity.

## Relationship to foundations

DAEF is a graph-abstraction construct without a direct textbook precedent; the closest priors are workflow-graph schemas in business-process management and operation-type vocabularies in classical planning, but neither is referenced in the introducing paper.

## My understanding

DAEF's value lies in being **the operational definition of "same workflow"** that the skill-evolution literature has been informally invoking. Without it, claims like "the agent generalized the skill to a new task" are ambiguous between "same domain, different instance" and "different domain, same procedure". DAEF pins down the second meaning rigorously enough that a benchmark can be built around it. Whether the vocabulary generalizes to UI agents, robotics, or open-ended research workflows is the obvious next test.
