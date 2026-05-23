---
title: "OSExpert: Computer-Use Agents Learning Professional Skills via Exploration"
slug: osexpert-computer-use-agents-learning-professional
arxiv: "2603.07978"
venue: arXiv
year: 2026
tags:
  - computer-use-agent
  - gui-agent
  - environment-learning
  - skill-library
  - bottom-up-exploration
  - benchmark
  - procedural-memory
importance: 4
date_added: 2026-05-18
source_type: tex
tldr: "OSExpert introduces an environment-learning paradigm where a computer-use agent runs a GUI-DFS exploration to autonomously discover, verify and consolidate per-app skills (with primitive-based fine-grained actions and a LoRA fast planner), reaching ~20% higher success on the new OSExpert-Eval benchmark and closing ~80% of the efficiency gap to human experts."
contribution_type:
  - method
  - benchmark
  - system
datasets:
  - OSExpert-Eval
  - OSWorld
  - GIMP
  - LibreOffice
  - Tableau Public
  - MiniWord
code_url: "https://github.com/Lumos-Jiateng/OSExpert"
cited_by: []
---

## Problem & Context

General-purpose computer-use agents (CUAs) such as Agent-S3, OpenCUA, CoAct-1 and Computer-Use-Preview are reported to approach human-level performance on OSWorld and similar benchmarks. The authors argue that this paints an overly optimistic picture: existing benchmarks emphasize short, single-function tasks in familiar UIs, while real professional workflows demand (1) long-horizon composite skills, (2) generalization to unseen and creative UIs, (3) fine-grained low-level actions with pixel-precise control, and (4) end-to-end efficiency comparable to human experts. The paper attributes today's gap to how CUAs are trained — large-scale behavior cloning + RL on human demonstrations across ~100 environments — which fails to install environment-specific procedural knowledge and forces agents to re-plan and trial-and-error at inference time, yielding 5–50× higher latency than humans and sharp performance cliffs on long-horizon and unseen tasks. Prior self-evolving GUI agents (AppAgent-v2, SEAgent, Mobile-Agent-E, WebEvolver, UI-Evol) acquire skills only along user-query-driven trajectories, leaving coverage incomplete and skill sets brittle.

## Key idea

Treat each digital environment as a first-class learning target: before deployment, the agent enters a goal-free *environment-learning* phase in which it explores the application bottom-up, discovers unit functions, composes them into a curriculum of composite tasks, and verifies fine-grained action primitives, all without per-environment human annotation. The resulting verified *skill set* — together with a small LoRA-tuned single-pass planner and an explicit skill-boundary check — is then loaded at inference time so the agent executes verified procedures instead of re-deriving them step-by-step.

## Method

OSExpert has three components:

1. **GUI-DFS exploration (Section 3.1; Algorithm 1).** Three coordinated modules — *planning*, *action*, *feedback* — run a depth-first search over UI exploration-state nodes. Each node stores `(plan sequence Π, action sequence α)`. The environment is reset, `α` is replayed, the action module proposes `α'`, and the feedback module classifies the resulting state as Continue / Final / Error. Continue states push successor plans; Final states condense `(Π, α⊕α')` into a unit-function skill in `K`; Error states return targeted critiques and retry up to `R=4` times. After unit-function discovery the agent self-proposes a curriculum of composite tasks rooted in known skills, validates them, and appends them as composite skills.
2. **Fast planner + skill-boundary check (Section 3.2).** A Qwen-3-4B model is LoRA-tuned on `(query → planning sequence)` pairs harvested from exploration so it can emit an entire plan in a single forward pass; only environment-specific LoRA weights are stored. A *skill-boundary check* maps an input query against the failed entries of `K`: if the required skill was repeatedly unreachable during exploration, the agent stops early instead of running blind test-time scaling. The fast plan is followed greedily and falls back to the general planning stack on cache-miss.
3. **Fine-grained action primitives (Section 3.3).** A curated database of named action-sequence primitives + grounding modules (SAM2 for segmentation, Qwen-3-VL for grounding verification) is consulted whenever the feedback module raises an error attributed to fine-grained control. The chosen primitive is run with verification; successful primitive-instances are condensed into the skill set with a usage condition (e.g. GIMP scissor-select traces a contour via SAM2 + click sequence).

During exploration the high-level planner+feedback can be either GPT-5 + UI-TARS-1.5-7B or Qwen-3-VL-8B end-to-end; the inference-time base agent is Qwen-3-VL-8B with the Qwen-3-4B fast planner.

