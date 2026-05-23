---
title: "JARVIS-1: Open-World Multi-Task Agents With Memory-Augmented Multimodal Language Models"
slug: jarvis-open-world-multi-task-agents
arxiv: "2311.05997"
venue: "IEEE Transactions on Pattern Analysis and Machine Intelligence"
year: 2023
tags:
  - agentic-memory
  - multimodal-memory
  - lifelong-learning
  - minecraft-agent
  - interactive-planning
  - self-instruct
  - retrieval-augmented-planning
importance: 4
date_added: 2026-05-19
source_type: tex
s2_id: fde266447d68b2728752a7835e8c97e1af316ac4
tldr: "JARVIS-1 is a Minecraft agent that augments a multimodal language model planner with a multimodal memory of past gameplay experiences, enabling situation-aware interactive planning and self-instruct lifelong learning that achieves a 5x reliability gain over VPT on long-horizon diamond-pickaxe tasks."
contribution_type:
  - method
  - system
datasets:
  - Minecraft Universe Benchmark
code_url: "https://craftjarvis.org/JARVIS-1"
cited_by: []
---

## Problem & Context

Building generalist agents that complete long-horizon tasks in open-world environments — and *keep improving* as gameplay progresses — is a key milestone toward functional AGI. Prior work in Minecraft and similar settings adopts hierarchical goal execution: an LLM-based high-level planner emits sub-goals, and a low-level controller executes them. This line of work (Inner Monologue, DEPS, Voyager, GITM, Plan4MC) made meaningful progress but bumped into three persistent walls in open-world environments:

1. **Multimodal sensing for planning.** LLM-based planners can ingest text but not visual observations, so their plans are blind to situation (location, biome, inventory, day/night, tool durability).
2. **Long-horizon planning robustness.** Tasks like `ObtainDiamondPickaxe` involve 20+ sub-goals with strict precondition chains; canonical LLM planners produce plans once and cannot recover from mid-execution failures without external scaffolding.
3. **Lifelong learning at scale.** With effectively infinite tasks, an agent must propose its own learning goals and improve over time. Gradient-based finetuning is prohibitively expensive given the number of new tasks and experiences.

Before JARVIS-1, VPT achieved roughly 2.5% reliability on `ObtainDiamondPickaxe` within 20 minutes (RL-finetuned from imitation on YouTube videos), and DEPS achieved roughly 0.59% with an LLM planner without learned memory.

## Key idea

Augment a multimodal language model (MLM) planner with a **multimodal memory** of successful past planning experiences (task + observation snapshot → executed plan), and let the agent *self-instruct* its own exploration curriculum to fill this memory across the gameplay lifetime. The planner retrieves relevant memory entries using both textual task description and current visual observation, then uses them as in-context demonstrations to ground its plans in the current situation. Because adaptation happens in-context (no gradient updates), the agent can keep learning indefinitely from its own experience.

The system also wraps planning in a closed loop: an initial plan goes through *self-check* (simulate sub-goal preconditions, fix bugs upfront) and, during execution, *self-explain* (diagnose environment-feedback failures and re-plan).

## Method

JARVIS-1 chains the MineCLIP visual encoder (MineDojo) with an LLM (default GPT-4) into a multimodal language model that produces situated plans. The architecture has three modules:

**Planner (MLM-based, interactive).** Visual observation is translated to text by extracting Minecraft-wiki keywords (e.g., "acacia tree", "sheep") and using GPT to compose natural-language scene descriptions from the matched keywords plus biome and inventory templates. The MLM emits a $K$-step sub-goal plan $g_1, \ldots, g_K$. *Self-check* simulates each sub-goal's predicted inventory state and verifies preconditions, fixing plan flaws before execution. During execution, *self-explain* (Reflexion-style) takes environment-feedback failures, locates the bug in the original plan, and emits a repaired plan.

**Multimodal memory (key-value).** Keys are multimodal: $(\text{task instruction}, \text{visual observation/situation})$. Values are the plan that was successfully executed under that key. Multiple entries can share a task with different situations and plans.

**Query generation via reasoning.** Given a task, the MLM backward-searches sub-goals (bounded depth) — e.g., for "craft 1 enchanting table with empty inventory" it reasons backward to "obtain book", "obtain diamond", "obtain obsidian". Sub-goals present in memory join the current visual observation to form the multimodal query.

**Multimodal retrieval.** CLIP text encoder filters memory entries by textual similarity above a confidence threshold; surviving candidates are then ranked by CLIP visual similarity between query observation and stored state, and the top entry per sub-goal is retrieved as in-context demonstration. Formally, with retrieval $p_\eta$ and planning $p_\theta$, $p(y \mid x) \approx \sum_{z \in \text{top-k}(p(\cdot \mid x))} p_\eta(z \mid x) p_\theta(y \mid x, z)$, where $z$ is a retrieved memory entry.

**Self-instruct exploration and self-improve.** A self-instruct loop generates a dynamic curriculum: each round, the MLM assesses the agent's current capability and proposes tasks from a task pool. Distributed agents execute these tasks in parallel with a shared centralized memory (speculative execution). Experiences are committed back to memory until the memory reaches a target capacity. Subsequent gameplay benefits from the accumulated memory without any gradient updates.

## Experiment & Results

