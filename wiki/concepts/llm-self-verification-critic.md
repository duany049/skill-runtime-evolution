---
title: LLM Self-Verification Critic
aliases:
  - self-verification critic
  - LLM critic
  - GPT-4 self-verification
  - critic agent
  - task-success critic
tags:
  - self-verification
  - critic
  - llm-agent
  - feedback-loop
  - skill-evolution
maturity: emerging
definition: A separately-instantiated LLM agent that judges whether an embodied agent's program achieved its target task — returning a boolean success/failure verdict plus, on failure, a natural-language critique that names what went wrong — used as one feedback channel inside an iterative code-refinement loop.
key_papers:
  - voyager-open-ended-embodied-agent-large
first_introduced: "Voyager (Wang et al., 2023) as a named module distinct from self-reflection; conceptually inherits the critic role from actor-critic RL (Mnih et al. 2016; Schulman et al. 2017; Lillicrap et al. 2016)."
date_updated: 2026-05-20
related_concepts: []
linked_ideas: []
---

## Definition

An LLM self-verification critic is a **separately-prompted** LLM that, given (a) the original task description and (b) the agent's state after attempting the task, produces:

1. A **boolean verdict** — did the program achieve the task? — and
2. On failure, a **natural-language critique** describing what is missing or what went wrong, useful for the next refinement round.

The critic is structurally distinct from the actor (the LLM that generates the program). In Voyager both are GPT-4, but they receive different prompts, see different inputs, and serve different roles. The critic also differs from a separately-coded rule-based success checker: it generalizes across the open-ended set of tasks an automatic curriculum can propose, without per-task rule authoring.

The critic's output drives the iterative-prompting loop: success commits the program to the skill library; failure pushes the critique back into the actor's next prompt.

## Intuition

The classical alternative is **rule-based success checking** — a handwritten predicate per task ("did inventory.diamond > 0?"). This is fine when the task set is fixed, but Voyager's automatic curriculum proposes open-ended tasks the system designer never anticipated. A natural-language critic generalizes to any task the LLM can describe.

The reframing matters because it shifts success-checking from a *static design-time artifact* to a *learned inference-time judgement*. The same prompt template handles "mine 10 cobblestone" and "craft a stone shovel" and "combat zombie with sword" without any per-task code.

Separating critic from actor also avoids the "grading your own homework" failure mode of self-reflection: the actor has incentive to declare success; a separate critic has none.

## Variants

- **Same-model critic (Voyager).** Actor and critic are both GPT-4 but prompted independently. Cheapest separation.
- **Different-model critic.** Use a cheaper or specialized model as critic (cost / specialization tradeoff). Less studied.
- **Multi-critic / disagreement detection.** Run two critics; treat disagreement as low confidence and request another iteration. Not in Voyager but a natural extension.
- **Critic with critique only (no boolean).** Force the critic to suggest improvements; the actor decides whether to commit. Useful when the critic is calibration-poor.
- **Verdict-only critic (no critique).** Cheap but unhelpful on failure; the actor has no signal beyond "try again."

## Comparison

- **vs. rule-based success checkers:** rule-based scales linearly in design effort with task variety; an LLM critic generalizes for free but introduces noise.
- **vs. self-reflection (Reflexion):** Reflexion folds success-judgement and failure-explanation into one self-issued reflection from the actor. Voyager argues this is strictly weaker because the actor is not a disinterested judge and because the two signals (did-it-succeed vs why-not) have different optimal prompts.
- **vs. classical actor-critic RL:** the critic plays the same architectural role — judging the actor's behavior — but emits natural-language verdict and critique rather than a scalar value estimate.
- **vs. learned reward models (RLHF):** reward models score continuous policies during training; an LLM critic gates discrete task acceptance at inference time.
- **vs. [[verify-before-update]]:** verify-before-update guards *memory writes* on online runs; the LLM critic guards *skill commits* during skill-library construction. Same intuition (separate verifier), different target.

## Known limitations

- **Critic noise.** The paper documents miscalibration — the critic occasionally fails to recognize valid success signals (e.g., spider-string in inventory after defeating a spider) or accepts a borderline failure as success.
- **Same-model failure modes.** If actor and critic share the same model, they share systematic blind spots (the same hallucinations, the same recipe-tree confusions).
- **No calibration training.** The critic is zero-shot prompted; no fine-tuning, no feedback from downstream environmental ground truth to improve verdict accuracy over time.
- **Cost.** Adds one full LLM call per iteration on top of code generation.
- **Single-checkpoint judgement.** The critic sees the final state only; it cannot diagnose mid-execution failures that masked the underlying root cause.

## Open problems

- **Calibration:** how to align the critic's verdict distribution with downstream environmental ground truth without manually labeling success?
- **Critic-actor decorrelation:** would a critic from a different model family catch the same-model blind spots that Voyager's appendix documents?
- **Critic chains:** sequencing multiple specialized critics (one for safety, one for completion, one for efficiency) and aggregating their verdicts.
- **Online critic improvement:** can the critic learn from being wrong (cases where it accepted a failure or rejected a success that the environment later revealed)?
- **Critic for plans vs critic for code:** can the same critic role be re-targeted at planning failure (à la JARVIS-1's self-explain) and code-execution failure simultaneously?

## Relationship to foundations

Generalizes the *critic* in actor-critic RL (Mnih et al. 2016 A3C; Schulman et al. 2017 PPO; Lillicrap et al. 2016 DDPG) to the prompted-LLM setting, where verdict and critique are natural-language outputs rather than scalar values and gradients. Also adjacent to learned reward modeling in RLHF (Christiano et al. 2017; Ouyang et al. 2022), but used at inference time as a gate rather than during training as a signal.

## My understanding

The reason the LLM critic deserves a name distinct from "self-reflection" is the *role separation*. Reflexion folds "did I succeed?" and "what went wrong?" into a single reflection issued by the agent; Voyager's critic is **a separate prompt context that does not see the actor's chain-of-thought**, eliminating the obvious confirmation bias.

Empirically this is the single most impactful feedback channel in Voyager — ablating it costs 73% of unique items (Voyager Table 5), more than ablating the skill library. This is strong evidence that for code-generating embodied agents the bottleneck is not "can the LLM write the code" but "can the system tell when the code worked." It suggests that future skill-evolution systems should put as much engineering effort into the critic as into the actor.
