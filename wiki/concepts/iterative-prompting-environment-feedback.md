---
title: Iterative Prompting with Environment Feedback
aliases:
  - iterative prompting mechanism
  - feedback-driven code refinement
  - closed-loop code generation
  - three-channel code refinement loop
tags:
  - iterative-prompting
  - code-generation
  - feedback-loop
  - self-verification
  - llm-agent
  - embodied-control
maturity: emerging
definition: A closed-loop code-generation pattern in which an LLM repeatedly refines a candidate program for an embodied task using three feedback channels — runtime environment observations, execution errors from the interpreter, and a separate LLM critic's success/failure judgement — until the critic confirms success or a round budget is exhausted.
key_papers:
  - voyager-open-ended-embodied-agent-large
first_introduced: "Voyager (Wang et al., 2023); generalizes self-debug (Chen et al. 2023) and self-reflection (Shinn et al. 2023) by integrating environment feedback and a separate critic into a single loop."
date_updated: 2026-05-20
related_concepts: []
linked_ideas: []
---

## Definition

Iterative prompting with environment feedback is a code-refinement loop with three distinct feedback channels feeding back into successive prompts:

1. **Environment feedback.** Runtime observations emitted by the program via in-API logging calls (Voyager uses `bot.chat()`), e.g., "I cannot make an iron chestplate because I need: 7 more iron ingots." Conveys *progress* and *intermediate failure causes*.
2. **Execution errors.** Stack traces and syntax errors from the program interpreter. Conveys *implementation bugs*.
3. **Self-verification critique.** A separately-prompted LLM agent (Voyager uses GPT-4) that takes the agent's post-execution state and the original task description, returns a boolean success judgement plus, on failure, a natural-language critique. Conveys *task-completion* assessment.

Each round: the LLM generates a candidate program; it executes in the environment; observations, errors, and critic verdict are concatenated into the next prompt; the LLM generates a refined program. The loop terminates when the critic returns success (commit the skill to the library) or after a fixed round budget (Voyager uses 4) at which point the curriculum is queried for a new task.

## Intuition

Single-shot LLM code generation fails on the long tail of embodied tasks. The reasons are heterogeneous — syntax bugs, API misuse, incorrect inventory state, off-by-one preconditions, wrong success criterion — and each requires a different signal to fix. Iterative prompting *factors* the feedback into three channels because the LLM's repair strategy differs by channel:

- An **execution error** says "you wrote invalid code"; the fix is syntactic/static.
- An **environment feedback** message says "your code ran but the world is not what you expected"; the fix is logical/dynamic.
- A **critic critique** says "your code finished but didn't actually achieve the task"; the fix is at the level of goal interpretation.

Separating these channels lets the model attend to the right kind of error without conflating them.

## Variants

- **Self-debug (Chen et al. 2023).** Code-only loop: execute, compare output to expected, fix. No environment, no separate critic. Strong for pure-Python competitive-programming problems; insufficient for embodied tasks where success cannot be checked by output equality.
- **Reflexion (Shinn et al. 2023).** Single self-reflection trace from the agent itself after task failure. Conflates "did it succeed" with "what went wrong"; no separate critic; no execution-error channel separate from environment.
- **Iterative prompting with all three channels (Voyager).** Adds a critic agent and a runtime-observation channel; the paper's ablation (removing self-verification drops items 73%) shows the critic channel is individually load-bearing.
- **Plan-level analogue: [[interactive-planning-with-self-check]] (JARVIS-1).** Same closed-loop intuition but verifies *plans* against an internal world model upfront rather than refining *code* against runtime feedback after the fact.

## Comparison

- **vs. one-shot code generation (Code-as-Policies, ProgPrompt):** brittle on long-horizon embodied tasks where errors are heterogeneous; no recovery mechanism.
- **vs. Reflexion-style self-reflection:** Reflexion combines "did I succeed" and "why did I fail" into one self-reflection step. Voyager's separate critic does both more comprehensively and avoids the agent grading its own homework.
- **vs. ReAct:** ReAct interleaves reasoning and action but does not factor feedback into distinct channels and does not have a separate critic for task-completion.
- **vs. test-time RL methods (JITRL, Skill-Pro):** RL methods update logits or prompt-text via numeric optimization; iterative prompting uses natural-language feedback only and never updates parameters.

## Known limitations

- **Critic errors.** The self-verification critic is itself an LLM and occasionally misjudges success — the paper notes failure to recognize "spider string in inventory" as evidence of beating a spider.
- **Cost.** Up to 4 generation rounds per task, plus a critic call per round, plus environment execution — substantially more expensive than one-shot.
- **Round budget is heuristic.** The fixed 4-round limit is empirically chosen; tasks that need 5+ rounds are abandoned even when the agent was about to succeed.
- **Channel imbalance.** All three channels are concatenated raw into the prompt; in long episodes the prompt may overflow context, requiring truncation policies that are not specified.
- **No memory across tasks.** The iteration loop is per-task; failure patterns learned on one task do not propagate to others except via skill commitment.

## Open problems

- Can the **round budget be adaptive** based on the critic's confidence trajectory rather than fixed?
- How to **distill critic verdicts** into reusable lessons for future tasks (separate from the executable skill library)?
- **Multi-agent verification:** does using two critics in disagreement-detection mode catch the spider-string-style critic errors?
- How to **share refinement signal** across tasks — if iteration N fails because of an inventory-checking bug, can subsequent tasks pre-emptively guard against it?

## Relationship to foundations

Generalizes the self-debug intuition from pure-code to embodied control by adding the runtime-feedback channel and the critic. Also draws on classical actor-critic decompositions in RL (Mnih et al. 2016; Schulman et al. 2017; Lillicrap et al. 2016) — the critic-as-success-judge plays the role of a value head, but instantiated as natural language rather than scalar.

## My understanding

The architectural insight is the **three-channel factorization**. Prior loops either had one channel (Reflexion's self-reflection, Code-as-Policies' execution result) or conflated several (ReAct's combined reasoning+action). Voyager's separation — env feedback vs execution errors vs critic verdict — gives the LLM a structured failure signal that maps cleanly onto a structured repair strategy.

The 73% item drop when removing self-verification (Voyager ablation Table 5) is the strongest single piece of evidence that the *critic channel*, not the curriculum or the library, is where most code-generation reliability comes from. This reframes "agent reliability" as a verifier-design problem more than a generation problem.
