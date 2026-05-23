---
title: Agent Skill
aliases:
  - skill
  - agent skills
  - LLM agent skill
tags:
  - agent-skills
  - procedural-memory
  - skill-format
maturity: emerging
definition: A structured document that encodes activation conditions, execution steps, and task-specific knowledge for use by an LLM agent on a specialized task.
key_papers:
  - skilllearnbench-benchmarking-continual-learning-methods-agent
first_introduced: Anthropic, 2025 (agent skills as a standardized format)
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

An *agent skill* is a structured artifact — typically a Markdown document with a defined frontmatter — that tells an LLM agent *when* a particular task-shaped competence applies, *how* to execute it step by step, and *what* domain knowledge or tools it depends on. Skills sit in the agent's external memory (a skill library) and are retrieved into context at execution time when their activation conditions match the agent's current task.

## Intuition

A skill is best understood by contrast: it is not a tool (which is a single callable function), not a system prompt (which is global), and not a chat-style example (which is illustrative but unstructured). A skill is a *recipe*: it specifies a class of problems it applies to, the procedure for tackling them, and the ingredients (tools, data, sub-skills) needed. Once defined, the same skill can be invoked across many *instances* of the same task type without re-derivation.

This recipe framing is what enables a *library* of skills to compose into general agent capability: new tasks trigger new skills, and over time the agent accumulates a procedural knowledge base far larger than fits in any one prompt.

## Variants

- **Human-authored skills.** Hand-written by domain experts; serve as the upper-bound reference in evaluation (e.g., SkillLearnBench uses them to define what "skill-dependent" means).
- **Auto-generated skills.** Produced by continual-learning methods (One-Shot, Self Feedback, Teacher Feedback, Skill Creator, SkillRL, etc.) from a task description plus execution experience.
- **Code-based skill libraries.** Voyager-style executable skill libraries where each skill is callable code rather than a Markdown recipe. Same abstract role; different encoding.
- **Natural-language workflows.** Trajectory-induced procedural patterns (AgentRR / ExpeL) that are skills in everything but name and frontmatter.

## Comparison

The closest neighbour concept is *procedural memory* in general; agent skill is the specific instantiation that has been crowned an open standard (`agentskills_spec`). What makes the skill standard worth its own concept page is the surrounding ecosystem assumption: a portable format, an activation-condition section, and a community-shared library.

Distinct from a *workflow*: a workflow describes a single execution trace, whereas a skill describes a *class* of executions that share a procedure.

## Known limitations

- The Markdown skill format presumes the LLM can faithfully follow procedural prose. SkillLearnBench finds adoption rates of 60–85% across methods — meaning 15–40% of generated skills are simply ignored at execution time.
- Skills currently encode procedure but not negative constraints — when *not* to apply, when to abort. Open-ended tasks often suffer because a skill that should not apply still does.
- No standard exists for skill *retirement* or *versioning*; libraries accumulate stale procedures.

## Open problems

- How to balance specification *richness* (more detail = more coverage) against *adoptability* (longer / more rigid skills are more often ignored).
- Whether skills should encode activation *probability* / *confidence* rather than binary conditions.
- Cross-agent portability — a skill that helps Claude may hurt Gemini, as SkillLearnBench documents.

## Relationship to foundations

Agent skills are a contemporary instantiation of long-standing ideas in procedural-memory and case-based reasoning. The novelty is the format standardization plus the LLM-agent execution substrate, not the underlying memory abstraction.

## My understanding

Treat "agent skill" as the standardized *artifact* and "continual skill learning" as the *generation process*. Most of the open research is on the generation side; the artifact format is settling toward a community standard with diminishing room for innovation.
