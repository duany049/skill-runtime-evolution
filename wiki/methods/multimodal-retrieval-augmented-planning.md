---
name: "Multimodal Retrieval-Augmented Planning"
slug: multimodal-retrieval-augmented-planning
type: inference
tags:
  - retrieval-augmented-generation
  - multimodal-memory
  - llm-planning
  - in-context-learning
  - two-stage-retrieval
source_papers:
  - jarvis-open-world-multi-task-agents
parent_methods: []
child_methods: []
date_updated: 2026-05-19
---

## Problem setting

An embodied agent must produce plans grounded in both task description and visual situation. Pure parametric LLM planning ignores situation; pure RAG over a static text corpus ignores both situation and the agent's own past successes. The method is for any setting where (a) plan correctness depends on visual context, (b) the agent accumulates execution-validated experiences over time, and (c) the planner LLM stays frozen and adaptation happens via in-context retrieval.

## Mechanism

Approximate the posterior over plans $p(y \mid x)$ — where $x$ is the task instruction and $y$ is a plan — by marginalizing over retrieved experiences $z$ from a multimodal memory:

$$
p(y \mid x) \approx \sum_{z \in \text{top-k}(p(\cdot \mid x))} p_\eta(z \mid x) \, p_\theta(y \mid x, z)
$$

where $p_\eta$ is the retrieval model and $p_\theta$ is the planning model. The memory is keyed by $(\text{task}, \text{visual observation})$ pairs, and the retrieval model is a two-stage scorer:

1. **Textual filter** — embed the task instruction with CLIP-text; embed each memory entry's task key with CLIP-text; keep entries whose similarity is above a confidence threshold.
2. **Visual rank** — compute $p_\eta(z \mid x) \propto \text{CLIP}_v(s_z)^\top \text{CLIP}_v(s_x)$, where $s_x$ is the current visual query and $s_z$ is the stored visual key of each surviving candidate; rank by inner-product similarity.

The top-1 surviving entry per sub-goal is retrieved and concatenated into the planner's in-context prompt as a demonstration. Plans of retrieved entries are not executed verbatim — they serve as in-context examples to bias the planner's output.

## Procedure

1. **Sub-goal decomposition.** Given the task instruction, use the planner LLM to backward-reason over sub-goals (bounded depth). Each emitted sub-goal becomes its own retrieval query.
2. **Per-sub-goal multimodal query.** Concatenate the sub-goal text with the current visual observation to form the multimodal query.
3. **Stage 1 — textual filtering.** Encode both query and stored keys with CLIP-text; keep entries above a fixed similarity threshold.
4. **Stage 2 — visual ranking.** Encode query and surviving entries' visual states with CLIP-vision; rank by inner-product similarity.
5. **Top-1 retrieval per sub-goal.** Take the top-1 entry per sub-goal as the in-context demonstration for that sub-goal.
6. **Prompt assembly.** Concatenate task description, current situation, and the retrieved demonstrations into the planner prompt; generate the plan.
7. **Plan emission.** The planner emits a unified $K$-step plan; downstream self-check verifies sub-goal preconditions before execution.

## Assumptions

- Both visual and textual encoders are available and well-calibrated to the target domain (MineCLIP for Minecraft).
- Sub-goal decomposition via LLM reasoning is reliable enough to produce useful queries; otherwise stage-2 retrieval ranks against the wrong situation.
- The multimodal memory is large enough that the top-1-per-sub-goal entries are actually relevant; on a near-empty memory the method degrades to plain LLM planning.
- The planner LLM tolerates concatenated multi-shot in-context demonstrations without context-length problems.

## Limitations

- **Similarity ≠ utility.** The top-ranked entry is the most similar, not necessarily the most helpful for the current planning step.
- **Two-stage thresholding requires tuning.** The stage-1 similarity threshold is a hyperparameter; too tight and stage 2 starves, too loose and stage 2 sees noise.
- **Context budget.** One demonstration per sub-goal multiplies prompt length with plan depth; long horizons hit context limits in older LLMs.
- **No cross-sub-goal coherence.** Each sub-goal retrieves independently; nothing forces the retrieved demonstrations to be mutually consistent or composable.

## Tradeoff profile

- **Retrieval quality vs. latency.** Two-stage retrieval is more accurate than single-stage but doubles the encoder calls; for large memories, stage 1 must be implemented with an approximate nearest-neighbor index to stay tractable.
- **Demonstrations per sub-goal vs. context length.** Top-1 per sub-goal balances recall against context. Top-k > 1 improves recall but blows up prompt length.
- **Frozen LLM vs. continual finetuning.** The method makes adaptation purely in-context, trading parameter-efficient updates for retrieval cost that grows with memory size. The alternative (periodic LoRA on accumulated trajectories) trades the opposite way.
- **Domain-specialized encoders vs. generic CLIP.** Specialized encoders (MineCLIP) outperform generic CLIP on the target domain but bake in domain assumptions; the choice is a portability/quality tradeoff.
