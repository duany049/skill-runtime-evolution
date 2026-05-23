---
title: "Rethinking Agent Design: From Top-Down Workflows to Bottom-Up Skill Evolution"
slug: rethinking-agent-design-top-down-workflows
arxiv: "2505.17673"
venue: NeurIPS
year: 2025
tags:
  - bottom-up-agent
  - skill-evolution
  - llm-agent
  - learning-from-experience
  - open-ended-environment
  - mcts
  - visual-grounding
  - skill-library
importance: 4
date_added: 2026-05-18
source_type: tex
tldr: "Proposes a bottom-up agent paradigm where LLM agents autonomously acquire, refine, and prune a shared skill library from raw visual inputs and implicit rewards, evaluated zero-prior on Slay the Spire and Civilization V."
contribution_type:
  - method
  - position
  - system
datasets:
  - Slay the Spire
  - Civilization V
code_url: "https://github.com/AngusDujw/Bottom-Up-Agent"
cited_by: []
---

## Problem & Context

Most LLM-based agent frameworks (ReAct, Reflexion, AutoGPT, MetaGPT, ChatDev, Voyager) follow a **top-down design philosophy**: humans decompose a high-level goal into subtasks, define static or dynamic workflows, attach task-specific APIs, and assign agents to each node. The paper argues this paradigm inherits three structural constraints that block open-ended scaling: **staticness** (deployed replicas of a central prototype, updated by designers rather than learned adaptively); **prior dependency** (predefined APIs, workflows, and task-specific prompts that simply do not exist in open-ended environments); and **token inefficiency** (a large share of LLM tokens consumed enforcing the workflow rather than reasoning over lived experience). The authors situate this critique inside the broader [[era-of-experience]] vision of Silver and Sutton, which calls for a shift from curated human-trajectory imitation to learning from unstructured streams of experience, and inside the "second half of AI" framing that emphasizes evaluation on open-ended stream-based environments rather than static benchmarks.

## Key idea

Instantiate the era-of-experience vision through a **bottom-up agent paradigm** in which agents start with an empty skill library `S = ∅` and grow competence through a **trial-and-reasoning** loop: explore atomic actions, reflect on the visual consequences via an LLM, abstract the successful sequences into reusable skills, and prune or refine the library based on implicit reward (visual change, game progression). Skills, once discovered by any agent, propagate through a shared cloud-based library, so collective experience accelerates evolution. The paradigm is positioned as **complementary** to top-down design rather than as its replacement.

## Method

The agent models the environment as a POMDP `(X, A, T, R)` with visual-only observations `x_t` and atomic actions `A` (mouse clicks, drags, key presses). A **skill** is a sequence `σ = (a_1, ..., a_k)` paired with an LLM-generated semantic descriptor `d_σ`. The skill library `S` evolves as a population-level optimization:

`max_S E[ R_skill(σ, S, x_t, T) ]`, where `R_skill = R_diversity + R_efficiency + R_semantics`.

Three modules drive evolution:

- **Skill Augmentation.** When no candidate skill applies (`S_t = ∅`), the agent grows skills incrementally: starting from `k = 1` atomic actions, it appends a random action `σ^(k) = Augment(σ^(k-1), a_k)` and **retains only sequences that trigger observable visual changes**. This visual-change filter prunes the otherwise prohibitive action space.
- **Skill Invocation.** Given `x_t`, the LLM samples a candidate set `S_t = M(p_select, x_t, S)`, then **Monte Carlo Tree Search (MCTS)** simulates rollouts and selects the best skill `σ*` to execute.
- **Skill Evaluation and Refinement.** Implicit semantic reward `R_semantics = M(p_differ, σ, x_t, x_{t+1})` is computed from the LLM-judged visual diff between pre- and post-execution screenshots. Persistently low-scoring skills are pruned across the population; alternatively, the LLM rewrites a skill as `σ' = M(p_refine, (x_t, σ, T))` when viable candidates are sparse.

Visual grounding for atomic-action targeting uses the **Segment Anything Model (SAM)** to identify clickable UI elements and reduce the pixel-level click space. Prompts for augmentation, selection, refinement, and description are **deliberately environment-agnostic** — the identical codebase runs across both games with no game-specific priors.

## Experiment & Results

**Environments.** Two open-ended games chosen for the absence of explicit rewards, subgoals, or APIs: Slay the Spire (Ascension 0) and Civilization V ("Prince" difficulty). Agent perceives via screenshots and acts via simulated mouse events, no privileged state. LLM backend: GPT-4o; episode cap 1,000 steps (~6.5h). Each agent runs 3 episodes per environment.

