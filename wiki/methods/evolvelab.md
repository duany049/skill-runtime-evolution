---
name: EvolveLab
slug: evolvelab
type: benchmark
tags:
  - agentic-memory
  - codebase
  - benchmark
  - memory-architecture
  - testbed
source_papers:
  - memevolve-meta-evolution-agent-memory-systems
parent_methods: []
child_methods:
  - memevolve
code_repo: https://github.com/bingreeky/MemEvolve
date_updated: 2026-05-20
---

## Problem setting

EvolveLab is the unified codebase and evaluation testbed under which twelve published self-improving agent memory systems are re-implemented against a single abstract base class, then evaluated on a common set of agentic benchmarks under a shared harness. It serves a dual role:
1. As an *empirical foundation* for [[memevolve]]'s evolutionary process — providing the slot-typed `BaseMemoryProvider` interface that any candidate architecture (E, U, R, G) must conform to so descendants are guaranteed executable.
2. As a *community resource* — a fair-comparison testbed where new memory systems can be implemented against the same interface and evaluated under the same protocol as the 12 baselines, removing the per-paper re-implementation tax that currently fragments the field.

The implementations target self-improving (not personalized) memory: memory that distills knowledge and skills from continual environment interaction to improve task performance, as opposed to memory that tracks user preferences across sessions.

## Mechanism

**Unified interface.** Every re-implemented memory system inherits from a singular abstract base class, `BaseMemoryProvider`, which enforces the four-slot contract — see [[modular-memory-design-space]] — by exposing typed entry points for:
- `Encode(ε)` → structured experience representation
- `Store(M, e)` → updated persistent memory state
- `Retrieve(M, s, Q)` → task-relevant context
- `Manage(M)` → offline / asynchronous consolidation

Slot implementations are programmatic functions; substitution of any slot yields a new executable architecture with no glue-code modification required.

**Implemented baselines (12).** The codebase ships re-implementations of:

| # | System | Date | Mul. | Gran. | Online | Encode | Store | Retrieve | Manage |
|---|---|---|---|---|---|---|---|---|---|
| I | Voyager | 2023.5 | single | traj | online | Traj. & Tips | Vector DB | Semantic Search | N/A |
| II | ExpeL | 2023.8 | single | traj | online | Traj. & Insights | Vector DB | Contrastive Comparison | N/A |
| III | Generative Agents | 2023.10 | multi | traj | online | Traj. & Insights | Vector DB | Semantic Search | N/A |
| IV | DILU | 2024.2 | single | traj | online | Traj. | Vector DB | Semantic Search | N/A |
| V | AWM | 2024.9 | single | traj | both | Workflows | Vector DB | Semantic Search | N/A |
| VI | Mobile-E | 2025.1 | single | step | offline | Tips & Shortcuts | Vector DB | Semantic Search | N/A |
| VII | Dynamic Cheatsheet | 2025.4 | single | traj | online | Tips & Shortcuts | JSON | Semantic Search | N/A |
| VIII | SkillWeaver | 2025.4 | single | traj | offline | APIs | Tool Library | Function Matching | Skill Pruning |
| IX | G-Memory | 2025.6 | multi | traj | online | Tips & Workflow | Graph | Graph/Semantic Search | Episodic Consolidation |
| X | Agent-KB | 2025.7 | multi | step | offline | Tips & Workflow | Hybrid DB | Hybrid Search | Deduplication |
| XI | Memp | 2025.8 | single | step | online | Tips & Workflow | JSON | Semantic Search | Failure-driven Adjustment |
| XII | EvolveR | 2025.10 | single | step | online | Tips & Workflow | JSON | Contrastive Comparison | Update & Pruning |

The "Manage" slot is empty (N/A) for most pre-2025 systems, revealing that offline consolidation is a relatively recent design dimension.

