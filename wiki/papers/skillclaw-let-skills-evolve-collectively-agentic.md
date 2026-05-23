---
title: "SkillClaw: Let Skills Evolve Collectively with Agentic Evolver"
slug: skillclaw-let-skills-evolve-collectively-agentic
arxiv: "2604.08377"
venue: arXiv preprint
year: 2026
tags:
  - skill-evolution
  - agentic-memory
  - procedural-memory
  - multi-user-agents
  - llm-agents
  - skill-library
  - collective-evolution
importance: 4
date_added: 2026-05-18
source_type: tex
s2_id: 3d795acefb5939ec88767d9a01dffb765fb90111
tldr: "SkillClaw aggregates session trajectories from many users of OpenClaw-style agents, then uses an agentic evolver to refine or create shared skills that propagate back to all agents, demonstrating consistent gains on WildClawBench across Social, Search, Creative, and Safety categories."
contribution_type:
  - system
  - method
datasets:
  - WildClawBench
code_url: "https://github.com/AMAP-ML/SkillClaw"
cited_by: []
---

## Problem & Context

LLM-agent platforms such as OpenClaw ship reusable **skills** — structured procedural artifacts that encode how to drive tools, format API arguments, and chain multi-step workflows. Once installed from a centralized skill hub, these skills remain effectively **static**: improvements discovered while a single user struggles through trial-and-error never leave that session, even though the *same* failure modes (wrong argument format, mismatched tool call, missing validation step) recur across users and over time.

Existing approaches to agent adaptation fall short for this regime:

- **Memory-based methods** (Reflexion, ExpeL, MemP, Mem0, ReasoningBank, SimpleMem) store past trajectories for retrieval, but the records stay tied to specific instances and are hard to generalize.
- **Skill-based methods** (SkillRL, MemEvolve, Evolver, MemSkill) compress experience into structured instructions, yet treat the resulting library as a *static* resource that does not evolve through usage.
- **Local refinement** can improve individual agent instances, but improvements stay isolated and do not accumulate across users — the system as a whole never gets smarter.

The key challenge the authors identify is therefore not just intra-session improvement, but **cumulative cross-user skill evolution**: a mechanism that converts ordinary multi-user interactions into continuous, validated updates to a shared skill repository.

## Key idea

> *Different users exercising the same skill under diverse contexts produce complementary views of that skill's behavioral boundary.*

A single user cannot reliably distinguish a generalizable fix from an idiosyncratic patch; aggregating trajectories across users provides the grounding. SkillClaw operationalizes this in a closed loop:

```
Multi-user Interaction → Session Collection → Skill Evolution → Skill Synchronization
```

- All deployed agents share a common skill repository.
- Each user interaction is recorded as a structured **session trajectory** preserving the full causal chain `prompt → action → feedback → ... → response`.
- Trajectories are **grouped by referenced skill**, exposing both consistent success patterns and recurring failure modes for the *same* skill under varied conditions — a natural ablation with the skill as the controlled factor.
- An **agentic evolver** (an LLM agent with a structured harness, not a hard-coded pipeline) reasons over each group and emits one of three actions: `Refine`, `Create`, or `Skip`.
- Sessions that invoke no skill (group $\mathcal{G}(\varnothing)$) are mined for *missing but reusable* procedures.
- Updates are validated overnight in real user environments (full toolchain, multi-step interactions). Only updates with better task success and execution stability are `Accept`-ed; users always interact with the *best validated* pool from the previous night, never unverified candidates.

The three distinguishing properties claimed are: **collective evolution** (shared improvement across users), **full automation** (no manual curation or explicit user input beyond normal agent usage), and **agentic adaptability** (open-ended reasoning rather than predefined update rules).

## Method

Let $\mathcal{S} = \{s_1, \dots, s_M\}$ be the shared skill set and $\mathcal{T} = \{\tau_i\}$ the trajectories aggregated across users. The goal is

$$\mathcal{S}' = \Phi(\mathcal{S}, \mathcal{T}).$$

**Stage 1 — Isolated sessions to shared evidence.** Each raw session is converted into a structured representation preserving the full action-feedback trace; lightweight metadata flags (i) which skills were referenced, (ii) whether tool errors occurred, (iii) a coarse quality estimate. The trajectories are then grouped:

$$\mathcal{G}(s) = \{\tau_i \mid s \in \mathcal{K}_i\}, \quad \mathcal{G}(\varnothing) = \{\tau_i \mid \mathcal{K}_i = \emptyset\}.$$

