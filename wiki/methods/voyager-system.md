---
name: "Voyager System"
slug: voyager-system
type: system
tags:
  - minecraft-agent
  - lifelong-learning
  - llm-agent
  - skill-library
  - automatic-curriculum
  - iterative-prompting
  - code-as-action
  - gpt-4
source_papers:
  - voyager-open-ended-embodied-agent-large
parent_methods: []
child_methods: []
code_repo: "https://voyager.minedojo.org"
date_updated: 2026-05-20
---

## Problem setting

Open-ended lifelong learning in Minecraft. The agent starts with no inventory in a procedurally-generated world with no predefined goal, and must autonomously propose tasks, learn skills, and progressively traverse Minecraft's hierarchical tech tree (wood → stone → iron → diamond) using only natural-language interaction with a frozen LLM. No human-authored task curriculum, no demonstrations, no gradient updates to the LLM, no pixel input — the world is observed via Mineflayer's symbolic JavaScript API (inventory dictionary, biome label, nearby block list, entity list, position, health/hunger).

## Mechanism

Voyager is the integration of three modules that together form a closed lifelong-learning loop around a black-box LLM:

1. **[[automatic-curriculum]]** — a GPT-4 prompt that proposes the next task conditioned on the agent's state, history of completed and failed tasks, and a directive to "discover as many diverse things as possible." Temperature 0.1 for diversity.
2. **[[executable-skill-library]]** — an ever-growing key-value store of JavaScript programs (using Mineflayer APIs) indexed by `text-embedding-ada-002` embeddings of natural-language descriptions. Retrieval injects top matches into the next code-generation prompt as in-context examples; commits happen only after the critic confirms success.
3. **[[iterative-prompting-environment-feedback]]** — a code-refinement loop with three feedback channels (environment `bot.chat()` messages, JS interpreter errors, [[llm-self-verification-critic]] verdict + critique). Up to 4 rounds per task; the curriculum is queried for a new task on round-budget exhaustion or critic success.

A separate GPT-4 agent acts as the critic: same model as the actor but a different prompt context (it sees the task description and the agent's post-execution state, not the actor's chain-of-thought). GPT-3.5 handles cheap NLP self-Q&A inside the curriculum prompt for budget reasons.

## Procedure

1. **Curriculum query.** Prompt GPT-4 with directive + agent state + completed/failed task history + Q&A context. Receive next task.
2. **Skill retrieval.** Embed (task plan + environment feedback); retrieve top-k library entries by cosine similarity. Inject as in-context examples.
3. **Code generation.** Prompt GPT-4 with task, retrieved skills, control-primitive API list, current state, chain-of-thought instructions. Emit candidate JavaScript program.
4. **Execute.** Run the program in MineDojo via Mineflayer. Collect (a) any execution errors from the JS interpreter and (b) any `bot.chat()` messages emitted by control primitives.
5. **Self-verification.** Prompt a separate GPT-4 instance with original task + post-execution state. Receive (success_bool, critique_string).
6. **Branch:**
   - On success → generate a natural-language description for the program, embed the description, commit (description_embedding → program) to the skill library. Return to step 1.
   - On failure within round budget → concatenate (last-program, environment feedback, execution errors, critique) into the next prompt; go to step 3.
   - On round budget exhausted → log failure; go to step 1 for a new task.

Up to 4 rounds per task. All temperatures are 0 except curriculum (0.1).

## Assumptions

- Strong base LLM (GPT-4-class) with substantial Minecraft world knowledge in pretraining — ablation with GPT-3.5 produces 5.7x fewer unique items.
- Symbolic environment state (Mineflayer API) is available; no native visual perception is required, but spatial-detail tasks (building Nether Portals, houses) need a human acting as visual critic / curriculum.
- An executable interpreter is available for the action language (JavaScript, in Voyager's case) and exposes runtime errors back to the LLM.
- The task space is well-described by natural-language proposals; "mine 10 cobblestone" is meaningful to both the curriculum, the actor, and the critic.

## Limitations

- **Cost.** GPT-4 calls per round + critic call per round + multiple rounds per task make Voyager substantially more expensive than one-shot baselines; ~15x the per-call cost of GPT-3.5.
- **Critic miscalibration.** The self-verification critic occasionally rejects valid success (spider-string-in-inventory example) and may accept marginal failure.
- **Skill library accumulates, does not evolve.** Once a skill is committed it is not rewritten after later failures expose bugs. No retirement / deduplication policy.
- **Single environment.** All empirical results are in Minecraft; transfer to robotics, web automation, or other open-ended domains is argued but not demonstrated.
- **No native vision.** Spatial-detail tasks fail without human visual feedback.
- **Hallucinations.** Curriculum sometimes proposes nonexistent items ("copper sword"); GPT-4 occasionally invokes nonexistent control primitives.
- **No multi-agent or shared-library setting.** A single agent's skill library is studied; multi-agent shared-skill libraries are out of scope.

## Tradeoff profile

- **Compute vs. data:** Heavy at inference (multiple GPT-4 calls per task); zero gradient updates, zero training data, zero parameter access required.
- **Generality vs. domain scaffolding:** General architectural template (curriculum + library + iterative-prompting critic) but Minecraft-specific scaffolding (Mineflayer APIs, MineDojo simulator, Minecraft recipe priors baked into GPT-4). Ports to other domains require reimplementing the control primitives and re-grounding the LLM in the new API.
- **Reliability vs. cost:** Each added feedback channel (env feedback, execution errors, critic) improves reliability at the cost of one more LLM call per round; the ablations show all three are net-positive but the critic dominates (73% drop without).
- **Accumulation vs. evolution:** The append-only skill library favors interpretability and resistance to catastrophic forgetting at the cost of leaving committed bugs unfixable.
- **Frozen-LLM vs. trained-policy:** Pays a steady inference cost forever; in exchange, the system trivially upgrades when a better LLM lands (no retraining).
