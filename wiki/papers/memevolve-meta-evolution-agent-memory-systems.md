---
title: "MemEvolve: Meta-Evolution of Agent Memory Systems"
slug: memevolve-meta-evolution-agent-memory-systems
arxiv: "2512.18746"
year: 2025
tags:
  - agentic-memory
  - meta-evolution
  - self-evolving-agents
  - memory-architecture
  - skill-evolution
  - procedural-memory
importance: 4
date_added: 2026-05-18
source_type: tex
tldr: "Meta-evolves the *architecture* of an agent's memory system (not just its contents) by decomposing memory into four modules (Encode/Store/Retrieve/Manage) and running a bilevel optimization that improves both stored experience and the memory pipeline itself."
contribution_type:
  - method
  - system
  - benchmark
datasets:
  - GAIA
  - WebWalkerQA
  - xBench-DeepSearch
  - TaskCraft
code_url: "https://github.com/bingreeky/MemEvolve"
cited_by: []
---

## Problem & Context

Self-evolving agent memory systems have proliferated rapidly: trajectories, tips, workflows, skill libraries, knowledge graphs, MCP/API tool stores, and code-level repositories have all been proposed as memory substrates. Each system bakes in a fixed pipeline for ingesting, abstracting, and retrieving experience, then assumes that pipeline will sustain long-term self-improvement merely by being exposed to new data.

The authors argue this is the field's blind spot: the *architecture* of the memory system is itself static while only the *contents* evolve. Distinct tasks reward distinct memory affordances (web browsing benefits from reusable APIs; reasoning benefits from self-critique), so a single hand-crafted memory pipeline cannot be Pareto-optimal across task families. They draw an analogy to human learners — "skillful" learners extract reusable schemas from experience but apply the same extraction strategy everywhere, whereas "adaptive" learners dynamically alter their *meta-learning* strategy based on subject domain. Current memory systems sit at the skillful tier; the missing rung is adaptive.

## Key idea

Treat the memory architecture itself as the object of evolutionary search. Maintain a population of candidate memory architectures, each instantiated as a concrete (Encode, Store, Retrieve, Manage) tuple. Use a bilevel loop: an **inner loop** lets each candidate accumulate experience over agent rollouts (first-order evolution of memory contents), and an **outer loop** uses task success/cost/latency feedback to retain top candidates and mutate them into new architectures (second-order evolution of memory structure). The framework, called **MemEvolve**, is grounded in a companion modular codebase **EvolveLab** that re-implements 12 published memory systems (Voyager, ExpeL, Generative Agents, DILU, AWM, Mobile-E, Dynamic Cheatsheet, SkillWeaver, G-Memory, Agent-KB, Memp, EvolveR) under a unified `BaseMemoryProvider` interface, so any architecture can be expressed as a "genotype" over the same four-slot design space.

## Method

**Modular design space.** Any memory system is decomposed as `Ω = (E, U, R, G)` — Encode (raw experience → structured representation), Store (commit into persistent state), Retrieve (context-conditioned recall), Manage (offline consolidation / forgetting). All 12 baselines are recast in this space; new architectures are produced by mutating one or more slots.

**Dual-evolution.** At iteration *k*, hold a candidate set `{Ωⱼ⁽ᵏ⁾}`. For each candidate:
- *Inner loop:* run the agent over a batch `Tⱼ⁽ᵏ⁾` of 60 task trajectories (40 new + 20 reused for inter-iteration calibration), starting from an empty memory base; the candidate's Encode/Store/Manage rules populate `Mⱼ⁽ᵏ⁾`, and Retrieve conditions the policy. Collect a 3-D feedback vector `fⱼ = (success, −token_cost, −latency)`.
- *Outer loop:* rank candidates by non-dominated (Pareto) sort over `fⱼ`, break ties by primary success metric, keep top-*K* survivors (`K=1` in the main experiments). For each survivor, run **diagnose-and-design**:
  - *Diagnose:* an LLM inspects the parent's trajectory replay and produces a structured defect profile across the four components — retrieval failures, ineffective abstractions, storage inefficiencies.
  - *Design:* conditioned on the defect profile, propose *S=3* descendant architectures (modifying only the four permissible slots so descendants remain executable). Distinct descendants vary encoding strategies, storage rules, retrieval constraints, or management policies.

