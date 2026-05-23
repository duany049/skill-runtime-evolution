---
title: Modular Memory Design Space
aliases:
  - four-slot memory decomposition
  - Encode-Store-Retrieve-Manage decomposition
  - E/U/R/G design space
  - BaseMemoryProvider interface
tags:
  - agentic-memory
  - memory-architecture
  - modular-design
  - codebase
maturity: emerging
definition: "A four-slot decomposition of any self-improving agent memory system as Ω = (Encode, Store, Retrieve, Manage), used both as a taxonomy that admits a dozen heterogeneous published systems under a unified interface and as a genotype that meta-evolutionary search can mutate."
key_papers:
  - memevolve-meta-evolution-agent-memory-systems
first_introduced: "MemEvolve / EvolveLab (OPPO AI Agent Team & LV-NUS lab, 2025)"
date_updated: 2026-05-20
related_concepts:
  - procedural-memory-framework
  - modular-procedural-memory
  - memory-update-policies
linked_ideas: []
---

## Definition

The modular memory design space factorizes any self-improving agent memory system into four functionally distinct, programmatically substitutable slots:

- **Encode (E)** — transforms raw experiences (trajectory segments, tool outputs, self-critiques) into structured representations: `e_t = E(ε_t)`. Choices range from raw-trace compression (Voyager, MemoryBank) to LLM-distilled lessons (Reflexion, ExpeL) to API extraction (SkillWeaver) to workflow induction (AWM).
- **Store (U)** — integrates encoded experiences into persistent memory: `M_{t+1} = U(M_t, e_t)`. Substrates include vector databases (ExpeL, AWM), knowledge graphs (G-Memory, Zep), tool libraries (SkillWeaver), JSON dicts (Cheatsheet, Memp), and hybrid databases (Agent-KB).
- **Retrieve (R)** — provides task-relevant context: `c_t = R(M_t, s_t, Q)`. Implementations span semantic search, contrastive comparison (ExpeL, EvolveR), function matching (SkillWeaver), and graph traversal (G-Memory).
- **Manage (G)** — offline / asynchronous consolidation, abstraction, and forgetting: `M'_t = G(M_t)`. Examples include skill pruning (SkillWeaver), episodic consolidation (G-Memory), deduplication (Agent-KB), failure-driven adjustment (Memp), and periodic update/pruning (EvolveR). Many early systems leave Manage empty.

Each concrete memory system is then a tuple `Ω = (E, U, R, G)` over programmatic implementations of these slots — a "genotype" that admits direct comparison, ablation, and mutation. The design space is realized as a unified `BaseMemoryProvider` abstract base class (the EvolveLab codebase) under which twelve published systems — Voyager, ExpeL, Generative Agents, DILU, AWM, Mobile-E, Dynamic Cheatsheet, SkillWeaver, G-Memory, Agent-KB, Memp, EvolveR — are re-implemented with the same interface and the same evaluation harness.

## Intuition

The published self-improving memory literature is a Cambrian explosion of incompatible architectures: each paper hand-rolls its ingest/abstract/retrieve pipeline, embeds it in its own scaffolding, and reports results on a non-overlapping benchmark slice. The community has had no shared coordinate system for asking "which slot of system A would help system B?" or "what is the empirical Pareto frontier across the design space?"

The four-slot decomposition is the move that makes those questions tractable. By picking a granularity where the slots are (i) coarse enough to map an arbitrary published system onto, but (ii) fine enough that mutating one slot still yields an executable system, the framework turns the literature into a structured taxonomy and the design space into a navigable surface.

A second consequence: once slots are interchangeable through a common interface, the design space becomes machine-searchable. This is what allows MemEvolve's meta-evolutionary loop to propose new memory architectures by mutating one slot at a time — descendants remain executable by construction because they conform to the same `BaseMemoryProvider` contract.

## Variants

