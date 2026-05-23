---
title: Hybrid Memory Retrieval
aliases:
  - scratchpad and vector memory
  - deterministic plus selective retrieval
  - dual-path memory retrieval
tags:
  - agentic-memory
  - retrieval
  - memory-architecture
maturity: emerging
definition: A two-path memory observation scheme combining a deterministic always-retrieved scratchpad entry with a selective query-based vector retrieval, used to rank information by importance for an LLM agent.
key_papers:
  - atommem-learnable-dynamic-agentic-memory-atomic
first_introduced: AtomMem (Huo et al., 2026)
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

Hybrid memory retrieval is the design pattern of presenting an LLM agent with memory through two parallel channels at every step. A **deterministic** channel returns a fixed scratchpad entry `m_t^scr` regardless of query — its retrieval schedule is mandatory, and functionally it holds the global task state and step-wise pivotal information. A **selective** channel uses a textual query `q_t` (the previous step's Read parameter) to fetch `TopK` entries from a vector database by semantic similarity. The agent's observation `o_t = {o_t^env, m_t^scr, M̂_t}` combines both. Empirically the two channels are complementary — removing either causes a moderate drop, removing both is catastrophic, indicating they preserve fundamentally different information.

## Variants

- **Scratchpad-only retrieval** — keeps the deterministic channel, drops the vector store; weakest among ablations on long-context QA.
- **Storage-only retrieval** — keeps the vector store, drops the scratchpad; barely benefits from RL, suggesting the scratchpad is what carries the global state needed for stable policy learning.
- **Embedding model sensitivity** — selective retrieval quality scales with embedding model size (Qwen3-embedding-0.6B → 4B → 8B improves end-task averages by ~1.7 points).

## Why it matters

Pure vector retrieval has an unavoidable failure mode: when the query is weak or the relevant entry was never written, retrieval returns noise. A deterministic scratchpad short-circuits this by always returning the global task state, while the selective channel keeps the system extensible to large memory sets. The hybrid arrangement is what enables the **learned CRUD policy** to make stable per-step decisions even when retrieval over the vector store is imperfect.

## Related ideas

(none yet)
