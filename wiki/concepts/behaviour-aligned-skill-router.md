---
title: Behaviour-Aligned Skill Router
aliases:
  - behaviour-similar retrieval
  - behavior-aligned router
  - execution-utility router
  - behavioural retrieval
tags:
  - skill-router
  - retrieval
  - offline-rl
  - contrastive-learning
  - llm-agents
maturity: emerging
definition: A skill retrieval policy trained so that "similar" means "would execute successfully on this query", rather than "shares vocabulary with this query"; typically realised as a contrastive embedding model fine-tuned on synthetic (query, target-skill) pairs annotated by an LLM judge.
key_papers:
  - memento-skills-let-agents-design-agents
first_introduced: "Memento-Skills (2026); the behavioural-similarity framing was articulated in this paper."
date_updated: 2026-05-18
related_concepts:
  - read-write-reflective-learning
linked_ideas: []
---

## Definition

A behaviour-aligned skill router is a retrieval module whose similarity score over (query, skill) pairs is calibrated to *execution outcome* rather than lexical or semantic overlap. Concretely, given an LLM agent with a library of executable skills, a behaviour-aligned router answers the question: "**Among all skills in my library, which one, if executed against this query, is most likely to produce the right answer?**" — not "which one has the most overlapping vocabulary with this query".

It is typically realised as an embedding model $\mathrm{enc}_\theta$ trained with a contrastive objective (e.g. multi-positive InfoNCE) on data where positives are (query, skill) pairs whose execution succeeds and hard negatives are (query, skill) pairs that share domain terminology but execute incorrectly. The router score can be interpreted as a soft $Q$-function in a one-step MDP, yielding a Boltzmann routing policy.

## Intuition

Semantic embedding models are trained on text-similarity proxies (paraphrase, NLI, citation, click-through). These reward saying *similar things* — they have no signal about whether two skills *do* similar things when executed. The result is the high-cosine-low-utility failure mode: a router returns the password-reset skill for a refund query because both mention "user account" and "session".

The behaviour-aligned view re-bases the supervision signal. Each training pair is created by:
1. Sampling a skill $d$ from the library;
2. Generating a synthetic user query $q$ that *should* be solved by $d$;
3. Verifying with an LLM judge that executing $d$ on $q$ would indeed succeed.

Hard negatives are crafted from same-domain but execution-incompatible skills. The router learns to push true behavioural positives to the top of its rank list and demote lexically near but operationally wrong candidates.

## Variants

- **Single-step offline RL InfoNCE** (Memento-Skills's Memento-Qwen): multi-positive InfoNCE on Qwen3-Embedding-0.6B, with the router score interpreted as a soft $Q$-function.
- **Cross-encoder reranker**: a higher-capacity model reranks the top-$k$ candidates produced by a fast bi-encoder; can use the same behavioural training data.
- **Hybrid lexical + behavioural**: sparse BM25 recall fused with a behaviour-aligned dense retriever via reciprocal-rank fusion before a behavioural reranker (the retrieval pipeline configuration Memento-Skills actually ships).

## Comparison

| Routing approach | What "similar" means | Recall@1 on Memento-Skills 140-query set |
|---|---|---|
| BM25 | shared bag-of-words tokens | 0.32 |
| Qwen3-Embedding (off-the-shelf semantic) | shared semantic content | 0.54 |
| Memento-Qwen (behaviour-aligned) | shared execution outcome | **0.60** |

End-to-end (route hit rate / judge success): 0.29/0.50 (BM25), 0.53/0.79 (Qwen3), **0.58/0.80** (Memento-Qwen).

## Known limitations

- **Synthetic-query distribution drift.** The training queries are LLM-generated; if real user queries are stylistically very different ("pls fix the thing from last time thx"), the router's behavioural signal can degrade.
- **Hard-negative quality.** Behavioural hard negatives are expensive to mine — same-domain-but-wrong-skill pairs require an LLM judge that itself can be wrong.
- **Recall@1 of 0.60** is still far from saturation; production deployments need cheap escalation when the top-1 pick is wrong.
- **Backbone dependence.** All reported results use Qwen3-Embedding-0.6B; cross-backbone generalisation is unmeasured.
- **Stationarity assumption.** The training distribution assumes a fixed skill catalogue; live deployments grow the catalogue, so the router itself may need periodic re-training.

## Open problems

- How to expand the training set with real-user query distribution without compromising judge quality.
- Whether the soft $Q$-function interpretation can be extended to multi-step routing (when the answer requires composing several skills).
- Curriculum design for hard-negative mining as the skill library grows.

## Relationship to foundations

The training objective is multi-positive InfoNCE, a standard contrastive loss. The single-step MDP framing — state $q$, action $d$, reward $r(q,d)$, horizon 1 — is a degenerate case of offline RL where the "policy" is the Boltzmann distribution over actions induced by the learned score. The KL-regularised view (`Boltzmann routing as maximiser of $\mathbb{E}[Q] - \tau \mathrm{KL}(\pi \,\|\, \pi_0)$ with uniform prior $\pi_0$`) sits inside the soft-RL family.

## My understanding

The strongest pedagogical contribution here is the **explicit name and frame**: previously, "behavioural retrieval" lived implicitly inside skill-routing papers as an ablation choice; Memento-Skills makes it a first-class concept with a clean reduction to one-step offline RL. The empirical evidence (Qwen3 → Memento-Qwen Recall@1 0.54 → 0.60; route hit rate 0.53 → 0.58) is meaningful but not dramatic — semantic embeddings already capture most of the behavioural signal. The gap to BM25 is much larger, suggesting that the practical takeaway is "any decent dense retriever beats BM25; behavioural fine-tuning gives a small extra kick on top". The framing is more durable than the specific delta.