- **Slot granularity** — finer (e.g. splitting Encode into "extraction" + "abstraction" sub-stages) vs. coarser (collapsing Store and Manage into one). MemEvolve picks the four-way split as a compromise; finer splits could expose more design corners at the cost of harder search.
- **Slot-coupling discipline** — strict interface (slot outputs typed and slot-internal state hidden, as in EvolveLab) vs. loose (slots may share global state). Strict discipline is what makes mutation safe.
- **Single-vs-multi-store** — current decomposition treats `M_t` as one structure; a typed multi-store variant (episodic/semantic split, working-memory side-channel) would require either a richer Store slot or a new top-level slot.
- **Per-slot programmability** — slots filled by hand-written code (current EvolveLab) vs. slots filled by LLM-emitted code or prompts (a step toward fully agentic memory pipelines).

## Comparison

- vs. **Memp's Build/Retrieve/Update framework** ([[procedural-memory-framework]]): a three-axis lifecycle decomposition for procedural memory specifically. The MemEvolve four-slot space generalizes the same intuition to all self-improving memory (not just procedural) and separates Encode/Store (Build's two sub-steps) and renames Update → Manage for offline consolidation. Memp's framework is used as a cartography tool for measurement; the EvolveLab design space is additionally used as a genotype for search.
- vs. **LEGOMem's role-aware procedural memory** ([[modular-procedural-memory]]): LEGOMem decomposes a multi-agent memory along *agent roles* (orchestrator vs. task agent); the modular memory design space decomposes along *functional pipeline stages*. The two are orthogonal — a role-aware system could in principle have its own four slots per role.
- vs. **MemEngine taxonomy** (Zhang et al., cited as the precursor to the modular framing): MemEngine introduces an ingestion/abstraction/retrieval three-way taxonomy. EvolveLab refines this into the (E, U, R, G) four-way split and operationalizes it as a unified codebase covering 12 baselines.
- vs. **bespoke memory architecture papers** (Voyager, ExpeL, AWM, ...): each paper presents one architecture as a monolith. The design space rewrites each as a slot tuple, which (a) exposes which design choice each paper actually contributes and (b) lets ablation be performed by slot replacement rather than full re-implementation.

## Known limitations

- The four-slot abstraction is taken as given. Memory affordances that fundamentally don't fit `(E, U, R, G)` — typed multi-store hierarchies, episodic-semantic splits, attention-style soft retrieval, hierarchical working memory — are systematically excluded or forced into an awkward fit.
- Re-implementation faithfulness is claimed for all 12 baselines but not separately ablated; underperformance of any single baseline could partially reflect re-implementation drift rather than the architecture itself (the MemEvolve paper flags ExpeL specifically as a candidate).
- The codebase enforces a *programmatic* slot interface, which biases the design space toward implementations expressible as concise functions; richer learned components (e.g. trained retrieval policies, neural managers) require extra plumbing.
- "Manage" is often empty in early systems; whether this slot is a fundamentally separate axis or a special case of Store / asynchronous Encode is not resolved.

## Open problems

- Is there a principled way to *learn* the slot decomposition itself from a corpus of memory systems, rather than hand-fixing it at four?
- How should the design space accommodate hierarchical memory (memory-of-memory, meta-guideline banks on top of experience banks) without inflating the slot count?
- What is the right typing discipline so that slot mutations preserve correctness automatically, eliminating the need for executability checks during meta-evolution?
- Whether the same decomposition transfers across task families (deep research → embodied → code execution) or whether each family demands its own slot palette.

## Relationship to foundations

Inherits from software-engineering modular-design principles (Parnas-style information hiding, abstract base classes) and from neuroscience-inspired multi-component memory models (Atkinson-Shiffrin sensory/short/long; Tulving episodic/semantic), but operationalizes the decomposition at the level of an LLM-agent runtime rather than as a cognitive model. The unified `BaseMemoryProvider` ABC is the engineering artifact that makes the decomposition load-bearing.

## My understanding

The contribution that may outlast MemEvolve's specific evolutionary recipe is this design space itself. By re-implementing 12 disparate published systems under a single ABC and a single eval harness, the EvolveLab work gives the field a shared coordinate system — and shared coordinates is what makes a community of research possible. Whether the right number of slots is four (vs. three, five, or context-dependent) is a second-order question; the load-bearing move is treating "the memory system" as a typed composition of pluggable parts rather than an indivisible artifact. The most interesting near-term direction is augmenting the slot palette without losing the unified-codebase property — admitting hierarchical memory or typed multi-store designs while still keeping mutation safe.
