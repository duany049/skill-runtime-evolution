---
title: Workflow Induction
aliases:
  - workflow extraction
  - sub-routine induction
  - routine extraction
tags:
  - procedural-memory
  - agent-skills
  - workflow-induction
  - lifelong-learning
maturity: emerging
definition: The process of extracting reusable, abstract sub-routines (workflows) from past agent trajectories so they can be reused as procedural memory for future tasks.
key_papers:
  - agent-workflow-memory
first_introduced: agent-workflow-memory
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

Workflow induction is the operation that maps a set of past agent experiences E = {(task instruction q, action trajectory P)} to a set of workflows W = {(description d, abstracted step sequence)}. The induction module I: E → W is the central mechanism by which procedural memory is *acquired* — distinct from procedural memory *use* (retrieval and integration into the agent prompt) and *evaluation* (judging whether an experience was successful enough to induct from).

Two design dimensions distinguish induction methods. **Granularity**: at what level of decomposition is a sub-routine extracted? Whole-task replay (each experience becomes one workflow) at one extreme; fine-grained primitives at the other. AWM lands in the middle, inducing common sub-tasks like "search for a product" rather than full instructions. **Abstraction**: what example-specific details are removed? Pure replay keeps everything; full abstraction replaces all entities with typed placeholders ({product-name}, {location}). Abstraction trades fidelity for transfer.

## Intuition

Humans solve complex tasks by chunking past experience into reusable routines, then composing those routines into new procedures. Workflow induction is the mechanical analog: don't ask an agent to re-derive "how to find an item on Amazon" every time; have it abstract that procedure once, store it as a named workflow, and call it as a sub-component when a future task needs item-finding. The induction problem is non-trivial because the right *unit* of abstraction is task-dependent, the right *abstraction level* depends on transfer requirements, and there is no oracle for which trajectories deserve to become durable workflows.

## Variants

- **Rule-based induction**: extract action sequences from each experience, deduplicate by action signature, and drop steps with environment-invalid actions. Cheap, deterministic, no LM call at induction time.
- **LM-based induction**: prompt the agent LM to read one or more experiences and emit named sub-routines with abstracted parameters. Captures finer granularity and removes example-specific bias at the cost of an extra LM call.
- **Offline induction**: run once over a fixed training set, produce a static workflow library used for all subsequent test queries.
- **Online induction**: induce continuously on the stream of agent experiences as they happen, with success judged by an LM evaluator. Yields a snowballing memory where workflows can compose on top of earlier workflows.

## Comparison

Workflow induction differs from neighboring operations:

- **Skill discovery** (Voyager-style) focuses on growing a library of executable code skills; workflow induction targets *recipe-like sub-routines* expressed in NL plus actions rather than self-contained programs.
- **Demonstration retrieval** (Synapse) augments the agent with concrete past examples; workflow induction *abstracts* the examples first, replacing entities with placeholders to reduce demonstration-specific bias.
- **Trajectory summarization** condenses a single trajectory into a textual gist; workflow induction extracts cross-experience patterns and is intended to be re-invoked, not just re-read.
- **Human-authored workflows** (SteP) bypass induction entirely and depend on a domain expert.

## Known limitations

- Quality of induced workflows depends on the success signal: a noisy LM evaluator can canonize incorrect routines and degrade downstream performance.
- Granularity is currently a prompt-engineering decision; there is no learning signal to pick the right abstraction level adaptively.
- Pure online induction cannot revise or retract workflows once stored.
- Cross-domain transfer of induced workflows is empirically weaker than same-domain reuse; induction in the test distribution remains the strongest setting.

## Open problems

- Can the *granularity* of induction be learned rather than prompted?
- What is the right policy for **workflow retirement** when induced routines turn out to be wrong or no longer apply?
- How does induction interact with retrieval? A large unified workflow library demands a smarter access policy than per-website partitioning.
- Can induction be coupled with **counter-example mining** so failed trajectories also contribute (as anti-patterns)?

## Relationship to foundations

Workflow induction is a procedural-memory acquisition mechanism, sitting alongside classical procedural memory ideas in cognitive science (chunking, skill compilation) and inductive-programming traditions (library learning, DreamCoder-style abstraction). It differs from purely symbolic library learning by operating directly over LM-generated NL+action trajectories rather than typed program syntax.

## My understanding

Induction is the *load-bearing* operation in any procedural-memory system: retrieval and integration are downstream of having a meaningful library to retrieve from. AWM's empirical contribution is that LM-based induction over self-generated trajectories — even with no human-curated training set — produces a library good enough to beat hand-authored workflows on WebArena. That makes induction the locus of the most promising future work: better evaluators, better abstraction control, and explicit mechanisms for revising the library when induction goes wrong.
