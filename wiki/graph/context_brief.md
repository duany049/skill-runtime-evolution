# Query Pack (general)

_Auto-generated compressed context. Do not edit._

## Methods (28 total)
- [prompting] Agent Workflow Memory (AWM)
- [training] AtomMem
- [system] AutoRefine
- [inference] AveFact Retrieval
- [system] Bottom-Up Skill Evolution
- [system] Contextual Experience Replay
- [system] DeepSolver
- [training] Designer-Driven Skill Bank Evolution
- [training] Evolvable Skill Bank Controller
- [benchmark] EvolveLab
- [training] Group Relative Self-Distillation
- [system] JARVIS-1 Agent
- [inference] JitRL (Just-In-Time Reinforcement Learning)
- [system] Live-Evo
- [system] MemEvolve
- [inference] MemWeaver Dual-Channel Memory Retrieval
- [inference] Multimodal Retrieval-Augmented Planning
- [optimization] Non-Parametric PPO
- [benchmark] OSExpert-Eval Benchmark
- [system] OSExpert Framework
- [prompting] Proceduralization
- [evaluation] Progressive Skill Evaluation
- [inference] Query-Specific Skill Refinement
- [evaluation] Reflective Step-wise Rewards
- [benchmark] SciSkillBench
- [benchmark] SkillFlow Benchmark
- [benchmark] SkillLearnBench
- [system] Voyager System
## Open Gaps
_Auto-generated open questions. Do not edit._
- [paper/agent-workflow-memory] Can workflows be **revised** post-hoc when later evidence contradicts them (the LM evaluator was wrong, or a website changed)?
- [paper/agent-workflow-memory] What is the right **retrieval policy** for a unified workflow library spanning many websites and tasks? Per-website grouping is a stopgap.
- [paper/agent-workflow-memory] How does AWM compose with **action-space modifications** like SteP's hand-written skills — are induced workflows and authored skills additive or redundant?
- [paper/agent-workflow-memory] What does the **scaling behavior** look like? AWM's curve plateaus on WebArena after ~40 queries; is that an artifact of test-set size or a real saturation?
- [paper/agent-workflow-memory] Is there a principled **cross-domain transfer** signal for workflows, or does cross-domain success require always re-inducing from test queries (the online setting)?
- [paper/agentic-memory-learning-unified-long-term] How does AgeMem behave under truly persistent multi-session deployment where LTM grows across many episodes and forgetting/consolidation become first-order concerns? The benchmarks here are within-episode long-horizon, not cross-session.
- [paper/agentic-memory-learning-unified-long-term] Can the curriculum be applied without HotpotQA-style supporting-fact labels? The paper claims the three-stage structure only needs temporal separation between exposure and execution, but doesn't demonstrate 
## Papers (26 total)
- [4] MemEvolve: Meta-Evolution of Agent Memory Systems — Meta-evolves the *architecture* of an agent's memory system (not just its contents) by decomposing memory into four modules (Encode/Store/Retrieve/Manage) and running a bilevel optimization that improves both stored experience and the memory pipeline itself.
- [5] VOYAGER: An Open-Ended Embodied Agent with Large Language Models — Voyager is the first GPT-4-driven lifelong learning agent in Minecraft, combining an automatic curriculum, an ever-growing executable skill library, and iterative code refinement with self-verification to outperform prior LLM agents by 3.3x in unique items and unlock the diamond tech tree.
- [4] AtomMem: Learnable Dynamic Agentic Memory with Atomic Memory Operation — AtomMem reframes agent memory management as a learnable decision-making problem over atomic CRUD operations, trained end-to-end with GRPO to outperform static-workflow baselines on long-context QA and web benchmarks.
- [4] How Well Do Agentic Skills Work in the Wild: Benchmarking LLM Skill Usage in Realistic Settings — First systematic study showing skill benefits for LLM agents degrade sharply as evaluation moves from idealized curated-skill settings to realistic settings requiring retrieval from a 34k-skill pool, with query-specific refinement recovering much of the lost performance only when retrieved skills already have reasonable relevance.
- [4] JARVIS-1: Open-World Multi-Task Agents With Memory-Augmented Multimodal Language Models — JARVIS-1 is a Minecraft agent that augments a multimodal language model planner with a multimodal memory of past gameplay experiences, enabling situation-aware interactive planning and self-instruct lifelong learning that achieves a 5x reliability gain over VPT on long-horizon diamond-pickaxe tasks.
- [4] Just-In-Time Reinforcement Learning: Continual Learning in LLM Agents Without Gradient Updates — JitRL is a training-free framework that performs RL-style policy optimization at test 
## Recent Relationships (51 total)
  papers/atommem-learnable-dynamic-agentic-memory-atomic --introduces_concept--> concepts/hybrid-memory-retrieval
  papers/memweaver-weaving-hybrid-memories-traceable-long --introduces_concept--> concepts/tri-layer-memory-consolidation
  papers/live-evo-online-evolution-agentic-memory --introduces_concept--> concepts/online-self-evolving-agentic-memory
  papers/live-evo-online-evolution-agentic-memory --introduces_concept--> concepts/meta-guideline-bank
  papers/live-evo-online-evolution-agentic-memory --introduces_concept--> concepts/verify-before-update
  papers/skillclaw-let-skills-evolve-collectively-agentic --introduces_concept--> concepts/collective-skill-evolution
  papers/skillclaw-let-skills-evolve-collectively-agentic --introduces_concept--> concepts/agentic-evolver
  papers/autorefine-trajectories-reusable-expertise-continual-llm --introduces_concept--> concepts/subagent-pattern
  papers/memp-exploring-agent-procedural-memory --introduces_concept--> concepts/procedural-memor
