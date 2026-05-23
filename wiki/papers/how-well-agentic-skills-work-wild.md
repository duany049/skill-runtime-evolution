---
title: "How Well Do Agentic Skills Work in the Wild: Benchmarking LLM Skill Usage in Realistic Settings"
slug: how-well-agentic-skills-work-wild
arxiv: "2604.04323"
venue: COLM 2026 (preprint)
year: 2026
tags:
  - agentic-skills
  - skill-retrieval
  - skill-refinement
  - benchmarking
  - llm-agents
  - skill-usage
importance: 4
date_added: 2026-05-18
source_type: tex
tldr: First systematic study showing skill benefits for LLM agents degrade sharply as evaluation moves from idealized curated-skill settings to realistic settings requiring retrieval from a 34k-skill pool, with query-specific refinement recovering much of the lost performance only when retrieved skills already have reasonable relevance.
contribution_type:
  - benchmark
  - analysis
  - method
datasets:
  - SkillsBench
  - Terminal-Bench 2.0
code_url: "https://github.com/UCSB-NLP-Chang/Skill-Usage"
cited_by: []
---

## Problem & Context

Agentic skills — reusable file-system-based knowledge artifacts (`SKILL.md` plus optional helper files) introduced by Anthropic and adopted across Claude Code, Codex, and an emerging open-source ecosystem — promise to extend LLM agents with domain-specific workflows and best practices. Prior benchmarking work (notably `SkillsBench`) showed that providing curated skills to an agent improves task pass rates, but did so under two strong idealizations: (1) hand-crafted skills overfit to the specific evaluation task (often spelling out the exact solution), and (2) the curated skills are placed directly in the agent's context, bypassing the practical retrieval problem of finding relevant skills inside a large, noisy collection. Before this paper, there was no rigorous evidence that skills retain their value when an agent must search a real-world skill pool and adapt skills that are not specifically authored for the task. Three concrete challenges — skill selection (recognizing which provided skills are worth loading), skill retrieval (searching a large pool), and skill adaptation (extracting useful information from partial-fit skills) — were absent from existing evaluations.

## Key idea

Skill benefits are fragile under realistic conditions. The paper introduces a *progressive evaluation* framework that gradually layers in the three real-world challenges on top of `SkillsBench`, and shows that pass rates degrade monotonically from a force-loaded curated upper bound (55.4% for Claude Opus 4.6) down to near no-skill-baseline levels (38.4% vs. 35.4% baseline) once curated skills are removed from the retrieval pool. The same paper then introduces *skill refinement* — transforming retrieved skills into more useful forms — and shows that query-specific refinement (where the agent explores the task, attempts an initial solution, and synthesizes a tailored skill across multiple retrieved candidates) substantially recovers lost performance *only when* the initially retrieved skills already have reasonable relevance and coverage. Refinement is a *multiplier* on existing skill quality, not a *generator* of new knowledge.

## Method

The paper combines a large-scale skill infrastructure with a controlled empirical protocol.

**Skill collection.** 34,198 real-world skills assembled from `skillhub.club` and `skills.sh`, downloaded from origin GitHub repositories, filtered by permissive licenses (MIT, Apache 2.0), deduplicated by file content, and de-noised by removing ill-formatted entries.

**Skill search engine.** Two indices per skill — metadata (name + description) and full `SKILL.md` content — using Qwen3-Embedding-4B for dense embeddings and BM25 for sparse keyword matching. Five retrieval strategies are compared: direct semantic, agentic keyword, agentic semantic, agentic hybrid w/o content, and agentic hybrid w/ content (an iterative agent that formulates queries, inspects candidates, and refines its strategy).

**Progressive evaluation.** Six evaluation conditions on `SkillsBench` ordered from idealized to realistic: curated + forced load, curated, curated + distractors, retrieved (w/ curated), retrieved (w/o curated), no skills. Each condition is run 3 times per task on 84 tasks across three models: `Claude Opus 4.6` (with Claude Code), `Kimi K2.5` (with Terminus-2), `Qwen3.5-397B-A17B` (with Qwen-Code).

**Skill refinement.** Two strategies. *Query-agnostic* refines each retrieved skill independently and offline using Anthropic's `skill-creator` meta-skill (skills synthesize test queries, run agents with/without the skill, self-evaluate, iterate). *Query-specific* lets the agent first read the task, attempt an initial solution without ground-truth verification, then reflect, compose, and emit a single tailored skill that merges and discards content from the retrieved candidates.

**Generalization.** Skill retrieval and refinement are also evaluated on `Terminal-Bench 2.0` (89 tasks, not designed for skills, no curated skills available), demonstrating the protocol works beyond skill-purpose-built benchmarks.

## Experiment & Results

