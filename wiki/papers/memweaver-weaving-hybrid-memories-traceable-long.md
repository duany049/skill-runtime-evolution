---
title: "MemWeaver: Weaving Hybrid Memories for Traceable Long-Horizon Agentic Reasoning"
slug: memweaver-weaving-hybrid-memories-traceable-long
arxiv: "2601.18204"
venue: arXiv.org
year: 2026
tags:
  - agentic-memory
  - long-term-memory
  - memory-architecture
  - knowledge-graph
  - retrieval-augmented-generation
  - long-horizon-reasoning
importance: 3
date_added: 2026-05-19
source_type: tex
s2_id: 6185f73684109d7f87f92d7f4db44c310b7ddf77
tldr: "MemWeaver consolidates long-horizon agent experience into a tri-layer memory — a temporally grounded knowledge graph, an experience-abstraction layer, and a passage-evidence layer — and serves it via dual-channel retrieval that cuts input context >95% over long-context baselines while improving multi-hop and temporal QA on LoCoMo."
contribution_type:
  - method
  - system
datasets:
  - LoCoMo
code_url: https://github.com/Chengkai-Huang/MemWeaver_code
cited_by: []
---

## Problem & Context

LLM agents deployed in long-horizon, multi-session settings (conversational assistants, personalized agents) need external memory because context windows cannot hold the entire interaction history. Prior memory designs fall into three families, each with a structural weakness:

- **Flat retrieval memory** (MemoryBank, raw passage stores) stores past turns as embeddings. It lacks relational structure and is brittle on multi-hop or time-constrained queries.
- **Structured memory** (knowledge-graph variants like GraphRAG, HippoRAG) models entities and relations but suffers from noisy extraction and accumulated logical conflicts when no session-level verification is performed.
- **Abstraction / experience memory** (Reflexion-style, summary-based) distills preferences and strategies, but the abstractions are weakly grounded — agent decisions cannot be traced back to concrete evidence.

A second cross-cutting limitation: most systems treat memory as a *passive retrieval buffer*. They retrieve, but do not systematically *consolidate* (verify, normalize timestamps, deduplicate, abstract) as interactions accumulate. Long-context baselines (full-history prompting) work but require >22K tokens per query and still struggle on temporal questions because the model must time-reason inside an unstructured blob.

The authors frame the central question as: how can an agent systematically consolidate long-term interaction history into representations that simultaneously support temporal consistency, compositional reasoning, cross-session generalization, and evidence-grounded decision making?

## Key idea

Memory should be a **consolidation process**, not a static store. MemWeaver decomposes long-term memory into three loosely coupled layers, each targeted at a different reasoning capability, and threads them together via shared semantic representations and explicit structural links:

1. **Graph Memory (GM)** — a directed knowledge graph of entities and semantic relations, where every relation triple carries normalized absolute-time metadata, an optional condition, and a provenance pointer. A session-level review step (LLM-issued `add` / `update` / `deny` operations) reconciles conflicts before triples are committed to the dense triple index.
2. **Experience Memory (ExpM)** — DBSCAN-clustered episodic dialogue windows, each cluster validated for coherence by an LLM, then distilled into reusable "experience items" only when supported by multiple consistent interactions. Online routing uses cosine similarity with two thresholds and a buffered re-clustering policy to keep the layer evolving cheaply.
3. **Passage Memory (PM)** — a dense index over original text spans, attached to GM entity nodes via `contains` edges and to ExpM items via `about` edges, providing the verbatim evidence channel for traceability.

At inference, a **dual-channel retrieval** strategy fuses both worlds: structured retrieval over GM yields precise relational facts, and textual retrieval over PM+ExpM (anchored on entities surfaced by GM, plus a global recall side-channel) provides supporting evidence. The combined `(C_KG, C_TXT)` context is fed to any backbone LLM. The structural innovation is the explicit linking among the three layers, which makes the system traceable end-to-end: any retrieved fact can be followed back to its supporting passages and experience items.

## Method

**Memory state.** Formally the agent memory is `M = {G, E, P}`. Each dialogue turn `x_i = <q_i, a_i, s_i, t_i>` is encoded by a shared sentence encoder φ (all-MiniLM-L6-v2). Memory writing is incremental — `M_{i+1} = Update(M_i, x_i)` — and memory-based reasoning produces an answer `y = LLM(Q, C_KG, C_TXT | P_ans)`.

