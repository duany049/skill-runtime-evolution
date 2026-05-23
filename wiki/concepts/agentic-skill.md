---
title: Agentic Skill
aliases:
  - agent skill
  - SKILL.md
  - reusable knowledge artifact
tags:
  - agentic-skills
  - llm-agents
  - reusable-knowledge
  - skill-library
maturity: emerging
definition: A file-system-based reusable knowledge artifact for LLM agents, packaged as a `SKILL.md` file with structured metadata and content (and optional helper files), encoding domain-specific workflows, API usage patterns, or best practices that an agent can load on demand.
key_papers:
  - how-well-agentic-skills-work-wild
first_introduced: anthropic skill spec (agentskills2026); academic systematization in Liu et al. 2026 (this paper) and concurrent SkillsBench, SoK-Agentic-Skills, SkillNet
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

An *agentic skill* is a file-system-resident knowledge artifact consisting of a `SKILL.md` file with structured metadata (name, description, possibly trigger conditions) and content (workflows, API patterns, code snippets), optionally accompanied by helper files. The format is a standardized contract: any agent harness that understands the convention can discover, retrieve, load into context, and apply the skill's content to a downstream task. Skills are intended to be reusable across tasks (unlike one-shot prompts) and across agents (unlike fine-tuned weights), and they are intended to be authored, stored, and shared the way source-code files are — via repositories, package managers, and aggregation platforms (`skillhub.club`, `skills.sh`).

## Intuition

A skill sits between two extremes. On one side, in-context prompting offers maximum flexibility but no reuse — every task pays the full instruction-writing cost. On the other side, fine-tuning the model offers maximum reuse but no portability and high cost. Skills are the in-between: portable like prompts, reusable like code. They are *not* tools (which run external code), *not* memories (which are agent-internal traces), and *not* trained behaviors — they are structured documents that an agent reads to learn a procedure or pattern. The bet implicit in the agentic-skill design is that a large enough library of well-written skills, plus retrieval, plus a competent reader, is competitive with model improvement for many practical tasks.

## Variants

- *Curated / task-specific skills* — hand-crafted for a single task or task family, often overfit to spell out a solution. Used in `SkillsBench`-style idealized evaluations.
- *General-purpose skills* — encode a reusable workflow or pattern (e.g. "USGS water-level API usage") without targeting any specific downstream task. Dominate real-world skill repositories.
- *Meta-skills* — skills *about* skills (e.g. Anthropic's `skill-creator`), used to author or improve other skills. Used as the engine for query-agnostic refinement in [[how-well-agentic-skills-work-wild]].
- *Refined skills* — produced by query-specific refinement: a single tailored skill synthesized from multiple retrieved candidates at inference time.

## Comparison

vs. *tools / actions* (programmatic, called via API, executed): skills are read-only documents; tools execute code.

vs. *procedural memory*: skills are explicit, externally-authored, retrieval-mediated; procedural memory is typically extracted from past trajectories by the agent itself.

vs. *instruction manuals / SOPs*: skills carry a metadata header for retrieval and a standardized structure; instructions are free-form.

## Known limitations

- Skill *utility* is not the same as skill *loading*: agents can fail to load relevant skills (selection problem) or load skills without effectively using them (e.g. Kimi K2.5 high load + low pass-rate in [[how-well-agentic-skills-work-wild]]).
- Skill quality is heterogeneous: real-world collections include many ill-formatted, ambiguous, or domain-misaligned skills.
- The skill ecosystem is recent and the format conventions (metadata keys, file layout) are still settling, complicating cross-collection studies.

## Open problems

- What metadata format maximizes retrievability without inflating skill files?
- Should skills carry versioning, dependency, or compatibility metadata?
- How does the optimal skill granularity (one-skill-per-workflow vs. composite) depend on task type?
- How should skills be retired or archived as the ecosystem grows?

## Relationship to foundations

Agentic skills inherit ideas from retrieval-augmented generation (RAG over knowledge) and from instruction tuning (structured natural-language guidance), but operationalize them at the *agent* level rather than the *model* level: the artifact lives outside the weights and outside the prompt, retrieved on demand by an agent's reasoning loop.

## My understanding

The decisive question for agentic skills is not whether they work in principle but whether the realistic-condition fragility documented in [[how-well-agentic-skills-work-wild]] is a permanent ceiling or a fixable engineering problem. The paper's coverage-based analysis suggests the latter — retrieval and authoring quality are the levers — but neither is a solved problem. Until those land, "skills work" should be read as "skills can work when curation is tight," not as "skills are a reliable productivity layer for general-purpose agents."