The skill-conditioned grouping is what enables (1) evaluating an existing skill under diverse real-world usage and (2) surfacing recurring procedures that no current skill covers.

**Stage 2 — Agentic skill evolution (Algorithm 1).** For each $\mathcal{G}(s)$ the evolver examines both successes and failures *jointly*: successful sessions define the **invariants** that must not be broken; failed sessions define the **targets** that need correction. This joint view is the explicit guard against the naive failure mode of "fix one problem, break another." It selects from `{Refine, Create, Skip}`:

- **Refine** an existing skill's text to correct identified errors / strengthen robustness.
- **Create** a new skill when $\mathcal{G}(s)$ reveals a recurring, teachable sub-procedure no existing skill covers.
- **Skip** when evidence is insufficient.

For $\mathcal{G}(\varnothing)$, the evolver focuses on discovering missing but reusable procedures, with conservative editing to keep updates targeted.

**Stage 3 — Validation and deployment loop.** Candidate updates $s'$ are run in idle user environments overnight on tasks selected from that day's interaction data, using the full toolchain. An LLM judge compares $s$ vs $s'$ on overall task success and execution stability. `Accept`-ed updates merge into the shared pool and are synced to all agents for the next day; `Reject`-ed updates are kept as records only. Because only improvements pass, the deployed pool is **monotonic** by construction:

$$\text{Interaction} \to \text{Evidence} \to \text{Evolution} \to \text{Validation} \to \text{Deployment}.$$

The harness supplies inputs (grouped evidence, current skill definitions, permitted actions) but does not constrain the evolver's reasoning, which is what lets one framework handle heterogeneous failure modes without hand-crafted rules per type.

## Experiment & Results

**Setup.** WildClawBench: 60 complex real-world agent tasks across six domains — Productivity Flow, Code Intelligence, Social Interaction, Search & Retrieval, Creative Synthesis, Safety & Alignment — executed end-to-end in a full Linux container with multimodal tool usage. Task lengths run 15–50 steps; evaluation aggregates 3–27 metrics with hard constraints (critical errors → zero score). All execution, evolution, and validation use **Qwen3-Max**. The simulation runs 6 day–night rounds with 8 concurrent users; Day 1 is the baseline skill set. Results are reported on four categories (Social, Search, Creative, Safety); two are deferred.

**Main results (user-side daytime, Day 1 → Day 6 best pool, abs / rel gain):**

| Category            | Day 1   | Day 6   | Abs gain | Rel gain |
|---------------------|---------|---------|----------|----------|
| Social Interaction  | 54.01%  | 60.34%  | +6.33    | +11.72%  |
| Search & Retrieval  | 22.73%  | 34.55%  | +11.82   | +52.00%  |
| Creative Synthesis  | 11.57%  | 21.80%  | +10.23   | +88.41%  |
| Safety & Alignment  | 24.00%  | 32.00%  | +8.00    | +33.33%  |

Each category exhibits a distinct evolution trajectory rather than a uniform pattern:

