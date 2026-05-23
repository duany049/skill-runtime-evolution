---
title: Skill Retrieval
aliases:
  - agent skill search
  - skill search
  - skill selection
tags:
  - skill-retrieval
  - retrieval
  - llm-agents
  - agentic-skills
maturity: emerging
definition: The task of finding, ranking, and selecting agentic skills from a large repository so that an LLM agent can load only the skills relevant to the current task, encompassing both algorithmic ranking and the agent's policy decision of which retrieved candidates to actually consume.
key_papers:
  - how-well-agentic-skills-work-wild
first_introduced: how-well-agentic-skills-work-wild (this paper) — first formalization as a distinct stage with its own evaluation metrics
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

*Skill retrieval* is the stage between a user task and skill consumption: given a task description and a skill collection of size N (in the 10⁴–10⁵ range for realistic deployments), produce a top-k subset of skills and have the agent decide which of those k to actually load into context. This decomposes into two coupled sub-problems: an *information-retrieval* problem (rank skills by relevance to the task, measured by Recall@k against a ground-truth skill set) and a *selection-policy* problem (the agent's downstream choice of which retrieved skills to use, measured by skill-loading rate and task pass rate). [[how-well-agentic-skills-work-wild]] establishes both as first-class evaluation targets.

## Intuition

A skill collection is qualitatively different from a typical RAG corpus: skills are workflows and patterns, not documents; relevance is partially executable (a skill "matches" if its instructions actually help the task); and a wrong skill can mislead the agent more than no skill at all. A naive direct-query retriever using the task description as the search string is therefore a weak baseline; iterative, agentic strategies that let the model reformulate queries and inspect candidates substantially outperform direct retrieval. The bottleneck is rarely lexical overlap and often the agent's ability to recognize a useful skill from name + description alone.

## Variants

- *Direct retrieval* — single fixed query (task description) against a dense or sparse index; the top-k is the output.
- *Agentic search* — the agent calls a retrieval tool, reformulates queries based on returned snippets, and iterates until it commits to a set.
  - *Keyword* (BM25 only)
  - *Semantic* (dense embeddings only)
  - *Hybrid* (combined BM25 + dense)
  - *Hybrid w/ content* — similarity computed over both metadata (name + description) and full `SKILL.md` content.

## Comparison

vs. *general RAG*: skill retrieval optimizes for a downstream agent's *use*, not for a reader's *answer*. A skill is only relevant if the agent loads it *and* uses it productively.

vs. *tool selection* (function-call routing): tools have fixed APIs and clear success/failure signals; skills are open-ended documents and "wrong skill" failures are softer and harder to detect.

vs. *procedural-memory retrieval*: in skill retrieval the corpus is external and shared; in procedural memory it is internal and learned from past trajectories.

## Known limitations

- Ground-truth skill labels (used to compute Recall@k) only exist on skill-curated benchmarks; for realistic skill libraries there is no ground truth, only proxy metrics.
- Recall@k may overstate practical utility — a skill that ranks at k=10 but is never loaded by the agent contributes zero to pass rate.
- Adding more retrieved skills does not monotonically help: noise can crowd out relevant skills in the agent's selection step.

## Open problems

- Learned reranking using skill-usage outcomes as supervision is largely unexplored.
- Joint retrieval + selection optimization (instead of decomposed stages) could exploit signals lost at the ranking step.
- Multi-task and multi-skill retrieval (the agent loads complementary skills that together cover the task) is under-formalized.

## Relationship to foundations

Skill retrieval builds on retrieval-augmented generation, dense-passage retrieval, and BM25 sparse retrieval, transplanted into the agentic-skill setting. The novel element is the second-stage selection policy run by the LLM agent itself, which has no direct analog in standard RAG.

## My understanding

The high-value research direction is not better Recall@k — agentic hybrid search already gets to ~68% Recall@10 — but closing the gap between retrieval quality and end-task pass rate, which mostly lives in the agent's selection step and in the actual usability of retrieved skills. The paper's coverage-score analysis is the clearest hint that "relevance" needs to be operationalized as "the agent can extract usable information," not as lexical or semantic similarity.