**Graph Memory writing.** For each new turn the system first creates a passage node preserving the raw text plus speaker/timestamp metadata. Two-stage LLM extraction follows: prompt `P_ent` extracts entities, then `P_rel` extracts relation candidates under entity constraints. A dedicated time-normalization function η lifts relative timestamps ("yesterday", "last month") into normalized absolute time expressions `t̂`, which become part of each relation's metadata `m(r) = {t̂, c, π}`. After the per-turn write, a session-level review prompt `P_review` serializes all new entities/relations from the current session and asks an LLM to output `add` / `update` / `deny` operations to fill, correct, or remove triples. Redundant triples are pruned, then the dense triple index `I_T` is rebuilt as the structured-retrieval entry point.

**Experience Memory induction.** DBSCAN clustering (eps=0.3, min_samples=2, cosine distance) groups dialogue units. Each candidate cluster passes an LLM coherence screen; consistent ones get a `center_text` theme summary and are finalized. Within each cluster, an LLM induces small experience items, each linked to multiple supporting dialogue-unit IDs (provenance is mandatory). Duplicate and small-talk items are filtered out.

For online updates, the system implements three-way routing on cosine similarity of new turn `e_i` to cluster center `μ_j`:

- `cos ≥ τ_high (0.8)` → directly merge into best cluster
- `τ_low ≤ cos < τ_high` → submit shortlist of candidate clusters to an LLM router with prompt `P_route` for disambiguation; if router returns `none`, queue to pending buffer
- `cos < τ_low (0.5)` → queue to pending buffer for future re-clustering when the buffer fills

Each cluster maintains an `add_buffer`; when it exceeds threshold 4, the cluster center is recomputed and experience items are re-induced, amortizing LLM cost.

**Dual-channel retrieval.** Given query Q:

- **Structured KG retrieval.** Embed semantic relation edges (excluding structural `contains` / `about`); retrieve seed triples by cosine similarity; expand a bounded-hop subgraph; filter candidates by similarity; pass to an LLM selector with prompt `P_select`; union the LLM-chosen set with top-`k_r` high-similarity triples to avoid overly narrow context. Produces `C_KG`.
- **Textual evidence retrieval.** For entities appearing in selected triples, pull attached passage nodes via `contains` and experience nodes via `about`; in parallel, query a global dense retriever over all dialogue units for recall; rank by similarity, deduplicate by content and dialogue ID. Produces `C_TXT`.
- **Context assembly.** Serialize the selected subgraph as `C_KG`, concatenate top passages and experience items as `C_TXT`, feed `f_LLM(Q, C_KG, C_TXT)`. Default retrieval budgets `k_r = k_p = k_e = 6`.

**Memory construction model.** Crucially, all offline memory operations (entity/relation extraction, temporal normalization, session-level verification, experience induction) are performed by DeepSeek-V3.2 (non-thinking), regardless of which backbone runs inference. This decouples memory-write quality from inference-time model capacity — small backbones like Qwen2.5-1.5B still query a high-quality memory.

## Experiment & Results

**Benchmark.** LoCoMo (Snap Research), a long-term conversational QA benchmark with ~9K tokens and up to 35 sessions per dialogue. Five question categories; evaluation focused on the four answerable ones — Single-Hop, Multi-Hop, Temporal, Open-Domain. 7,512 QA pairs total.

**Backbones.** GPT-4o-mini (API), Llama3.2-3B, Llama3.2-1B, Qwen2.5-1.5B (Ollama-served). Memory built once with DeepSeek-V3.2; all backbones query the same memory state.

**Baselines.** LoCoMo (long-context prompting over the full dialogue), MemoryBank (flat retrieval), ReadAgent (selective summarize-and-retrieve), A-Mem (atomic agentic memory).

**Headline numbers (GPT-4o-mini).** MemWeaver vs A-Mem (the strongest baseline):

- Multi-Hop F1: 26.00 vs 23.68 (LoCoMo long-context: 24.35)
- Temporal F1: 50.83 vs 38.77 (LoCoMo: 22.54) — +12.06 absolute over A-Mem
- Open-Domain F1: 20.73 vs 12.50
- Single-Hop F1: 39.20 vs 35.13 (LoCoMo: 42.39 — long-context still wins here)
- Average input tokens: 672 vs LoCoMo's 21,625 — >95% reduction

**Smaller backbones benefit more.** Qwen2.5-1.5B Temporal F1 rises from 14.54 (A-Mem) to 46.07 (MemWeaver), a +31.5 absolute jump. The authors attribute this to structured retrieval externalizing temporal cues that small models cannot reliably reason over inside an unstructured long context.

**Ablation (Table 2).** Removing ExpM causes moderate consistent drops (e.g., Temporal F1 50.83 → 48.67 on GPT-4o-mini). Removing GM is catastrophic on Multi-Hop and Temporal: 26.00 → 13.97 and 50.83 → 9.58. The graph is the load-bearing component; experience abstractions are an incremental but consistent helper.

