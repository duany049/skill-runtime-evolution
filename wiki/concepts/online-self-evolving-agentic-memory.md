---
title: Online Self-Evolving Agentic Memory
aliases:
  - live self-evolving memory
  - online memory evolution
  - streaming agentic memory
tags:
  - agentic-memory
  - online-learning
  - continual-learning
  - self-evolving-agents
maturity: emerging
definition: A regime of agentic memory where the agent updates its memory continuously from a stream of incoming tasks under distribution shift, rather than learning the memory from a static train split and freezing it at test time.
key_papers:
  - live-evo-online-evolution-agentic-memory
first_introduced: Live-Evo (Zhang et al., 2026)
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

Online self-evolving agentic memory is the special case of self-evolving memory in which the memory updates run *as new tasks arrive over time*, on a real stream rather than on a static benchmark folded into pseudo-sequential chunks. The regime adds two pressures absent from offline self-evolution: (1) the task distribution can drift, so old experiences may become stale or actively misleading; and (2) the supervision signal arrives sequentially and is often continuous (e.g. probabilistic forecasting scores) rather than binary success/failure.

## Intuition

Most self-evolving agent work treats memory as a function of a training partition: walk through the partition, accumulate experiences, freeze, then evaluate on held-out tasks. The shape of the resulting system implicitly assumes i.i.d. tasks. In contrast, live benchmarks such as Prophet Arena and FutureX release new tasks weekly drawn from a non-stationary real world. Under those conditions the agent must do two things the offline regime does not require:

- decide *when to forget*, because past experience can become misleading rather than merely irrelevant;
- decide *how to use* a now-growing experience set, because brute concatenation of retrieved trajectories no longer scales and offers no principled prioritization.

The frontier within this regime is grounding both decisions (forget? how to use?) in the dense, continuous environmental feedback that live benchmarks naturally provide, rather than in LLM self-verification.

## Variants

- *Per-task online update* — every incoming task immediately produces a weight update on the experiences it retrieved (Live-Evo's reinforce/decay loop).
- *Batch-and-verify online update* — updates are accumulated over a batch (e.g. a week), and only the worst-performing tasks contribute new entries to the experience bank, gated by re-evaluation (Live-Evo's Verify-Before-Update).
- *Fold-based pseudo-online* — earlier and arguably misnamed variants (e.g. evaluations that split a static benchmark into folds and learn fold-by-fold). These approximate online learning but do not exhibit true distribution shift.

## Comparison

vs. *offline self-evolving memory*: offline systems learn memory on a fixed corpus and freeze before test; online systems update during deployment. Offline assumes i.i.d. tasks; online does not.

vs. *RAG with caching*: caching adds and reuses retrieved content, but does not adjust retrievability based on outcome feedback. Online self-evolving memory makes retrievability itself learnable from feedback.

vs. *continual learning of model weights*: weight-level continual learning updates the policy network itself; agentic memory evolution keeps the policy frozen and updates an external, interpretable, editable store of experiences and meta-guidelines.

## Known limitations

- Requires reasonably dense feedback to drive the online update. With sparse or delayed signal, the update loop degenerates.
- Memory growth and drift over very long horizons (months to years) is not yet characterized; existing studies report 10-week-scale evaluations.
- Doubled inference cost when contrastive (memory-on vs memory-off) evaluation is used to derive a per-task supervision signal.

## Open problems

- Adaptive forgetting rates: the right decay schedule almost certainly depends on detected distribution shift, but principled methods for tying decay to drift signals are missing.
- Regime change vs. smooth drift: most current systems handle gradual drift; abrupt regime change (e.g. a market shock that invalidates a whole class of experiences at once) is not addressed.
- Joint evolution of $\mathcal{E}$ and $\mathcal{M}$: how should higher-level meta-guidelines themselves be pruned or invalidated as their underlying experiences are decayed out?

## Relationship to foundations

Sits on top of classical online and continual learning ([[continual-learning-evaluation]]), but the operational mechanisms — natural-language experience entries, LLM-driven retrieval and reflection, meta-guideline composition — are distinctly agentic and do not reduce to parameter updates.

## My understanding

The substantive content of this concept is the *forcing function*: by switching from "static benchmark folded into pseudo-streams" to "real live benchmark with continuous feedback", the design space of memory systems changes qualitatively. Decay, write-gating, contrastive supervision, and meta-guideline learning all become first-class concerns rather than optional features. Treating "online" as a discipline (not a setting) reframes what counts as a complete self-evolving memory system.
