---
title: Automatic Curriculum
aliases:
  - LLM-generated curriculum
  - bottom-up curriculum
  - novelty-search task proposer
  - GPT-4 automatic curriculum
tags:
  - automatic-curriculum
  - curriculum-learning
  - novelty-search
  - lifelong-learning
  - llm-agent
  - open-ended-exploration
maturity: emerging
definition: A curriculum mechanism in which a large language model is prompted to propose the next task for an embodied agent conditioned on the agent's current state, completed/failed task history, and a high-level directive to maximize diversity — producing an in-context, bottom-up curriculum that tracks the agent's capability frontier without any hand-authored task graph.
key_papers:
  - voyager-open-ended-embodied-agent-large
first_introduced: "Voyager (Wang et al., 2023) as a named module driven by GPT-4 prompting; conceptually rooted in PAIRED / intrinsically-motivated curriculum learning (Wang et al. 2019; Portelas et al. 2020; Forestier et al. 2022)."
date_updated: 2026-05-20
related_concepts: []
linked_ideas: []
---

## Definition

An automatic curriculum, in the Voyager sense, is a prompt-driven module that asks a large language model to **propose the next task** for an embodied agent. The prompt bundles four ingredients:

1. **Diversity directives and difficulty constraints** ("discover as many diverse things as possible... the next task should not be too hard since I may not have the necessary resources or have learned enough skills to complete it yet");
2. **The agent's current state** (inventory, equipment, nearby blocks/entities, biome, time, health/hunger, position);
3. **Lists of previously completed and failed tasks**, reflecting the capability frontier;
4. **Additional context** answered by a cheaper LLM that self-asks and self-answers questions about the current state (Voyager uses GPT-3.5 for this).

The output is a single next-task description, sampled at a small but nonzero temperature for diversity. The curriculum **unfolds bottom-up**: it is not a fixed DAG of milestones but a state-conditioned policy over the unbounded task space.

## Intuition

In open-ended environments the task space is effectively unbounded and the order in which tasks become reachable depends on what the agent already has in inventory, what biome it stands in, and which skills are already in its library. Hand-authoring such a curriculum is either too sparse (overshoots the frontier and the agent fails) or too dense (undershoots and the agent stops growing). Asking the LLM to propose the next task conditioned on state and history sidesteps both: the proposal is *capability-conditioned* (history of failures) and *opportunity-conditioned* (current biome and inventory).

The reframing — "novelty search via prompting" — uses GPT-4's prior knowledge of Minecraft (recipe trees, biome contents) as the world model that drives diversity. No explicit novelty bonus, no intrinsic-motivation reward; the directive itself does the work in context.

## Variants

- **State-and-history-conditioned (Voyager).** Full prompt with state, completed and failed tasks, plus a separately-asked Q&A context block. Temperature 0.1 for proposals.
- **Self-instruct exploration (JARVIS-1).** Capability-conditioned sampling from a *discrete* task pool with distributed execution and shared memory. Differs from Voyager's automatic curriculum in (a) sampling from a fixed pool rather than free-form proposal, and (b) running many parallel agents.
- **Warm-up scheduled curriculum (Voyager appendix).** Information made available in the prompt grows with completed-task count, ensuring early prompts emphasize basics and later prompts richer state.

## Comparison

- **vs. fixed skill-tree curriculum** (most prior Minecraft RL work): predefined milestones cannot adapt to which biome the agent woke up in or which dependencies it already satisfied.
- **vs. [[self-instruct-exploration]]:** Voyager proposes from open vocabulary; JARVIS-1 samples from a discrete task pool. Voyager runs sequentially; JARVIS-1 runs many environments in parallel against shared memory.
- **vs. intrinsic-motivation RL (Go-Explore, RIDE, etc.):** intrinsic-motivation works at the action level via shaped rewards; an automatic curriculum works at the *task-description level* and composes naturally with LLM planners.
- **vs. PAIRED / adversarial curriculum:** PAIRED trains an adversary network to propose environments; the automatic curriculum here is prompt-only and uses LLM priors instead of a trained adversary.

## Known limitations

- **Hallucinated tasks.** The curriculum sometimes proposes nonexistent items (the paper notes "copper sword"); LLM priors are not perfectly calibrated to the game's recipe tree.
- **Ablation impact only proven in one environment.** The 93% drop without the curriculum (Voyager Table 5) is measured in Minecraft; transfer to environments with less LLM prior knowledge is untested.
- **No explicit retry policy.** On repeated failure the curriculum is queried for another task, but there is no principled budget for revisiting hard tasks.
- **Single proposer.** One LLM call per task means the curriculum's diversity is bounded by the model's exploration of its own prior under a fixed prompt.

## Open problems

- How does the curriculum behave with **weaker / open-source LLMs** that have less Minecraft prior — does diversity collapse?
- Can the diversity directive be **learned or adapted** over the course of a run rather than held fixed?
- How should the curriculum interact with **multi-agent or shared-skill** settings, where many agents draw from the same library?
- **Cross-domain transfer:** the prompt-template is Minecraft-specific (inventory, biome, blocks). What is the general schema for state in non-game embodied domains?

## Relationship to foundations

Sits between classical curriculum learning (PAIRED, Portelas et al., Forestier et al.) and prompt-based LLM agents. The reformulation — "use the LLM both as world-knowledge prior and as task-sampler conditioned on agent state" — is the move that makes curriculum learning work in environments where no analytic difficulty function is available.

## My understanding

The contribution of Voyager's automatic curriculum is less the algorithm (sample-a-task-given-state is straightforward) and more the demonstration that, given GPT-4's prior, *the prompt itself is the curriculum-policy*. Ablating it costs 93% of unique items — strong evidence that the bottleneck for prior LLM agents in open-ended worlds was not model capability but the lack of a state-conditioned task generator.

The unsolved part is novelty pressure under weaker models. With GPT-4, the diversity directive in context is enough to keep proposals fresh; with smaller models that lack the Minecraft prior, the curriculum may drift to a narrow region of task space. Pairing this prompt-based curriculum with an explicit novelty signal (or with a retrieval-augmented task bank) is the natural next step.
