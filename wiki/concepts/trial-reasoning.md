---
title: Trial-and-Reasoning
aliases:
  - trial and reasoning
  - exploration and reflection
  - trial-and-reasoning loop
tags:
  - llm-agent
  - skill-discovery
  - bottom-up-agent
  - exploration
maturity: emerging
definition: "A skill-acquisition loop in which an agent explores combinations of atomic actions, reflects on their observed environmental effects via an LLM, and crystallizes the sequences that trigger recognizable changes into reusable skills — used as the foundational learning mechanism for bottom-up agents instead of imitation, RL with explicit reward, or designer-decomposed workflows."
key_papers:
  - rethinking-agent-design-top-down-workflows
first_introduced: "rethinking-agent-design-top-down-workflows (2505.17673)"
date_updated: 2026-05-18
related_concepts:
  - bottom-up-agent-paradigm
  - implicit-visual-reward
linked_ideas: []
---

## Definition

Trial-and-reasoning is a two-phase loop applied per timestep: (1) **trial** — the agent generates an action or extends a known skill with a new atomic action, executing it against the live environment; (2) **reasoning** — an LLM compares the pre- and post-execution observations and judges whether the action produced a recognizable, useful effect. Sequences that pass the recognition test are retained as candidate skills with a generated semantic descriptor; those that do not are discarded. The loop scales by progressively increasing sequence length (`k = 1, 2, 3, …`), so complex behaviors emerge from compositions of validated primitives rather than from monolithic action proposals.

## Intuition

Trial-and-reasoning is the bottom-up agent's substitute for two classical learning regimes:

- **Imitation learning** assumes a curated expert trajectory exists; trial-and-reasoning produces its own data from interaction.
- **RL with explicit reward** assumes a scalar reward signal is available; trial-and-reasoning uses LLM-judged environmental change as the reasoning target instead, removing the need for handcrafted reward functions.

The "reasoning" half is what distinguishes this loop from generic random exploration. Random exploration retains everything; trial-and-reasoning retains only sequences whose effects can be **named and explained** by the LLM as semantically meaningful. The naming step is also how skills become reusable: each retained skill gets a natural-language descriptor `d_σ`, which is later consumed when the LLM selects candidates `S_t = M(p_select, x_t, S)`.

## Variants

- **Per-step augmentation.** Triggered when no existing skill applies; explores incrementally longer sequences until one produces a recognizable effect.
- **Refinement variant.** When viable skills are sparse, the LLM rewrites a skill `σ' = M(p_refine, (x_t, σ, T))`, keeping the new version only if it shows improved semantic alignment or efficiency.

## Comparison

| Mechanism | Source of feedback | Source of selection |
|---|---|---|
| Imitation learning | Expert demonstrations | Static curated dataset |
| RL with explicit reward | Scalar reward function | Policy gradient / value estimate |
| Voyager (top-down with priors) | Task-specific prompts | Designer-defined subgoals |
| Trial-and-reasoning | Visual change / progression | LLM-judged semantic alignment |

## Known limitations

- **Recognition threshold is brittle.** Sequences with subtle or delayed effects fail the "recognizable change" filter and are pruned even if they were strategically valuable.
- **Search cost grows with skill length.** Although composition reduces the space, longer-`k` skills still require many trials per useful retention.
- **LLM judgement noise.** The reasoning step inherits LLM hallucination and prompt sensitivity: a poorly-judged effect can either pollute the library with false positives or drop genuinely useful skills.

## Open problems

- Designing the recognition function so that delayed and strategic effects are captured (not just frame-to-frame visual diffs).
- Reducing exploration overhead via skill priors transferred from similar environments or memory-based generalization.
- Quantifying how much of the loop's competence is due to LLM reasoning vs. the visual-change filter (ablation suggests both matter, but their interaction is unclear).

## Relationship to foundations

The trial half is a constrained form of random exploration familiar from RL; the reasoning half is an LLM-driven generalization of credit assignment over interaction streams. The loop sits closer to the "era of experience" vision than to classical RL because the credit-assignment substrate is open-ended natural-language reasoning rather than a scalar reward.

## My understanding

The honest read is that trial-and-reasoning is "Voyager's curriculum loop, with the visual-feedback judge upgraded from a Minecraft-specific verifier to a general LLM-vision call". That is not faint praise — the genericization is what makes the bottom-up paradigm portable across environments — but it does mean the open question is whether the LLM-vision judge is accurate enough at scale. The ablation in the source paper shows removing visual-change filtering hurts but is not catastrophic; the more critical knob seems to be MCTS-driven invocation, not the discovery loop itself. Future work should ablate the **reasoning** half more directly: replace `M(p_differ, ...)` with deterministic image-similarity scores and see how badly the library degrades.
