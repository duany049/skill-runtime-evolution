---
title: Reusable Workflow
aliases:
  - workflow
  - agent workflow
  - task workflow
  - task routine
tags:
  - procedural-memory
  - agent-skills
  - workflow-induction
  - skill-library
maturity: emerging
definition: An abstracted, named sub-routine — typically (description, step_sequence) with example-specific values replaced by placeholders — used as a unit of procedural memory for LM-based agents.
key_papers:
  - agent-workflow-memory
first_introduced: agent-workflow-memory
date_updated: 2026-05-18
related_concepts:
  - workflow-induction
linked_ideas: []
---

## Definition

A reusable workflow is the *artifact* produced by workflow induction and consumed by the agent. It pairs an NL description d of what the workflow accomplishes with a step sequence (p_1, p_2, ..., p_k) where each step contains a textual observation, the agent's reasoning thought, and an executable action — and where example-specific entities (e.g., "dry cat food") have been replaced with typed placeholders (e.g., `{product-name}`). The workflow is the unit at which procedural memory is *stored*, *retrieved*, and *integrated* into the agent context.

## Intuition

Demonstrations are concrete and brittle: an example that finds "dry cat food" on Amazon does not transfer to "noise-cancelling headphones" without re-reasoning about the entity. A workflow trades the specificity of demonstrations for the *reusability* of an abstracted recipe. The bet is that **agents benefit more from a small library of well-abstracted routines than from a large corpus of literal examples**, both because the abstracted form generalizes across tasks and because it reduces element-selection bias that follows from retrieving concrete examples.

## Variants

- **Code-format workflows**: steps stored as executable programs (`CLICK(submit-id)`, `stop()`); aligns naturally with browser-action agents.
- **Text-format workflows**: actions verbalized in NL (`"CLICK the submit button"`). AWM's ablation shows roughly comparable performance to code format on Mind2Web.
- **State-described workflows**: each step carries an NL description of the environment state. AWM's default.
- **HTML-augmented workflows**: each step carries filtered HTML in addition to (or instead of) the NL state description. AWM's ablation shows HTML alone is marginally weaker than NL; combining them is *worse* than either alone due to context-length blowup and filter noise.
- **Per-website workflows**: AWM partitions the workflow store by website, keeping each agent invocation grounded in the small set of routines relevant to the current site.

## Comparison

- **vs. demonstrations / few-shot examples**: workflows are abstracted and named; demonstrations are concrete and untyped. Workflows have higher reuse but lower fidelity.
- **vs. tools / skills (Voyager)**: tools are executable code units with explicit signatures; workflows are recipes that combine actions and NL reasoning, intended to be read by the LM rather than directly executed.
- **vs. human-authored workflows (SteP)**: induced workflows are derived from agent experience without expert authorship; SteP's hand-written workflows are higher precision but require domain expertise per task family.
- **vs. raw trajectories**: trajectories are unprocessed; workflows are deduplicated, abstracted, and named.

## Known limitations

- Workflows can be over-specific (failed abstraction) or over-general (lost too much context to be useful). AWM's induction prompt is a fixed heuristic.
- Once stored, a workflow is currently irrevocable; if it was induced from a falsely-judged success, it can keep harming downstream tasks.
- Workflows do not currently carry retrieval keys beyond their NL description, so retrieval defaults to "include all workflows for this website."
- Long-horizon use accumulates workflows monotonically without compression or retirement.

## Open problems

- What is the right **granularity** for a workflow — sub-task, sub-skill, primitive? AWM lands on sub-task by prompt design; a learned policy is open.
- How should workflows be **revised** when later evidence contradicts the induced routine?
- How does the workflow library **compose**? AWM shows workflows can chain ("find a place" → "get the zip code of a place"); the general composition operator is not formalized.
- What **metadata** does a workflow need beyond (description, steps) to support better retrieval — provenance trajectories, success counts, applicability conditions?

## Relationship to foundations

Reusable workflows are a particular formalization of *procedural memory* — sitting alongside the macro-operator and chunking traditions in classical AI. They are also a particular *skill* representation, but unlike Voyager-style code skills they are LM-readable recipes rather than executable functions.

## My understanding

The workflow as defined by AWM is a useful but not finished representation. It captures enough abstraction to drive measurable transfer on WebArena and Mind2Web, but the lack of revision, the heuristic granularity, and the bespoke per-website partitioning all read as places where the representation will evolve. The interesting question is whether future systems will keep workflows as the unit, or fragment them into smaller composable primitives that an LM stitches together at use time.