**Configuration.** `K_max=3` outer iterations; survivor budget `K=1`; descendants per parent `S=3`; meta-evolution LLM is GPT-5-mini; held-out generalization probes use Kimi K2 and DeepSeek V3.2 without re-evolution.

## Experiment & Results

**Setup.** Four agentic benchmarks — GAIA (165 tasks across 3 levels), WebWalkerQA (170 sampled queries), xBench-DeepSearch (100 tasks), TaskCraft (120 of 300 queries used for meta-evolution; 40 per round). Two host frameworks: SmolAgent (2-agent) and Flash-Searcher (single-agent deep research). Two held-out multi-agent transfer targets: CK-Pro and OWL. Three backbones: GPT-5-mini (main), Kimi K2 and DeepSeek V3.2 (cross-LLM transfer).

**Main numbers** (Table 1):
- Flash-Searcher + GPT-5-mini pass@1: 69.0 → 73.33 on GAIA (+4.24); 69.0 → 74.0 on xBench-DS (+5.0); 71.18 → 74.71 on WebWalkerQA (+3.53); 69.67 → 72.0 on TaskCraft (+2.33).
- Flash-Searcher + GPT-5-mini pass@3 with MemEvolve: 80.61 on GAIA, 78.0 on xBench-DS, 81.18 on WebWalkerQA, 79.33 on TaskCraft — surpassing OWL-Workforce and CK-Pro on multiple axes.
- Largest single jump: Kimi K2 + Flash-Searcher on WebWalkerQA: 52.35 → 69.41 (+17.06).
- SmolAgent + GPT-5-mini xBench-DS pass@3: 51.0 → 68.0 (+17.0) after MemEvolve.

**Cross-domain / cross-LLM / cross-framework transfer.** Memory systems evolved on *TaskCraft only* are then frozen and applied to WebWalkerQA, xBench-DS, and the held-out CK-Pro / OWL frameworks. All show consistent gains (typically 2.0–9.09 % relative), and the same evolved memory transfers to Kimi K2 / DeepSeek V3.2 with no manual adaptation. The authors argue this means MemEvolve learns *task-regime-agnostic* design principles within a shared deep-research family — not that any memory transfers across radically different families (e.g. embodied).

**Comparison vs. 7 human-designed memories** (Table 2, Flash-Searcher + GPT-5-mini, on GAIA / xBench / WebWalkerQA):
- MemEvolve: GAIA 73.33 / xBench 74.0 / WebWalker 74.71. Cost per task GAIA \$0.085 (vs. \$0.086 No-Memory); delay 693s GAIA (similar scale to AWM's 585s and Cheatsheet's 560s).
- Several baselines degrade on at least one benchmark: DILU drops GAIA −2.42; Cheatsheet drops GAIA and xBench; ExpeL underperforms on all three (origin in ALFWorld/HotpotQA — prompts unsuited to long-horizon deep research).
- MemEvolve is the only system with non-negative gains across all three benchmarks (+3.54 to +5.0).

**Discovered architectures.** Three named systems emerge along different evolutionary trajectories:
- **Lightweight** — from a MemoryBank-style few-shot trajectory baseline, evolves into a stage-aware system delivering memory at varying granularities (plan-level, tool-use-level, working-memory).
- **Riva** — AgentKB-style initialization but without the costly offline KB; evolves more agentic encode/retrieve while staying online and lightweight.
- **Cerebra** — extends Riva with tool distillation and periodic working-memory maintenance.

The recurring evolutionary signature is *agentic-ization*: encoding and retrieval shift from pre-defined pipelines to LLM-driven decisions, often with hierarchical memory and meta-guardrails introduced as outer rounds proceed.

## Limitations