**Benchmark.** 200+ tasks from the Minecraft Universe Benchmark (MCU) grouped into 11 categories (Wood, Stone, Iron, Gold, Diamond, …). Agent uses the native human Minecraft GUI (mouse/keyboard, 20 fps observations). At least 30 seeds per task; survival mode with empty starting inventory.

**Main comparative result (Table 1, average success rates per task group).** Against Instruct GPT, ReAct, Inner Monologue, DEPS:

- JARVIS-1 leads across all task groups, with the advantage widening as task complexity grows from Wood → Stone → Iron → Diamond.
- On Diamond tasks specifically: **8.99%** (JARVIS-1 with memory) vs **2.42%** (DEPS) — nearly 3x improvement.
- JARVIS-1 needs only 2–3 re-planning rounds vs DEPS's 6+ rounds, saving LLM tokens and execution budget.

**`ObtainDiamondPickaxe` (Figure 7, long-horizon).**
- 20-min budget: JARVIS-1 **6.22%** vs VPT-RL **2.5%** (2.5x improvement; "5x reliability" claim refers to a separate setting).
- 60-min budget: JARVIS-1 improves to **12.5%** while VPT barely changes (2.5% → 3%) — VPT-RL exhibits "perplexing behaviors" (wrong tool selection, useless crafts) after pickaxe damage, while JARVIS-1 re-plans from current inventory.
- Human baseline (skilled players, 10-min): ~15% diamond, ~12% diamond-pickaxe.

**Ablation on language model backbone (Figure 5).** ChatGPT roughly matches GPT-4 in JARVIS-1's framework — memory compensates for raw model capability. Pretrained LLaMA-2-70B underperforms substantially in long-horizon tasks (lacks Minecraft knowledge); finetuned LLaMA-2-13B on collected Minecraft text matches ChatGPT.

**Ablation on memory size (Figure 6).** Across 4 training epochs, accumulated successful trajectories grow to 425; success rates on key technology-tree items increase monotonically with memory size.

**Ablation on retrieval method (Figure 6, ablation_retrieval).**
- Text Memory only < Text Memory + Reasoning < Multimodal Memory + Reasoning.
- Reasoning before retrieval improves accuracy; multimodal (text + visual state) retrieval beats text-only.

**Self-improvement curriculum variants.** GPT-generated curriculum > human-written > random-generated, all evaluated after 4 epochs of self-instruct exploration.

## Limitations

- **Controller bottleneck.** The authors observe that diamond-tier failures often trace to the low-level controller's inability to execute short-horizon text instructions perfectly, not the planner — the memory-augmented planner has overrun the controller's capacity.
- **Retrieval cost scales with memory size.** The Language Model parameters never update; retrieval over the growing experience store gets more expensive over time.
- **Closed-API dependence by default.** Best results use GPT-4 (closed); open-source alternatives need either substantial parameter counts or task-specific finetuning to match.
- **Scene description via keyword matching.** Visual understanding is mediated by Minecraft-wiki keyword extraction rather than end-to-end visual reasoning, capping the granularity of "situation".
- **Domain-specific scaffolding.** Self-instruct uses a task pool from the Minecraft Universe Benchmark; generalization to other open worlds without a comparable task ontology is untested.

## Open questions

- How should procedural memory be *consolidated* as it grows — are all 425 trajectories worth keeping, or do redundant entries dilute retrieval quality? The paper does not study eviction or compression.
- Can self-instruct curriculum quality be measured intrinsically (without downstream task evaluation) to make the loop fully autonomous?
- When should the system fold accumulated experience back into the LLM's parameters via finetuning vs. keep everything in-context indefinitely?
- How does memory generalize across agent backbones — does the same memory help both GPT-4 and a finetuned LLaMA?
- How robust is multimodal retrieval to distribution shift (e.g., new biomes, new mod packs) the agent has never seen during exploration?

## My take

JARVIS-1 is the canonical worked example of "memory-augmented LLM agent succeeds where a fixed LLM planner stalls" in an open-world embodied setting. Its enduring contribution is less the Minecraft numbers and more the architectural template:

1. Decouple **what to remember** (task + situation key, plan value) from **how to use it** (in-context retrieval, not finetuning).
2. Decouple **planning** (MLM with self-check/self-explain) from **adaptation** (memory grows; LLM weights frozen).
3. Bootstrap memory with **self-instruct** so the curriculum tracks the agent's actual capability.

These three decouplings are now common scaffolding in subsequent open-world and tool-use agents. The paper sits squarely at the intersection of [[agentic-memory]], [[skill-evolution]], and [[procedural-memory]] — it's a procedural-memory system disguised as a Minecraft agent.

The weak link in retrospect is retrieval cost and memory consolidation: as the system runs longer, the in-context store grows unboundedly, and there is no mechanism for forgetting or summarization. That gap is exactly what later "skill consolidation" and "memory compression" work tries to fill.

## Related

- [[multimodal-memory]] — introduces this concept
- [[interactive-planning-with-self-check]] — introduces this concept
- [[self-instruct-exploration]] — introduces this concept
- [[jarvis-1-agent]] — system method
- [[multimodal-retrieval-augmented-planning]] — method
- [[zihao-wang]] — first author
- [[team-craftjarvis]] — author team
