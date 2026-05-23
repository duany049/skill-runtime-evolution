---
title: Multimodal Memory
aliases:
  - multimodal experience memory
  - visual-textual memory
  - situation-conditioned memory
tags:
  - agentic-memory
  - multimodal-memory
  - retrieval-augmented-generation
  - episodic-memory
maturity: emerging
definition: A key-value experience store for embodied agents in which the keys are multimodal (task description plus visual observation or situation snapshot) and the values are previously successful plans, retrieved at planning time via a hybrid text-and-vision similarity score to provide situation-grounded in-context demonstrations.
key_papers:
  - jarvis-open-world-multi-task-agents
first_introduced: "JARVIS-1 (2023)"
date_updated: 2026-05-19
related_concepts: []
linked_ideas: []
---

## Definition

Multimodal memory is an experience store for embodied agents in which the keys are multimodal — typically a task description combined with a visual observation or compact situation snapshot — and the values are plans, trajectories, or sub-routines that were successfully executed under that key. At retrieval time the agent computes a hybrid similarity score that weighs both the textual task overlap and the visual/state alignment, returning the top-matching experiences as in-context demonstrations for the current planner.

The motivation is that in partial-observation open worlds, the same task description (e.g., "obtain a diamond pickaxe") admits many different correct plans depending on the agent's current situation (biome, inventory, time of day, tool durability). A text-only memory either over-retrieves irrelevant entries or under-conditions the retrieved plan on the current visual context. Multimodal keys make the experience store *situation-aware*.

## Intuition

A unimodal text memory says "the last time someone asked you to obtain a diamond pickaxe, here is what worked." A multimodal memory says "the last time someone asked you to obtain a diamond pickaxe *and the scene looked like this and the inventory had these items*, here is what worked." The second is far more reusable because plan correctness in open worlds is conditioned on situation as much as on task.

The retrieval implementation mirrors classical retrieval-augmented generation (RAG) but operates over the agent's *own past experience*, not over a static external corpus. The library grows with the agent and reflects what it has actually achieved.

## Variants

- **Two-stage retrieval.** Filter candidates by textual task-instruction similarity above a confidence threshold; rank survivors by visual-state similarity. JARVIS-1 uses this.
- **Joint embedding.** Map (text, image) keys into a shared embedding space and use a single nearest-neighbor lookup. Less commonly used in the open-world agent literature.
- **Compressed keys.** Replace raw observation with a learned situation summary (inventory tokens, biome enum, recent action history) to shrink the memory footprint.

## Comparison

- **vs. text-only experience memory** (e.g., text-RAG agents): faster retrieval, but blind to visual situation; under-conditions plans in open worlds.
- **vs. episodic memory of past dialogues:** focuses on (situation → successful plan) tuples rather than raw conversation turns; the unit of storage is execution-validated.
- **vs. parametric memory (LoRA / RL finetuning):** updates are append-only at the data layer, not in model weights, so the underlying LLM stays frozen and the agent can keep adding entries indefinitely.
- **vs. procedural-memory libraries** (e.g., Voyager's skill library): procedural libraries store named, reusable code skills; multimodal memory stores raw plan demonstrations keyed by situation. Different unit, different retrieval contract.

## Known limitations

- **Unbounded growth.** Memory size grows monotonically as the agent explores; retrieval cost scales with size, and there is no built-in eviction or consolidation policy.
- **Quality depends on exploration.** If the self-instruct curriculum is narrow, the memory will be narrow too — out-of-distribution situations at deployment time will retrieve poor matches.
- **Retrieval is similarity-based, not utility-based.** The agent retrieves what *looks similar*, not what *would help most*. Calibration of retrieval to downstream success is open.
- **Visual representation choice matters.** CLIP-style encoders trained on internet images may misrepresent Minecraft-like rendered scenes; domain-specific encoders (MineCLIP) help but bake in domain assumptions.

## Open problems

- How should the memory be **consolidated** as it grows — what's the equivalent of long-term-memory compression for raw multimodal trajectories?
- Can retrieval be made **utility-calibrated** (retrieve what will help, not what looks similar)?
- How does multimodal memory **transfer across embodiments** (different action spaces, different rendering engines)?
- How should multimodal memory **interoperate with parametric updates** — when is in-context retrieval enough, when must accumulated experience be folded back into weights?

## Relationship to foundations

Builds on retrieval-augmented generation (Lewis et al. 2020) but extends the key from text to a (text, vision) pair, and grounds the retrieval source in agent-generated experience rather than an external static corpus.

## My understanding

Multimodal memory is a concrete instantiation of "memory beyond context window" tailored to embodied agents. The interesting design move is that the memory is *task-and-situation keyed*: the agent does not just remember what worked, it remembers *under what visual circumstances* it worked, which lets the same task admit different correct plans across deployments. This is the conceptual handle that distinguishes it from generic agent memory or text-only RAG.

The unsolved frontier is consolidation: every system that has shipped multimodal memory hits the same wall around memory size scaling, and the architectural answers (compression, deduplication, eviction policy) are still ad hoc.