- **Social Interaction** improves earliest and sharpest — a single accepted Night 1 rewrite of `03_task6` (executive summarization workflow) drives the entire gain; later candidates are all `Reject`-ed.
- **Search & Retrieval** improves in stages: Night 1's `validate-file-existence` lifts the floor (input reliability), Night 3 confirms the current best pool, yielding the largest relative gain in the table (+52%). The pattern is *input-first, strategy-later*.
- **Creative Synthesis** jumps once (Night 1's `validate-tmp-workspace-inputs`) and plateaus: the bottleneck was environment setup (working directory, symlinks, multimodal pipelines), not generation itself.
- **Safety & Alignment** improves later through a *reliability-driven* sequence: `git-push-with-auth-fallback` accepted Nights 1–2, `git-clone-to-directory` accepted Night 3, retest accepted Night 4. Nights 5–6 candidates rejected.

**Controlled validation (Skill Evolve Lite, Table 8):** three isolated queries that target specific failure modes — `basic extraction` 21.7% → 69.6% (+47.8), `deadline parsing` 41.1% → 48.0% (+6.9), `save report` 28.3% → 100.0% (+71.7). Average 30.4% → 72.5% (+42.1). Tasks dominated by procedural knowledge (save report, basic extraction) gain much more than tasks dominated by nuanced reasoning (deadline parsing) — direct mechanism-level evidence for *why* skill evolution helps.

**Case studies** (Figures 2–5) illustrate qualitative changes: Slack message analysis (filtering → selective retrieval pipeline plus correct API port), ICCV 2025 oral paper counting (strict "first affiliation" definition + OpenAccess alignment + targeted re-checks), SAM3 under incomplete environments (workspace inspection + non-blocking missing paths + CUDA→CPU adaptation), multi-criteria product selection (constraint-aware verification + calibrated "no candidate fully satisfies" reporting).

## Limitations

- **Small-scale evaluation.** Only 4 of 6 WildClawBench categories are reported (Productivity Flow and Code Intelligence deferred to a future version). 8 simulated users, 6 rounds, 60 tasks — the authors themselves frame this as "a small-scale test of collective skill evolution, with limited user queries, feedback signals, and interaction depth."
- **Single backbone.** All execution, evolution, and validation use Qwen3-Max. Whether the agentic evolver, the validator's accept/reject judgements, and the resulting skill text generalize across model families is untested.
- **Validation cost.** Overnight validation re-runs candidate skills with full tool interaction in real environments, which the paper explicitly notes "introduces additional token cost." No accounting of this overhead vs the daytime serving cost is provided.
- **Plateau effect not analyzed.** Three of four categories plateau after a single Night 1 acceptance — the framework has limited "headroom" within the 6-day window once the dominant bottleneck is fixed. Whether longer horizons or richer trajectories would unlock further evolution is left as conjecture.
- **No comparison to memory/skill baselines.** The paper situates SkillClaw against Reflexion, ExpeL, MemP, SkillRL, MemEvolve, Voyager-style libraries, etc. in related work, but reports no head-to-head comparison on WildClawBench. The reported gains are vs the Day-1 static skill pool only.
- **Validator vulnerability.** The validator is itself an LLM ("the system uses the model to compare the outcomes"). A miscalibrated judge could accept locally-better-but-globally-worse updates; the paper does not characterize judge error rates.
- **Privacy / heterogeneity not addressed.** Aggregating trajectories across users is the core mechanism, but the paper does not discuss what is shared in those trajectories, anonymization, or how users with very different task distributions are reconciled into one shared skill pool.

## Open questions

- Does collective skill evolution still help when users have **disjoint** task distributions, or does the shared pool degenerate into a lowest-common-denominator skill set?
- How does the **validator** scale when candidate-update volume grows? The current design re-runs candidate skills end-to-end overnight; this is O(candidates × tasks × steps) and may not stay feasible at platform scale.
- Can the **agentic evolver** itself be evolved? The paper fixes the evolver's harness; whether the harness rules should themselves be updated from accumulated meta-evidence is open.
- What is the **privacy contract** for cross-user trajectory aggregation in production? The framework as described assumes raw `prompt → action → feedback → response` traces are uploadable; many real deployments cannot do that.
- How does this interact with **forgetting / retirement**? The paper only describes additive updates (Refine, Create, Skip). Whether evolved skills should also be **deprecated** as task distributions drift, and what evidence triggers retirement, is not addressed.
- Is **monotonic deployment** truly safe? Accepting only updates that beat the current best on yesterday's task slice is a greedy criterion; a candidate could be locally rejected but globally better on tasks not present that day.

## My take

SkillClaw is best read as an **engineering reframing** of skill evolution rather than a fundamentally new learning mechanism: the loop `aggregate trajectories → group by skill → agentic edit → overnight validate → sync` is conceptually straightforward, but the specific design choices that make it work — preserving full causal chains, grouping by referenced skill as a natural ablation, reasoning over success and failure *jointly* to keep invariants intact, and validator-gated monotonic deployment — are individually defensible and together form a credible recipe for production agent platforms. The paper is up-front about the small-scale setting, and the Skill Evolve Lite controlled validation (Table 8) is the most useful evidence: the +42.1 average gain across three procedurally-dominated queries is a direct mechanism check that the framework does what it claims for the failure modes it targets. The plateau-after-Night-1 pattern in three of four categories is the most interesting tension — it suggests either that 6 days × 8 users is too thin a slice to surface secondary bottlenecks, or that the validator is too conservative once a strong skill enters the pool. Either way, the system-level claim ("static skill libraries should become living artifacts continuously updated from deployment evidence") is well-motivated, and the closed-loop design is one of the cleaner concrete instantiations of that claim in the recent agentic-memory literature.

## Related

- [[collective-skill-evolution]] — concept introduced by this paper.
- [[agentic-evolver]] — concept introduced by this paper.
