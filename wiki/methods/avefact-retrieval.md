---
name: AveFact Retrieval
slug: avefact-retrieval
type: inference
tags:
  - retrieval
  - procedural-memory
  - vector-search
  - keyword-extraction
source_papers:
  - memp-exploring-agent-procedural-memory
parent_methods: []
child_methods: []
code_repo: https://github.com/zjunlp/MemP
date_updated: 2026-05-18
---

## Problem setting

When retrieving procedural memories for a new task, the choice of *retrieval key* materially shapes which memories surface. Using the raw task description (Query) anchors retrieval to surface lexical similarity. Using random sampling discards relevance entirely. AveFact is a middle ground that emphasizes *core task elements* extracted by an LLM rather than the verbatim query.

## Mechanism

AveFact constructs each retrieval key by averaging the embeddings of LLM-extracted keywords from the task description:

1. Given a task description $t$, prompt a base LLM to extract a list of keywords $K(t) = \{k_1, k_2, \ldots, k_n\}$ that summarize the task's core elements (entities, verbs, constraints).
2. Embed each keyword: $\phi(k_i)$.
3. The retrieval key is the average $\bar\phi(t) = \frac{1}{n}\sum_i \phi(k_i)$.
4. At retrieval, score stored memories by cosine similarity between their $\bar\phi$ and the new task's $\bar\phi$. Return top-k.

The averaging step is meant to attenuate spurious lexical variation in the raw query while preserving the centroid of task semantics.

## Procedure

1. **Indexing:** for each stored memory, precompute its AveFact key during the Build phase.
2. **Querying:** at inference, run the keyword extractor on $t_{\text{new}}$ and compute $\bar\phi(t_{\text{new}})$.
3. **Scoring:** rank stored memories by $\cos(\bar\phi(t_{\text{new}}), \bar\phi(t_i))$.
4. **Return:** top-k memories, passed downstream to (e.g.) [[proceduralization]].

## Assumptions

- The LLM keyword extractor reliably surfaces the task's task-relevant elements.
- The embedding model produces semantically meaningful keyword vectors at the keyword granularity (not just sentence granularity).
- Averaging is a reasonable aggregation; outlier keywords don't dominate the centroid.

## Limitations

- Adds an LLM call per indexed item and per query.
- Sensitive to the keyword-extractor prompt; no formal robustness analysis.
- The averaging step can wash out distinctive constraints — consistent with the observed Hard-Constraint regression on TravelPlanner.
- Empirically beats Random Sample and raw-Query on Commonsense scores, but not always on Hard-Constraint scores.

## Tradeoff profile

| | Random Sample | Key=Query | AveFact |
|---|---|---|---|
| Retrieval cost | lowest | low | medium |
| Build cost | none | none | LLM call per item |
| Semantic precision | none | medium | higher on CS, lower on HC |
| Robustness to query surface form | n/a | low | medium |

Best when tasks vary in surface phrasing but cluster around shared core elements. Worst when the task hinges on a single distinctive constraint that an averaged centroid will dilute.
