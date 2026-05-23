---
title: Verify Before Update
aliases:
  - verify-then-store
  - re-evaluation gating
  - utility-gated memory write
tags:
  - agentic-memory
  - memory-management
  - online-learning
  - quality-control
maturity: emerging
definition: A memory write-back protocol where a candidate experience or guideline is committed to long-term memory only after a re-evaluation step shows that using it yields a statistically significant performance gain over the prior memory configuration.
key_papers:
  - live-evo-online-evolution-agentic-memory
first_introduced: Live-Evo (Zhang et al., 2026)
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

Verify-Before-Update gates each write to the experience bank by an explicit re-evaluation step. Concretely: given a candidate experience $e^{\text{new}}_q$ produced by summarizing a failure trajectory, the system re-runs the task with $e^{\text{new}}_q$ available and only commits the candidate to $\mathcal{E}$ when the re-evaluated outcome strictly improves on the original memory-on score. Reflection alone is insufficient to add an entry; demonstrated downstream gain is required.

## Intuition

Self-evolving systems that store every reflection accumulate noisy, sometimes hallucinated entries that later mislead retrieval. The protocol replaces "store everything the agent reflects on" with "store only the reflections whose downstream effect is measurably positive." It moves the supervision signal from the act of reflecting (free) to the act of re-deciding (expensive but causal). The mechanism is cheap when the evaluation signal is dense and continuous; it would be brittle when scores are sparse or noisy at the per-task level.

## Variants

- *Hard gate*: commit if and only if the re-evaluation score strictly exceeds the prior memory-on score (Live-Evo's instantiation, with a tunable `min_brier_improvement` threshold of 0.05).
- *Soft gate*: weight the new entry by the magnitude of improvement; commit always but with a small weight when the gain is marginal.
- *Multi-trial verification*: re-evaluate $n$ times and gate on a statistical test rather than a single comparison; not realized in the original paper.

## Comparison

vs. *unverified write-back* (e.g. Reflexion, ExpeL, Mem0): reflection is generated and added to memory immediately; the system trusts the LLM's self-assessment.

vs. *outcome-only memory updates*: outcome-based methods adjust the weight of existing experiences but do not gate the *creation* of new entries.

vs. *human curation*: human approval gates writes by external judgment; verify-before-update gates writes by mechanical re-evaluation against the environment.

## Known limitations

- **Conservative.** Subtle or slow-acting heuristics may produce only modest per-task gains and never clear the threshold, even when they would help in aggregate.
- **Cost.** Every candidate experience requires an extra task re-execution, doubling inference cost on each candidate.
- **Threshold sensitivity.** The required margin (e.g. `min_brier_improvement = 0.05`) is a hyperparameter; too tight starves the bank, too loose admits noise.
- Single-trial verification confounds the candidate's true utility with per-task stochasticity.

## Open problems

- Adaptive thresholds: looser when the bank is small (cold-start) and tighter when mature.
- Verifying meta-guidelines as well as raw experiences; Live-Evo gates $\mathcal{E}$ writes but not $\mathcal{M}$ writes.
- Multi-trial / bootstrap-based verification that controls false-positive admissions under noisy evaluation.

## Relationship to foundations

A form of empirical risk minimization applied online to memory writes: rather than asking the agent to predict a candidate's utility, the protocol measures it. Analogous in spirit to off-policy evaluation in reinforcement learning, but at the per-entry granularity of the external memory store rather than at the policy level.

## My understanding

The protocol is what turns "self-evolving memory" from a marketing term into a measurable mechanism: a written entry has to *prove* its keep. The big asymmetry of the design is that it gates writes very aggressively but leaves the existing bank untouched — combined with the weight-decay loop, this means stale entries can persist as long as their retrieval weight has not yet decayed enough to push them out, even when newer, better entries fail to clear the strict admission threshold. The conservative-bias point in the paper's limitations section follows directly from this asymmetry.
