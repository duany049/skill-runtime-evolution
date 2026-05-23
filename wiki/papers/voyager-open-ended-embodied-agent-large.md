---
title: "VOYAGER: An Open-Ended Embodied Agent with Large Language Models"
slug: voyager-open-ended-embodied-agent-large
arxiv: "2305.16291"
venue: TMLR
year: 2023
tags:
  - lifelong-learning
  - embodied-agent
  - skill-library
  - llm-agent
  - minecraft
  - automatic-curriculum
  - code-generation
  - procedural-memory
importance: 5
date_added: 2026-05-18
source_type: tex
tldr: "Voyager is the first GPT-4-driven lifelong learning agent in Minecraft, combining an automatic curriculum, an ever-growing executable skill library, and iterative code refinement with self-verification to outperform prior LLM agents by 3.3x in unique items and unlock the diamond tech tree."
contribution_type:
  - method
  - system
datasets:
  - MineDojo
code_url: "https://voyager.minedojo.org"
cited_by: []
---

## Problem & Context

Building generally capable embodied agents that continuously explore, plan, and develop new skills in open-ended worlds is a long-standing AI grand challenge. Before this paper, the landscape split into two unsatisfying camps:

- **Low-level RL / imitation learning controllers** (VPT, DreamerV3, hierarchical-RL Minecraft agents) operate on primitive actions and struggle with systematic exploration, interpretability, and generalization. They typically require large-scale demonstrations or pre-training, and are tied to fixed task curricula.
- **LLM-based planners for embodied tasks** (Code-as-Policies, ProgPrompt, SayCan, Inner Monologue) and for NLP agents (ReAct, Reflexion, AutoGPT) can leverage world knowledge from pre-trained LLMs to generate plans or executable policies, but **none are lifelong learners** — they cannot progressively acquire, update, accumulate, and transfer knowledge over extended time spans. They also lack any persistent skill memory.

Minecraft is the natural testbed: an open-ended 3D world with no predefined end goal, requiring agents to traverse vast terrains and unlock a hierarchical tech tree from raw materials to diamond tools. An effective lifelong-learning agent should (1) propose tasks suited to its current skill level and world state, (2) refine skills from environmental feedback and commit mastered skills to memory, and (3) continually explore on its own initiative — capabilities prior LLM agents collectively lacked.

## Key idea

Drive lifelong embodied learning entirely through **blackbox prompting of GPT-4** by combining three interlocking modules:

1. an **automatic curriculum** that asks GPT-4 to propose the next task conditioned on the agent's state, completed/failed tasks, and a directive to "discover as many diverse things as possible" — an in-context form of novelty search;
2. an **ever-growing skill library** of executable JavaScript programs (Mineflayer API), each indexed by the embedding of its natural-language description, allowing retrieval-augmented composition of new skills from old ones;
3. an **iterative prompting mechanism** that refines generated code using three feedback channels — environment observations, code-execution errors, and a separate GPT-4 self-verification critic — until the critic confirms success, at which point the program is committed to the library.

Programs (rather than tokens or low-level controls) as the action space give the agent temporally extended, interpretable, and compositional skills, which **alleviates catastrophic forgetting** because old skills remain in the library and can be reused or composed without retraining.

## Method

**System pipeline.** The automatic curriculum proposes a task → the skill library retrieves relevant prior programs → GPT-4 generates a candidate program with chain-of-thought reasoning → the program executes in MineDojo via Mineflayer JavaScript APIs → environment feedback and execution errors are fed back into the next round of code generation → a separate GPT-4 self-verification agent judges success and either commits the skill to the library (indexed by embedding of its description) or provides critique for another iteration. Up to 4 rounds per task; otherwise the curriculum is queried for a new task.

**Automatic curriculum prompt.** Inputs: (a) directives encouraging diverse exploration with difficulty constraints, (b) the agent's current state (inventory, equipment, biome, nearby blocks/entities, time, health/hunger, position), (c) lists of previously completed and failed tasks, and (d) self-asked context questions answered by GPT-3.5 for budget. Temperature 0.1 for task diversity; all other temperatures are 0.

**Skill library.** Each new skill is stored as JavaScript code with a natural-language description. Skills are retrieved by querying the library with the embedding (text-embedding-ada-002) of the current task plan plus environment feedback, and the top matches are injected into the code-generation prompt as in-context examples. Complex skills (e.g., `combatZombieWithSword()`) emerge by composing simpler ones (`craftStoneShovel()`).

**Iterative prompting.** Three feedback types: (1) **environment feedback** via `bot.chat()` (e.g., "I cannot make an iron chestplate because I need: 7 more iron ingots"), (2) **execution errors** from the JS interpreter, and (3) **self-verification** by a separate GPT-4 critic that decides task success and emits a critique on failure — strictly more comprehensive than Reflexion's self-reflection because it both grades success and reflects on mistakes.

