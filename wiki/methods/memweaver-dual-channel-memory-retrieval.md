---
name: MemWeaver Dual-Channel Memory Retrieval
slug: memweaver-dual-channel-memory-retrieval
type: inference
tags:
  - retrieval-augmented-generation
  - knowledge-graph
  - agentic-memory
  - hybrid-retrieval
  - long-horizon-reasoning
source_papers:
  - memweaver-weaving-hybrid-memories-traceable-long
parent_methods: []
child_methods: []
code_repo: https://github.com/Chengkai-Huang/MemWeaver_code
date_updated: 2026-05-19
---

## Problem setting

Given a long-horizon agent memory state `M = {G, E, P}` (a temporally grounded knowledge graph G, an experience-abstraction layer E, and a passage layer P with cross-layer structural edges `contains` / `about`) and a user query Q, the goal is to assemble a *compact* inference context — small enough to keep a long-context baseline's >22K-token input down to under 1K tokens — that simultaneously delivers (a) precise compositional relational facts and (b) verbatim evidence supporting those facts. Existing single-channel retrievers force a tradeoff: pure dense retrieval over passages misses relational structure; pure graph traversal lacks recall and verbatim grounding.

## Mechanism

The retriever exposes two independent channels that are unioned, deduplicated, and assembled into the LLM context:

**Channel A — Structured KG Retrieval.** Embed semantic relation edges in G (excluding the structural `contains` / `about` edges, which are wiring rather than knowledge). Retrieve seed triples by cosine similarity to the query embedding φ(Q); expand a bounded-hop neighborhood from involved entities to gather candidate triples `R_cand`; filter by similarity to bound context size; pass the candidates to an LLM selector with fixed prompt `P_select`. The selector's output `R_LLM` is unioned with the top-`k_r` highest-similarity triples for backfill:
  `R★ = R_LLM ∪ Top_{k_r}(R_cand; Q)`
The union prevents over-narrow contexts that pure LLM-selection can produce. `R★` is serialized to text as `C_KG`.

**Channel B — Textual Evidence Retrieval.** Two sources fused:
  1. *Anchored:* for entities appearing in `R★`, follow structural edges — `contains` to attached passage nodes, `about` to attached experience nodes.
  2. *Global:* an independent dense retriever queries Q against all dialogue units for a recall side-channel.
All retrieved texts are ranked by cosine similarity to Q and deduplicated by dialogue-unit ID and content. The top items form `C_TXT`, with retrieval budgets `k_p` for passages and `k_e` for experience items.

**Context assembly.** The two channels are concatenated into a single prompt: `f_LLM(Q, C_KG, C_TXT)`. Both channels share the same sentence encoder φ (all-MiniLM-L6-v2 in the reference implementation) so similarity scores are comparable across channels for deduplication.

The structural innovation is that Channel B uses Channel A's selected entities as anchors — the textual evidence is not retrieved independently but is conditioned on what the graph channel surfaced. This is what makes the assembled context coherent rather than two parallel blocks of unrelated retrieval.

## Procedure

1. Encode query: `q = φ(Q)`.
2. Channel A:
   a. Retrieve seed triples by cosine similarity from the dense triple index `I_T`.
   b. Expand bounded-hop neighborhood in G to produce candidate set `R_cand`.
   c. Filter `R_cand` by similarity threshold to bound size.
   d. Send `(Q, R_cand)` to LLM with prompt `P_select`; receive `R_LLM`.
   e. Compute `R★ = R_LLM ∪ Top_{k_r}(R_cand; q)`.
   f. Serialize `R★` to text → `C_KG`.
3. Channel B:
   a. For each entity in `R★`, follow `contains` edges → anchored passages; follow `about` edges → anchored experience items.
   b. Query global passage index with q → top recall-side passages.
   c. Rank all retrieved texts by cos(φ(text), q); deduplicate by dialogue-unit ID + content match.
   d. Take top-`k_p` passages and top-`k_e` experience items → `C_TXT`.
4. Form prompt `(Q, C_KG, C_TXT)` and call backbone LLM `f_LLM` to produce answer.

Default retrieval budgets reported: `k_r = k_p = k_e = 6`. Category-specific tuning is permitted but the method is robust across {3, 6, 9, 12}.

## Assumptions

- A pre-built tri-layer memory exists with `contains` and `about` structural edges in place (built offline, e.g., by [[tri-layer-memory-consolidation]] writers).
- A shared sentence encoder maps queries, triples, passages, and experience items into a comparable vector space.
- An LLM is available to act as the triple selector (any model capable of constrained selection; in practice a chat model with a fixed prompt template).
- Dialogue units have stable identifiers so deduplication by ID is meaningful.
- Triple-level temporal metadata is normalized to absolute time at write time — the retriever does not perform temporal normalization at inference.

## Limitations

- Retrieval-time latency (~41.57 ± 12.85 ms reported on the reference setup) is ~2–3× higher than flat retrieval (~16–17 ms), driven by graph expansion and the LLM-selector call. Not engineered for sub-millisecond serving.
- The LLM-selector call introduces a model dependency on a third LLM (in addition to the offline builder and the inference backbone) — though in practice the selector is usually the same model as the backbone, increasing total inference token cost.
- The bounded-hop expansion uses a single-hop neighborhood by default; deeper compositional questions may require multi-hop expansion at additional cost.
- Recall depends on graph quality: if a relevant entity was not extracted at write time, neither channel can recover it (Channel B's global retrieval may, but lacks the structural anchor).
- The dual budgets `(k_r, k_p, k_e)` are tunable but the method does not auto-adapt to query type — Single-Hop questions probably need different budgets than Temporal questions, but the paper applies category-specific tuning rather than online adaptation.

## Tradeoff profile

- **Latency vs context length:** ~2.5× retrieval-time cost for ~30× input-token reduction vs long-context baseline. Net inference cost typically lower for backbones whose latency scales with input length.
- **Quality vs simplicity:** Dual-channel union + LLM selector + global recall side-channel is significantly more complex than single-channel retrieval. The complexity buys robustness — global side-channel covers entity-extraction failures, LLM selector handles ambiguity that pure cosine fails at, backfill prevents over-narrow contexts.
- **Selectivity vs recall:** The method is biased toward selectivity (filter, LLM-select, dedupe), trading some recall for context tightness. Hyperparameter sensitivity is mild, suggesting the bias is reasonable.
- **Builder dependence:** Inference quality with this retriever inherits the offline builder's extraction quality; the retriever cannot correct for missing entities or relations.
- **Best fit:** long-horizon settings (multi-session dialogues, long agent trajectories) where context length is the binding constraint and the underlying graph is high-quality. Less attractive when memory is small enough to fit in context anyway, or when sub-50ms retrieval latency is required.