**Main results (Table 1).** Under zero-prior, baselines (GPT-4o, Claude 3.7, UITARS-1.5) cannot progress in either game (1 floor / 5 score in Slay the Spire; 0 turns / 0 techs in Civilization V for most). Even with hand-engineered subgoals + UI priors, baselines reach at most 8 floors / 46 score and 17 turns / 3 techs. The bottom-up agent, under **zero prior**, clears **13 floors / 81 score / 98.56% execution responsive rate** in Slay the Spire and **50 turns / 8 techs / 92.27% responsive rate** in Civilization V, at $7.14 and $6.89 token cost respectively.

**Skill evolution (Table 2).** Across 4 training rounds of 100 steps in Slay the Spire, the library grows 0 → 59 → 70 → 96 → ~110 skills, with pruning rates of 1.67%, 31.25%, 0%, 6.25%. Progression rises from 6 → 8 floors, score 36 → ~48, responsive rate 93.14% → 96.58%, with token cost falling slightly. Skills converge after Round 4.

**Ablation (Table 3).** Removing **visual-change filtering** drops progression to 5 floors / 33 score / 89.29% (token cost falls). Removing **MCTS** is catastrophic: 1 floor / 5 score / 64.52% responsive — MCTS is critical for long-horizon planning. Removing **skill descriptions** drops to 5 floors / 22 score, hurting library reuse most.

## Limitations

- **Exploration overhead.** Zero-prior bootstrapping requires 2-2.5× more environment steps than prior-assisted baselines (~12 vs. 6 hours per 1,000 steps).
- **Reset and evaluation protocols.** Open-ended games lack reliable reset/seed mechanisms, introducing variance and complicating systematic comparison.
- **Perception of subtle changes.** Implicit reward via visual diff misses defensive or long-horizon strategies whose effects are not immediately visible.
- **Skill abstraction.** Skills remain flat record-and-replay action sequences, not parameterized functions; modularization conflicts with the environment-agnostic prompting goal.
- **Asynchronous multi-agent updates.** A shared globally-consistent skill library is assumed; concurrent edits and conflicting refinements are not addressed.
- **Skills are not cross-environment transferable.** Visual semantics, action consequences, and UI layout differences keep skills environment-specific despite shared architecture.
- **Turn-based games only.** Real-time environments are deferred because of current LLM inference latency.

## Open questions

- How can reinforcement-learning credit assignment over extended horizons supplement visual-diff implicit reward, so that defensive / long-term strategic skills are rewarded?
- How can record-and-replay skills be lifted into callable, parameterized functions without reintroducing environment-specific priors?
- Which decentralized consistency protocol (eventual consistency, versioned skills, trust-weighted refinement consensus) keeps the shared skill library coherent under massively parallel asynchronous edits?
- Can transfer or memory-based generalization across similar environments cut the 2-2.5× exploration overhead?
- What evaluation protocols allow controlled reset and reproducible comparison across open-ended games?

## My take

The paper is a clean position-plus-instantiation contribution. The position (top-down vs. bottom-up dichotomy, three structural constraints of top-down agents) is sharp and the proof-of-concept is convincing: zero-prior numbers crush prior-assisted baselines on two structurally different games using an identical codebase, which is the cleanest possible argument that the bottom-up framing actually carries weight rather than being mere reframing of Voyager-style skill libraries. The ablation pins down which mechanism does the work — MCTS-driven selection is the dominant component, not the visual-change filter or skill descriptions. The framing of `R_skill = R_diversity + R_efficiency + R_semantics` is closer to a design template than a learned objective, since only `R_semantics` is operationalized in the experiments, but the template is reusable. The most exposed weakness is **scale of evidence**: two turn-based games with GPT-4o is a narrow base for claiming a paradigm shift, and the "shared cloud library" only ever runs as a single-agent system in the experiments — the "collective evolution" story is unverified. If I were extending this, the first move would be to (a) test cross-agent skill diffusion empirically with 4-8 parallel agents writing to a shared library, and (b) introduce a learned implicit reward beyond visual diff to capture the defensive-play class of skills the authors flag.

## Related

- Concepts introduced: [[bottom-up-agent-paradigm]], [[trial-reasoning]], [[implicit-visual-reward]]
- Method introduced: [[bottom-up-skill-evolution]]
- People: [[jiawei-du]], [[joey-tianyi-zhou]]
