---
title: Skill Patch
aliases:
  - skill diff
  - incremental skill update
  - patch-based skill revision
tags:
  - skill-update
  - procedural-memory
  - file-level-interface
  - skill-library
maturity: emerging
definition: "A minimal three-field record (`summary`, `upsert_files`, `delete_paths`) that incrementally mutates a skill library after a single task — sufficient to add, revise, or delete SKILL.md files and helper scripts while preserving a fully auditable update history."
key_papers:
  - skillflow-benchmarking-lifelong-skill-discovery-evolution
first_introduced: "SkillFlow (2026)"
date_updated: 2026-05-18
related_concepts:
  - agentic-lifelong-learning
linked_ideas: []
---

## Definition

A skill patch $\Delta_t$ is the output of a model under a fixed prompt template $g$, conditioned on the current library $\mathcal{S}_{t-1}$, the latest task's execution trajectory $\tau_t$, and the verifier-derived rubric $r_t$:
$$\Delta_t = \text{Model}_g(\mathcal{S}_{t-1}, \tau_t, r_t).$$
The patch has three fields:
- `summary` — a short human-readable description of the lesson learned.
- `upsert_files` — a mapping from file path to file content; existing files at the path are overwritten, new paths are created.
- `delete_paths` — a list of file paths to remove from the library.

The patch is applied as $\mathcal{S}_t = \text{Apply}(\Delta_t, \mathcal{S}_{t-1})$ — a file-level mutation, not a free-form rewrite. This file-level interface is the *minimal auditable surface* the SkillFlow protocol uses to record skill evolution.

## Intuition

Earlier skill-update mechanisms range from "regenerate the whole library each time" to "append-only logging of every trajectory". Both endpoints make analysis hard: total regeneration destroys history, append-only logs bloat the library and confound usage with utility. Skill patches sit in the middle — each patch is a small, structured diff that **preserves edit history** while still allowing arbitrary insertions, revisions, and deletions. Because the patch is file-level, common failure modes (uncontrolled skill growth, redundant helpers, fragmentation) are directly inspectable: you can read the sequence of patches and see exactly when an incorrect abstraction was written, when (or whether) it was revised, and whether any obsolete files were deleted.

The minimality is deliberate. By keeping the patch to three fields instead of a rich diff language, the protocol avoids confounding **skill quality** with **instruction-following ability** of the patch model. A model that struggles to follow a complex patch schema would produce malformed skills even if its abstraction capability is sound.

## Variants

- Three-field file-level patch (canonical, from SkillFlow).
- Append-only skill log (degenerate case: `delete_paths = ∅`, no overwrites).
- Full-rewrite patch (degenerate case: every file in `upsert_files`, no history).

## Comparison

- vs. *episodic memory write*: stores the trace; the patch stores an *abstraction* over the trace.
- vs. *semantic memory entry*: a fact in semantic memory has no executable surface; a skill patch's `upsert_files` typically includes executable assets (helper scripts, `SKILL.md` invocation instructions).
- vs. *fine-tuning update*: changes the library, not the model weights; reversible and inspectable per-step.

## Known limitations

- The patch can encode an *incorrect* skill just as easily as a correct one. SkillFlow's Finding 2 documents systematic downstream drift when bad skills enter the library.
- `delete_paths` exists but most evaluated models rarely use it; uncontrolled skill growth (Finding 4) is the empirical norm for weaker models.
- The fixed prompt template that elicits patches is not learned — its sensitivity to wording is unmeasured.

## Open problems

- Training signals (RL, supervised) that teach a model to write *deletion-aware* patches.
- Patches that include test cases or assertions, allowing self-verification before application.
- Patch quality metrics independent of downstream task success (so a patch can be judged on its own merits before being committed).

## Relationship to foundations

The construct generalizes the version-control diff/patch model (additions, modifications, deletions) to learned procedural memory. No deeper foundational dependency is invoked by the introducing paper.

## My understanding

The skill patch is the **smallest structured artifact** that makes lifelong skill evolution auditable. Its real contribution is methodological: by forcing every library mutation through a single typed record, it converts vague claims like "the agent learned" into a sequence of inspectable diffs. Once that diff sequence is available, downstream analyses — when did the agent write the bad skill, was it ever repaired, did deletions ever occur — become straightforward. Empirically, the patch interface also reveals that most models can write skills but few can repair them: the patch sequence shows updates that *should* delete or revise bad entries simply re-adding similar variants instead. That is the cleanest failure mode I have seen documented in the lifelong-skill literature.