**Evaluation harness.** Out-of-the-box support for multiple agentic benchmarks: GAIA (165 tasks across 3 levels), xBench-DeepSearch (100 tasks), WebWalkerQA (170 sampled queries), DeepResearchBench, plus TaskCraft used as the meta-evolution corpus. Two evaluation modes:
- **Online mode** — the experiential memory base is updated on-the-fly as the agent processes a continuous stream of tasks; this matches the deployment-time behavior of most self-improving memory systems.
- **Offline mode** — the memory system first accumulates experience from a static set of trajectories, then is frozen and assessed on separate held-out tasks; cleaner causal comparison but does not exercise continual-update dynamics.

Multiple grading protocols: exact string matching and flexible LLM-as-a-Judge for free-form answers. Agent frameworks supported: SmolAgent (two-agent), Flash-Searcher (single-agent deep research), Cognitive Kernel-Pro (three-agent, held out), OWL (hierarchical multi-agent, held out).

## Procedure

To benchmark a new memory system in EvolveLab:
1. Subclass `BaseMemoryProvider` and implement the four slot methods.
2. Register the implementation with the framework configuration (model backbone, benchmark, eval mode).
3. Run the harness — per-task success, token cost, latency, and step count are logged in a standardized format.
4. Compare against the 12 in-codebase baselines on identical splits and identical backbone.

To use EvolveLab as a substrate for meta-evolution (the [[memevolve]] use case):
1. Express both baseline and descendant architectures as `BaseMemoryProvider` subclasses (or compose existing slot implementations).
2. The meta-operator's `Design` step emits new slot implementations conforming to the interface; executability is guaranteed by construction.

## Assumptions

- Self-improving (not personalized) memory is the target — EvolveLab does not currently provide harness support for chat-personalization workloads.
- The four-slot decomposition is sufficient to express each baseline faithfully; the paper claims faithfulness for all 12 but does not isolate re-implementation drift from the architecture's intrinsic behavior (relevant for systems where the baseline underperforms, e.g. ExpeL on deep-research benchmarks).
- Benchmarks shipped are biased toward agentic deep-research and web-navigation; embodied / mathematical / code-execution benchmarks are out of scope.
- The agent framework wrapping the memory provider is one of the supported scaffolds (SmolAgent, Flash-Searcher, CK-Pro, OWL); arbitrary scaffolds would require additional adapter work.
- LLM-as-a-Judge grading is reasonable for the supported tasks; for tasks where exact-match is critical, the exact-match protocol must be selected.

## Limitations

- Faithfulness of re-implementations is asserted but not ablated against authors' original codebases — underperformance of any baseline could conflate framework drift with architectural weakness.
- The four-slot ABC excludes memory affordances that don't fit the (E, U, R, G) decomposition (typed multi-stores, episodic-semantic splits, attention-style soft retrieval).
- "Online" vs. "offline" modes are coarse-grained; partial-online regimes (e.g. periodic batch updates) are not first-class.
- Benchmark coverage skews toward deep research — embodied benchmarks (ALFWorld, etc.), code generation (HumanEval, SWE-bench), and mathematical reasoning (MATH, GSM-symbolic) are not natively supported.
- The codebase requires programmatic slot implementations; integrating learned components (trained retrievers, neural managers) requires extra adapter plumbing.

## Tradeoff profile

- **Unified vs. faithful.** Forcing 12 disparate systems into one ABC ensures fair comparison but may sand off implementation details that mattered in the original papers; this is the canonical cost of unified codebases.
- **Coverage vs. specialization.** Targeting four agentic benchmarks gives a broad picture of memory systems' behavior on long-horizon deep research but does not characterize them on embodied or symbolic tasks; the paper explicitly disclaims cross-family transfer.
- **Programmatic slots vs. learned modules.** The current ABC privileges concise functional slot implementations; learned retrieval policies or neural managers integrate awkwardly without additional plumbing.
- **Slot count vs. expressiveness.** Four slots is a compromise — sufficient to cover the 12 baselines and to keep mutation safe (each descendant differs in one slot), but coarse enough that hierarchical or typed-multi-store memory architectures require some shoehorning.