## Experiment & Results

The paper introduces **OSExpert-Eval**: 113 tasks across six environments (LibreOffice Writer/Calc/Impress, GIMP, Tableau Public, in-house MiniWord) covering long-horizon composite skills, unseen-UI generalization (Tableau, MiniWord), and fine-grained actions; tasks are capped at 30 interaction steps with single runs (three independent runs in the main tables) and no Best-of-N.

Headline numbers (avg success rate, ± std over three runs):

- Long-horizon GIMP: OSExpert 0.33 vs. Agent-S3 w/ GPT-5 0.06, CoAct-1 0.00, OpenCUA-7B 0.00.
- Long-horizon LibreOffice: OSExpert (GPT-5+UI-TARS) 0.31 vs. Agent-S3 0.08.
- Unseen-UI Tableau: OSExpert 0.25 vs. Agent-S3 0.03; MiniWord: OSExpert 0.37 vs. CoAct-1 0.06.
- Fine-grained GIMP: OSExpert 0.28 vs. Agent-S3 0.00 (no fine-grained primitives → 0.14, so the primitive database is responsible for the GIMP fine-grained gain).
- Fine-grained LibreOffice: OSExpert 0.26 vs. baselines 0.05–0.10.

Efficiency (avg seconds per task, three runs):

- Human expert: 11–48 s across subsets.
- Agent-S3 / CoAct-1: 600–1260 s (≈5–50× slower than humans).
- OSExpert: 27–44 s — within ~2× of humans, closing roughly ~80% of the efficiency gap.
- Ablations: removing the fast planner adds ~10 s; removing the skill-boundary check roughly doubles latency (e.g. GIMP long-horizon 32 s → 84 s) — boundary check supplies most of the efficiency gain, fast planner is secondary.

Qualitative analysis: gains on long-horizon and unseen UIs come primarily from environment-specific procedural knowledge in the skill set (verified plans reduce cascade failures); gains on fine-grained subsets come from the primitive database.

## Limitations

The authors list five (Section: Limitations):

- **Exploration cost.** GUI-DFS still requires repeated resets, replays, and verification; for applications with deep menus / high branching, exploration is time- and compute-intensive.
- **Base-model dependence.** Skill-set quality is bounded by the planner / action / feedback models used during exploration; weaker backbones yield noisier skills.
- **Coverage / brittle failure modes.** Under finite budget, DFS does not exhaustively cover all paths; failures recorded for early-stopping may reflect policy limits, not true infeasibility.
- **Manual primitive design.** Fine-grained handling still requires humans to enumerate recurring manipulation patterns and seed action primitives + grounding tools.
- **Evaluation scope.** OSExpert-Eval is desktop / GUI only — no mobile, no real-time, no purely web-native automation.

## Open questions

- Can the failed-skill labels in `K` be distinguished between *agent-policy failures* and *true environment infeasibility*? Otherwise the skill-boundary check risks ossifying weak-agent blind spots.
- The fast planner only emits planning sequences; per-step perception and action selection still call the base agent. What is the right unit of caching — plan, plan+action, or full trajectory?
- Composite skills are generated by the agent's own curriculum. How sensitive is downstream performance to curriculum quality, and how does it degrade with weaker exploration backbones?
- Can the database of fine-grained action primitives be discovered / induced from interaction rather than hand-curated?
- How well does the skill set transfer across application versions, themes, or sibling apps (Photoshop ↔ GIMP)? The paper measures only intra-environment reuse.

## My take

OSExpert is a clean instance of the "build skills, not agents" position (citing Anthropic's Zhang et al.) operationalized as a concrete pre-inference training loop. The strongest part is the empirical separation of *knowledge* vs. *efficiency* gains: the skill-boundary check alone explains most of the 80% latency reduction, which is a useful, unflashy finding that argues for capability-aware early stopping as a generally-useful CUA building block. The weakest seam is the dependence on GPT-5 + UI-TARS as exploration backbones — when the Qwen-3-VL-8B-only configuration is used, several numbers collapse to the same value as the strong-explorer setting, which raises questions about how much of the gain truly comes from the exploration loop versus from a strong planner being able to plan reliably on these apps in the first place. The benchmark itself looks small (113 tasks, six environments) but the four-dimensional split is genuinely under-served by existing benchmarks (cf. Table 1).

## Related

- [[procedural-memory]]
- [[skill-evolution]]
- [[agentic-memory]]
- [[gui-dfs-exploration]]
- [[environment-learned-agent]]
- [[skill-boundary-check]]
