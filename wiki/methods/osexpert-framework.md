---
name: OSExpert Framework
slug: osexpert-framework
type: system
tags:
  - computer-use-agent
  - gui-agent
  - environment-learning
  - skill-library
  - lora
  - fine-grained-action
source_papers:
  - osexpert-computer-use-agents-learning-professional
parent_methods: []
child_methods: []
code_repo: "https://github.com/Lumos-Jiateng/OSExpert"
date_updated: 2026-05-18
---

## Problem setting

Equip a general-purpose computer-use agent with environment-specific procedural knowledge so that it can complete long-horizon, fine-grained tasks in professional desktop applications (GIMP, LibreOffice, Tableau, MiniWord) with success rate and end-to-end latency approaching a human expert, *without* per-environment human demonstrations.

## Mechanism

OSExpert is a three-component pipeline that runs in two phases.

**Phase 1 — Environment learning (offline, per app):**

1. *GUI-DFS exploration* (Algorithm 1). A planner/action/feedback triad runs depth-first traversal over UI exploration-state nodes; verified `(plan, action)` pairs are condensed into unit-function skills in `K`; failures are recorded with retry budget `R=4`.
2. *Curriculum composition.* The agent self-proposes composite tasks rooted in already-discovered unit skills and runs a second exploration pass to verify them; verified composites are appended to `K`.
3. *Fine-grained primitive matching.* When the feedback module attributes a failure to fine-grained control, the agent searches a curated database of action primitives + grounding tools (SAM2, Qwen-3-VL grounding) for a candidate, runs it with verification, and stores the successful procedure (with its trigger condition) in `K`.
4. *Fast-planner training.* `(query → plan sequence)` pairs harvested from exploration are used to LoRA-fine-tune a Qwen-3-4B fast planner; only the LoRA weights are stored per environment.

**Phase 2 — Inference (online):**

5. *Skill-boundary check.* The incoming query is mapped against `K`'s failed entries; if a required skill is marked failed, the agent stops early.
6. *Fast plan + execute.* The Qwen-3-4B fast planner emits a complete plan in one forward pass; the base agent (Qwen-3-VL-8B) executes step by step using per-step perception.
7. *Fallback.* On execution failure, the system reverts to the general planning stack with relevant skills from `K` injected into the context for test-time scaling.

## Procedure

A single `/ingest`-relevant deployment looks like:

1. Pick a target application; assemble the planner/action/feedback modules and the primitive database.
2. Run GUI-DFS until the stack empties (or a time/cost budget is hit).
3. Run curriculum composition over the unit-function skills.
4. Train the fast planner on the collected `(query, plan)` pairs.
5. Serialize `K` + LoRA weights as the per-environment artifact.
6. At inference, load `K` + LoRA, run the skill-boundary check, then fast-plan-and-execute with fallback.

## Assumptions

- The target application allows programmatic reset, replay, and screen capture.
- An action module (e.g. UI-TARS-1.5-7B or Qwen-3-VL-8B) is available for low-level UI grounding.
- A feedback model that can classify Continue / Final / Error and attribute fine-grained failures is available.
- A database of fine-grained action primitives + grounding tools exists for the modality of interest.

## Limitations

- Compute-heavy exploration; not amortizable across applications that share only generic patterns.
- Skill-set quality is bounded by backbone capability — weak backbones produce noisy `K`.
- Manual curation of fine-grained primitives is still required.
- Skills are tied to a specific UI version; major UI changes can invalidate large portions of `K`.

## Tradeoff profile

OSExpert pushes the cost from inference time (where current CUAs spend 5–50× more wall-clock than humans on long-horizon tasks) to a one-off environment-learning phase. On OSExpert-Eval the trade is favorable: ~20% absolute success-rate gain on professional workflows and ~80% reduction in efficiency gap to humans, with the boundary check supplying most of the efficiency win and the primitive database supplying most of the fine-grained-action win. The framework is most valuable when the target environment will be used repeatedly (so the exploration cost amortizes) and least valuable for one-shot, throwaway tasks.