**Models.** GPT-4 (`gpt-4-0314`) for code generation and self-verification; GPT-3.5 (`gpt-3.5-turbo-0301`) for cheap NLP self-Q&A; `text-embedding-ada-002` for skill embeddings.

## Experiment & Results

**Setup.** MineDojo simulator + Mineflayer JS APIs. Baselines re-interpreted to be executable in Minecraft: ReAct, Reflexion, AutoGPT (re-implemented with GPT-4 for task decomposition). No pixel-input methods compared because the controllers are not apples-to-apples.

**Main exploration results.** Within 160 prompting iterations, Voyager discovers **63 unique items**, a **3.3x improvement** over the strongest baseline. ReAct and Reflexion barely progress because the open-ended exploration goal is too abstract without an automatic curriculum; AutoGPT improves but lags.

**Tech tree mastery.** Voyager unlocks wooden tools **15.3x faster**, stone **8.5x faster**, iron **6.4x faster** (all measured in prompting iterations) than baselines, and is the **only method that unlocks the diamond level** of the tech tree.

**Map traversal.** Voyager navigates **2.3x longer distances** than baselines across multiple terrains; baselines tend to get stuck in local regions.

**Zero-shot generalization.** With inventory cleared and a fresh world, Voyager solves all evaluated unseen tasks within 50 iterations while baselines solve none. Plugging Voyager's learned skill library into AutoGPT also boosts AutoGPT's performance, confirming the library is a transferable, plug-and-play asset.

**Ablations.** Ablating the automatic curriculum collapses item count by **93%**; removing self-verification drops it by **73%** (the single most impactful feedback channel); replacing GPT-4 with GPT-3.5 for code generation yields **5.7x fewer** unique items; removing the skill library causes the agent to plateau in late iterations.

**Multimodal extension.** Voyager has no built-in visual perception (text-only GPT-4 API at the time). When humans act as visual critic or curriculum, Voyager additionally constructs complex 3D structures such as a Nether Portal and a house.

## Limitations

- **Cost.** GPT-4 API is ~15x more expensive than GPT-3.5; results depend on that quality gap.
- **Code-generation failures.** The agent occasionally gets stuck and fails to generate correct skills despite iteration; the curriculum mitigates by retrying later.
- **Self-verification errors.** The critic occasionally misjudges success (e.g., failing to recognize spider string as evidence of beating a spider).
- **Hallucinations.** The curriculum sometimes proposes nonexistent items ("copper sword"); GPT-4 sometimes invokes nonexistent control primitives or treats cobblestone as fuel.
- **No native visual perception.** The system depends on Mineflayer's symbolic state; spatial-detail tasks fail without human visual feedback.
- **Single-environment study.** Empirical results are only in Minecraft; transfer to robotics or other open-ended worlds is argued but not demonstrated.

## Open questions

- Can the skill library scale to thousands of skills without retrieval collapse or library bloat? The paper does not stress-test scaling.
- Are the skills genuinely *evolved* (refined post-commit) or only *one-shot generated*? The library appends but does not appear to revise committed skills.
- How does Voyager interact with open-weight LLMs that exhibit weaker code-generation? Can finetuning close the gap that GPT-4 currently fills?
- What is a principled retirement / deduplication policy for the skill library as it grows? The paper acknowledges but does not study this.
- How portable is the framework to non-game embodied domains where reward and feedback channels are noisier and partially observable?

## My take

Voyager is the canonical reference point for LLM-driven lifelong learning with a programmatic skill library, and a load-bearing antecedent for almost every later "agent that learns skills from its own trajectories" line of work. Its real contribution is less the individual modules (curriculum, library, critic each have ancestors) and more the **integration**: showing that a tight feedback loop between a curriculum, a code-based skill memory, and a self-critic can produce open-ended, compositional skill growth from a frozen LLM. That is the lesson that survives even if specific design choices (JS skills, embedding retrieval, GPT-4 critic) get replaced.

The ablations are unusually informative. The 93% drop without curriculum and 73% drop without self-verification are strong evidence that the bottleneck in prior LLM agents was *not* base-model capability but missing scaffolding. This frames "skill evolution" and "agentic memory" research as primarily a systems-architecture problem rather than a model-scaling problem.

What the paper does *not* show — and what subsequent work has had to grapple with — is whether the skill library actually *evolves* (skills get refined post-commit) versus merely *accumulates*. This gap is exactly what motivates the wiki's `skill-evolution` topic.

## Related

- [[automatic-curriculum]]
- [[executable-skill-library]]
- [[iterative-prompting-environment-feedback]]
- [[voyager-system]]
- [[llm-self-verification-critic]]
- [[guanzhi-wang]]
- [[linxi-jim-fan]]
- [[anima-anandkumar]]
- [[yuke-zhu]]
