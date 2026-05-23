---
title: Agent Skill Runtime Self-Evolution
scope: How LLM-based agents acquire, store, refine, and evolve reusable skills and memory at runtime — without parameter updates — through interaction with environments and users.
key_topics:
  - procedural-memory
  - skill-evolution
  - agentic-memory
  - continual-learning-evaluation
paper_count: 26
date_updated: 2026-05-18
---

## Overview

Modern LLM agents are powerful as zero-shot reasoners but brittle as continual learners: once deployed, their weights are frozen, and any "experience" they appear to accumulate lives in prompts or external stores. This Summary tracks a recent line of work that treats the *agent runtime itself* as the substrate for adaptation — encoding what was learned into procedural memory, skill libraries, or modulated decoding, all without gradient updates.

## Core areas

- [[procedural-memory]] — encoding "how-to" knowledge extracted from past trajectories into reusable, executable procedures.
- [[skill-evolution]] — autonomous discovery, refinement, retirement, and recomposition of skills across a deployed agent's lifetime.
- [[agentic-memory]] — broader memory architectures (episodic + semantic + procedural) and mechanisms for selective recall, consolidation, and updating.
- [[continual-learning-evaluation]] — benchmarks and protocols for measuring whether agents actually improve with experience under realistic conditions.

## Evolution

Early skill-learning agents (Voyager, JARVIS-1) demonstrated that LLMs can grow skill libraries through self-driven exploration in open-world games. The line then split: one branch refined what to *store* (procedural memory abstractions, atomic memory operations, modular memory units), another refined how to *update* skills from continued use (bottom-up evolution, cross-user aggregation, online evolution from feedback), and a third asked the meta-question — *does this actually generalize* under realistic, long-horizon, multi-user conditions (benchmarks for skill usage, lifelong skill discovery, real-world continual learning).

## Current frontiers

- Test-time policy optimization without gradient updates (just-in-time RL, logit-level modulation).
- Cross-user collective skill evolution and library curation.
- Failed-trajectory learning, especially in GUI/computer-use settings.
- Benchmarks that disentangle skill *use* from skill *generation* and skill *maintenance*.

## Key references

Anchor papers in this area are tracked by their respective topics. See [[procedural-memory]], [[skill-evolution]], [[agentic-memory]], and [[continual-learning-evaluation]] for paper lists.

## Related

This Summary anchors the domain. Individual contributions are linked from the topic pages above.