**Resource cost (Table 3).** Total memory footprint 13.31 MB (GM 5.24, ExpM 8.07) — heavier than MemoryBank (7.23 MB) and A-Mem (18.29 MB intermediate). Retrieval latency 41.57 ± 12.85 ms, vs 16–17 ms for the flatter baselines — a 2–3× retrieval-time cost traded for the accuracy and context-length wins.

**Hyperparameter sweep.** MemWeaver is stable across `(k_r, k_p, k_e)` in {3, 6, 9, 12}; performance varies mildly with no abrupt collapses. Bigger retrieval budgets give diminishing or negligible gains — selectivity matters more than volume.

**Human evaluation.** Annotators rated retrieved-knowledge support quality on 25 sampled questions per category with GPT-4o-mini. MemWeaver beat A-Mem across all four answerable categories on perceived evidence sufficiency.

## Limitations

- Memory-write quality remains coupled to the underlying LLM (DeepSeek-V3.2 here) — different builder models produce different abstractions and entity normalizations. Robustness across builders is untested.
- Text-only setting; no multimodal interactions (image, audio).
- The paper does not characterize failure modes when DBSCAN parameters or similarity thresholds drift from the tuned values; the hyperparameter sweep covers `k_*` but not the writer-side knobs (`eps`, `min_samples`, `sim_high`, `sim_low`, buffer threshold).
- Retrieval latency increase (~2.5× vs flat baselines) is acknowledged but not engineered down.
- Single benchmark (LoCoMo) — generalization to non-dialog long-horizon traces (e.g., agent task trajectories on AppWorld, WebArena) is left for future work.

## Open questions

- How does memory-write quality degrade when the offline builder is a smaller / weaker model? Is the asymmetry between builder and backbone the actual driver of the small-backbone gains, or is it the structural decoupling?
- Can temporal normalization survive multi-timezone or fictional-time settings? The paper assumes the dialogue's timestamps are coherent.
- The session-level review step performs `add` / `update` / `deny`. What happens at very long horizons where deny accumulates faster than evidence — does the graph become brittle?
- How does the system handle conflicting facts that genuinely change over time (a user's preference shifting)? "Temporal grounding" implies the older fact should remain but be flagged as superseded; the paper does not elaborate the policy.
- ExpM is induced from clusters of similar dialogue units. What guarantees that the induced experience items capture *behavior patterns* rather than just topical repetition? The current coherence check is semantic-similarity-driven, not behavioral.

## My take

The paper's structural commitment is the right one: the *tri-layer + explicit cross-layer links + traceable provenance* design treats memory as plumbing rather than a black box. The headline win is not actually accuracy — it is the 95% context-length reduction at parity-or-better accuracy, which is the deployment story for long-horizon agents.

The most interesting finding is small-backbone amplification: Qwen-1.5B with MemWeaver on Temporal questions (46.07) is comparable to GPT-4o-mini with naive long-context (22.54). This argues that for time-aware multi-hop reasoning, *memory structure carries more weight than parametric scale*, at least up to a point. That points to a research direction: how cheap can the inference-side LLM get if the memory-write side is allowed to be expensive and offline?

The session-level LLM verification step is also under-emphasized in the writeup but probably load-bearing. Most prior KG-from-text systems suffer because every extracted triple is committed; introducing a deny-capable review pass is a small architectural change with large robustness consequences.

The weak spot is benchmarking. LoCoMo is the only evaluation. The framework's claim is about long-horizon agentic reasoning, but agentic reasoning over tool-use trajectories (AppWorld, WebArena, BFCL) is qualitatively different from conversational QA. Whether the tri-layer organization generalizes to that setting is the obvious next experiment.

For procedural-memory / skill-evolution research adjacent to this thread, the relevant takeaway is the ExpM design: experience items are explicitly required to be supported by multiple coherent interactions and stored with provenance back to source passages. That provenance discipline is what most "procedural memory" systems do not enforce, and it is the lever that makes corrections and audit possible.

## Related

- [[tri-layer-memory-consolidation]] — the consolidation-centric memory framing this paper introduces
- [[memweaver-dual-channel-memory-retrieval]] — the named inference-time retrieval method
- [[agentic-memory]] — umbrella topic for memory architectures beyond the context window
- [[procedural-memory]] — adjacent topic; ExpM as procedural-memory-by-clustering rather than skill-extraction
- [[continual-learning-evaluation]] — LoCoMo as a long-horizon QA evaluation; relevant to evaluation-protocol discussions
