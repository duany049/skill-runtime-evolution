---
name: "JARVIS-1 Agent"
slug: jarvis-1-agent
type: system
tags:
  - minecraft-agent
  - memory-augmented-mlm
  - hierarchical-agent
  - llm-planner
  - multimodal-memory
source_papers:
  - jarvis-open-world-multi-task-agents
parent_methods: []
child_methods: []
code_repo: "https://craftjarvis.org/JARVIS-1"
date_updated: 2026-05-19
---

## Problem setting

Open-world embodied tasks in Minecraft, ranging from short-horizon ("chop trees") to long-horizon ("obtain a diamond pickaxe", ~20+ sub-goals with precondition chains). Observation is a stream of game frames at 20 fps plus inventory/biome metadata; actions are native mouse/keyboard inputs. The agent must generalize across 200+ tasks from the Minecraft Universe Benchmark without per-task imitation data or RL finetuning.

## Mechanism

JARVIS-1 is a three-module memory-augmented multimodal-language-model agent:

1. **Multimodal language model planner** — chains MineCLIP (visual encoder) with an LLM (default GPT-4) into an MLM that consumes (visual observation, task instruction, situation metadata) and emits a $K$-step sub-goal sequence.
2. **Multimodal memory** — a key-value store where keys are $(\text{task}, \text{visual observation})$ pairs and values are previously successful plans. Retrieval is two-stage: textual filtering by CLIP-text similarity, then ranking by CLIP-visual similarity.
3. **Goal-conditioned low-level controller** — executes each emitted sub-goal as native mouse/keyboard actions; the paper inherits the controller from prior work (Steve-1-style instruction-following or VPT-style policy).

The planner wraps execution in an interactive loop: an initial plan goes through *self-check* (simulate sub-goal preconditions, repair plan upfront), then executes; on environment failure, *self-explain* attributes the bug to a specific sub-goal and emits a repaired plan.

Lifelong improvement comes from *self-instruct exploration*: each round, the MLM assesses current capability and proposes a candidate task batch from a Minecraft Universe Benchmark task pool. Distributed agent instances execute the batch in parallel against different environment seeds (speculative execution) and write successful trajectories back to a shared centralized memory. The loop runs until the memory reaches a target capacity (425 trajectories after 4 epochs in the published configuration).

## Procedure

1. **Visual-to-text translation.** Extract Minecraft-wiki keywords from the current observation; use GPT to compose natural-language scene descriptions; append biome and inventory templates.
2. **Query generation.** Given the task instruction, reason backward via the MLM to identify sub-goals (bounded depth); each sub-goal contributes to the textual query.
3. **Memory retrieval.** Filter memory entries by CLIP-text similarity (above confidence threshold); rank surviving candidates by CLIP-visual similarity between query observation and stored state; retrieve top-1 per sub-goal as in-context demonstrations.
4. **Plan generation.** Prompt the MLM with task, situation description, and retrieved demonstrations; emit a $K$-step sub-goal sequence.
5. **Self-check.** Simulate each sub-goal's inventory transition; verify the next sub-goal's preconditions; if violated, prompt the MLM to repair the offending sub-goal; iterate until self-check passes.
6. **Execute.** Pass sub-goals to the low-level controller; observe environment feedback.
7. **Self-explain (on failure).** Prompt the MLM with the environment-feedback error message; identify the responsible sub-goal; emit a repaired plan; resume execution.
8. **Memory write (on success).** Append (task, observation snapshot, executed plan) to the shared memory.

Self-instruct exploration runs the same procedure in a loop with LLM-proposed tasks to grow the memory.

## Assumptions

- The environment exposes enough structure (Minecraft's recipe system, inventory state) for the LLM to reason about preconditions during self-check.
- CLIP encoders (visual and text) provide meaningful similarity in the target rendering domain; MineCLIP is used because generic CLIP underperforms on Minecraft visuals.
- A low-level controller capable of executing short-horizon text instructions (e.g., Steve-1 or VPT-derived) is available.
- The LLM has substantial Minecraft world knowledge in pretraining (true for GPT-4; less true for un-finetuned LLaMA-2-70B).
- A discrete task pool exists from which self-instruct can sample candidate tasks.

## Limitations

- **Controller bottleneck.** On diamond-tier tasks the authors observe that failures cluster at the controller's inability to execute sub-goals perfectly, not at the planner. JARVIS-1's planning quality has overrun controller capacity.
- **Retrieval cost grows with memory.** No eviction or consolidation policy; in-context store can grow unboundedly.
- **Closed-API dependence by default.** Best results use GPT-4; open-source backbones require Minecraft-specific finetuning to compete.
- **Visual perception via keyword matching.** Scene description is mediated by Minecraft-wiki keyword extraction; finer-grained visual understanding is bottlenecked here.
- **Discrete task pool requirement.** Self-instruct samples from a benchmark task ontology; environments without such a pool need additional scaffolding.

## Tradeoff profile

- **Compute vs. data:** Heavy at inference (multiple LLM calls per sub-goal for self-check / self-explain) and during distributed self-instruct exploration; zero gradient updates after pretraining.
- **Generality vs. domain scaffolding:** General architectural template (MLM + memory + interactive planning) but specific scaffolding (MineCLIP, Minecraft-wiki keyword library, MCU task pool) is Minecraft-bound. Ports to other open worlds require rebuilding the visual encoder and task pool.
- **Memory size vs. latency:** Larger memory → higher task success but linearly slower retrieval; no built-in mechanism to trade these off.
- **Plan rounds vs. environment cost:** Self-check costs LLM calls but saves environment steps that would otherwise fail; favorable on long-horizon tasks (2–3 rounds vs DEPS's 6+), unfavorable on trivial short-horizon tasks where the overhead is not amortized.