- Outer-loop selection is rank-1 plus 3 descendants per round for 3 rounds — population pressure is shallow; broader genetic search might find better Pareto frontiers but at heavy cost.
- Cross-task generalization is demonstrated only within a deep-research/web-browsing task family. The authors explicitly disclaim transfer to fundamentally different families (embodied, code-execution, mathematical reasoning).
- Diagnose-and-design relies on a strong LLM (GPT-5-mini) for both the inner-loop policy and the meta-operator. The framework's compute and API cost during evolution itself is not reported in detail beyond per-task post-evolution metrics.
- Twelve re-implementations are claimed to be faithful but ExpeL underperforming everywhere may partly reflect re-implementation drift, which is not separately ablated.
- No reported variance across evolutionary seeds — a single evolutionary trajectory per (framework, benchmark) is reported, so it is unclear how stable the discovered architectures are.

## Open questions

- Does MemEvolve's "agentic-ization" trend (encode/retrieve becoming LLM-driven) generalize once backbones shift to non-frontier or open-weight smaller models, or does it collapse when the meta-operator LLM is weak?
- What is the marginal value of the *evolution* of architecture vs. simply picking the best single human-designed memory per task family? The Table 2 baselines suggest the gap is real but narrow (best baseline AWM matches MemEvolve on WebWalkerQA within 2.36 pp).
- The four-slot design space is taken as given. Are there memory affordances (e.g. typed multi-store hierarchies, episodic-semantic split, attention-style soft retrieval) that fundamentally don't fit `(E, U, R, G)` and are systematically excluded?
- Can the diagnose-and-design loop become self-improving — i.e. can the meta-operator itself be evolved, not just the memory it produces?
- How does negative transfer interact with the framework? When an evolved memory transfers to a new framework and *hurts* one sub-task while helping the aggregate, no mechanism is provided to detect that.

## My take

The paper's core move — promoting "the memory system" from a designed artifact to a search object — is the right framing for this stage of the agentic-memory field. The four-slot decomposition is more contribution than it gets credit for: by re-implementing 12 disparate systems under one ABC, the authors give the community a tractable design coordinate system. Whether the *specific* dual-evolution recipe wins long-term is less certain — the search is shallow, the meta-operator depends on a single strong LLM, and within-family generalization is demonstrated but cross-family is explicitly disclaimed. The most interesting empirical signal is not the absolute SOTA numbers but the *pattern* — agentic encoding/retrieval keeps emerging across evolutionary trajectories — which suggests a substantive design principle for the next generation of memory systems regardless of whether anyone uses MemEvolve itself.

## Related

- Builds on the unified-codebase line of memory-systems work, particularly MemEngine (referenced as the source for the ingestion/abstraction/retrieval taxonomy MemEvolve refines).
- Companion to AgentKB (used as evolutionary initialization for Riva and Cerebra without inheriting the offline KB) and AWM, Dynamic Cheatsheet, SkillWeaver, G-Memory, Memp, EvolveR (all re-implemented in EvolveLab).
- Adjacent to Darwin-Gödel Machine and Huxley-Gödel Machine on the architectural-self-modification axis, but operates over memory pipelines rather than the agent's code itself.

**Concepts introduced.** [[meta-memory-evolution]] (architecture as a search object) and [[dual-evolution-loop]] (the bilevel inner/outer optimization skeleton that operationalizes it) are the load-bearing conceptual moves. The [[modular-memory-design-space]] — the four-slot `(Encode, Store, Retrieve, Manage)` decomposition realized as the `BaseMemoryProvider` interface — is the engineering substrate that makes both possible.

**Methods.** The system itself is [[memevolve]]; the unified codebase / testbed it depends on is [[evolvelab]], under which all twelve baselines (Voyager, ExpeL, Generative Agents, DILU, AWM, Mobile-E, Dynamic Cheatsheet, SkillWeaver, G-Memory, Agent-KB, Memp, EvolveR) are re-implemented.

**Authors.** Core contributors include [[guibin-zhang]] and Haotian Ren; corresponding senior authors are Wangchunshu Zhou and [[shuicheng-yan]].

**Wiki neighborhood.** Sits in the [[agentic-memory]] and [[skill-evolution]] topics, both of which have multiple precursor entries that MemEvolve unifies under one design space. Closely connected to [[procedural-memory-framework]] (Memp's three-axis B/R/U cartography, generalized here to a four-slot search space) and to [[closed-loop-skill-evolution]] (MemSkill — same nested-loop skeleton applied at the skill-bank level rather than the memory-architecture level).
