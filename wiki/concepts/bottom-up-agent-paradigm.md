---
title: Bottom-Up Agent Paradigm
aliases:
  - bottom-up agent
  - bottom-up paradigm
  - experience-driven agent
tags:
  - llm-agent
  - skill-evolution
  - learning-from-experience
  - open-ended-environment
maturity: emerging
definition: "An agent-design paradigm where competence is acquired bottom-up through autonomous interaction with an environment — exploring atomic actions, reasoning about their outcomes via an LLM, abstracting successful sequences into a skill library, and refining that library over time — without predefined goals, APIs, or task-specific priors."
key_papers:
  - rethinking-agent-design-top-down-workflows
first_introduced: "rethinking-agent-design-top-down-workflows (2505.17673)"
date_updated: 2026-05-18
related_concepts: []
linked_ideas: []
---

## Definition

The bottom-up agent paradigm describes a class of LLM-based agents that **start from an empty skill library** and grow competence **through trial-and-reasoning interaction** with the environment, rather than executing a designer-decomposed workflow over predefined APIs. The agent perceives via raw observations (e.g., pixel screenshots), acts via atomic primitives (mouse, keyboard), and uses an LLM to (a) propose action sequences, (b) judge their effects against the environment's visual or numeric feedback, and (c) crystallize successful sequences into reusable skills. The library evolves autonomously over deployment.

## Intuition

The paradigm is named in opposition to the **top-down agent paradigm** typical of ReAct, AutoGPT, MetaGPT, ChatDev, and similar systems, which inherits three constraints from traditional software engineering: *staticness* (deployed agents are identical replicas of a central prototype, updated only by designers), *prior dependency* (workflows assume predefined APIs, subgoals, and task-specific prompts), and *token inefficiency* (LLM tokens go to workflow enforcement, not reasoning over experience). Bottom-up reverses each of these: agents are active researchers rather than static replicas, they have no prior structure to lean on, and the LLM's tokens fund reasoning about lived interaction.

The paradigm extends the **era of experience** vision (Silver and Sutton) — a shift away from imitating curated human trajectories toward learning from unstructured streams of interaction — by giving it a concrete agentic instantiation that uses modern LLM reasoning as the core abstraction mechanism.

## Variants

- **Single-agent bottom-up.** One agent, one library, sequential exploration. The instantiation in [[rethinking-agent-design-top-down-workflows]].
- **Population-level bottom-up (envisioned).** Many agents share a cloud-hosted library; any agent's discoveries propagate to all, with consistency / refinement consensus protocols still open.
- **Hybrid bottom-up / top-down.** The paradigm is positioned as complementary to top-down, not replacing it: top-down workflows can host bottom-up skill acquisition inside well-defined task families.

## Comparison

| Property | Top-Down Agent | Bottom-Up Agent |
|---|---|---|
| Starting state | Pre-engineered workflow + APIs | Empty skill library `S = ∅` |
| Source of competence | Designer decomposition | Autonomous trial-and-reasoning |
| Reward signal | Often explicit / task-defined | Implicit (visual change, progression) |
| Update mechanism | Manual designer iteration | Deployment-time LLM reasoning |
| Token usage | Workflow enforcement | Reasoning over experience |
| Adaptation surface | Static replication | Active, per-agent evolution |
| Scalability driver | Workflow complexity | Collective experience pooling |

## Known limitations

- **Exploration overhead.** Without priors, bootstrapping useful behaviors from atomic actions is slow; reported as 2-2.5× more environment steps than prior-assisted baselines.
- **Implicit reward is myopic.** Visual-diff implicit reward misses defensive or long-horizon strategies whose effects are not immediately visible.
- **Skill abstraction is shallow.** Skills remain flat record-and-replay action sequences, not parameterized callable functions, limiting cross-environment transfer.
- **Coordination at scale is open.** Population-level shared libraries assume globally-consistent state; concurrent edits and conflicting refinements have no published protocol.

## Open problems

- Defining implicit reward that captures delayed and strategic effects, not just immediate visual change.
- Lifting recorded skill sequences into parameterized, callable abstractions without reintroducing environment-specific priors.
- Decentralized consistency protocols for shared skill libraries under massively parallel deployment.
- Cross-environment skill transfer when visual semantics, action consequences, and UI layouts differ.

## Relationship to foundations

The paradigm is conceptually grounded in reinforcement-learning notions of exploration and policy improvement, but replaces explicit reward and learned policies with LLM-reasoned skill abstraction. It also draws on the broader "era of experience" framing in AI research about moving beyond human-trajectory imitation toward grounded interaction.

## My understanding

The strongest move in the framing is that the three constraints of top-down (staticness, prior dependency, token inefficiency) are presented as *structural*, not incidental — they follow from the software-engineering heritage of the workflow approach, so no amount of better workflow design dissolves them. That makes "bottom-up" not just an alternative design but a different category of system. The weakness, currently, is that the empirical evidence sits inside a single-agent system in two turn-based games; the population-level skill-diffusion claim is rhetorical until experiments with multiple concurrent agents validate it. Watch for follow-up work that (a) operationalizes the diversity/efficiency reward terms beyond `R_semantics` and (b) tests cross-agent transfer on a shared library — those are where the paradigm either grows teeth or collapses to "Voyager in different games".
