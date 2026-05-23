---
title: Interactive Planning with Self-Check
aliases:
  - self-check planning
  - upfront plan verification
  - self-check and self-explain
  - closed-loop LLM planning
tags:
  - interactive-planning
  - llm-planning
  - self-verification
  - closed-loop-control
  - long-horizon-planning
maturity: active
definition: A closed-loop planning pattern in which an LLM-based planner produces an initial plan, simulates and verifies preconditions for each sub-goal to catch errors before execution (self-check), and on execution failure uses environment feedback to reason about and repair the offending step (self-explain).
key_papers:
  - jarvis-open-world-multi-task-agents
first_introduced: "JARVIS-1 (2023) as a named two-stage scheme; self-check builds on self-debug, self-explain builds on Reflexion."
date_updated: 2026-05-19
related_concepts: []
linked_ideas: []
---

## Definition

Interactive planning with self-check is a closed-loop planning pattern for LLM-based agents that interleaves two verification mechanisms around plan execution:

1. **Self-check (upfront verification).** Given an initial plan, the planner simulates each sub-goal sequentially, predicts the resulting state (especially inventory or environment side-effects), and verifies that the next sub-goal's preconditions are met. Plan flaws are caught and fixed before any action is taken in the environment.
2. **Self-explain (post-failure repair).** When execution fails despite self-check, the planner takes the environment-feedback error message, reasons about which sub-goal in the original plan is responsible, and emits a repaired plan that addresses the diagnosed cause.

The pattern shrinks the failure surface in two complementary ways: cheap upfront simulation eliminates many bugs without environment cost, and feedback-driven repair handles the bugs that survived simulation.

## Intuition

Canonical LLM planners produce a plan once at the start of the episode and pass it to a controller. When something goes wrong, the agent must either silently fail or fall back to expensive recovery (re-planning from scratch, returning to a known safe state). In open worlds with strict precondition chains — "you need a wooden pickaxe before you can mine stone, you need stone before you can mine iron…" — a single precondition violation cascades into a long recovery.

Self-check shifts as much error detection as possible *before* execution starts, by treating the planner itself as a forward simulator over a simplified world model (mainly inventory and tool state). Self-explain handles the residual: when the simulation missed something, environment feedback names the actual failure, and the planner uses that name to diagnose and patch.

## Variants

- **Self-check only.** Simulate plan, fix violations, execute, no feedback handling. Brittle to anything the simulation does not model.
- **Self-explain only (Reflexion-style).** No upfront verification; rely on execution failures to drive re-planning. Wastes environment steps on bugs that could have been caught for free.
- **Self-check + self-explain (interactive planning, JARVIS-1).** Both layers active; sub-quadratic in re-plan rounds compared to self-explain only.
- **Self-debug (code-generation analogue, Chen et al. 2023).** Same upfront-simulation idea applied to LLM-generated code: predict program output, compare to expected output, fix before running.

## Comparison

- **vs. fixed-plan / open-loop planning** (e.g., Huang et al. 2022 "Language Models as Zero-Shot Planners"): no feedback handling, no upfront simulation; brittle to long horizons.
- **vs. ReAct:** ReAct interleaves reasoning and acting at each step but does not separately verify sub-goal preconditions before execution. Interactive planning with self-check adds the verification layer.
- **vs. Inner Monologue:** Inner Monologue introduces environment feedback to the planner but does not include upfront plan simulation; in long-horizon Minecraft, the authors of JARVIS-1 attribute Inner Monologue's accumulated planning errors to the missing self-check stage.
- **vs. DEPS:** DEPS uses interactive re-planning with descriptions and explanations but does the simulation implicitly in the LLM and is limited to roughly 6 re-plan rounds before context overflow. The named self-check / self-explain split in JARVIS-1 explicitly factors these phases and converges in 2–3 rounds.

## Known limitations

- **Simulation fidelity.** Self-check is only as good as the LLM's mental model of the environment. State transitions that depend on subtle dynamics (e.g., tool durability under specific use patterns) escape simulation.
- **Compound error in self-explain.** When environment feedback is ambiguous, the LLM may misattribute the failure to the wrong sub-goal and emit a patch that does not address the real bug.
- **Cost.** Each self-check pass costs roughly one LLM call per sub-goal; long plans amortize this against avoided environment failures, but short plans pay the overhead without benefit.
- **Domain assumptions.** The pattern as stated assumes a planner that can name preconditions, which requires the environment to expose enough structure (e.g., Minecraft's recipe system). In less structured environments the self-check phase has nothing to check.

## Open problems

- Can self-check be **learned** rather than prompted — a separately trained verifier specialized to a domain's transition dynamics?
- How should self-check and self-explain **share information** across episodes — does a self-explain repair from one task generalize as a self-check rule for future tasks?
- How does the pattern **scale to environments without explicit preconditions** (open-ended robotics, natural-language tasks without enumerable sub-goals)?

## Relationship to foundations

Inherits the closed-loop control intuition from classical robotics (sense → plan → act → sense, repeat) and the in-context reasoning patterns from chain-of-thought / self-consistency. Self-check builds on self-debugging (Chen et al. 2023); self-explain builds on Reflexion (Shinn et al. 2023).

## My understanding

The crisp contribution of this pattern is *separating the two failure-handling phases*. Prior work either dumped both into a single re-planning loop (DEPS) or skipped one entirely (Inner Monologue, ReAct). Treating upfront simulation and post-failure repair as distinct mechanisms with different inputs and different cost profiles is what enables JARVIS-1's 2–3 re-plan rounds vs DEPS's 6+ rounds.

The unsolved part is generalization: the pattern is well-defined in Minecraft because the environment exposes preconditions via the recipe system. In environments where preconditions are not enumerable (open-ended robotics, web automation with non-deterministic side effects), self-check has no natural verification surface, and the pattern degrades to plain feedback-driven re-planning.