**Retrieval.** Agentic hybrid (with full skill content) achieves the best `Recall@5 = 65.5%` and `Recall@10 = 68.3%` on `SkillsBench`, far ahead of direct semantic retrieval (`Recall@3` 38.1% vs. agentic-semantic 56.8% — +18.7 points). Adding the full-content index gives modest but consistent gains over metadata-only at higher k. Agentic keyword search alone is the worst variant (`Recall@3 = 24.1%`).

**Progressive evaluation on SkillsBench (pass rates).** Claude Opus 4.6 degrades 55.4 (forced) → 51.2 (curated) → 43.5 (curated+distractors) → 40.1 (retrieved w/ curated) → 38.4 (retrieved w/o curated) → 35.4 (no skills). Kimi K2.5 and Qwen3.5-397B-A17B both *drop below their no-skill baseline* in the most realistic setting (Kimi 19.8% vs. 21.8% baseline; Qwen 19.7% vs. 20.5% baseline), suggesting weaker models can be actively misled by irrelevant retrieved skills while stronger models can ignore them. Skill loading rates expose the selection bottleneck: only 49% of Claude trajectories load all curated skills in the basic curated setting, falling to 31% with distractors.

**Refinement on SkillsBench.** Query-specific refinement boosts Claude from 40.1% → 48.2% under retrieved (w/ curated), recovering most of the gap to curated. Skill loading rate also jumps (Claude: 44% → 72%). Under retrieved (w/o curated), gains are modest or negative (Claude: 38.4 → 37.9, Qwen: 19.7 → 21.5, Kimi: 19.8 → 23.1). Query-agnostic refinement gives smaller, inconsistent gains (Claude: 40.1 → 42.0 w/ curated).

**Terminal-Bench 2.0 generalization.** Query-specific refinement improves all three models on a benchmark not designed for skills: Claude 57.7 (no skills) → 61.4 (retrieved) → 65.5 (+ query-specific), Kimi +5.6, Qwen +4.9. This is the headline 57.7% → 65.5% Claude Opus 4.6 result.

**Coverage analysis (LLM-judged 1-5).** Refinement gains track the coverage of initially retrieved skills: settings with ≥3.83 coverage benefit most; the SB-w/o-curated setting (≤3.49) shows little gain — refinement amplifies, it does not synthesize from nothing.

## Limitations

- All evaluation is on coding-flavored benchmarks (`SkillsBench`, `Terminal-Bench 2.0`); whether the same fragility holds for non-coding agent tasks is untested.
- The 34k skill collection is filtered to permissively licensed open-source repos, which is a specific quality distribution; behavior on private or enterprise skill libraries may differ.
- Query-specific refinement requires a full exploration pass per task at inference time, which is expensive; cost numbers vs. quality gain are not given.
- The LLM-judge coverage scores (used to explain when refinement fails) depend on `GPT-5.4` and could be biased in ways that confound the conclusion.
- Each retrieved-skill set is fixed to top-5; how performance scales with retrieval budget (top-10, top-20) is not studied.
- Kimi's query-agnostic experiments are omitted because Terminus-2 lacks subagent support, so cross-harness comparisons for that strategy are incomplete.

## Open questions

- Can query-agnostic refinement match query-specific gains if the offline skill-improvement loop has access to the task *distribution* (not the specific query) — i.e. is there a useful middle ground?
- How much of the gap between retrieved-w/o-curated and curated is closable purely by improving retrieval (e.g. higher Recall@k or learned reranking) without changing skill content?
- What design properties of agent harnesses (Claude Code vs. Terminus-2 vs. Qwen-Code) explain Kimi's high skill-loading rate without translation into pass-rate gains?
- Are there skill-collection-curation strategies (clustering, hierarchy, canonicalization) that reduce the realistic-vs-curated gap without per-query refinement at inference?
- Does the multiplier-vs-generator characterization of refinement generalize to non-coding domains where "skill quality" may not be as cleanly defined?

## My take

This is the cleanest empirical case for why hot benchmarks in the agent-skill space have been over-stating skill utility. The progressive evaluation design is the contribution that will outlive the specific numbers — it gives every future skill paper a checklist of which idealizations to remove. The multiplier-not-generator framing of refinement is also load-bearing: it predicts when offline skill-collection investment pays off and when it won't, which is exactly the question deployment teams care about. The weakest point is that all three benchmarks live near the coding agent slice; the fragility result might be stronger or weaker for non-coding skills (e.g. enterprise workflows, data-analysis recipes), and we don't know yet. The cross-model differences (Kimi drops below baseline; stronger models recover) hint that skill robustness is a model-capability axis worth measuring separately, not bundled into pass rate.

## Related

- [[agentic-skill]]
- [[skill-retrieval]]
- [[skill-refinement]]
- [[progressive-skill-evaluation]]
- [[query-specific-skill-refinement]]
- [[shiyu-chang]]
- [[tommi-jaakkola]]
